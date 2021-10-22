---
layout: post
title: "kubernetes使用及运维"
author-id: "zqmalyssa"
tags: [code, operations]
---

kubernetes已经成为目前最常用的应用编排组件，有必要写一写

### kubernetes的基本介绍

kubernetes所有的配置数据存储在etcd中；其他组件通过API Server和ETCD打交道

Pod是管理一组容器，Pod对外共享一个Ip，通过livenessProbe探针监测容器是否健康，另外一类是readinessProbe探针来判断容器是否启动完成

Service是是一些列pod的逻辑集合的抽象，同时他是访问这些pod的一个策略，有时候也被称为微服务。Service通过Label Selector和Pod建立关联关系，并由Service决定将访问转向到后端的哪个pod。当Service被创建后，系统自动创建一个同名的endpoints，该对象包含pod的ip地址和端口号集合。Service分为三类：ClusterIP，NodePort，LoadBalancer

kube-apiserver是整个集群管理的 API 接口，所有对集群进行的查询和管理都要通过 API 来进行集群内部各个模块之间通信的枢纽，所有模块之前并不会之间互相调用，而是通过和 API Server 打交道来完成自己那部分的工作集群安全控制：API Server 提供的**验证和授权**保证了整个集群的安全。Kubernetes所有的配置数据存储在etcd中，其他组件通过API Server和ETCD打交道
1. 它是集群管理的API入口
2. 是资源配额控制的入口
3. 提供了完备的集群安全机制

可以举kubelet，controller-manager和scheduler与api-server的交互

kube-scheduler是预选策略和优选策略，先预选出可用的node，然后在其中优选出可用的节点，它是一个调度器，Scheduler监听API Server，当需要创建新Pod时，Scheduler负责选择该Pod与哪个Node进行绑定，将此绑定信息通过API Server写入到ETCD

kube-controller-manager是ReplicationController，Node Controller，ResrouceQuota，Namespace Controller, ServiceAccount Controller, Token Controller, Service Controller以及Endpoint Controller，它通过API Server监控系统的共享状态，并尝试着将状态从现有状态修正到期望状态。Replication Controller对应容错副本控制、扩容、升级


kubelet是个node上都会启动一个kubelet服务进程，该进程用于处理master节点下发到本节点的任务，管理Pod及Pod中的容器，同时Kubelet进程会在API Server上注册节点自身的信息，定期向Master节点汇报节点的资源情况，并通过cAdvise监控容器和节点资源；它负责创建和销毁Pod，通过探针监测

kube-proxy的作用，主要有以下几点

1. 实时监控kube-api的建立，升级信息，增加或者删除backend Pod信息，来获取Pod和Vip(ClusterIp)的映射关系
2. 维护本地Netfilter、iptables、IPVS内核㢟，通过修改和更新这些组件的规则，来实现数据报文的转发规则，实现每个node上clusterIP的发布和路由维护，构建路由信息，通过转发规则转发报文到VIPs对应的Pod

kube-proxy也有三种工作模式，分别是

1. userspace代理模式
2. iptables模式
3. IPVS模式

其中usersapce是用到了linux中用户态和内核态的切换，所以性能损耗比较大，iptables模式是Pod较多时候，规则也非常多，但kube-proxy不去做负载均衡，之前kube-proxy可以重试，但是现在就是一条条路由规则，要转发的Pod没有响应，就会导致超时，需要配合K8s探针一起使用，IPVS模式用hash tables来存储网络转发规则，比iptables快，它主要工作在内核空间，性能会更好

#### 补充：各大组件之间的交互

这个在MS的时候可能会被大概率问及，所以需要掌握，有的地方可以结合源码

Pod中的进程如何知道API Server的访问地址，其实kubernetes API Server本身**也是个Service**，它的名字是"kubernetes"，它的ClusterIp是地址池中的第一个地址，它所服务的端口是HTTPS端口443。API Server还提供一类很特殊的REST接口，就是kubernetes Proxy API接口，这类接口的作用是代理REST请求，即API Server把收到的REST请求转发到某个Node上的kubelet守护进程的REST端口，由该kubelet进程负责响应。API Server作为集群的核心，负责集群各个功能模块之间的通信，集群内各个功能模块**通过API Server将信息存入etcd**，当需要获取和操作数据时，通过API Server提供的REST接口(GET, LIST, WATCH)来实现。例如，kubelet每隔一个周期就会调用API Server的REST接口报告自身的状态，API Server接收这些信息后，将节点状态信息更新到etcd，此外kubelet也通过API Server的**Watch接口监听Pod信息**，如果监听到新的Pod副本被调度绑定到本节点，则执行Pod对应容器的创建和启动逻辑，如果监听到Pod被删除，则删除本节点上相应的Pod容器，修改也是。另外，controller-manager与API Server的通信，Node Controller模块通过API Server提供的Watch接口，实时监控Node的信息，并做相应处理。还有比较重要的是scheduler的场景，Scheduler通过WATCH接口监听到新建Pod副本的信息后，它会检索所有符合该Pod要求的Node列表，开始执行Pod的调度逻辑，调度成功后将Pod绑定到目标节点上。为了减轻API Server的压力，各功能模块都采用缓存机制来缓存数据，各功能模块定时从API Server获得指定的资源对象信息(通过LIST和WATCH方法)，然后将这些信息保存到本地缓存。Controller Manager里包含Replication Controller，Node Controller，ResourcceQuata Controller，Namespace Controller，ServiceAccount Controller，Token Controller，Service Controller，Endpoint Controller等，这里面每种controller都负责一个具体的控制流程，而controller manager是这些controller的核心管理者。每个controller都是这样一个操作系统，他们通过API Server提供的接口实时监控整个集群里每个资源对象的当前状态，当发生异常时，尝试将系统从"现有状态"修正到"期望状态"。Node Controller通过API Server实时获取Node的相关信息。Endpoint对象是PodIp+Port的形式的，Endpoint对象在哪里被使用的？是在每个Node上的kube-proxy进程，获取每个Service的Endpoint，实现Service的负载均衡。Scheduler是将待调度的Pod和Controller新建的Pod按照特定的调度算法和调度策略绑定到集群中某个合适的Node上，**并将绑定信息写入etcd**（这里个人认为还是会通过API Server），kubelet通过API Server监听etcd目录，同步Pod列表，所以以非API Server方式创建的Pod都叫做static pod，kubelet使用探针或者cAdvisor来监控资源。将Service落实的是Kube-proxy组件，Service就是个反向代理。Service的NodePort和ClusterIP等概念是kube-proxy服务通过iptables的NAT转换来实现的，在运行过程中动态创建与Service的相关Iptables

#### 补充：集群安全机制

1. API Server认证管理(Authentication)，用CA认证，Token和Http Base
2. API Server授权管理(Authorizatio)，比较常用的是RBAC，可以像API对象一样去操作，可以运行时调整而无需重启API Server
3. Admission Control(准入控制)，通过上面两道关，还有这个准入控制器。
4. ServiceAccount，Secret是从属于ServiceAccount资源对象，属于ServiceAccount的一个部分，一个ServiceAccount里面可以包含多个不同的Secret对象

#### 网络概述

k8s网络通信：

1. 容器间通信：同一个pod内的多个容器间的通信，通过lo即可实现；
2. pod之间的通信，pod ip <---> pod ip，pod和pod之间要不经过任何转换即可通信；
	- 在同一个Node上的Pod之间的通信，Pod1和Pod2是通过veth连接同一个docker0网桥的，所以当然是通的，都不需要其他发现机制，如DNS，Consul或者etcd
	- 不同Node上的Pod的通信，那就必须想办法通过主机的这个IP地址来进行寻址和通信，kubernetes会记录所有正在运行Pod的IP地址，并将这些信息保存在etcd中(作为Service的endpoint)，要通信，达到两个条件，1是整个kubernetes集群中对pod的ip分配进行规划，不能有冲突。2是找到一种办法，将Pod的IP和所在Node的Ip关联起来，通过这个关联可以让Pod互相访问，这就是一些常用的组件了
3. pod和service通信:pod ip <----> cluster ip（即service ip，用get svc出来的）<---->pod ip，他们通过iptables或ipvs实现通信，另外大家要注意ipvs取代不了iptables，因为ipvs只能做负载均衡，而做不了nat转换。Service服务做为客户端和Pod(Backends)中间的代理层，并抽象一个clusterIP的虚拟IP，集群内部可以通过这个IP访问到具体Pod的服务。这里要注意的是ClusterIp是Iptables规则产生，不是绑定在网络接口上的，服务可以访问，但ping不通ClusterIP(就是在容器内ping命令不能，但是wget clusterIP:port就行，返回401但是是通的)，补充下，kubernetes如何查找service下面的pod呢？答案就是kubectl get endpoint命令
	- clusterIp是由--service-cluster-ip-range配置的地址分配的，只要不和docker0或者物理网络的子网冲突就行
4. Service与集群外部客户端的通信

在学习docker时知道docker有四种常用的网络模型
1. bridge：桥接式网络
2. joined：联盟式网络，共享使用另外一个容器的网络名称空间
3. opened：容器直接共享使用宿主机的网络名称空间
4. none：不使用任何网络名称空间
无论是哪一种网络方式都会导致如果我们跨节点之间的容器之间进行通信时必须要使用NAT机制来实现，任何pod在访问出去之前因为自己是私有网络中的地址，在离开本机时候必须要做源地址转换以确保能够拿着物理机的地址出去，而后每一个pod要想被别人所访问或者每一个容器在上下文当中它也访问不到我们还来做什么？从下图中可以看到，第一个节点上我们有container1，container1自己使用虚拟网卡（纯软件）的方式生成一个网络接口，它通常是一个v1th格式的网络接口，一半在容器上一半在宿主机上并关联到docker0桥上，所以他是一个私网地址，我们使用172.17.0.2这样的地址，很显然在这个地址出去访问其它物理机地址的时候物理机能收到请求没有问题因为它的网关有可能都已经通过打开核心转换要通过eth0即宿主机的网络发出来，但是对端主机收到以后响应给谁？是不是没法响应了，因为这是私网地址，而且是nat背后的私网地址，或者是位于某个服务器背后的私有地址，因此为了确保能够响应到我们任然能送回这个主机，我们需要在本地做原地址转换，同样的，如果container2希望被别人访问到我们需要在宿主机上的物理接口上做dnat将服务暴露出去，因此被别人访问时，假如有个物理机要访问docker2时就应该访问其宿主机上的eth0上的某一端口，再目标地址转换至container2的eth0的地址上来，所以如果我们刚好是container1和container2通信，那就是两级nat转换了，首先，对container1来讲，其报文离开本机的时候要做snat，然后这个报文接入物理网络发给container2的宿主机的时候我们还需要dnat一次然后到container2，container2响应的报文也是一样的逻辑。当前，dnat和snat都是都是自动实现的，不需要手动设置。这个过程是必不可少的这样的话就会导致我们网络通信性能很差不说，并且container1其实一直不知道自己访问的是谁，他访问的是container2但实际上他的目标地址指向的是eth0的地址，并且container2一直不知道访问自己的是谁，因为他收到的请求是左侧eth0的，他实际上是被container1所访问。

