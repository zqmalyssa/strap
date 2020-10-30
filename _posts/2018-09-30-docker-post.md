---
layout: post
title: "docker使用及运维"
author-id: "zqmalyssa"
tags: [code, operations]
---

docker现在使用已经非常广泛了，我们主要做docker和K8S的结合，以及与harbor的配合使用，本篇介绍docker的基本知识及常见运维

### docker的基本介绍

**docker的存储类型有如下几种**

| Storage Driver | 后端文件系统 | 不支持的后端文件系统 |
| :----: | :----:  | :----: |
| overlay | ext4 xfs | btrfs aufs overlay zfs eCryptfs |
| overlay2 | ext4 xfs | btrfs aufs overlay zfs eCryptfs |
| aufs | ext4 xfs | btrfs aufs eCryptfs |
| btrfs | btrfs only | N/A |
| devicemapper | direct-lvm | N/A |
| vfs | debugging only | N/A |
| zfs | zfs only | N/A |

overlay2从linux内核3.18起并入内核

devicemapper驱动在第一次对文件读写时性能稍微优越于overlay2，但是对已有文件的重复读写来看，overlay2性能优越于devicemapper，对于随机读写来说，overlay2性能优越于devicemapper。

由于devicemapper是基于block复制，并且每次修改数据时，都会进行复制，这会造成磁盘空间和内存的浪非甚至会出现磁盘和内存溢出等问题，对于overlay2驱动来说，只在第一次修改的时候复制文件，后续更在都在该文件上进行。自从Docker1.12起，Docker也支持overlay2存储驱动，目前社区更推荐使用overlay2。

通过对overlay2和overlay存储结构的对比，可以看出overlay2在centos kernel3.10上是支持的，overlay2所支持的多层lowerdir特性也被支持。

**docker的日志类型有如下几种**

| Logging Driver | 描述 |
| :----: | :----: |
| none | 容器没有日志可用，docker logs 什么都不返回 |
| json-file | 日志格式化为 JSON。这是 Docker 默认的日志驱动程序 |
| syslog | 将日志消息写入 syslog 工具。syslog 守护程序必须在主机上运行 |
| journald | 将日志消息写入 journald。journald 守护程序必须在主机上运行 |
| gelf | 将日志消息写入 Graylog Extended Log Format (GELF) 终端，例如 Graylog 或 Logstash |
| fluentd | 将日志消息写入 fluentd（forward input）。fluentd 守护程序必须在主机上运行 |
| awslogs | 将日志消息写入 Amazon CloudWatch Logs |
| splunk | Writes log messages to splunk using the HTTP Event Collector |
| etwlogs | 将日志消息写为 Windows 的 Event Tracing 事件。仅在Windows平台上可用 |
| gcplogs | 将日志消息写入 Google Cloud Platform (GCP) Logging |
| logentries | 将日志消息写入 Rapid7 Logentries |

以上参数都可以进行配置`/etc/docker/daemon.json`，比如

```html
{
  "log-driver": "json-file",
  "log-opts": {
    "labels": "production_status",
    "env": "os,customer"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
  "overlay2.override_kernel_check=true"], #这个最好加上，否则启动报错
  "graph": "/root/test/docker1.13.1" #配置docker的默认根目录
}
```

**docker的整体架构**

![docker_framework]({{ "/assets/img/docker/docker_framework.png" | relative_url}})

1、distribution 负责与docker registry交互，上传镜像以及v2 registry 有关的源数据
2、registry负责docker registry有关的身份认证、镜像查找、镜像验证以及管理registry mirror等交互操作
3、image 负责与镜像源数据有关的存储、查找，镜像层的索引、查找以及镜像tar包有关的导入、导出操作
4、reference负责存储本地所有镜像的repository和tag名，并维护与镜像id之间的映射关系
5、layer模块负责与镜像层和容器层源数据有关的增删改查，并负责将镜像层的增删改查映射到实际存储镜像层文件的graphdriver模块
6、graphdriver是所有与容器镜像相关操作的执行者

还有一个架构图更清晰点，感谢作者

![docker_framework2]({{ "/assets/img/docker/docker_framework2.png" | relative_url}})

从上图看，用户是使用Docker Client与Docker Daemon建立通信，并发送请求给后者。而Docker Daemon作为Docker架构中的主体部分，首先提供Server的功能使其可以接受Docker Client的请求；而后Engine执行Docker内部的一系列工作，每一项工作都是以一个Job的形式的存在。Job的运行过程中，当需要容器镜像时，则从Docker Registry中下载镜像，并通过镜像管理驱动graphdriver将下载镜像以Graph的形式存储；当需要为Docker创建网络环境时，通过网络管理驱动networkdriver创建并配置Docker容器网络环境；当需要限制Docker容器运行资源或执行用户指令等操作时，则通过execdriver来完成。而libcontainer是一项独立的容器管理包，networkdriver以及execdriver都是通过libcontainer来实现具体对容器进行的操作。当执行完运行容器的命令后，一个实际的Docker容器就处于运行状态，该容器拥有独立的文件系统，独立并且安全的运行环境等。

