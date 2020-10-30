---
layout: post
title: Service Mesh
tags: [servicemesh, microservice]
author-id: zqmalyssa
---

Service Mesh，近两年来比较火的东西，研究一下

#### Service Mesh

Kubernetes 和 Istio

Kubernetes 提供了部署、升级和有限的运行流量管理能力，但并不具备熔断、限流降级、调用链追踪等能力。Kubernetes 的本质是通过声明式配置对应用进行生命周期管理，其强项在于容器的调度和编排，而 Istio 的本质是提供应用间的流量和安全以及可观察性等。Istio 很好地补齐了 Kubernetes 在微服务治理上的诸多能力，为 Kubernetes 提供更好的应用和服务管理。因此，Istio 与 Kubernetes 是极为互补的。

为什么需要 Istio
微服务体系带给我们很有好处，但也带来一些问题：

服务发现
负载均衡
故障恢复
指标
监控
A / B 测试
金丝雀发布
熔断
限速
调用链追踪
访问控制
端到端认证

大家可成想过，要实现这些，需要花多少精力，对业务的入侵性又有多少？

Istio 通过负载平衡，服务到服务的身份验证，监控等功能，可以轻松创建已部署服务的网络，而服务的代码只需很少更改甚至无需更改。 Istio 通过在整个环境中部署特殊的 Sidecar 代理来支持服务，以拦截微服务之间的所有网络通信，然后使用 “控制平面” 功能配置和管理 Istio，其中包括：

1、HTTP、gRPC、WebSocket 和 TCP 通信的自动负载均衡。
2、通过丰富的路由规则、重试、故障转移和故障注入对流量行为进行细粒度控制
3、可插拔的策略层和配置 API，支持访问控制、速率限制和配额
4、集群内（包括集群的入口和出口）所有流量的自动度量，日志和跟踪。
5、具备强大的基于身份的验证和授权，确保群集中服务之间的通信安全

借助 Istio 可以轻松地处理微服务体系带来的一系列问题，让我们更专注于业务本身。

Istio 的核心功能

连接（Connect）
智能控制服务之间的流量和 API 调用，进行一系列测试，并通过灰度发布逐步升级。

安全（Secure）
通过托管身份验证、授权和服务之间通信加密，自动为服务通信提供安全保障。

控制（Control）
应用并确保执行流量控制策略，使得资源在客户侧的公平分配。

遥测（Observe）
对服务进行多样化、自动化的追踪、监控以及记录日志，以便实时了解服务运行状态。

Istio 以统一的方式提供了许多跨服务网络的关键功能：

流量管理
Istio 简单的规则配置和流量路由允许您控制服务之间的流量和 API 调用过程。Istio 简化了服务级属性（如熔断器、超时和重试）的配置，并且让它轻而易举的执行重要的任务（如 A/B 测试、金丝雀发布和按流量百分比划分的分阶段发布）。

有了更好的对流量的可视性和开箱即用的故障恢复特性，您就可以在问题产生之前捕获它们，无论面对什么情况都可以使调用更可靠，网络更健壮。

安全
Istio 的安全特性解放了开发人员，使其只需要专注于应用程序级别的安全。Istio 提供了底层的安全通信通道，并为大规模的服务通信管理认证、授权和加密。有了 Istio，服务通信在默认情况下就是受保护的，可以让您在跨不同协议和运行时的情况下实施一致的策略——而所有这些都只需要很少甚至不需要修改应用程序。

Istio 是独立于平台的，可以与 Kubernetes（或基础设施）的网络策略一起使用。但它更强大，能够在网络和应用层面保护pod 到 pod 或者服务到服务之间的通信。

策略
Istio 允许您为应用程序配置自定义的策略并在运行时执行规则，例如：