所以这种通信方式实在是让人觉得效率低而且很难去构建我们自己真正需要的网络通讯模型。因此k8s作为一个编排工具来讲他本身就必须需要让我们容器工作在多个节点之上，而且是pod的形式，各pod之间是需要进行通信的，而且在k8s之上要求我们pod之间通信大概存在以下情形
1. 容器间通信：同一个Pod内的多个容器间的通信： lo
2. pod间通信：pod间通信k8s要求他们之间的通信必须是所见即所得，即一个pod的IP到另一个pod的IP之间通信不经过任何NAT转换，要直达
3. pod与service通信：即pod IP  与cluster IP之间直接通信，他们其实不在同一个网段，但是他们通过我们本地的IPVS或者iptables规则能实现通信，而且我们知道1.11上的kube-proxy也支持IPVS类型的service，只不过我们以前没有激活过。即pod IP与cluster IP通信是通过系统上已有的iptables或ipvs规则来实现的，这里特别提醒一下ipvs取代不了iptables，因为ipvs只能拿来做负载均衡，做nat转换这个功能就做不到
4. service与集群外部客户端的通信；使用ingress 或nodeport或loadblance类型的service来实现
我们此前说过利用一个新的环境变量能够在部署的时候就能够实现IPVS后来发现这种方式不行，其实在我们使用kubeadm部署k8s集群时不需要这么做，最简单的办法就是直接改kube-proxy的配置文件。我们去往每一个节点都已经默认加载了ipvs内核模块，调度算法模块等等，还有连接追踪模块。我们来看一下kube-system名称空间的configmap，可以看到一个叫做kube-proxy，这里面定义了我们kube-proxy到底使用哪种模式在工作，将其中的mode定义为ipvs再重载一下即可

```html
# kubectl get configmap -n kube-system
# kubectl get configmap kube-proxy -n kube-system -o yaml
```
**k8s最有意思的是他的网络实现不是自己来实现的，而是要靠网络插件。即CNI（容器网络接口）接口插件标准接入进来的其它插件来实现，即k8s自身并不提供网络解决方案，他允许去使用去托管第三方的任何解决方案(主要是docker的CNM模型和CoreOS公司的CNI模型)，代为解决k8s中能解决这四种通信模型中需要执行通信或解决任何第三方程序都可以，这种解决方案是非常非常多的，有几十种，目前来讲比较流行的就是flannel，calico，以及二者拼凑起来的canel，事实上我们k8s自己也有据说性能比较好的插件比如kube-router等等，但无论是哪一种插件他们试图去解决这四种通信时所用到的解决方案无非就这么几种**
1. 虚拟网桥：brdige，用纯软件的方式实现一个虚拟网络，用一个虚拟网卡接入到我们网桥上去。这样就能保证每一个容器和每一个pod都能有一个专用的网络接口，从而实现每一主机组件有网络接口。每一对网卡一半留在pod之上一半留在宿主机之上并接入到网桥中。甚至能接入到真实的物理网桥上能顾实现物理桥接的方式
2. 多路复用： MacVLAN，基于mac的方式去创建vlan，为每一个虚拟接口配置一个独有的mac地址，使得一个物理网卡能承载多个容器去使用。这样子他们就直接使用物理网卡并直接使用物理网卡中的MacVLAN机制进行跨节点之间进行通信了
3. 硬件交换：使用支持单根IOV（SR-IOV）的方式，一个网卡支持直接在物理机虚拟出多个接口来，所以我们称为单根的网络连接方式，现在市面上的很多网卡都已经支持单根IOV的虚拟化了。它是创建虚拟设备的一种很高性能的方式，一个网卡能够虚拟出在硬件级多个网卡来。然后让每个容器使用一个网卡

相比来说性能肯定是硬件交换的方式效果更好，不过很多情况下我们用户期望去创建二层或三层的一些逻辑网络子网这就需要借助于叠加的网络协议来实现，所以会发现在多种解决方案中第一种叫使用虚拟网桥确实我们能够实现更为强大的控制能力的解决方案，但是这种控制确实实现的功能强大但多一点，他对网络传输来讲有额外的性能开销，毕竟他叫使用隧道网络，或者我们把它称之为叠加网络，要多封装IP守护或多封装mac守护，不过一般来讲我们使用这种叠加网络时控制平面目前而言还没有什么好的标准化，那么用起来彼此之间有可能不兼容，另外如果我们要使用VXLAN这种技术可能会引入更高的开销，这种方式给了用户更大的腾挪的空间

如果我们期望去使用这种所谓的CNI插件对整个k8s来讲非常简单，只需要在kubelet配置文件或在启动时直接通过一个目录路径去加载插件配置文件， /etc/cni/net.d/，因此我们只要把配置文件扔到这个目录中去他就可以被识别并加载为我们插件使用

```html
# ls /etc/cni/net.d/
```
任何其它网络插件部署上来也是如此，把配置文件扔到这个目录下从而被kubelet所加载，从而能够被kubelet作为必要时创建一个pod，这个pod应该有网络和网卡，那么它的网络和网卡怎么生成的呢？kubelet就调用这个目录下的由配置文件指定的网络插件，由网络插件代为实现地址分配，接口创建，网络创建等各种功能。这就是CNI，CNI是一个JSON格式的配置文件，另外他有必要有可能还会调IP地址管理一些二层的模块来实现一些更为强大的管理功能。

通常来讲CNI本身只是一种规范，CNI的插件是说我们的容器网络插件应该怎么定义，网络接口应该怎么定义等等，目前来讲我们CNI插件分为三类，了解即可。不同的网络插件在实现地址管理时可能略有不同，而flannel，calico，canel，kube-router等等都是解决方案，据统计，flannel目前来讲份额还是最大的。**但是其有个缺陷，在k8s上网络插件不但要实现网络地址分配等功能，网络管理等功能，他还需要实现网络策略，可以去定义pod和pod之间能不能通信等等，大家知道我们在k8s上是存在网络名称空间的，但是这个名称空间他并不隔离Pod的访问，虽然隔离了用户的权限，比如我们定义说rolebinding以后我们一个用户只能管理这个名称空间的资源，但是，他却能够指挥着这个pod资源去访问另一个名称空间的pod资源，pod和pod之间在网络上都属于同一网段，他们彼此之间没有任何隔离特性，万一你的k8s集群有两个项目，彼此之间不认识，万一要互害的时候是没有隔离特性的。所以我们需要确保网络插件能够实现辅助设置pod和pod之间是否能够互相访问的网络策略，但是flannel是不支持网络策略的。calico虽然部署和配置比flannel麻烦很多，但是calico即支持地址分配又支持网络策略，因此其虽然复杂，但是却很受用户接受。还有一点就是calico在实现地址转发的方式中可以基于bgp的方式实现三层网络路由，因此在性能上据说比flannel要强一些。但是flannel也支持三种网络方式。**默认是叠加的，我们使用host gw其实可能比我们想象的功能要强大的多　　

这是介绍的网络插件，我们经常使用时可以兼顾使用flannel的简单，借助于calico实现网络策略，也没必要直接部署canel，可以直接部署flannel做平时的网络管理功能若需要用到网络策略时再部署一个calico，只让其部署网络策略，搭配起来使用，而不用第三方专门合好的插件

flannel默认情况下是用的vxlan的方式来作为后端网络传输机制的。正常情况下，两个节点之间怎么通信呢？如果基于宿主机之间桥接直接通信那么他们应该就可以直接借助于物理网卡实现报文交换，但现在是我们要做成一个叠加网络，使得两边主机之上应该各自有一个专门用于叠加报文封装的隧道，通常称之为flannel.0，flannel.1之类的接口，他们的ip很独特，为10.244.0.0等等，掩码也很独特，为32位，是用来封装隧道协议报文的，并且可以看到我们物理网卡mtu为1500，而他的mtu为1450要留出来一部分，因为我们要做叠加隧道封装的话会有额外的开销，他把额外的开销给留出来了，但这些开销不是没有，这些报文额外的空间不是不被占用，而是被占用了留给我们隧道使用而已，使用`ip addr`可以查看到flannel.1和cni.0接口

![flannel]({{ "/assets/img/flannel/flannel.png" | relative_url}})

另外还有cni的接口，对应在flannel.1之上对应有一个10.244.0.1的接口，被当前主机作为这个隧道协议上在本地通信时使用的接口，但只有创建完pod以后有容器运行以后这个接口才会出现，默认情况下可能只有flannel.1这个接口存在

基于此我们可以简单理解一下，正常情况下我们的网络两个接口直接我们是假设期望他们能直接通信的，但是他两之间无法直接通信，要借助于我们的隧道接口进行通信，比如我们称为叫flannel，这也就意味着在他两个之间我们要构建一个额外添加了开报文传输的一个传输通道，使得他两之间的通信借助这个传输通道来实现，就好像他在直接通信一样，这个后端通信我们所使用的功能就是flannel接口，每一次在这个通讯之外再封装一个二层或三层或四层的直接报文，默认情况下我们用的是VXLAN机制，承载外层的隧道我们用的是VXLAN（扩展的虚拟局域网）协议，其在实现报文通信时他是一种类似于四层隧道一样的协议，也就意外着我们正常传输一个以太网帧的时候他就能在链路层直接进行传输，那怎么传呢？当一个帧发过来的时候，他要经过这个隧道接口再格外封装一层报文守护。第一，他要封装一个VXLAN守护。第二、VXLAN守护之外是UDP协议守护，他们用UDP进行传输。第三、UDP协议之外才是IP守护。第四、IP之外又有一个以太网守护。所以这几种守护完全都是额外开销。所以基于VXLAN可能性能上会有点低，但是好处就在于我们在这里可以独立管理一个网络，而且彼此之间和物理网络是不相干扰的