1、docker client，docker client 是docker架构中用户用来和docker daemon建立通信的客户端
2、docker daemon：docker daemon 是docker架构中一个常驻在后台的系统进程，功能是：接收处理docker client发送的请求
3、docker server：docker server 在docker架构中时专门服务于docker client的server
4、docker engines：Docker运行的核心模块。它扮演Docker container存储仓库的角色，并且通过执行job的方式来操纵管理这些容器。
5、job：一个Job可以认为是Docker架构中Engine内部最基本的工作执行单元。Docker可以做的每一项工作，都可以抽象为一个job。
6、docker registry：Docker Registry是一个存储容器镜像的仓库。而容器镜像是在容器被创建时，被加载用来初始化容器的文件架构与目录。
7、Graph：Graph在Docker架构中扮演已下载容器镜像的保管者，以及已下载容器镜像之间关系的记录者。一方面，Graph存储着本地具有版本信息的文件系统镜像，另一方面也通过GraphDB记录着所有文件系统镜像彼此之间的关系。
8、driver：Driver是Docker架构中的驱动模块。通过Driver驱动，Docker可以实现对Docker容器执行环境的定制。由于Docker运行的生命周期中，并非用户所有的操作都是针对Docker容器的管理，另外还有关于Docker运行信息的获取，Graph的存储与记录等。因此，为了将Docker容器的管理从Docker Daemon内部业务逻辑中区分开来，设计了Driver层驱动来接管所有这部分请求。在Docker Driver的实现中，可以分为以下三类驱动：graphdriver、networkdriver和execdriver。
  - graphdriver主要用于完成容器镜像的管理，包括存储与获取。即当用户需要下载指定的容器镜像时，graphdriver将容器镜像存储在本地的指定目录；同时当用户需要使用指定的容器镜像来创建容器的rootfs时，graphdriver从本地镜像存储目录中获取指定的容器镜像。在graphdriver的初始化过程之前，有4种文件系统或类文件系统在其内部注册，它们分别是aufs、btrfs、vfs和devmapper。而Docker在初始化之时，通过获取系统环境变量”DOCKER_DRIVER”来提取所使用driver的指定类型。而之后所有的graph操作，都使用该driver来执行。
  - networkdriver的用途是完成Docker容器网络环境的配置，其中包括Docker启动时为Docker环境创建网桥；Docker容器创建时为其创建专属虚拟网卡设备；以及为Docker容器分配IP、端口并与宿主机做端口映射，设置容器防火墙策略等。networkdriver的架构如下：
  - execdriver作为Docker容器的执行驱动，负责创建容器运行命名空间，负责容器资源使用的统计与限制，负责容器内部进程的真正运行等。在execdriver的实现过程中，原先可以使用LXC驱动调用LXC的接口，来操纵容器的配置以及生命周期，而现在execdriver默认使用native驱动，不依赖于LXC。具体体现在Daemon启动过程中加载的ExecDriverflag参数，该参数在配置文件已经被设为”native”。这可以认为是Docker在1.2版本上一个很大的改变，或者说Docker实现跨平台的一个先兆。execdriver架构如下：
9、libcontainer：libcontainer是Docker架构中一个使用Go语言设计实现的库，设计初衷是希望该库可以不依靠任何依赖，直接访问内核中与容器相关的API
  - 正是由于libcontainer的存在，Docker可以直接调用libcontainer，而最终操纵容器的namespace、cgroups、apparmor、网络设备以及防火墙规则等。这一系列操作的完成都不需要依赖LXC或者其他包。libcontainer架构如下：
10、docker container：Docker container（Docker容器）是Docker架构中服务交付的最终体现形式。

#### docker的核心技术

核心技术主要包括linux操作系统的命名空间（Namespaces），控制组（Control Groups），联合文件系统（Union File System）和Linux虚拟网络支持

docker采用标准的C/S架构，docker daemon一般在宿主机后台运行，作为服务端接受来自客户端的请求。在设计上，docker daemon是一个非常松耦合的架构，通过专门的Engine模块来分发管理各个来自客户端的任务

