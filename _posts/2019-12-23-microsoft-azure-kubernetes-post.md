---
layout: post
title: 微软的Azure使用
author-id: "zqmalyssa"
tags: [kubernetes, cloud computing, azure]
---

这边是microsoft的azure的使用体检，主要面向的是kubernetes

#### 1.kubernetes集群升级的注意事项

为尽量减少对正在运行的应用程序造成中断，AKS 节点已进行仔细隔离和排空。 在此过程中，执行以下步骤： 
1. Kubernetes 计划程序阻止在要升级的节点上计划其他 Pod。 
2. 计划在群集中的其他节点上运行该节点上运行的 Pod。 
3. 创建运行最新 Kubernetes 组件的节点。 
4. 准备好新节点并将其加入群集后，Kubernetes 计划程序开始在该群集上运行 Pod。 
5. 删除旧节点，群集的下一个节点随即开始隔离和排空进程。

#### 2.Azure的常见用法

创建资源组

```html
az group create --name myResourceGroup --location eastus
```

创建集群，直接建建不了，创了一台虚机后就能建了
```html
az aks create --resource-group myResourceGroup --name myAKSCluster --node-count 1 --enable-addons monitoring --generate-ssh-keys
```
删除集群
```html
az aks delete --resource-group myResourceGroup --name myAKSCluster --no-wait
```

删除整个资源组
```html
az group delete --name myResourceGroup --yes --no-wait
```

获取认证，可以执行`kubelet`

```html
az aks get-credentials -g myResourceGroup --name myAKSCluster
```

查看集群信息

```html
az aks show --resource-group myResourceGroup --name myAKSCluster --output table
```

查看可用的集群版本
```html
az aks get-upgrades --resource-group myResourceGroup --name myAKSCluster
```

升级版本
```html
az aks upgrade --resource-group myResourceGroup --name myAKSCluster --kubernetes-version KUBERNETES_VERSION
```
#### 3.Azure Active Directory

Azure Active Directory 用于azure kubernetes的外部标识解决方案，Azure AD 是基于数十年的企业标识管理经验 推出的基于云的多租户目录，也是一种将核心目录服务、应用程序访问管理和标识保护相结合的标识管理服务，借助 Azure AD，可以将本地标识集成到 AKS 群集中，提供帐户管理和安全性的单一源

获取kubelet配置上下文，运行

```html
az aks get-credentials
```
随后在用户使用 kubectl 与 AKS 群集进行交互时，系统会提示他们使用自己的 Azure AD凭据登录，AKS 群集中的 Azure AD 身份验证使用 OpenID Connect，后者是构建在 OAuth 2.0 协议顶层的标识层。 OAuth 2.0 定义获取和使用访问令牌以访问受保护资源的机制，而 OpenID Connect 实现身份验证，作为对 OAuth 2.0授权过程的扩展


#### 4.网络

可以有两种网络模型

1. kubenet，AKS集群创建时就配置的网络资源
	- 节点获取虚拟网络子网的IP地址，然后使用NAT
	- 节省IP地址空间，使用kubernetes内部或外部负载均衡器从集群外部到达Pod，必须手动管理和维护用户定义的路由，每个集群最多400个结点
2. Azure容器网络接口(cni)，AKS集群连接到现有的虚拟网络资源和配置
	- 每个Pod都可以从子网获取IP地址，并且可以直接访问，IP地址在网络空间中必须事先规划好，且唯一
	- Pod可获取完全虚拟网络连接，并可通过连接的网络通过其专用IP地址直接访问，需要更多的IP地址空间

使用入口控制器，单个IP地址可以将流量分配给多个应用程序，入口有两个组件：

1. 入口资源
2. 入口控制器

#### 5.存储

卷的类型

1. Azure磁盘，创建kubernetes DataDisk资源，是readwriteonce迷失，仅适合单个pod
2. Azure文件，可跨多个结点和Pod共享数据，主要是以上两个
3. emptyDir，Pod的临时空间，pod删除就消失，可以存放在本地结点也可以存放在内存中
4. secret，用于将敏感数据注入Pod，例如密码，只有同一命名空间中的Pod能访问该机密
5. configMap，将键值对属性注入Pod，例如应用程序的配置信息，是一种资源，只有同一命名空间中的Pod能访问该资源

卷作为 Pod 生命周期的一部分定义和创建，且仅在删除 Pod 之前存在。 如果在维护事件期间（尤其是在 StatefulSets 中）于另一台主机上重新计划 Pod，则 Pod 通常会预期其存储会被保留。 永久性卷 (PV) 是由 Kubernetes API 创建和管理的存储资源，可以在单个 Pod 的生命周期之外存在。