但是这只是我们对应的flannel网络的后端传输的第一种方式。所以flannel支持多种后端传输方式
1. VXLAN：如上所述

2. host-gw（Host Gateway）：主机网关，一样的方式，我们有多个节点，每一个节点上都需要运行pod，因此对第一个节点和第二个节点上的pod大家各自有一个专有网段，但是我们把主机自己所在的这个网络接口当做是网关来使用，从而能给这些pod之间各自配一个ip地址并通过这个网关对外进行传递。我们在物理机上创建一个虚拟接口，然后我们每一个pod也有一个虚拟接口，这个虚拟接口在传输报文时不是通过隧道承载传递出去的，而是当其需要传输报文时一看目标主机不是本地的，他就应该把报文传给网关，即我们本地的专用虚拟接口，这个网关看到报文以后他会查我们本地的路由表，然后再经过物理网卡就发出去了，这个路由表中记录了到达哪个网络就跳到哪儿。假如主机一pod网络我们使用的是10.244.0.0/24，主机二pod网络使用的是10.244.1.0/24。所以假设我们主机一上的pod IP为10.244.0.2，主机二上的pod IP为10.244.1.2，当我们主机一pod发报文出去时发现对端主机是10.244.1.2，不在自己的网段内，然后就将报文送到网关，网关路由表记录了要到达10.244.1.0/24网络需要送给对端的物理网卡。所以这个报文他会通过本机的物理网卡直接传给主机二的物理网卡。主机二物理网卡一看这就是本机上的另外一个网卡（即这个虚拟接口），所以报文送给该接口，然后报文就到达了主机二了。通过这种方式就相当于是直接将节点当网关，不过这样的话主机的路由表会很大，假如是一个拥有三千节点的集群，大概至少要有三千个路由条目才行。这种比VXLAN方式性能好多了，因为几乎没有什么额外开销。但是其有一个缺陷就是要求各节点必须工作于同一个二层网络中。这样会使得同一个网段中主机非常多，万一发一个泛洪报文，因为彼此之间没有隔离所以每一个主机都有可能受到波及和影响。如果节点不在同一二层网络中就不能使用该模式。我们部署k8s集群所有主机没必要在同一网络中，我们有时候部署很大的k8s集群时中间是有路由器隔离的

但是我们又期望用这种高性能又期望各个节点又可以不在同一网段时要怎么办呢？其实他们还支持第二种方式，在VXLAN上也支持类似于host-gw这种模式，VXLAN还可以这样玩，如果说节点和节点之间通信时大家在同一个网络中，那么我们就直接使用host-gw的方式进行通信，因此我们不用隧道转发不用隧道承载，但是有些我们目标节点与当前节点，所谓当前pod所在的节点与目标pod所在的节点中间隔了路由设备的时候那么他就会自动转为VXLAN的方式使用叠加网络进行通信。这就是柔和了VXLAN和host-gw的一种解决方案。即VXLAN的Directrouting模式

UDP方式：即纯粹的UDP方式进行转发，这种方式比VXLAN性能更差，因为这是用普通的UDP报文转发的还不是用VXLAN专用的报文转发的。因此性能比前两种方式低很多很多，即前两种都不支持的情况下才使用。flannel刚刚被发明出来的时候linux内核尚且不支持VXLAN，linux内核没有VXLAN模块，而那时候host-gw又有很高的技术门槛，所以早期flannel就用的最差的方式UDP，因此大家产生了一种偏见认为flannel性能很差

接下来我们来改一改flannel用法，以及了解一下他的配置参数是怎么配置的，不过要注意的是我们要配置flannel的话，我们定义他使用VXLAN之类配置的话他的信息定义起来也是JSON格式的比较简单，我们要想使用VXLAN这种方式改为使用Directrouting，即能用直接路由就用不能用就用隧道进行转发的方式，这个时候我们需要自己定义flannel自己的configmap配置文件。大家可以发现flannel本身和k8s没有什么关系，他只是k8s的插件而已，都是第三方的，意思是说为了让k8s运行他事先都得存在才行。没有我们插件k8s没法跑起来，他要事先存在就很头疼。传统方式我们是将flannel部署在集群中

**我们之前讲过部署k8s有两种传统方式，第一种方式为直接部署在节点上，使用systemctl来启动服务之类的，这种方式启动起来比较难。第二种方式为使用kubeadm把我们k8s自己的很多组件除了kubelete和docker以外都运行为pod，只不过他们作为k8s核心组件时都是运行为静态pod的，叫static pod。同样的逻辑，如果我们要把这些系统的组件部署为守护进程的话flannel一般也可以部署为系统的守护进程。因为他是被kubelet所调用的。因此但凡有kubelet的节点上都应该部署flannel，因为kubelet存在就应该运行pod，而pod就应该需要网络，而网络是kubelet调flannel或者其它网络插件来实现为网络做设置的**

那么flannel自身是部署为系统级的守护进程还是部署在k8s之上的pod呢？两种方式都支持，只不过对第二种方式来讲我们必须把flannel配置为共享他所运行的节点的网络名称空间的pod，所以flannel自己的控制器如果作为pod来部署的话一定是一个demonset，一个节点上只运行一个pod副本，并且这个pod副本直接共享宿主机的网络名称空间，因为这样的pod才能设置宿主机的网络名称空间。 不然他就没法代为在pod中运行又能改宿主机的网络接口了。比如他创建一个cni，每一个pod启动时他要创建另外一段虚拟网卡，另外一半还要在虚拟机上，还能添加到某个桥上去，这个功能如果不能共享宿主机网络名称空间显然是做不到的，另外，如果我们把flannel托管运行在k8s之上作为pod运行的话他虽然是pod但相当于模拟运行了系统级的守护进程。而既然作为pod运行我们要配置它，配置pod应用我们可以使用configmap和secret，当我们使用kubeadm部署时我们的flannel自身也是托管运行在k8s上作为pod存在的

可以看到我们flannel是被配置为daemonset控制器的，因为我们集群中的节点数量为3因此运行了三个副本

```html
# kubectl get daemonset -n kube-system
# kubeadm get configmap XXX -n kube-system -o yaml
```

flannel的配置参数：大体上来讲配置参数如下（上述json字符串中也可看到）
1. Network：flannel使用的CIDR格式的网络地址，用于为Pod配置网络功能。他通常会是全局的，他通常会是16位或8位掩码的，然后这个全局网络会划分成子网，每一个节点使用一个子网，比如他使用10.244.0.0/16，但是我们的节点1，节点2，节点3分别分一个子网，比如master：10.244.0.0/24。node01：10.244.1.0/24。...直到node255：10.244.255.0/24，不过这就意味着我们整个集群支持255个节点，想要更多的节点那就需要使用更大的网络，比如10.0.0.0/8，这样就能有10.0.0.0/24-10.255.255.0/24个节点。就可以有2的16次方，即65536个。因此我们一般不会用这么大的网段，比如我们可以使用16或12位掩码，最起码要留出来service使用的网络。比如service使用10.9.0.0/12，而pod使用10.10.0.0/12。

2. subnetLen：把Network切分子网供各节点使用时，使用多长的掩码进行切分，默认为24位。默认为什么是使用上述方式切子网呢？那是因为我们的子网默认使用24位掩码去切分子网。 留24位就意味着8位是可以当主机地址的，也就意味着将来在一个节点上可以同时运行两百多个Pod,如果我们运行不了那么多运行20个就够了那么我们的掩码可以再长一点，比如使用28位掩码，这样就能支持更多节点了。

3. SubnetMin：指明我们子网中的地址段中从最大最小是多少可以分给节点使用，默认就是从第一个开始，比如10/244.0.0/16中就是从10.244.0.0/24开始，10.244.255.0/24结束。但是我们可以限制，从10.244.10.0/24开始，前面10个不让其使用

4. SubnetMAx：同c中，指明最大可以以哪个网段结束，比如10.244.100.0/24，后面155个不让其使用。

5. Backend：指明各pod之间通信时使用什么样的方式来进行通信。有三种方式，vxlan，host-gw，udp。vxlan还有两种方式：默认的vxlan和Directrouting类型的VXLAN。


分别找运行在两个node上的Pod，登录容器中

```html
# kubectl exec -it [pod_name] /bin/sh
# ping 节点2Pod的Ip
```
然后在cni0接口和flannel.1的接口分别抓包

```html
# tcpdump -i cni0 -nn icmp
# tcpdump -i flannel.1 -nn icmp
```
实际上这个报文被送达到flannel这个接口时他要先从cni0进来，从flannel.1出去，然后再借助于物理网卡发出去，到达flannel.1的时候就已经被封装为vxlan报文了，因此我们再来抓flannel.1接口的包

然后我们看我们node1物理网卡和node2之间通信的报文，可以看到有overlay的报文

修改flannel的配置为Directrouting，尝试使用edit具体Pod的方法，发现并没有生效，`ip route`发现其他节点的子网使用的还是flannel.1的接口，使用delete配置文件yaml的方法生效，`ip route`后发现变成宿主机的物理接口
```html
# tcpdump -i ens33 -nn icmp
```
在物理接口上抓包可以看到并没有显示说二者是被overlay网络重载，和我们物理桥接是很想象的，所以我们flannel也能实现两个pod之间的桥接网络。这种通信性能肯定是很ok的，但是如果两个节点是跨网段的，他会自动降级为overlay

若我们将flannel的配置文件的net-conf.json中的Type内容改为host-gw时那它就是host-gw模式，host-gw是不支持Directrouting的，所以部署时将vxlan改为host-gw即可，不过host-gw各主机不能跨网段，跨网段是通信不了的　

**calico网络组件**

```html
# calicoctl node status
```
状态全是UP即可

**cilium网络组件**

下面有cilium网络组件的安装例子

**flannel网络组件补充**

flannel实际上实现了以下两点

1. 它能协助kubernetes，给每一个Node上的Docker容器分配互相不冲突的IP地址
2. 它能在这些IP地址上建立一个覆盖网络(Overlay network)，通过这个覆盖网络，将数据包原封不动的传递到目标容器