**命名空间（Namespace）**
是linux内核针对实现容器虚拟化而引入的一个强大特性，每个容器都可以拥有自己单独的命名空间，运行在其中的应用都像是在独立的操作系统中运行一样，命名空间保证了容器之间彼此互不影响，
1、进程命名空间，各个命名空间有一套自己的进程管理方法，
2、网络命名空间，通过网络命名空间，又可以使网络隔离，一个网络命名空间为进程提供了一个完全独立的网络协议栈的视图，包括网络设备接口，IPv4，IPv6协议栈，IP路由表，防火墙规则，sockets等等，这样每个容器的网络就能隔离出来了，docker采用虚拟网络设备的方式，将不同命名空间的网络设备连接到一起，默认情况下，容器中的虚拟网卡将同本地主机上的docker0网桥连接在一起
```html
brctl show //可以查看到桥接到宿主机docker0网桥上的虚拟接口
```
3、IPC命名空间，包括信号量，消息队列和共享内存等，
4、挂载命名空间，类似chroot，将一个进程放到一个特定的目录执行
5、UTS命名空间，允许每个容器拥有独立的主机名和域名，从而可以虚拟出一个有独立主机名和网络空间的环境，就跟网络上一台独立的主机一样
6、用户命名空间，每个容器可以有不同的用户和组ID，也就是可以使用容器内特定的用户运行程序，而非本地系统存在用户，每个容器内都有root账号

**控制组**

控制组是（CGroups）是Linux内核的一个特性，主要用来对共享资源进行隔离，限制，审计等，只有能控制分配到容器的资源，docker才能避免多个容器同时运行时的系统资源竞争
它可以对容器的内存，CPU，磁盘IO等资源进行限制和计费管理，功能点：

1、资源限制
2、优先级
3、资源审计
4、隔离
5、控制，挂起、恢复、重启动等

/sys/fs/cgroup/memory/docker目录下看到对docker组应用的各种限制项

**联合文件系统**

UnionFS是一种轻量级的高性能分层文件系统，它支持将文件系统的修改信息作为一次提交，并层层叠加，同时可以将不同目录挂载到同一个虚拟文件系统下

联合文件系统是实现docker镜像的技术基础，镜像可以通过分层来进行继承，例如，用户基于基础镜像来制作各种不同的应用镜像，这些镜像共享同一个基础镜像层，提高了存储效率，此外，当用户改变了一个Docker镜像，则一个新的层(layer)会被创建，因此，用户不用替换整个原镜像或者重新建立，只需要添加新层就可以。用户分发镜像的时候，也只需要分发被改动的新层内容（增量部分），这让docker的镜像管理变得十分轻量级和快速

docker支持的联合文件系统种类包括AUFS，btrfs，vfs和DeviceMapper

**docker的网络实现**

docker的网络实现其实就是利用linux上的网络命名空间和虚拟网络设备（veth pair）实现的，熟悉这两部分，就有利于理解docker网络的实现过程，网路可以作为一个专门块说下

![docker_net]({{ "/assets/img/docker/docker_net.png" | relative_url}})

要实现网络通信，机器至少需要一个网络接口（物理接口和虚拟接口）与外界相通。并可以收发数据包，此外，如果不同子网间需要通信，需要额外的路由机制

docker中的网络接口默认的都是虚拟的接口，虚拟接口最大的优势就是转发效率高，这是因为Linux通过在内核中进行数据复制来实现虚拟接口之间的数据转发，即发送接口的发送缓存中的数据包将被直接复制到接收接口的接收缓存中，而无需通过外部物理网络设备进行交换，对于本地系统和容器系统来看，虚拟接口跟一个正常的以太网相比并无区别，只是它速度要快的多

docker容器网络就很好的利用了linux虚拟网络技术，它在本地主机和容器内分别创建一个虚拟接口，并让它们彼此连通（这样的一对接口叫作veth pair）

网络的创建过程：

docker创建一个容器的时候，会具体执行如下操作：

1、创建一对虚拟接口，分别放到本地主机和新容器的命名空间
2、本地主机一端的虚拟接口连接到默认的docker0网桥或者指定网桥上，并具有一个以veth开头的名字，如veth1234
3、容器一端的虚拟接口将放到新创建的容器中，并修改名字作为eth0，这个接口只在容器的命名空间可见
4、从网桥可用地址段中获取一个空闲地址分配给容器的eth0（172.17.0.2/16）。并配置默认路由网关为docker0网卡的内部接口docker0的IP地址（172.17.42.1/16）

完成这些之后，容器就可以使用它所能看到的eth0虚拟网卡来连接其他容器和访问外部网络，docker运行的时候可以通过--net参数来指定容器的网络配置，有4个可选值bridge，host，container和none

--none之后如何配置网络呢