速率限制能动态的限制访问服务的流量
Denials、白名单和黑名单用来限制对服务的访问
Header 的重写和重定向
Istio 还容许你创建自己的策略适配器来添加诸如自定义的授权行为。
可观察性
Istio 健壮的追踪、监控和日志特性让您能够深入的了解服务网格部署。通过 Istio 的监控能力，可以真正的了解到服务的性能是如何影响上游和下游的；而它的定制 Dashboard 提供了对所有服务性能的可视化能力，并让您看到它如何影响其他进程。

Istio 的 Mixer 组件负责策略控制和遥测数据收集。它提供了后端抽象和中介，将一部分 Istio 与后端的基础设施实现细节隔离开来，并为运维人员提供了对网格与后端基础实施之间交互的细粒度控制。

所有这些特性都使您能够更有效地设置、监控和加强服务的 SLO。当然，底线是您可以快速有效地检测到并修复出现的问题。

Istio 架构

Istio 为目前最新的 1.5 版本。该版本已经去掉了 Mixer 组件

Istio 服务网格从逻辑上分为数据平面和控制平面。

1、数据平面由一组智能代理（Envoy）组成，被部署为 sidecar。这些代理协调和控制所有的微服务之间的所有网络通信。同时，它们也收集和遥测所有网格的流量。
2、控制平面管理并配置代理来进行流量路由。

![istioframework]({{ "/assets/img/istio/istio_mesh.png" | relative_url}})

Istio 中的流量分为数据平面流量和控制平面流量。数据平面流量是指工作负载的业务逻辑发送和接收的消息。控制平面流量是指在 Istio 组件之间发送的配置和控制消息用来编排网格的行为。Istio 中的流量管理特指数据平面流量。

Envoy
Istio 使用 Envoy 代理的扩展版本。Envoy 是用 C++ 开发的高性能代理，用于协调服务网格中所有服务的入站和出站流量。Envoy 代理是唯一与数据平面流量交互的 Istio 组件。

Envoy 代理被部署为服务的 sidecar，在逻辑上为服务增加了 Envoy 的许多内置特性，例如:

动态服务发现
负载均衡
TLS 终端
HTTP/2 与 gRPC 代理
熔断器
健康检查
基于百分比流量分割的分阶段发布
故障注入
丰富的指标

这种 sidecar 部署允许 Istio 提取大量关于流量行为的信号作为属性。反之，Istio 可以使用这些属性来执行决策，并将它们发送到监控系统，以提供整个网格的行为信息。

sidecar 代理模型还允许您向现有的部署添加 Istio 功能，而不需要重新设计架构或重写代码。

由 Envoy 代理启用的一些 Istio 的功能和任务包括:

流量控制功能：通过丰富的 HTTP、gRPC、WebSocket 和 TCP 流量路由规则来执行细粒度的流量控制。
网络弹性特性：重试设置、故障转移、熔断器和故障注入。
安全性和身份验证特性：执行安全性策略以及通过配置 API 定义的访问控制和速率限制。
Pilot
Pilot 为 Envoy sidecar 提供服务发现、用于智能路由的流量管理功能（例如，A/B 测试、金丝雀发布等）以及弹性功能（超时、重试、熔断器等）。

Pilot 将控制流量行为的高级路由规则转换为特定于环境的配置，并在运行时将它们传播到 sidecar。Pilot 将特定于平台的服务发现机制抽象出来，并将它们合成为任何符合 Envoy API 的 sidecar 都可以使用的标准格式。

这种松耦合允许 Istio 在 Kubernetes、Consul 或 Nomad 等多种环境中运行，同时维护相同的 operator 接口来进行流量管理。

您可以使用 Istio 的流量管理 API 来指示 Pilot 优化 Envoy 配置，以便对服务网格中的流量进行更细粒度地控制。

Citadel
Citadel 通过内置的身份和证书管理，可以支持强大的服务到服务以及最终用户的身份验证。您可以使用 Citadel 来升级服务网格中的未加密流量。使用 Citadel，operator 可以执行基于服务身份的策略，而不是相对不稳定的 3 层或 4 层网络标识。从 0.5 版开始，您可以使用 Istio 的授权特性来控制谁可以访问您的服务。