flannel创建一个名为flannel0的网桥，而且这个网桥的一端连接到docker0网桥，另一端连接一个叫作flanneld的服务进程，flanneld进程并不简单，它首先上连etcd，利用etcd来管理可以分配的IP地址段资源，同时监控etcd中每个Pod的实际地址，并在内存中建立一个Pod节点路由器。然后下连docker0和物理网络，使用内存中的Pod节点路由器，将docker0发给它的数据包包装起来，利用物理网络的连接将数据包投递到目标flanneld，从来完成Pod到Pod之间的直接的地址通信

flannel底层通信协议可以选UDP，Vxlan，ASW VPC等，源flanneld加包，目标flanneld解包，最终docker0看到的就是原始数据，让你感觉不到flanneld的存在，flanneld中有个etcdctl set网络的命令，就是flanneld分配个每个docker的虚拟Ip地址段，会覆盖docker0网桥，所以之前先停docker

#### 存储描述

kubernetes使用PersistentVolume（PV）和PersistentVolumeClaim（PVC）来实现存储管理

PV是对底层网络共享存储的抽象，将共享存储定义为一种资源，GLUSTERFS，iSCSI等共享存储，通过插件式的机制完成与共享存储的对接，以供应用访问及使用

PVC则是用户对于存储资源的一个申请，就像Pod消费Node的资源一样，PVC会消费PV的资源，PVC可以申请特定的存储空间和访问模式

StorageClass是新的资源对象，用于标记存储资源的特性和性能

#### pause容器

每个Pod里都有一个特殊的容器`pause-amd64`，官网解释`it’s part of the infrastructure. This container is started first in all Pods to setup the network for the Pod`，pause容器主要为每个业务容器提供以下功能

1. PID命名空间：Pod中的不同应用程序可以看到其他应用程序的进程ID
2. 网络命名空间：Pod中的多个容器能够访问同一个IP和端口范围
3. IPC命名空间：Pod中的多个容器能够使用SystemV IPC或POSIX消息队列进行通信
4. UTS命名空间：Pod中的多个容器共享一个主机名；Volumes（共享存储卷）

Pod中的各个容器可以访问在Pod级别定义的Volumes。

Pause是C语言实现的进程，核心代码就是28行，有个无限循环for(;;)，里面是一个系统调用pause();

#### K8s里的资源