```html

//首先启动一个容器
docker run -it --rm --net=none base /bin/bash

//在本地查找容器的进程Id，并为它创建网络命名空间

docker inspect -f '{{.State.Pid}}' XXXXX
pid = 2778
mkdir -p /var/run/netns
ln -s /proc/$pid/ns/net /var/run/netns/$pid

//检查桥接网卡的Ip和子网掩码信息

ip addr show docker0
21: docker0: ...
inet 172.17.42.1/16 scope global docker0


//创建一对"veth pair"接口A和B，绑定A接口到网桥docker0，并启用它
ip link add A type veth peer name B
brctl addif docker0 A
ip link set A up

//将B接口放到容器的网路命名空间，命名eth0，启动它并配置一个可用IP（桥接网段）和默认网关
ip link set B netns $pid
ip netns exec $pid ip link set dev B name eth0
ip netns exec $pid ip link set eth0 up
ip netns exec $pid ip addr add 172.17.42.99/16 dev eth0
ip netns exec $pid ip route add default via 172.17.42.1

```
上面就是手动配置docker网络的过程，容器终止后，docker会清空容器，容器内的网路接口会随网络命名空间一起被清除，A接口也自动从docker0卸载并清除

#### docker的网络

Docker的本身技术依赖于近年Linux内核虚拟化技术的发展，所以Docker对Linux内核的特性有很强的依赖，docker与linux网络有关的技术主要如下：

1. Network Namespace(网路命名空间)
	- Linux的网络栈中引入网络命名空间的概念，实现不同容器之间的网络隔离，每个命名空间有自己独立的路由表和独立的Iptables/Netfilter设置来
2. Veth设备对
	- 命名空间之间需要通信，用什么，就是Veth设备对，它就是用来打通不同命名空间的网络的，docker还把veth设备的名字改成了eth0，以假乱真
3. Iptables/Netfilter
4. 网桥
	- 网桥是一个二层的虚拟网络设备，把若干网络接口"连接"起来，以使得网口之间的报文能够互相转发，根据MAC地址学习到转发端口。Linux内核是通过一个虚拟的网桥设备(Net Device)来实现桥接的，这个虚拟设备可以绑定若干个以太网接口设备，网桥有自己的操作命令
5. 路由

docker支持以下四种网络模式：

1. host模式，指定--net=host，告诉docker不要将容器网络放到隔离的命名空间，即不要容器化容器内的网络。此时容器使用本地主机的网络，它拥有完全的本地主机接口访问权限，容器进程可以跟主机其他root进程一样打开低范围的端口，可以访问本地网络服务比如D-bus，还可以让容器做一些影响整个主机系统的事情，比如重启主机，因此这个选项用的时候要非常小心，如果进一步使用--privileged=true参数，容器甚至会被允许直接配置主机的网络堆栈
2. container模式，指定--net=container:NAME_or_ID，让docker将新建容器的进程放到一个已存在容器的网络栈中，新容器进程有自己的文件系统，进程列表，和资源限制，但会和已存在的容器共享IP地址和端口等网络资源，两者进程可以直接通过lo环回接口通信
3. none模式，指定--net=none，让docker将新容器放到隔离的网络栈中，但是不进行网络配置，之后，用户可以自己进行配置
4. bridge模式，指定--net=bridge，这个就是默认模式，daemon第一次启动时会创建一个虚拟网桥，docker0。在docker网桥上为容器创建新的网络栈

针对这个docker0，在私有网路空间给这个网桥分配一个子网，针对docker创建出来的每一个容器，都会创建一个虚拟的以太网设备(Veth设备对)，其中一端关联到网桥上，另一端使用linux的网络命名空间技术，映射到容器内的eth0设备，然后从网桥的地址段内给eth0接口分配一个IP地址，docker的网络模型在跨主机访问时面临着问题，docker把这块问题交给更专业的人去做，不像openstack的vswitch那样，因为太特么复杂了

之前也讲了docker的网络，网络这部分其实是可以单拎一块出来说的，还有些细节的东西

Docker启动时会在主机上自动创建一个docker0虚拟网桥，实际上是一个Linux网桥，可以理解为一个软件交换机，它会在挂载其上的接口之间进行转发。同时docker随机分配一个本地未占用的私有网段（RFC1918）中的一个地址给docker0接口，比如典型的172.17.42.1，掩码255.255.0.0。此后启动的容器内的网口也会自动分配一个同一网段(172.17.0.0/16)的地址

veth pair接口的描述看上面核心组件

有些参数看一下：

```html
--ip-forward=true | false 启动进制net.ipv4.ip.forward，即打开转发功能
--iptables=true | false 禁止docker添加iptables规则
```

配置容器DNS和主机名

1、相关配置文件

实际上，容器中主机名和DNS配置信息都是通过三个系统配置文件来维护的：/etc/resolv.conf、/etc/hostname和/etc/hosts。

启动容器后，在容器中使用mount命令后可以查看三个文件的挂载信息，其中/etc/resolv.conf文件在创建容器时候，默认会与宿主机/etc/resolv.conf文件内容保持一致，