PersistentVolume 可以由群集管理员静态创建，或者由 Kubernetes API 服务器动态创建。动态预配使用 StorageClass 来标识需要创建的 Azure存储类型。若要定义不同的存储层（例如高级和标准），可创建 StorageClass。 StorageClass 还定义 reclaimPolicy。 删除 Pod 后且可能不再需要永久性卷时，此 reclaimPolicy 可控制基础 Azure 存储资源在此情况下的行为。 可删除基 础存储资源，也可保留基础存储资源以便与未来的 Pod 配合使用。

在 AKS 中会创建两个初始 StorageClass：

1. default：使用 Azure 标准存储创建托管磁盘。 回收策略指示在删除使用它的持久卷时，将删除基础 Azure 磁盘
2. managed-premium：使用 Azure 高级存储创建托管磁盘。 回收策略再次指示在删除使用它的持久卷时，将删除基础 Azure 磁盘

PersistentVolumeClaim 会请求特定StorageClass、访问模式和大小的磁盘或文件存储。可用存储资源分配给请求它的 Pod 后，PersistentVolume 就会绑定到 PersistentVolumeClaim。 永久性卷与声明 之间存在 1:1 的映射

#### 6.缩放

1. Pod
	- 手动缩放
	- 水平Pod自动缩放（HPA）

2. 集群
	- 自动缩放，缩减时避免使用单副本Pod的应用，否则可能会导致一些中断
	- 使用ACI可以解决突发的请求量



#### 7.集群管理人员

1. 多租户和隔离功能
	- 计划，包括排斥(taint)和容许(toleration)
	- 网络包括用于控制传入和传出Pod的流量流的网络策略
	- 身份验证和授权，RBAC和AD集成
	- 容器包括Pod的安全策略，Pod上下文

2. 高级计划
	- 使用排斥和容许提供专用结点，想要部署GPU，要阻止其他负荷，将排斥应用到指明了只能计划特定Pod的结点，然后将容许应用到可以容许结点排斥的Pod，kubernetes只会在容许和排斥相符合的结点上执行Pod，Pod未定义相应的容许时，无法在此node上调度Pod
	```html
	kubectl taint node aks-nodepool1 sku=gpu:NoSchedule
	```

	- 使用结点选择器和关联控制Pod，逻辑方式隔离工作负荷，排斥和容许时以硬分割的形式，nodeSelector的会优先执行，而其他Pod也可以跑在这个有label的node上，不像排斥容许


#### 8.安全

保护Pod对资源的访问权限，可以使用securityContext

```html
apiVersion: v1 
kind: Pod 
metadata: 
	name: security-context-demo 
spec: 
	containers: 
		- name: security-context-demo 
		  image: nginx:1.15.5 
        securityContext: 
          runAsUser: 1000 
          fsGroup: 2000 
          allowPrivilegeEscalation: false 
          capabilities: 
            add: ["NET_ADMIN", "SYS_TIME"]
```

Pod以ID为1000的用户身份和ID为2000的组运行，无法提升特权，无法使用root，允许Linux功能访问网络接口和主机的实时（硬件）时钟


#### 9.结点池

结点池在集群下，一个集群下有多个结点池，所以getNode的时候可以出现1结点池的结点，也可以出现2结点池的结点

使用开发空间
```html
az aks use-dev-spaces --resource-group myResourceGroup --name myAKSCluster
```

#### 10.ACR

创建容器注册表，这就是个harbor
```html
az acr create --resource-group MyResourceGroup --name MyDraftACR --sku Basic
```

#### 11.应用部署

```html
helm init --service-account tiller --node-selectors "beta.kubernetes.io/os"="linux"
```

#### 12.手动伸缩和自动伸缩

手动pod
```html
kubectl scale --replicas=5 deployment/azure-vote-front
```
自动pod
```html
kubectl autoscale deployment azure-vote-front --cpu-percent=50 --min=3 --max=10
```

手动结点
```html
az aks scale --resource-group myResourceGroup --name myAKSCluster --node-count 3
```

更新应用程序
```html
kubectl set image deployment azure-vote-front azure-vote-front=<acrLoginServer>/azure-vote-front:v2
```

缩放或升级 AKS 群集时，将对默认节点池执行操作。 你还可以选择缩放或升级特定节点池。 对于升级操作，将在节点池中的其他节点上计划正在运行的容器，直到成功升级所有节点。

#### 13.Ingress

loadbalance是4层网络，而Ingress controller是7层网络。

若要快速缩放 AKS 群集，可以与 Azure 容器实例 (ACI) 集成。 虚拟节点组件（基于virtual Kubelet）安装在 AKS 群集中，该群集将 ACI 显示为虚拟 Kubernetes 节 点。 然后，Kubernetes 可以计划通过虚拟节点作为 ACI 实例运行的 Pod，而不是直接在 AKS 群集中的 VM 节点 上运行的 Pod。 虚拟节点目前在 AKS 中预览。

#### 14.Service Mesh

service mesh服务网格

1. Istio
2. Linked
3. Consul