Galley
Galley 是 Istio 的配置验证、提取、处理和分发组件。它负责将其余的 Istio 组件与从底层平台（例如 Kubernetes）获取用户配置的细节隔离开来。

Kubernetes 关注 Pod，Istio 关注 Service。
Istio 是个平台，而 Envoy 是非常重要的一个组件，因为许多事情其实是它在做，Istio 更多的是充当适配和规则转发的角色。这也正是 Service Mesh 是 sidecar 模式导致的。因此，Istio 选择了 Envoy 这种用 C++ 开发的高性能代理。在 Service Mesh 世界里，得 sidecar 者得天下。
如果部署服务时，遇到配置派发不成功，应该先检查 Galley 组件是否正常，因为 Galley 负责了配置验证。
通过架构的描述可知，Istio 在性能与架构的选择中，选择了优先架构。

在架构上，Istio 1.5 重建了控制平面，将原有的多个组件整合为一个单体结构 istiod，并废弃了 Mixer 组件。也就是将pilot，galley这些东西整合成了一个istiod

#### Istio开发环境搭建

Istio 依托于 Kubernetes，因此，首先我们先安装 Kubernetes。Kubernetes 有许多安装的方法，包括：Minikube、kubeadm、Docker Desktop。本文选用较为便捷的 Docker Desktop。

一堆界面操作后，包括配置镜像加速，然后进行配置

```html
kubectl config use docker-desktop

kubectl get ns

//部署dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml

//查看dashboard
kubectl proxy

//地址是这个
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

然后需要获取token

创建 ServiceAccount

```html
# sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard


kubectl apply -f sa.yaml
```
创建 ClusterRoleBinding 为 dashboard sa 授权集群权限 cluster-admin

```html
# clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard

kubectl apply -f clusterrolebinding.yaml
```

获取token

```html
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

然后token黏贴到dashboard就可以了

接下来就是搞istio了，mac和linux就比较方便了

```html
curl -L https://istio.io/downloadIstio | sh -

cd istio-1.5.0/

export PATH=$PWD/bin:$PATH

//验证安装
istioctl version --remote=false

//选择启动auto-completion option
cp tools/istioctl.bash ~
source ~/istioctl.bash

```

选用 demo 配置文件部署 Istio。也可以是其他的

```html
istioctl manifest apply --set profile=demo
```

通过以下命令，为 default 命名空间开启 sidecar 自动注入。

```html
kubectl label namespace default istio-injection=enabled
```

验证istio，就是部署bookinfo例子程序

```html
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```
验证service和pod

```html
kubectl get services

kubectl get pods -w

```

验证服务访问

```html
kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
```

部署 gateway

```html
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

kubectl get gateway
```

获取访问路径

```html
kubectl get services -n istio-system
```

通过查看 istio-ingressgateway 的 EXTERNAL-IP 为 localhost，可得知访问地址为 http://localhost/productpage


#### Istio在kubernetes内网集群环境搭建