2、容器内修改配置文件

容器里可以直接编辑三个文件，但是修改时临时的，只在运行的容器中保留，容器终止或者重启后并不会被保存下来，也不会被docker commit提交

3、通过参数指定

可以自定义容器的配置，--hostname参数，记录其他容器主机名--link=CONTAINER_NAME:ALIAS，选项会在创建容器的时候，添加一个所连接容器的主机名到容器内/etc/hosts文件

指定DNS服务器，--dns=IP_ADDRESS。添加DNS服务器到容器的/etc/resolv.conf中，容器会用指定的服务器来解析所有不在/etc/hosts中的主机名

指定DNS搜索域 --dns-search=DOMAIN，设定容器的搜索域，当设定搜索域为.example.com时，在搜索一个名为host的主机时，DNS不仅搜索host，还会搜索host.example.com

容器访问控制

容器的访问控制，主要通过linux上的iptables防火墙软件来进行管理和实现，

1、容器访问外部网络

docker0可以让容器访问到宿主机，更进一步，容器要想通过宿主机访问到外部网络，需要宿主机进行转发

在宿主机Linux系统中，检查转发是否打开

```html
sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1

//如果0，手动打开
sysctl -w net.ipv4.ip_forward=1
```
更简单的指定--ip-forward进行转发

2、容器之间的访问

容器之间的访问需要以下两个方面的支持
a.网络拓扑是否已经连通，默认情况下，所有容器都会连接到docker0网络，这意味着默认情况下拓扑是互通的
b.本地系统的防火墙iptables是否允许访问通过，这取决于防火墙的默认规则是允许（大部分情况）还是禁止

访问所有端口

当启动docker服务的时候，默认会添加一条"允许"转发策略到iptables的FORWARD链上，通过配置--icc=true | false（默认值true）参数可以控制默认的策略，同时，如果启动docker服务时手动指定--iptables=false参数则不会修改宿主机系统上的iptables规则

访问指定端口

在通过-icc=false禁止容器间相互访问后，扔可以通过--link=CONTAINER_NAME:ALIAS选项来允许访问指定容器的开发端口

```html
--icc=false --iptables=true
//forward chain会drop所有流量
```
之后启动容器时用--link=CONTAINER_NAME:ALIAS选项，docker会在iptables中为两个互联容器分别添加一条ACCEPT规则，允许相互访问开放的端口，取决于Dockerfile中的EXPOSE行

映射容器端口到宿主机的实现

默认情况下，容器可以主动访问到外部网络的连接，但是外部网络无法访问到容器

容器访问外部网络实现

假设容器内部的网络地址为172.17.0.2，本地网络地址为10.0.2.2，容器要能访问外部网络，源地址不能为172.17.0.2，需要进行源地址映射（SourceNAT，SNAT），修改为本地系统的IP，10.0.2.2。映射是通过iptables源地址伪装操作实现的，查看NAT表上的POSTROUTING链的规则，该链负责网包要离开主机前，改写其源地址

```html
iptables -t nat -nvL POSTROUTING

pkts  btyes  target  port  opt  in  out  source  destination
0     0     MASQUERADE all --   *   !docker0  172.17.0.0/16  0.0.0.0/0
```
上述规则将所有源地址在172.17.0.0/16网段，且不是从docker0接口发出的流量（即从容器中出来的流量），动态伪装成从系统网卡发出，MASQUERADE行动跟传统SNAT行动的好处是它能动态从网卡获取地址

外部访问容器实现

容器允许外部访问，可以在docker run时候通过 -P或-p参数来启用

不管用那种办法，其实就是在本地的iptables的nat表中添加相应的规则，将访问外部IP地址的网包进行目标地址DNAT，将目标地址修改成容器的IP地址，以一个开放80端口的web容器为例，使用-P时，会自动映射本地49000~49900范围内的端口随机端口到容器的80端口
```html
iptables -t nat -nvL

Chain PRERPUTING
pkts  btyes  target  port  opt  in  out  source  destination
567     3024   DOCKER all --   *   *  0.0.0.0/0  0.0.0.0/0
ADDRTYPE match dst-type LOCAL

Chain DOCKER
pkts  btyes  target  port  opt  in  out  source  destination
0     0     DNAT      tcp   --   !docker0  *  0.0.0.0/0  0.0.0.0/0
tcp dpt:49153 to:172.17.0.2:80
```

可以看到，nat表中涉及两条链，PRERPUTING链负责包到达网络接口时，改写其目的地址，其中规则将所有流量都扔到DOCKER链，而DOCKER链中将所有不是从docker0进来的网包（意味着不是本地主机产生的），将目标端口为49153的，修改其目标地址为172.17.0.2，目标端口修改为80