1、statefulset的使用，可以参考部署redis的[文章](https://zqmalyssa.github.io/strap/2019/10/12/redis-post.html)


### kubernetes中的operator

你是否曾经想过 SRE 团队是如何有效地成功管理复杂的应用？在 Kubernetes 生态系统中，Kubernetes Operator 可以给你答案

Kubernetes Operator 这一概念是由 CoreOS 的工程师于 2016 年提出的，这是一种原生的方式来构建和驱动 Kubernetes 集群上的每一个应用，它需要特定领域的知识。它提供了一种一致的方法，通过与 Kubernetes API 的紧密合作，自动处理所有应用操作过程，而不需要任何人工干预。换句话说，Operator 是一种包装、运行和管理 Kubernetes 应用的方式。（是对应用的管理，应该是对有状态应用的管理）

Kubernetes Operator 模式遵循 Kubernetes 的核心原则之一：控制理论（control theory）。在机器人和自动化领域，它是一种持续运行动态系统的机制。它依赖于一种快速调整工作负载需求的能力，进而能够尽可能准确地适应现有资源。其目标是开发一个具有必要逻辑的控制模型，以帮助应用程序或系统保持稳定。在 Kubernetes 世界中，这部分由 controller 处理。

在循环中，Controller 是个特殊的软件，它可以对集群的变化做出响应，并执行适应动作。第一个 Kubernetes controller 是一个 kube-controller-manager。它被认为是所有 Operator 的前身，Operator 是后来建立的。

简单来说，Controller Loop 是 Controller 动作的基础。想象一下，有一个非终止的进程（在 Kubernetes 中称为和解循环）在不断地发生（就是desired state和current state）

这个过程至少观察一个 Kubernetes 对象，该对象包含有关所需状态的信息。比如：

Deployment

Services

Secrets

Ingress

Config Maps

这些对象由 JSON 或 YAML 中的 manifest 组成的配置文件定义。然后 controller 根据内置逻辑，通过 Kubernetes API 进行持续调整，模仿所需状态，直到当前状态变成所需状态。

通过这种方式，Kubernetes 通过处理不断的更改来处理 Cloud Native 系统的动态性质。为达到预期状态而执行的修改实例包括：

注意到节点宕机时，要求更换新的节点。

检查是否需要复制 pods。

如果需要，创建一个新的负载均衡器。

综上，Operator 基于 Kubernetes 的资源和控制器概念之上构建，但同时又包含了应用程序特定的领域知识。创建Operator 的关键是CRD（自定义资源）的设计

#### Kubernetes Operator 如何工作？

Operator 是一个特定应用程序的 controller，它扩展了一个 Kubernetes API，替代运维工程师或 SRE 工程师来创建、配置和管理复杂的应用程序。在 Kubernetes 官方文档中对此有以下描述：

Operator 是 Kubernetes 的软件拓展，它利用自定义资源来管理应用程序及其组件。Operator 遵循 Kubernetes 的原则，尤其遵循 control loop。（就是应用及其关联的任意组件）

到目前为止，你已经了解 Operator 会利用观察 Kubernetes 对象的 controller。这些 controller 有点不同，因为它们正在追踪自定义对象，通常称为自定义资源（CR）。CR 是 Kubernetes API 的扩展，它提供了一个可以存储和检索结构化数据的地方——你的应用程序的期望状态。整个操作原理如下图所示：

![operator]({{ "/assets/img/kubernetes/operator.png" | relative_url}})

Operator 会持续跟踪与特定类型的自定义资源相关的集群事件。可以跟踪的关于这些自定义资源的事件类型有

Add

Update

Delete

当 Operator 接收任何信息时，它将采取行动将 Kubernetes 集群或外部系统调整到所需的状态，作为其在自定义 controller 中的和解循环（reconciliation loop）的一部分。


`如何添加一个自定义资源`

自定义资源通过添加对你的应用有帮助的新型对象来扩展 Kubernetes 功能。Kubernetes 提供了两种向集群添加自定义资源的方法：

通过 API Aggregation 添加，这是一种高级方法，需要你建立自己的 API 服务器，但你有更多的控制权限。

通过自定义资源定义（CRD）添加，一种不需要复杂编程知识就可以创建的简单方式，作为 Kubernetes API 服务器的扩展。（常用）

自定义资源定义（CRD）:

自定义资源定义（CRD）的出现已经有一段时间了，第一个主要的 API 规范是与 Kubernetes 1.16.0 一起发布的。下面的 manifest 介绍了一个例子：

```html

apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: application.stable.example.com
spec:
  group: stable.example.com
  version: v1
  scope: Namespaced
  names:
    plural: application
    singular: applications
    kind: Application
    shortNames:
    - app


```

这个 CRD 可以让你创建一个名为“Application”的 CR（我们将会在下一个部分使用它）。前两行定义了 apiVersion 和你要创建的对象种类。

Metadata 描述了资源名称，但这里最重要的部分是“spec”字段。它让你可以指定组、版本以及可见性范围——命名空间或集群范围。

然后，你可以用多种格式定义名称，并创建一个方便的缩写，让你执行命令 kubectl get app 来获取现有的 CR。

自定义资源:

以上 CRD 可以让你创建以下自定义资源的 manifest。

```html

apiVersion: stable.example.com/v1
kind: Application
metadata:
  name: application-config
spec:
  image: container-registry-image:v1.0.0
  domain: teamx.yoursaas.io
  plan: premium


```

如你所见，在这里包含了运行特定情况下的应用程序所需的所有必要信息。这个自定义资源将被我们的 Operator 观察到——准确地说，是被 Operator 的自定义 controller 观察到。根据 controller 中的内置逻辑，将模仿所需的状态。它可以为我们的应用程序创建部署、服务和必要的 ConfigMaps。运行它，并在特定的域上通过 ingress 暴露它。这只是一个简单的用例，但你可以根据自己的需求对它进行任何设计。

Operator 还可以配置在 Kubernetes 之外的资源。你可以在不离开 Kubernetes 平台的情况下控制外部路由器的配置或在云中创建数据库。

`Kubernetes Operators：案例研究`

为了对 Kubernetes Operator 有一个整体清晰的认识，我们来看看 Prometheus Operator，它是最早也是最流行的 Operator 之一。它简化了 Prometheus、Alertmanager 以及相关监控组件的部署和配置。

Prometheus Operator 的核心功能是监控 Kubernetes API 服务器上指定对象的变化，并确保当前的 Prometheus 部署与这些对象相匹配。Operator 作用于以下自定义资源定义（CRD）：

Prometheus： 定义了所需 Prometheus 部署

Alertmanager： 定义了所需的 Alertmanager 部署

ServiceMonitor： 它声明性地指定了应该如何监控 Kubernetes 服务的组。Operator 会根据 API 服务器中对象的当前状态自动生成 Prometheus scrape 配置。

PodMonitor： 声明性地指定了应如何监控一组 pod。Operator 会根据 API 服务器中对象的当前状态自动生成 Prometheus scrape 配置。

PrometheusRule： 定义了一组所需的 Prometheus 告警和/或记录规则。Operator 会生成一个规则文件，可供 Prometheus 实例使用。

所以，Prometheus Operator 会自动检测 Kubernetes API 服务器中对上述任何对象的更改，并确保匹配的部署和配置保持同步。

#### operator的实践

Operator Framework 同样也是 CoreOS 开源的一个用于快速开发 Operator 的工具包，该框架包含两个主要的部分：

Operator SDK: 无需了解复杂的 Kubernetes API 特性，即可让你根据你自己的专业知识构建一个 Operator 应用。

Operator Lifecycle Manager OLM: 帮助你安装、更新和管理跨集群的运行中的所有 Operator（以及他们的相关服务）

大致的workflow是这样

Operator SDK 提供以下工作流来开发一个新的 Operator：

1. 使用 SDK 创建一个新的 Operator 项目
2. 通过添加自定义资源（CRD）定义新的资源 API
3. 指定使用 SDK API 来 watch 的资源
4. 定义 Operator 的协调（reconcile）逻辑
5. 使用 Operator SDK 构建并生成 Operator 部署清单文件

我们平时在部署一个简单的 Webserver 到 Kubernetes 集群中的时候，都需要先编写一个 Deployment 的控制器，然后创建一个 Service 对象，通过 Pod 的 label 标签进行关联，最后通过 Ingress 或者 type=NodePort 类型的 Service 来暴露服务，每次都需要这样操作，是不是略显麻烦，我们就可以创建一个自定义的资源对象，通过我们的 CRD 来描述我们要部署的应用信息，比如镜像、服务端口、环境变量等等，然后创建我们的自定义类型的资源对象的时候，通过控制器去创建对应的 Deployment 和 Service，是不是就方便很多了，相当于我们用一个资源清单去描述了 Deployment 和 Service 要做的两件事情。

这里我们将创建一个名为 AppService 的 CRD 资源对象，然后定义如下的资源清单进行应用部署

```html

apiVersion: app.example.com/v1
kind: AppService
metadata:
  name: nginx-app
spec:
  size: 2
  image: nginx:1.7.9
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30002

```

注意这里没有之前的deployment，service等一堆yaml的定义了

开发的环境要求：

要开发 Operator 自然 Kubernetes 集群是少不了的，还需要 Golang 的环境，这里的安装就不多说了，然后还需要一个 Go 语言的依赖管理工具包：dep，由于 Operator SDK 是使用的 dep 该工具包，所以需要我们提前安装好，可以查看资料：https://github.com/golang/dep，另外一个需要说明的是，由于 dep 去安装的时候需要去谷歌的网站拉取很多代码，所以正常情况下的话是会失败的，需要做什么工作大家应该清楚吧？要科学。

安装 operator-sdk

```html

mac的

$ brew install operator-sdk
......
$ operator-sdk version
operator-sdk version: v0.7.0
$ go version
go version go1.11.4 darwin/amd64

```

环境准备好了，接下来就可以使用 operator-sdk 直接创建一个新的项目了，命令格式为： operator-sdk new

按照上面我们预先定义的 CRD 资源清单，我们这里可以这样创建：

```html

#创建项目目录
$ mkdir -p operator-learning  
# 设置项目目录为 GOPATH 路径
$ cd operator-learning && export GOPATH=$PWD  
$ mkdir -p $GOPATH/src/github.com/cnych
$ cd $GOPATH/src/github.com/cnych
# 使用 sdk 创建一个名为 opdemo 的 operator 项目
$ operator-sdk new opdemo
......
# 该过程需要科学上网，需要花费很长时间，请耐心等待
......
$ cd opdemo && tree -L 2
.
├── Gopkg.lock
├── Gopkg.toml
├── build
│   ├── Dockerfile
│   ├── _output
│   └── bin
├── cmd
│   └── manager
├── deploy
│   ├── crds
│   ├── operator.yaml
│   ├── role.yaml
│   ├── role_binding.yaml
│   └── service_account.yaml
├── pkg
│   ├── apis
│   └── controller
├── vendor
│   ├── cloud.google.com
│   ├── contrib.go.opencensus.io
│   ├── github.com
│   ├── go.opencensus.io
│   ├── go.uber.org
│   ├── golang.org
│   ├── google.golang.org
│   ├── gopkg.in
│   ├── k8s.io
│   └── sigs.k8s.io
└── version
	 └── version.go

23 directories, 8 files

```

到这里一个全新的 Operator 项目就新建完成了，说下项目的结构：


使用operator-sdk new命令创建新的 Operator 项目后，项目目录就包含了很多生成的文件夹和文件。

Gopkg.toml Gopkg.lock — Go Dep 清单，用来描述当前 Operator 的依赖包。

cmd - 包含 main.go 文件，使用 operator-sdk API 初始化和启动当前 Operator 的入口。

deploy - 包含一组用于在 Kubernetes 集群上进行部署的通用的 Kubernetes 资源清单文件。

pkg/apis - 包含定义的 API 和自定义资源（CRD）的目录树，这些文件允许 sdk 为 CRD 生成代码并注册对应的类型，以便正确解码自定义资源对象。
pkg/controller - 用于编写所有的操作业务逻辑的地方

vendor - golang vendor 文件夹，其中包含满足当前项目的所有外部依赖包，通过 go dep 管理该目录。

`我们主要需要编写的是pkg目录下面的 api 定义以及对应的 controller 实现。`

`添加API`

接下来为我们的自定义资源添加一个新的 API，按照上面我们预定义的资源清单文件，在 Operator 相关根目录下面执行如下命令

```html

operator-sdk add api --api-version=app.example.com/v1 --kind=AppService

```

`添加控制器`

上面我们添加自定义的 API，接下来可以添加对应的自定义 API 的具体实现 Controller，同样在项目根目录下面执行如下命令

```html

operator-sdk add controller --api-version=app.example.com/v1 --kind=AppService

```

这个时候pkg中的apis和controller就都有所变化了，增加了文件，相当于脚手架搭建完成

`自定义API`

打开源文件pkg/apis/app/v1/appservice_types.go，需要我们根据我们的需求去自定义结构体 AppServiceSpec，我们最上面预定义的资源清单中就有 size、image、ports 这些属性，所有我们需要用到的属性都需要在这个结构体中进行定义：

```html

type AppServiceSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "operator-sdk generate k8s" to regenerate code after modifying this file
	// Add custom validation using kubebuilder tags: https://book.kubebuilder.io/beyond_basics/generating_crd.html
	Size  	  *int32                      `json:"size"`
	Image     string                      `json:"image"`
	Resources corev1.ResourceRequirements `json:"resources,omitempty"`
	Envs      []corev1.EnvVar             `json:"envs,omitempty"`
	Ports     []corev1.ServicePort        `json:"ports,omitempty"`
}

import (
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    appv1 "github.com/cnych/opdemo/pkg/apis/app/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

```

这里的 resources、envs、ports 的定义都是直接引用的"k8s.io/api/core/v1"中定义的结构体，而且需要注意的是我们这里使用的是ServicePort，而不是像传统的 Pod 中定义的 ContanerPort，这是因为我们的资源清单中不仅要描述容器的 Port，还要描述 Service 的 Port。

然后一个比较重要的结构体AppServiceStatus用来描述资源的状态，当然我们可以根据需要去自定义状态的描述，我这里就偷懒直接使用 Deployment 的状态了：

```html

type AppServiceStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "operator-sdk generate k8s" to regenerate code after modifying this file
	// Add custom validation using kubebuilder tags: https://book.kubebuilder.io/beyond_basics/generating_crd.html
	appsv1.DeploymentStatus `json:",inline"`
}

```

定义完成后，在项目根目录下面执行如下命令：

```html

$ operator-sdk generate k8s

```

该命令是用来根据我们自定义的 API 描述来自动生成一些代码，目录pkg/apis/app/v1/下面以zz_generated开头的文件就是自动生成的代码，里面的内容并不需要我们去手动编写。

这样我们就算完成了对自定义资源对象的 API 的声明。

`实现业务逻辑`

上面 API 描述声明完成了，接下来就需要我们来进行具体的业务逻辑实现了，编写具体的 controller 实现，打开源文件pkg/controller/appservice/appservice_controller.go，需要我们去更改的地方也不是很多，核心的就是Reconcile方法，该方法就是去不断的 watch 资源的状态，然后根据状态的不同去实现各种操作逻辑，核心代码如下：

```html

func (r *ReconcileAppService) Reconcile(request reconcile.Request) (reconcile.Result, error) {
	reqLogger := log.WithValues("Request.Namespace", request.Namespace, "Request.Name", request.Name)
	reqLogger.Info("Reconciling AppService")

	// Fetch the AppService instance
	instance := &appv1.AppService{}
	err := r.client.Get(context.TODO(), request.NamespacedName, instance)
	if err != nil {
		if errors.IsNotFound(err) {
			// Request object not found, could have been deleted after reconcile request.
			// Owned objects are automatically garbage collected. For additional cleanup logic use finalizers.
			// Return and don't requeue
			return reconcile.Result{}, nil
		}
		// Error reading the object - requeue the request.
		return reconcile.Result{}, err
	}

	if instance.DeletionTimestamp != nil {
		return reconcile.Result{}, err
	}

	// 如果不存在，则创建关联资源
	// 如果存在，判断是否需要更新
	//   如果需要更新，则直接更新
	//   如果不需要更新，则正常返回

	deploy := &appsv1.Deployment{}
	if err := r.client.Get(context.TODO(), request.NamespacedName, deploy); err != nil && errors.IsNotFound(err) {
		// 创建关联资源
		// 1. 创建 Deploy
		deploy := resources.NewDeploy(instance)
		if err := r.client.Create(context.TODO(), deploy); err != nil {
			return reconcile.Result{}, err
		}
		// 2. 创建 Service
		service := resources.NewService(instance)
		if err := r.client.Create(context.TODO(), service); err != nil {
			return reconcile.Result{}, err
		}
		// 3. 关联 Annotations
		data, _ := json.Marshal(instance.Spec)
		if instance.Annotations != nil {
			instance.Annotations["spec"] = string(data)
		} else {
			instance.Annotations = map[string]string{"spec": string(data)}
		}

		if err := r.client.Update(context.TODO(), instance); err != nil {
			return reconcile.Result{}, nil
		}
		return reconcile.Result{}, nil
	}

	oldspec := appv1.AppServiceSpec{}
	if err := json.Unmarshal([]byte(instance.Annotations["spec"]), oldspec); err != nil {
		return reconcile.Result{}, err
	}

	if !reflect.DeepEqual(instance.Spec, oldspec) {
		// 更新关联资源
		newDeploy := resources.NewDeploy(instance)
		oldDeploy := &appsv1.Deployment{}
		if err := r.client.Get(context.TODO(), request.NamespacedName, oldDeploy); err != nil {
			return reconcile.Result{}, err
		}
		oldDeploy.Spec = newDeploy.Spec
		if err := r.client.Update(context.TODO(), oldDeploy); err != nil {
			return reconcile.Result{}, err
		}

		newService := resources.NewService(instance)
		oldService := &corev1.Service{}
		if err := r.client.Get(context.TODO(), request.NamespacedName, oldService); err != nil {
			return reconcile.Result{}, err
		}
		oldService.Spec = newService.Spec
		if err := r.client.Update(context.TODO(), oldService); err != nil {
			return reconcile.Result{}, err
		}

		return reconcile.Result{}, nil
	}

	return reconcile.Result{}, nil

}

```

上面就是业务逻辑实现的核心代码，逻辑很简单，就是去判断资源是否存在，不存在，则直接创建新的资源，创建新的资源除了需要创建 Deployment 资源外，还需要创建 Service 资源对象，因为这就是我们的需求，当然你还可以自己去扩展，比如在创建一个 Ingress 对象。更新也是一样的，去对比新旧对象的声明是否一致，不一致则需要更新，同样的，两种资源都需要更新的。

另外两个核心的方法就是上面的resources.NewDeploy(instance)和resources.NewService(instance)方法，这两个方法实现逻辑也很简单，就是根据 CRD 中的声明去填充 Deployment 和 Service 资源对象的 Spec 对象即可。

```html

func NewDeploy(app *appv1.AppService) *appsv1.Deployment {
	labels := map[string]string{"app": app.Name}
	selector := &metav1.LabelSelector{MatchLabels: labels}
	return &appsv1.Deployment{
		TypeMeta: metav1.TypeMeta{
			APIVersion: "apps/v1",
			Kind:       "Deployment",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name:      app.Name,
			Namespace: app.Namespace,

			OwnerReferences: []metav1.OwnerReference{
				*metav1.NewControllerRef(app, schema.GroupVersionKind{
					Group: v1.SchemeGroupVersion.Group,
					Version: v1.SchemeGroupVersion.Version,
					Kind: "AppService",
				}),
			},
		},
		Spec: appsv1.DeploymentSpec{
			Replicas: app.Spec.Size,
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: labels,
				},
				Spec: corev1.PodSpec{
					Containers: newContainers(app),
				},
			},
			Selector: selector,
		},
	}
}

func newContainers(app *v1.AppService) []corev1.Container {
	containerPorts := []corev1.ContainerPort{}
	for _, svcPort := range app.Spec.Ports {
		cport := corev1.ContainerPort{}
		cport.ContainerPort = svcPort.TargetPort.IntVal
		containerPorts = append(containerPorts, cport)
	}
	return []corev1.Container{
		{
			Name: app.Name,
			Image: app.Spec.Image,
			Resources: app.Spec.Resources,
			Ports: containerPorts,
			ImagePullPolicy: corev1.PullIfNotPresent,
			Env: app.Spec.Envs,
		},
	}
}
```

```html

func NewService(app *v1.AppService) *corev1.Service {
	return &corev1.Service {
		TypeMeta: metav1.TypeMeta {
			Kind: "Service",
			APIVersion: "v1",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name: app.Name,
			Namespace: app.Namespace,
			OwnerReferences: []metav1.OwnerReference{
				*metav1.NewControllerRef(app, schema.GroupVersionKind{
					Group: v1.SchemeGroupVersion.Group,
					Version: v1.SchemeGroupVersion.Version,
					Kind: "AppService",
				}),
			},
		},
		Spec: corev1.ServiceSpec{
			Type: corev1.ServiceTypeNodePort,
			Ports: app.Spec.Ports,
			Selector: map[string]string{
				"app": app.Name,
			},
		},
	}
}

```

这样我们就实现了 AppService 这种资源对象的业务逻辑。

`调试`

如果我们本地有一个可以访问的 Kubernetes 集群，我们也可以直接进行调试，在本地用户~/.kube/config文件中配置集群访问信息，下面的信息表明可以访问 Kubernetes 集群：

首先，在集群中安装 CRD 对象：

```html

$ kubectl create -f deploy/crds/app_v1_appservice_crd.yaml
customresourcedefinition "appservices.app.example.com" created
$ kubectl get crd
NAME                                   AGE
appservices.app.example.com            <invalid>
......

```

当我们通过kubectl get crd命令获取到我们定义的 CRD 资源对象，就证明我们定义的 CRD 安装成功了。其实现在只是 CRD 的这个声明安装成功了，但是我们这个 CRD 的具体业务逻辑实现方式还在我们本地，并没有部署到集群之中，我们可以通过下面的命令来在本地项目中启动 Operator 的调试：

```html
$ operator-sdk up local                                                     
INFO[0000] Running the operator locally.                
INFO[0000] Using namespace default.

```

上面的命令会在本地运行 Operator 应用，通过~/.kube/config去关联集群信息，现在我们去添加一个 AppService 类型的资源然后观察本地 Operator 的变化情况，资源清单文件就是我们上面预定义的（deploy/crds/app_v1_appservice_cr.yaml，注意是CR哈，之前create是CRD）

```html
apiVersion: app.example.com/v1
kind: AppService
metadata:
  name: nginx-app
spec:
  size: 2
  image: nginx:1.7.9
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30002

#直接创建这个资源对象

$ kubectl create -f deploy/crds/app_v1_appservice_cr.yaml
appservice "nginx-app" created		


```

我们可以看到我们的应用创建成功了，这个时候查看 Operator 的调试窗口会有如下的信息出现：

```html

......
{"level":"info","ts":1559207416.670523,"logger":"controller_appservice","msg":"Reconciling AppService","Request.Namespace":"default","Request.Name":"nginx-app"}
{"level":"info","ts":1559207417.004226,"logger":"controller_appservice","msg":"Reconciling AppService","Request.Namespace":"default","Request.Name":"nginx-app"}
{"level":"info","ts":1559207417.004331,"logger":"controller_appservice","msg":"Reconciling AppService","Request.Namespace":"default","Request.Name":"nginx-app"}
{"level":"info","ts":1559207418.33779,"logger":"controller_appservice","msg":"Reconciling AppService","Request.Namespace":"default","Request.Name":"nginx-app"}
{"level":"info","ts":1559207418.951193,"logger":"controller_appservice","msg":"Reconciling AppService","Request.Namespace":"default","Request.Name":"nginx-app"}
......

```

然后我们可以去查看集群中是否有符合我们预期的资源出现：

```html

$ kubectl get AppService
NAME        AGE
nginx-app   <invalid>
$ kubectl get deploy
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-app                2         2         2            2           <invalid>
$ kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP        76d
nginx-app    NodePort    10.108.227.5   <none>        80:30002/TCP   <invalid>
$ kubectl get pods
NAME                                      READY     STATUS    RESTARTS   AGE
nginx-app-76b6449498-2j82j                1/1       Running   0          <invalid>
nginx-app-76b6449498-m4h58                1/1       Running   0          <invalid>

```

看到了吧，我们定义了两个副本（size=2），这里就出现了两个 Pod，还有一个 NodePort=30002 的 Service 对象，我们可以通过该端口去访问下应用：

清理：

```html
$ kubectl delete -f deploy/crds/app_v1_appservice_crd.yaml
$ kubectl delete -f deploy/crds/app_v1_appservice_cr.yaml

```

`部署`

执行下面的命令构建 Operator 应用打包成 Docker 镜像：

```html

$ operator-sdk build cnych/opdemo                         
INFO[0002] Building Docker image cnych/opdemo           
Sending build context to Docker daemon  400.7MB
Step 1/7 : FROM registry.access.redhat.com/ubi7-dev-preview/ubi-minimal:7.6
......
Successfully built a8cde91be6ab
Successfully tagged cnych/opdemo:latest
INFO[0053] Operator build complete.

$ docker push cnych/opdemo

#镜像推送成功后，使用上面的镜像地址更新 Operator 的资源清单：

$ sed -i 's|REPLACE_IMAGE|cnych/opdemo|g' deploy/operator.yaml
# 如果你使用的是 Mac 系统，使用下面的命令
$ sed -i "" 's|REPLACE_IMAGE|cnych/opdemo|g' deploy/operator.yaml

现在 Operator 的资源清单文件准备好了，然后创建对应的 RBAC 的对象

# Setup Service Account
$ kubectl create -f deploy/service_account.yaml
# Setup RBAC
$ kubectl create -f deploy/role.yaml
$ kubectl create -f deploy/role_binding.yaml

权限相关声明已经完成，接下来安装 CRD 和 Operator：

# Setup the CRD
$ kubectl apply -f deploy/crds/app_v1_appservice_crd.yaml
$ kubectl get crd
NAME                                   CREATED AT
appservices.app.example.com            2019-05-30T17:03:32Z
......
# Deploy the Operator
$ kubectl create -f deploy/operator.yaml
deployment.apps/opdemo created
$ kubectl get pods
NAME                                      READY   STATUS    RESTARTS   AGE
opdemo-64db96d575-9vtq6                   1/1     Running   0          2m2s

```
到这里我们的 CRD 和 Operator 实现都已经安装成功了。

现在我们再来部署我们的 AppService 资源清单文件，现在的业务逻辑就会在上面的opdemo-64db96d575-9vtq6的 Pod 中去处理了

```html

$ kubectl create -f deploy/crds/app_v1_appservice_cr.yaml

```

Operator SDK 为我们创建了一个快速启动的代码和相关配置，如果我们要开始处理相关的逻辑，我们可以在项目中搜索TODO(user)这个注释来实现我们自己的逻辑，比如在我的 VSCode 环境中，看上去是这样的：

### kubernetes的安装

这边记录一次kubernetes1.17.0和cilium1.7.4的结合部署过程，使用kubeadm去部署

首先，机器拿到手，做一些前置任务

```html
//关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

//关闭selinux
sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久
setenforce 0  # 临时

//关闭swap
swapoff -a  # 临时
vim /etc/fstab  # 永久 #号注释
sed -i 's/.*swap.*/#&/' /etc/fstab

//每台主机添加hosts
cat >> /etc/hosts << EOF
192.168.1.15 k8s-master
192.168.1.11 k8s-node1
192.168.1.14 k8s-node2
EOF

//将桥接的IPv4流量传递到iptables的链
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system  # 生效

//互信和NTP自己搞一下，不是必须，三台机器
ssh-keygen -t rsa
剩余的机器
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.3.21
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.3.22

```

安装docker

```html
//阿里的docker-ce源
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum list docker-ce --showduplicates|sort -r
yum install docker-ce-18.06.1.ce -y

//配置镜像加速，在自己的阿里云上有地址
```
安装kubernetes

```html
//配置阿里云的仓库地址
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum install -y kubelet-1.17.0 kubeadm-1.17.0 kubectl-1.17.0 //指定版本安装

systemctl enable kubelet && systemctl start kubelet

然后初始化了

kubeadm init \
--apiserver-advertise-address=10.6.57.55 \
--kubernetes-version v1.17.0 \
--pod-network-cidr=10.217.0.0/16

注意，要安装cilium，pod的network最好是上面的地址，这个时候如果初始化拉不到镜像，只能你懂得

docker load -i coredns-1.6.5.tar.gz
docker load -i etcd-3.4.3-0.tar.gz
docker load -i kube-apiserver-1.17.0.tar.gz
docker load -i kube-controller-manager-1.17.0.tar.gz
docker load -i kube-proxy-1.17.0.tar.gz
docker load -i kube-scheduler-1.17.0.tar.gz
docker load -i pause-3.1.tar.gzube

ok，这个时候网络插件不装的话节点是不会ready的

mkdir -p $HOME/.kube
scp root@172.23.216.48:/etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

这个时候就可以使用kubectl的命令了，如果别的节点（node）需要使用，则

scp /etc/kubernetes/admin.conf  qmzhang@10.6.57.56:/home/qmzhang
scp /etc/kubernetes/admin.conf  qmzhang@10.6.57.56:/home/qmzhang

本master节点需要pod调度的话
kubectl taint nodes --all node-role.kubernetes.io/master-

别的node节点加入到集群
kubeadm join 10.6.57.55:6443 --token h11cld.xqil46qfhlavykl3 \
    --discovery-token-ca-cert-hash sha256:91d29eeb3ff0eb96d36ffb51dc0c223ffef47969efa77516f12fd5adb348cd70

```
安装cilium的网络插件，看[官网](https://docs.cilium.io/en/stable/kubernetes/intro/)吧，在线的yaml文件会随着版本变化的

```html
//最新版本的 1.7.4
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/1.7.4/install/kubernetes/quick-install.yaml

//镜像拉不下来，你懂得
docker load -i cilium/cilium:v1.7.4.tar.gz
docker load -i cilium/operator:v1.7.4.tar.gz

下面测试连接的镜像

docker load -i son-mock-1.0.tar
docker load -i alpine-curl-0.1.8.tar

//测试连通性的，镜像如果还是拉不下，还是你懂的
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.7.4/examples/kubernetes/connectivity-check/connectivity-check.yaml

//如果之前安装过其他网络插件，如flannel，需要清除，但其实有共存的方法，先清干净，不然是create不出来的
rm -rf /var/lib/cni/
rm -rf /run/flannel
rm -rf /etc/cni/

ip link
ip link | grep flannel
ifconfig | grep flannel

ifconfig <name of interface from ip link> down
ip link delete <name of interface from ip link>
ifconfig flannel.1 down
ip link delete flannel.1

```

到这里，k8加cilium的集群应该是安装完毕了

```html

//这边提一个关于证书的运维的问题，kubeadm默认生成的ca证书有效期是10年，其他证书（如etcd证书、apiserver证书等）有效期均为1年。通过以下命令查看：

openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text |grep ' Not '

Not Before: Jun  5 09:17:47 2020 GMT
Not After : Jun  5 09:17:48 2021 GMT

报错信息如下：Unable to connect to the server: x509: certificate has expired or is not yet valid

在kubernetes 1.15版本后提供了强大的证书管理功能

kubeadm alpha certs renew all // 全局的更新，没报错就成功

kubeadm alpha certs renew all --config /root/config.yaml // 保存配置的，后续可以配合kubeadm alpha certs check-expiration --config /root/config.yaml查看过期，否则就一个个查看

#apiserver证书有效期
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text |grep ' Not '

#apiserver-kubelet-client证书有效期
openssl x509 -in /etc/kubernetes/pki/apiserver-kubelet-client.crt -noout -text |grep ' Not '

#front-proxy-client证书有效期
openssl x509 -in /etc/kubernetes/pki/front-proxy-client.crt -noout -text |grep ' Not '

#apiserver-etcd-client证书有效期
openssl x509 -in /etc/kubernetes/pki/apiserver-etcd-client.crt -noout -text |grep ' Not '

#这里我们可以看到ca证书都是10年
openssl x509 -in /etc/kubernetes/pki/ca.crt -noout -text |grep ' Not '

#这里我们可以看到ca证书都是10年
openssl x509 -in /etc/kubernetes/pki/front-proxy-ca.crt -noout -text |grep ' Not '

全局的更新失败可以一个个更新，如下
#续订kubeconfig文件中嵌入的证书，供管理员和kubeadm自身使用。
kubeadm alpha certs renew admin.conf
#续订apiserver用于连接kubelet的证书。
kubeadm alpha certs renew apiserver-kubelet-client --config /root/config.yaml
#续订用于提供Kubernetes API的证书。
kubeadm alpha certs renew apiserver --config /root/config.yaml
#续订kubeconfig文件中嵌入的证书，以供控制器管理器（controller manager）使用。
kubeadm alpha certs renew controller-manager.conf --config /root/config.yaml
#为前端代理客户端续订证书。
kubeadm alpha certs renew front-proxy-client --config /root/config.yaml
#续订kubeconfig文件中嵌入的证书，以供调度管理器使用。
kubeadm alpha certs renew scheduler.conf --config /root/config.yaml

注意，要像安装的时候做如下操作后生效

cp -i /etc/kubernetes/admin.conf $HOME/.kube/config   -- y
chown $(id -u):$(id -g) $HOME/.kube/config

就可以使用kubectl get pod

#然后复制到其他控制节点，注意是master节点，node节点只要把最新的$HOME/.kube/config弄过去就行了

scp /etc/kubernetes/pki/* k8s-master02:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/* k8s-master03:/etc/kubernetes/pki/

scp /etc/kubernetes/scheduler.conf k8s-master02:/etc/kubernetes/scheduler.conf
scp /etc/kubernetes/scheduler.conf k8s-master03:/etc/kubernetes/scheduler.conf

scp /etc/kubernetes/controller-manager.conf k8s-master02:/etc/kubernetes/controller-manager.conf
scp /etc/kubernetes/controller-manager.conf k8s-master03:/etc/kubernetes/controller-manager.conf

# 复制config至其它控制节点
scp $HOME/.kube/config k8s-master02:/root/.kube/config
scp $HOME/.kube/config k8s-master03:/root/.kube/config


正常更新证书以后，官方建议是重启管理节点，然后即可恢复正常。

另外因为kubelet客户端证书也已经更新，所以建议重启每台管理节点上的kubelet服务。

可以参考[文档](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-alpha/)


话说Kuernetes v1.18已经支持证书自动轮换更新了，详细参考：

[地址](https://kubernetes.io/zh/docs/tasks/tls/certificate-rotation/)

```

### kubernetes的运维

**1.三台机器，某一次卡机后，恢复完机器，etcd，docker，harbor后，开始恢复k8s，因为只恢复坏掉节点不起作用，重新初始化下**


```html
# cp /etc/kubernetes/pki/* /etc/kubernetes/pki-backup/
# kubeadm reset
```
进行pki文件中的备份，然后重置

```html
# cp /etc/kubernetes/pki-backup/* /etc/kubernetes/pki/
# kubeadm init --config config.yaml
```
另外两个节点依次执行，初始化完了也就好了
```html
# kubectl taint nodes --all node-role.kubernetes.io/master-
```

因为是三个master节点，这样就能调度了

**如果出现了第一个节点初始化成功，但是其他节点初始化不成功的情况（卡在拉取镜像那边）**，可能需要检查下etcd了，这里的检查可以说是重新启动下etcd，即

```html
# systemctl stop etcd
# rm -rf /var/lib/etcd
# mkdir -p /var/lib/etcd
# systemctl start etcd
```
注意一定要建下目录，否则etcd起不来，这样会导致之前跑的K8S中的Pod全部丢失，然后再reset后重新初始化，发现其他两个节点可以正常初始化了，别忘了flannel也要重新create一下

**如果出现flannel或者coredns有10.96.0.1 io/timeout的情况**，可能也需要检查下etcd了，因为flannel会写数据至etcd，那么用上面的方式处理下，reset后重新初始化就可以了

**2.kubernetes的镜像拉取策略**

`imagePullPolicy:` 的参数有`always`,`IfNotPresent`,`Never`几种，其中`IfNotPresent`是默认值，另外两个一个是一直会拉取远程镜像，一个是从不拉取，配置如下

```html
spec:
      containers:
      - image: 10.106.128.17/library/redis:latest
        imagePullPolicy: IfNotPresent
        name: qwer-redis
```

**3.kubernetes的网络插件使用**

像kubernetes1.9版本可以在集群中安装flannel网络插件，如果机器上有两张网卡，flannel会选个默认的网卡，可能不是你想要的，这时候就可以配置一下

```html
containers:
      - name: kube-flannel
        image: registry.cn-shanghai.aliyuncs.com/gcr-k8s/flannel:v0.10.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=ens32
```
让flannel使用`ens32`的流量

还有，在flannel中有个配置，`hairpinMode`，这边是有坑的

```html
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```

hairpinmode有什么用呢？先看看什么是发卡流量，bridge不允许包从收到包的端口发出，比如bridge从一个端口收到一个广播报文后，会将其广播到所有其他端口。bridge的某个端口打开hairpin mode后允许从这个端口收到的包仍然从这个端口发出。这个特性用于NAT场景下，比如docker的nat网络，一个容器访问其自身映射到主机的端口时，包到达bridge设备后走到ip协议栈，经过iptables规则的dnat转换后发现又需要从bridge的收包端口发出，需要开启端口的hairpin mode。

这可能就是引起Pod访问自己的Cluster-IP网络不通，访问其他Pod的Cluster-IP是通的原因

**4.kubernetes中controller-manager的配置使用**

测试K8s高可用的时候，我们希望集群节点NotReady后可以比较快的进行pod漂移，这时候可以在controller-manager中进行配置

```html
spec:
  containers:
  - command:
    - kube-controller-manager
    - --address=127.0.0.1
    - --use-service-account-credentials=true
    - --controllers=*,bootstrapsigner,tokencleaner
    - --root-ca-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --leader-elect=true
    - --kubeconfig=/etc/kubernetes/controller-manager.conf
    - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
    - --allocate-node-cidrs=true
    - --cluster-cidr=10.244.0.0/16
    - --node-cidr-mask-size=24
    - --pod-eviction-timeout=1m0s
```
`pod-eviction-timeout`参数当节点不为ready情况后一分钟进行漂移，这里是采用kubernetes1.9版本，controller-manager的配置会写在/etc/kubernetes/manifests/kube-controller-manager.yaml中，因为使用了kubeadm进行初始化，会自动检测配置的变化，一般配置一下Pod的name和增加配置后，再改回name就能再`kube-system`中生效

```html
apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  creationTimestamp: null
  labels:
    component: kube-controller-manager
    tier: control-plane
  name: kube-controller-manager
  namespace: kube-system
```
其中name换成`kube-controller-manager-cmp-1`过段时间再换回来就行

**5.kubeadm初始化的报错[ERROR CRI]: unable to check if the container runtime at “/var/run/dockershim.sock” is running: exit status 1**

网上查阅，解决如下

```html
rm -f /usr/bin/crictl
```
**6.kubernets容器连通性测试**

登录到容器内部后，应该是可以使用wget命令去测试clusterIp + port是否能访问通的，理论上这些是要都通的

```html
wget 10.102.203.169:8011
```
注意这个端口号是内部端口号，ping这个clusterIp是ping不通的，因为这是个虚拟地址

其实在宿主机上也是可以用curl clusterIp + port的形式访问到服务，同样curl podIp + port的形式就更可以了，对service地址的curl其实就会被分发到对Pod地址的curl，这里后面可能是有多个pod的，所以定义service的时候port写的端口号是service的，而targetPort写的端口号是pod的，两个port是可以不同的！！！默认情况下，采用的是RoundRobin的方式负载的

**7.kubernets启动6443后访问API**

之前如果是8080版本，访问很简单

```html
curl localhost:8080/api/v1
```
如上就可以访问了，但是有了认证之后就不一样了，要与Kubernetes API进行交互，您需要具有正确权限的ServiceAccount，通过（Cluster）Role和RoleBinding获得。使用ServiceAccount的token进行身份验证。由于所有通信都通过TLS进行，因此您还需要自签名证书（ca.crt）。或者，允许不安全的连接(--insecure)，但不建议这样做。

创建ServiceAccount

```html
kubectl create serviceaccount api-explorer
```
用yaml文件创建role
```html
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: log-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "watch", "list"]
```
参考
```html
kubectl get clusterrole admin -o yaml
```
将clusterRole绑定到当前命名空间的serviceAccount中(默认是'default')

```html
kubectl create rolebinding api-explorer:log-reader --clusterrole log-reader --serviceaccount default:api-explorer
```
获取token，API等基础信息，这边要安装下jq

```html

SERVICE_ACCOUNT=api-explorer

# Get the ServiceAccount's token Secret's name
SECRET=$(kubectl get serviceaccount ${SERVICE_ACCOUNT} -o json | jq -Mr '.secrets[].name | select(contains("token"))')

# Extract the Bearer token from the Secret and decode
TOKEN=$(kubectl get secret ${SECRET} -o json | jq -Mr '.data.token' | base64 -d)

# Extract, decode and write the ca.crt to a temporary location
kubectl get secret ${SECRET} -o json | jq -Mr '.data["ca.crt"]' | base64 -d > /tmp/ca.crt

# Get the API Server location
APISERVER=https://$(kubectl -n default get endpoints kubernetes --no-headers | awk '{ print $2 }')

```

访问API，通过/openapi/v2查看API说明

```html
curl -s $APISERVER/openapi/v2  --header "Authorization: Bearer $TOKEN" --cacert /tmp/ca.crt | less
```

获取所有的Pod

```html
url -s $APISERVER/api/v1/namespaces/default/pods/ --header "Authorization: Bearer $TOKEN" --cacert /tmp/ca.crt | jq -rM '.items[].metadata.name'
```

kubectl get secret出来的是base64加密后的，kubectl describe secret 获取的是不加密的。上面的基础信息里，是用base64解密过后的，测试了下defalut那个是不行的，还是会403，这样做

```html
kubectl create clusterrolebinding test:anonymous --clusterrole=cluster-admin --user=system:anonymous
```

**8.kubernets使用命令**

获取运行的pod的对应yaml文件

```html
kubectl get pod qwer-zebra-748df886b9-2k85b -o=yaml --export > zebra-deployment.yaml
```

```html
kubectl get svc qwer-zebra -o=yaml --export > zebra-svc.yaml
```

**9.记一次L3 miss but route for [ip] not found**

这个是flanneld日志中报的错，所以再一次补充下flannel

Flannel是CoreOS开发,专门用于docker多机互联的一个工具,让集群中的不同节点主机创建的容器都具有全集群唯一的虚拟ip地址，Flannel使用go语言编写.Flannel为每个host分配一个subnet，容器从这个subnet中分配IP，这些IP可以在host间路由，容器间无需使用nat和端口映射即可实现跨主机通信，每个subnet都是从一个更大的IP池中划分的，flannel会在每个主机上运行一个叫flanneld的agent，其职责就是从池子中分配subnet

Flannel使用etcd存放网络配置、已分配 的subnet、host的IP等信息，etcd的初始配合可以如下，这是Vxlan的模式

```html
etcdctl --endpoints http://127.0.0.1:2379 set /coreos.com/network/config '{"Network": "10.0.0.0/16", "SubnetLen": 24, "SubnetMin": "10.0.1.0","SubnetMax": "10.0.20.0", "Backend": {"Type": "vxlan"}}'
```
Network就是Ip地址池，SubnetLen就是分配给单个宿主机的docker0的IP段的子网掩码长度，SubnetMin就是最小能够分配的IP段，SubnetMax就是最大能够分配的IP段，之间的差值其实就是可以支持的宿主机了，backend是数据包以什么方式转发，默认为udp模式，host-gw模式性能最好，但不能跨宿主机网络

这边flannel还提供了一个运行后的环境变量文件，包含了当前主机要使用flannel通讯的相关参数

```html
cat /run/flannel/subnet.env
```

可以使用flannel提供的脚本将subnet.env转写成Docker启动参数，创建好的启动参数默认生成在/run/docker_opts.env文件中

```html
# /opt/flannel/mk-docker-opts.sh -c

# cat /run/docker_opts.env
DOCKER_OPTS=" --bip=10.0.18.1/24 --ip-masq=false --mtu=1450"
```
修改docker的服务启动文件如下：

```html
# vim /lib/systemd/system/docker.service

EnvironmentFile=/run/docker_opts.env
ExecStart=/usr/bin/dockerd $DOCKER_OPTS -H fd://
```
flanneld配置backend为host-gw，与vxlan不同，host-gw不会封装数据包，而是在主机的路由表中创建到其他主机的subnet的路由条目，从而实现容器网络跨主机通信。需要说明的是，host-gw不能跨宿主机网络通信，或者说跨宿主机网络通信需要物理路由支持。

```html
etcdctl --endpoints http://127.0.0.1:2379 set /coreos.com/network/config '{"Network": "10.0.0.0/16", "SubnetLen": 24, "SubnetMin": "10.0.1.0","SubnetMax": "10.0.20.0", "Backend": {"Type": "host-gw"}}'
```

需要重启flanneld和docker

```html
systemctl restart flanneld docker
```
可以在宿主机上查看路由条目

```html
ip route
```

Flannel的补充完成，下面看看ARP的补充。arp的一些基本操作

显示本机arp缓冲区内容
```html
arp -a
```

从缓冲区删除指定的地址类型
```html
arp -d 10.144.121.11
```
-v，显示执行过程，-n数字方式显示

问题发生时，从pod中执行pod命令，分别要在宿主机的网卡（eth0，bond1），flannel.1，docker0这三个网卡上抓包，定位arp表，手动刷新异常arp记录并查看

flannel对外通信就是到arp表中找对应的mac地址

```html
ip neig show dev flannel.1
```

知道了本侧网关地址，还需要知道目的IP地址。vxlan默认实现第一次确实是通过广播的方式，但flannel再次采用通过arp方式获取对应关系：

```html
bridge fdb show dev flannel.1
```

传统的arp获取邻居的方式是通过广播获取，如果收到对端的arp相应则会标记对端为reachable，在超过reachable设定时间后，如果发现对端失效会标记为stale，之后会转入的delay以及probe进入探测的状态，如果探测失败会标记为Failed状态。而在flannel实现中，它利用了内核会将发送arp征询发送到用户空间的这一特征，flanneld获取到内核发送到用户空间的L3MISS,并且配合k8s或etcd返回这个IP地址对应的mac地址，设置reachable。

从分析可以看出，如果flanneld程序如果退出后，容器之间的通信将会中断。而一旦没有flanneld来维护这个arp表，那么这个arp记录不会自动更新，也就造成了这次问题的发生。

**10.记一次换flannel网段导致服务不可用的问题**

接上一次的问题继续，上回说到，下面命令显示异常

```html
ip neig show dev flannel.1
```
![flannel_neig]({{ "/assets/img/flannel/flannel_neig.png" | relative_url}})

这种情况会导致L3 miss，然后环境发现了一个情况，就是clusterIp的range跟flannel的range竟然都是172.200，怪怪的，决定重新配置一波

首先docker和flannel得重新搞下，因为现在都是172.200的段，去etcd中删除配置的config

```html
etcdctl get /kube-centos/network/config
etcdctl rm /kube-centos/network/config
etcdctl set /kube-centos/network/config '{"Network": "172.100.0.0/16", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}'
```
然后重启docker，flanneld组件，查看网段是否变化，如果没变，先删除接口再进行重启

```html
ip link delete flannel.1
```
因为出现了很诡异的现象，明显是有问题的

![flannel_error]({{ "/assets/img/flannel/flannel_error.png" | relative_url}})

这导致了K8S的集群跨节点通信异常，也就是节点只能访问本机flannel出来的podIp地址，访问不了其他节点的podIp地址，所以再搞下flannel.1，docker0根flannel.1哪个有问题就删哪个。期间记得清理下iptables，检查下K8s的组件运行情况

**11.busybox的使用**

有时候需要测试网络，可以用busybox的工具，但是不要用最新版的，有bug，用下面的

```html
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - name: busybox
    image: busybox:1.28.4
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always


kubectl exec -it busybox -- nslookup details

```