```html

首先可参看kubernetes文章中k8 with cilium的搭建

参考[官网](https://istio.io/docs/setup/getting-started/)的部署方式，如果内网环境能暂时通外网，会比较方便

curl -L https://istio.io/downloadIstio | sh -

cd istio-1.6.1

export PATH=$PWD/bin:$PATH

以上三步如果有问题，不通网的话，那么我是本地通往的环境用docker起了个centos的虚机，然后将拉下来的文件压缩上传到内网环境的，docker这个怎么玩看docker那篇文章吧

tar -cvxf XX.tar istio-1.6.1/ #压缩
tar -zvxf XX.tar #解压缩

//下面就是安装命令了，装之前也可以看下demo这个profile的配置
istioctl install --set profile=demo

istioctl profile dump demo

//这种安装必然拉镜像，不通你懂得，需要下面的镜像

docker load -i grafana-6.5.2.tar.gz
docker load -i istio-1.6.1.tar.gz #这个应该是client，不需要
docker load -i jaegertracing-1.16.tar.gz
docker load -i kiali-v1.18.tar.gz
docker load -i pilot-1.6.1.tar.gz
docker load -i prometheus-v2.15.1.tar.gz
docker load -i proxyv2-1.6.1.tar.gz

有镜像了在prometheus这步会有问题，原因是它的pod里面会有多一个container，proxy，也就是已经load进去的镜像，但是为什么还是会去拉呢，因为拉取策略是Always，这就尴尬了，

其实分析下istio的文件包，里面是有charts的，有values.yaml和templates文件夹，说明可以helm的，应该能改，这边先将这个pod干掉

kubectl delete po <your-pod-name> -n <name-space> --force --grace-period=0

强制一般是无用的

用delete deployment和delete svc的方式去删

删前我们先导出deployment的配置

kubectl get deployment prometheus -n istio-system -o yaml > pro_deployment.yaml
对比了在charts的中配置，重新生成了一个yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: istio-system
  labels:
    app: prometheus
    release: istio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
        release: istio
      annotations:
        sidecar.istio.io/inject: "false"
    spec:
      serviceAccountName: prometheus
      containers:
        - name: prometheus
          image: docker.io/prom/prometheus:v2.15.1
          imagePullPolicy: IfNotPresent
          args:
            - '--storage.tsdb.retention=6h'
            - '--config.file=/etc/prometheus/prometheus.yml'
          ports:
            - containerPort: 9090
              name: http
          livenessProbe:
            httpGet:
              path: /-/healthy
              port: 9090
          readinessProbe:
            httpGet:
              path: /-/ready
              port: 9090
          resources:
            requests:
              cpu: 10m
          volumeMounts:
          - name: config-volume
            mountPath: /etc/prometheus
          - mountPath: /etc/istio-certs
            name: istio-certs
        - name: istio-proxy
          image: docker.io/istio/proxyv2:1.6.1
          ports:
            - containerPort: 15090
              protocol: TCP
              name: http-envoy-prom
          args:
            - proxy
            - sidecar
            - --domain
            - $(POD_NAMESPACE).svc.cluster.local
            - "istio-proxy-prometheus"
            - --proxyLogLevel=warning
            - --proxyComponentLogLevel=misc:error
            - --controlPlaneAuthPolicy
            - NONE
            - --trust-domain=cluster.local
          env:
            - name: OUTPUT_CERTS
              value: "/etc/istio-certs"
            - name: JWT_POLICY
              value: first-party-jwt
            - name: PILOT_CERT_PROVIDER
              value: istiod
            # Temp, pending PR to make it default or based on the istiodAddr env
            - name: CA_ADDR
              value: istiod.istio-system.svc:15012
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: INSTANCE_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: SERVICE_ACCOUNT
              valueFrom:
                fieldRef:
                  fieldPath: spec.serviceAccountName
            - name: HOST_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.hostIP
            - name: ISTIO_META_MESH_ID
              value: cluster.local
            - name: ISTIO_META_CLUSTER_ID
              value: Kubernetes
          imagePullPolicy: IfNotPresent
          readinessProbe:
            failureThreshold: 30
            httpGet:
              path: /healthz/ready
              port: 15020
              scheme: HTTP
            initialDelaySeconds: 1
            periodSeconds: 2
            successThreshold: 1
            timeoutSeconds: 1
          volumeMounts:
            - mountPath: /var/run/secrets/istio
              name: istiod-ca-cert
            - mountPath: /etc/istio/proxy
              name: istio-envoy
            - mountPath: /etc/istio-certs/
              name: istio-certs
            - mountPath: /etc/istio/config
              name: istio-config-volume
      volumes:
      - name: istio-config-volume
        configMap:
          name: istio
          optional: true
      - name: config-volume
        configMap:
          name: prometheus
      - name: istio-certs
        emptyDir:
          medium: Memory
      - name: istio-envoy
        emptyDir:
          medium: Memory
      - name: istiod-ca-cert
        configMap:
          defaultMode: 420
          name: istio-ca-root-cert

注意拉取策略改成了IfNotPresent，然后再create，就跑起来了普罗米修斯的deployment

然后再运行下istioctl install --set profile=demo 就过了，虽然还是会有一个多余的prometheus出来，但是应该可以在etcd中删除

```html
这边做了etcd删除的demo，首先1.17的etcd是安在k8s里的，可以进容器里玩
etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key get / --prefix --keys-only
etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key get /registry/pods/istio-system --prefix --keys-only
etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key del /registry/pods/istio-system/prometheus-6dd77d88cf-89j9j

etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key get /registry/replicasets/istio-system --prefix --keys-only
etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key del /registry/replicasets/istio-system/prometheus-6dd77d88cf

删了pod，删了控制pod的rs都还是会自动弹出，为什么，因为还是有deployment，它又比rs高级，所以暂无办法
```

这个时候istio算是装完了，官网会默认开启某个namespace的自动注入，如下。这边就不建议了，因为后面部署bookinfo的sample的时候又会变成要Always去拉镜像，先取消，看下面的命令

kubectl label namespace default istio-injection=enabled
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml //这边就会有问题了

kubectl label namespace default istio-injection=disabled --overwrite=true //先取消
kubectl get namespace -L istio-injection //查看

然后运行

istioctl kube-inject -f istio-1.6.1/samples/bookinfo/kube/bookinfo.yaml >> bookinfo_with_sidecar.yaml

就是将之前的bookinfo的yaml变成有sidecar模式的yaml，这里面其实就是在每个应用的container后面又加了个proxy镜像的container，这时候，我们在这个生成的yaml改变镜像拉取策略，然后

kubectl create -f bookinfo_with_sidecar.yaml

这样就全部running起来了，当然如果你拉不到镜像，你又懂得，如下

docker load -i bookinfo-details-v1-1.15.1.tar.gz
docker load -i bookinfo-ratings-v1-1.15.1.tar.gz
docker load -i bookinfo-reviews-v2-1.15.1.tar.gz
docker load -i bookinfo-reviews-v1-1.15.1.tar.gz
docker load -i bookinfo-reviews-v3-1.15.1.tar.gz
docker load -i bookinfo-productpage-v1-1.15.1.tar.gz

kubectl get services
kubectl get pods
检查一下
kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
出现
<title>Simple Bookstore App</title>
就是bookinfo的sample部署成功了

这个时候集群内部是ok了，但是怎么给集群外部的浏览器访问界面呢，这边按照官网的方式来

kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

这就创建了一个istio ingress gateway。里面是包含一个gateway和一个virtualservice的

然后进行一些环境变量的配置，这边是否可选呢

kubectl get svc istio-ingressgateway -n istio-system

如果上面结果的EXTERNAL-IP有值，说明你有一个可用的外部负载均衡给ingress gateway使用，这是一种配置，如果没有，就是

export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
echo $GATEWAY_URL
echo http://$GATEWAY_URL/productpage

上面的输出就是集群外部浏览器的访问地址了，其实上面这些也不用强制做，就是给你看看

还有比如istio的遥测组件如kiali怎么访问呢，这边1.6跟之前的方式不太一样了，之前有人改类型，为nodeport

curl 10.96.91.88:20001/kiali/console //内部ok了

kubectl patch service istio-ingressgateway -n istio-system -p '{"spec":{"type":"NodePort"}}'
kubectl describe svc istio-ingressgateway -n istio-system
然后用gateway和vs的方式暴露

apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: grafana-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 15031
      name: http
      protocol: HTTP
    hosts:
    - "*"


apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: grafana-vs
spec:
  hosts:
  - "*"
  gateways:
  - grafana-gateway
  http:
  - route:
    - destination:
        host: grafana
        port:
          number: 3000

curl -I http://10.0.135.30:30828

测试连通性

而1.6中发现竟然没有这些服务的映射端口，那就用edit pod的方式将kiali的servic中type从clusterIP换成NodePort

然后直接正常Ip加上大端口的方式就能访问了

```