而使用-p 80:80的原理是一样的

注意这里的0.0.0.0/0意味着接受主机来自所有网络接口上的流量，用户可以通过-p IP:HOST_PORT:container_port或-p IP::port来指定绑定的外部网络接口，以指定更严格的访问规则
如果希望映射永久绑定到某个固定的IP地址，可以再docker的配置文件中指定DOCKER_OPTS="--ip=IP_ADDRESS"

**配置docker0网桥**

目前docker网桥是linux网桥，所以可以使用brctl show来查看网桥和端口连接信息

docker不支持在启动容器的时候指定IP地址，在容器中进行查看

```html
ip addr show eth0
ip route
```

除了默认的docker0网桥，用户可以指定网桥来连接各个容器

```html
//删掉老的
systemctl stop docker
ip link set dev docker0 down
brctl delbr docker0

//创建新的
brctl addbr bridge0
ip addr add 192.168.5.1/24 dev bridge0
ip link set dev bridge0 up

//查看
ip addr show bridge0

//配置docker
DOCKER_OPTS="-b=bridge0"

```
**创建一个点到点的连接**

默认情况下，docker会将所有容器连接到由docker0提供的虚拟子网中，用户有时候需要两个容器可以直连通信，而不用通过主机网桥进行桥接，解决办法就是创建一对peer接口，分别放到两个容器中，配置成点到点链路类型即可

```html
//首先启动两个容器
docker run -it --rm --net=none base /bin/bash
docker run -it --rm --net=none base /bin/bash

//找到进程号，然后创建网络命名空间的跟踪文件：
docker inspect -f '{{.State.Pid}}' XXXX1
docker inspect -f '{{.State.Pid}}' XXXX2

mkdir -p /var/run/netns
ln -s /proc/$pid/ns/net /var/run/netns/$pid
ln -s /proc/$pid/ns/net /var/run/netns/$pid

//创建出一对peer，然后配置路由
ip link add A type veth peer name B
ip link set A netns $pid
ip netns exec $pid ip addr add 10.1.1.1/32 dev A
ip netns exec $pid ip link set A up
ip netns exec $pid ip route add 10.1.1.2/32 dev A

ip link set B netns $pid
ip netns exec $pid ip addr add 10.1.1.2/32 dev B
ip netns exec $pid ip link set B up
ip netns exec $pid ip route add 10.1.1.1/32 dev B

```

两个容器就可以互ping了，并成功建立了连接，点到点的链路不需要子网和子网掩码。此外，也可以不指定--net=none，这样容器还可以通过原先的网络来通信

docker的网络有很多使用工具和项目

如pipework，简化操作，批量给不同容器添加网卡
playground，基于playground，用户可以提前配置好容器的拓
libswarm项目，包括很多子项目，如Docker Server，Docker Client，SSH隧道，ETCD配置，SkyDNS等


#### docker的版本问题

moby、docker-ce与docker-ee
最早的时候docker就是一个开源项目，主要由docker公司维护。

2017年年初，docker公司将原先的docker项目改名为moby，并创建了docker-ce和docker-ee。

这三者的关系是：

moby是继承了原先的docker的项目，是社区维护的的开源项目，谁都可以在moby的基础打造自己的容器产品
docker-ce是docker公司维护的开源项目，是一个基于moby项目的免费的容器产品
docker-ee是docker公司维护的闭源产品，是docker公司的商业产品。

v1.13.1之后，发布计划更改为:

Edge:   月版本，每月发布一次，命名格式为YY.MM，维护到下个月的版本发布
Stable: 季度版本，每季度发布一次，命名格式为YY.MM，维护4个月

如果是centos，下面安装命令会在系统上添加yum源:

```html
curl -fsSL https://get.docker.com/ | sh
cat /etc/yum.repos.d/docker-ce.repo
```
然后yum安装：

```html
yum install -y docker-ce
```
yum源文件和rpm包都在[网页](download.docker.com)中，可以自己下载安装:

#### docker的容器

docker容器是镜像的一个运行实例，所不同的是，它带有额外的可写文件层，如果认为虚拟机是模拟运行的一整套操作系统（提供了运行态环境和其他系统环境）和跑在上面的应用，那么docker容器就是独立运行的一个或一组应用，以及它们的必须运行环境

docker run做了什么

1. 检查本地是否存在指定的镜像，不存在就从公有仓库下载
2. 利用镜像创建并启动一个容器
3. 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
4. 从宿主主机配置的网桥接口中桥接一个虚拟接口道容器中去
5. 从地址池配置一个IP地址给容器
6. 执行用户指定的应用程序
7. 执行完毕后同期被终止

所以默认docker run完就退出了，如果要让docker容器在后台以守护态的形式运行，用户可以添加-d参数

`docker ps -a -q`会查看到所有处于终止状态的容器

`docker export`和`docker import`可以迁移容器，但这种容器快照将丢失所有的历史数据和元数据信息

`-P` 参数是允许访问容器需要暴露的端口

docker容器的生命周期介绍

![dockercontainer]({{ "/assets/img/docker/dockercontainer.png" | relative_url}})

彩色圆形 代表容器的五种状态：
created：初建状态
running：运行状态
stopped：停止状态
paused： 暂停状态
deleted：删除状态

长方形 代表容器在执行某种命令后进入的状态：
docker create ： 创建容器后，不立即启动运行，容器进入初建状态；
docker run ： 创建容器，并立即启动运行，进入运行状态；
docker start ： 容器转为运行状态；
docker stop ： 容器将转入停止状态；
docker kill ： 容器在故障（死机）时，执行kill（断电），容器转入停止状态，这种操作容易丢失数据，除非必要，否则不建议使用；
docker restart ： 重启容器，容器转入运行状态；
docker pause ： 容器进入暂停状态；
docker unpause ： 取消暂停状态，容器进入运行状态；
docker rm ： 删除容器，容器转入删除状态（如果没有保存相应的数据库，则状态不可见）

菱形 需要根据实际情况选择的操作
killed by out-of-memory（因内存不足被终止）
宿主机内存被耗尽，也被称为OOM：非计划终止
这时需要杀死最吃内存的容器
然后进行选择操作
container process exitde（异常终止）
出现容器被终止后，将进入Should restart?选择操作：
yes 需要重启，容器执行start命令，转为运行状态。
no 不需要重启，容器转为停止状态。

#### docker资源限制

主要就是CGroup

[ref](https://www.cnblogs.com/zhuochong/p/9728383.html)

```html
docker stats [containerId]
```

#### docker的安全

docker是在Linux操作系统层面上的虚拟化实现，运行在容器内的进程，跟运行在本地系统中的进程，本质上并无区别，不适合的安全策略将可能给本地系统带来风险，因此，docker的安全性在生产环境中是十分关键的衡量因素

docker容器的安全性，很大程度上其实依赖于linux系统自身，因此在评估docker的安全性时，主要考虑：

1、Linux内核的命名空间机制提供的容器隔离安全
  - 命名空间提供了最基础也是最直接的隔离，在容器中运行的进程不会被运行在本地主机上的进程和其他容器通过正常渠道发现和影响
  - 例如，通过命名空间机制，每个容器都有自己独有的网络栈，意味着它们不能访问其他容器的套接字（sockets）或接口。当然，容器默认可以与本地主机网络连接，如果主机系统上做了相应的交换设置，容器可以像跟主机交互一样的和其他容器交互。
2、Linux控制组机制对容器资源的控制能力安全
  - 控制组（CGroup）是Linux容器机制中的另外一个关键组件，它负责实现资源的审计和限制，当用docker run启动容器时，docker将在后台为容器创建一个独立的控制组策略集合。确保各个容器可以公平的分享主机的内存、CPU、磁盘、IO等资源，更重要的，当发生容器内的资源压力不会影响到本地主机系统或其他容器
3、Linux内核的能力机制所带来的操作权限安全
  - 默认情况下，docker启动的容器被严格限制只允许使用内核的一部分能力，包括chrown，dac_override，kill，setid等
4、Docker程序（特别是服务端）本身的抗攻击性
  - docker允许宿主机，容器的资源共享，将容器的root用户映射到本地主机上的非root用户，减轻容器和主机之间因权限提升而引起的安全问题，允许docker服务端在非root权限下运行，利用安全可靠的子进程来代理执行需要特权权限的操作，这些子进程将只允许在限定范围内进行操作，例如仅仅负责虚拟网络的设定或文件系统管理，配置操作等
5、其他安全增强机制（AppArmor、SeLinux）对容器安全性的影响
  - 还可以用一些安全软件如selinux等

### docker知识点补充

1、docker与虚拟机的区别

![dockervsvm]({{ "/assets/img/docker/dockervsvm.png" | relative_url}})

容器是在linux上本机运行，并与其他容器共享主机的内核，它运行的一个独立的进程，不占用其他任何可执行文件的内存，非常轻量，虚拟机运行的是一个完成的操作系统，通过虚拟机管理程序对主机资源进行虚拟访问，相比之下需要的资源更多。


### Win版的docker安装

Linux版本docker安装没话说，但是Win要说下，有点不同，首先如果是专业版或者企业版的Win10系统，docker官网有专门的应用[下载](https://hub.docker.com/editions/community/docker-ce-desktop-windows)

但是一般自己玩的都不符合，所以要跟Win7一样，借助ToolBox，这个两者注意是不兼容的，下载最新的[toolbox](http://mirrors.aliyun.com/docker-toolbox/windows/docker-toolbox/)

然后安装，下一步下一步就安装好了，然后可以点击`Docker Quickstart Termina`，但是会提示去下载最新的`boot2docker.iso`，然后一般就没有然后，国外网没办法，直接用迅雷链接会快很多，地址如下

https://github.com/boot2docker/boot2docker/releases/download/v18.09.9/boot2docker.iso

下载好后将` C:\Users\Administrator\.docker\machine\cache`目录清空，将下载的内容复制到此目录，然后必须先断网，启动`Docker Quickstart Termina`

等看见如下情况，再连上网

![waitIp]({{ "/assets/img/docker/waitIp.png" | relative_url}})

一般ok，如果有问题，卸载toolbox再来

此外，需要给本地docker配置镜像加速器，一般是阿里云的，https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors 要注册，地址里面获取自己的镜像加速地址

然后进行如下操作

```html
//看看docker-machine安装好没
docker-machine --version

//下面这边一般返回已经存在
docker-machine create --engine-registry-mirror=自己加速地址 -d virtualbox default

//返回已经存在，这样运行
docker-machine ssh default
sudo sed -i "s|EXTRA_ARGS='|EXTRA_ARGS='--registry-mirror=自己加速地址 |g" /var/lib/boot2docker/profile
exit

//然后重启一下，需要连网
docker-machine restart default
```

基本步骤就结束了`docker pull`去吧

### Mac版的docker安装

直接官网下载了一个dmg，安装成了一个docker desktop，然后kubernetes选项卡那边enable就可以了

配置镜像加速的话在docker-engine里面自己写点json就可以了，地址还是阿里云的地址

### docker的运维

#### 一、 docker环境修复

本地的三台机器，其中装harbor的那台机器卡机了，开机都不能，修复好后发现etcd崩了，K8S集群崩了，本机的docker也崩了

```html
# systemctl restart docker
```

尝试上面的启动无用，journalctl会出现很多错误，网上搜索了用`thin_check`去检查也没有用，反正是单节点，直接清掉重启吧


```html
# rm -rf /var/lib/docker
# systemctl restart docker
```
这样能启动了，但是原来的镜像是肯定没有了，需要重新load，包括harbor的，K8S需要的


```html
# ./install.sh
# docker-compose up -d
```

#### 二、 docker上的runc漏洞修复

docker打补丁修复runc漏洞，需要升级内核至4.X.X版本


#### 三、 docker的日志输出问题

K8S日志不输出了，怀疑是docker的问题，发现确实也不输出了

```html
# docker logs -f [container_id]
```
查看下目前docker的日志驱动
```
# docker info | grep 'Logging Driver'
```

发现是journald老版的驱动程序，查看docker启动时的配置文件
```
# cat /usr/lib/systemd/system/docker.service
# EnvironmentFile=/etc/sysconfig/docker
```
在配置文件中发现

```
# OPTIONS='--selinux-enabled --log-driver=journald --signature-verification=false'
```
将journald这段配置注释掉，就会变成`json-file`的日志驱动
```
# OPTIONS='--selinux-enabled --signature-verification=false'
```

重启docker后docker及K8S均有日志输出


#### 四、 docker的常见用法

1、可以使用`docker commit`去生产一个新的镜像，因为容器中修改了一些内容
```
# docker commit [容器ID]
```

#### 五、 docker构建centos虚拟机

```html
//拉镜像的方式
docker pull centos
docker pull centos:7
docker pull centos:7.2

docker run -it -d centos:7 /bin/bash

//需要安装一些基础的东西
yum install passwd openssl openssh-server openssh-clients lrzsz wget initscripts net-tools -y

/usr/sbin/sshd -D
这时报以下错误：
[root@ b3426410ff43 /]# /usr/sbin/sshd
Could not load host key: /etc/ssh/ssh_host_rsa_key
Could not load host key: /etc/ssh/ssh_host_ecdsa_key
Could not load host key: /etc/ssh/ssh_host_ed25519_key

ssh-keygen -q -t rsa -b 2048 -f /etc/ssh/ssh_host_rsa_key -N ''?
ssh-keygen -q -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key -N ''
ssh-keygen -t dsa -f /etc/ssh/ssh_host_ed25519_key -N ''

我这里并没有修改/etc/ssh/sshd_config
如果以下命令不能用，改动容器启动方式，正好把端口给暴露出来，做之前commit镜像，然后用新镜像启动
systemctl restart sshd

docker commit 07a4a6c467e2 sshd:centos

docker run -d -p 5001:22 --name centos76-javabase --privileged=true centos:latest /usr/sbin/init
加了权限后就能用systemd了，可以改下root的密码

passwd root

然后就可以用xshell去连了

```
