---
layout: post
title: chaosblade
tags: [chaosblade]
author-id: zqmalyssa
---

该说下chaosblade了

#### chaosblade

ChaosBlade 中文名混沌之刃，是阿里巴巴开源的一款遵循混沌工程原理和混沌实验模型的实验注入工具，是内部项目 MonkeyKing 对外开源的项目。目前支持的场景有：基础资源、Java 应用、C++ 应用、Docker 容器以及 Kubernetes 平台。该项目将场景按领域实现封装成单独的项目，不仅可以使领域内场景标准化实现，而且非常方便场景水平和垂直扩展，通过遵循混沌实验模型，实现 chaosblade cli 统一调用。

该项目于 2020 年 5 月 27 日发布了最新了 v0.6.0 版本，本系列文章的全部实践也将基于这个版本以及该版本的修复版本 v0.6.x 进行。

#### chaosblade-operator

ChaosBlade-Operator 是 ChaosBlade 的 Kubernetes 平台实验场景实现。其将混沌实验通过 Kubernetes 标准的 CRD 方式定义，很方便的使用 Kubernetes 资源操作的方式来创建、更新、删除实验场景，包括使用 kubectl、client-go 等方式执行，而且还可以使用上述的 chaosblade cli 工具执行。


ChaosBlade-Operator 启动后将会在每个节点部署一个 chaosblade-tool Pod，再部署一个 chaosblade-operator Pod，如果都运行正常，则安装成功。上面设置 --set webhook.enable=true 是为了 Pod 文件系统 I/O 故障实验，如果不需要进行该实验，则无需添加该设置。


helm install chaosblade-operator chaosblade-operator-1.4.0-v3.tgz --namespace kube-system

helm uninstall chaosblade-operator -n kube-system

快速开始一个场景，一个网络延迟场景

```html
apiVersion: chaosblade.io/v1alpha1
kind: ChaosBlade
metadata:
  name: delay-pod-network-by-names
spec:
  experiments:
  - scope: pod
    target: network
    action: delay
    desc: "delay pod network by names"
    matchers:
    - name: names
      value:
      - "java-springboottest-65c7798879-q5gvg"
    - name: namespace
      value:
      - "default"
    - name: interface
      value: ["eth0"]
    - name: time
      value: ["200"]
    - name: offset
      value: ["10"]

kubectl apply -f delay_pod_network_by_names.yaml // 执行

kubectl get blade delay-pod-network-by-names -o json // 查看

查看的结果如下：

{
    "apiVersion": "chaosblade.io/v1alpha1",
    "kind": "ChaosBlade",
    "metadata": {
        "annotations": {
            "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"chaosblade.io/v1alpha1\",\"kind\":\"ChaosBlade\",\"metadata\":{\"annotations\":{},\"name\":\"delay-pod-network-by-names\"},\"spec\":{\"experiments\":[{\"action\":\"delay\",\"desc\":\"delay pod network by names\",\"matchers\":[{\"name\":\"names\",\"value\":[\"java-springboottest-65c7798879-q5gvg\"]},{\"name\":\"namespace\",\"value\":[\"default\"]},{\"name\":\"interface\",\"value\":[\"eth0\"]},{\"name\":\"time\",\"value\":[\"200\"]},{\"name\":\"offset\",\"value\":[\"10\"]}],\"scope\":\"pod\",\"target\":\"network\"}]}}\n"
        },
        "creationTimestamp": "2021-12-19T09:13:08Z",
        "finalizers": [
            "finalizer.chaosblade.io"
        ],
        "generation": 1,
        "name": "delay-pod-network-by-names",
        "resourceVersion": "165274730",
        "selfLink": "/apis/chaosblade.io/v1alpha1/chaosblades/delay-pod-network-by-names",
        "uid": "e72dc477-d795-4be8-953f-3bb2bbce4002"
    },
    "spec": {
        "experiments": [
            {
                "action": "delay",
                "desc": "delay pod network by names",
                "matchers": [
                    {
                        "name": "names",
                        "value": [
                            "java-springboottest-65c7798879-q5gvg"
                        ]
                    },
                    {
                        "name": "namespace",
                        "value": [
                            "default"
                        ]
                    },
                    {
                        "name": "interface",
                        "value": [
                            "eth0"
                        ]
                    },
                    {
                        "name": "time",
                        "value": [
                            "200"
                        ]
                    },
                    {
                        "name": "offset",
                        "value": [
                            "10"
                        ]
                    }
                ],
                "scope": "pod",
                "target": "network"
            }
        ]
    },
    "status": {
        "expStatuses": [
            {
                "action": "delay",  // 必须
                "resStatuses": [
                    {
                        "id": "3cbdd557b782be5b",
                        "identifier": "default/svr7871hp360/java-springboottest-65c7798879-q5gvg/istio-proxy/e370c1fc19d1",
                        "kind": "pod", // 如果有，必须
                        "state": "Success",  // 如果有，必须
                        "success": true  // 如果有，必须
                    }
                ],
                "scope": "pod", // 必须
                "state": "Success",  // 必须
                "success": true,  // 必须
                "target": "network"  // 必须
            }
        ],
        "phase": "Running"  // Initial ->Running -> Updating -> Destroying -> Destroyed
    }
}



停止实验的话可以：


kubectl delete -f delay_pod_network_by_names.yaml 或者 kubectl delete blade delay-pod-network-by-names

```

这里的话可以看下operator的源码，对应的crd定义如下：

```html

apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: chaosblades.chaosblade.io
spec:
  group: chaosblade.io
  names:
    kind: ChaosBlade
    listKind: ChaosBladeList
    plural: chaosblades
    singular: chaosblade
    shortNames: [blade]
  scope: Cluster
  subresources:
    status: {}
  validation:
    openAPIV3Schema:
      description: ChaosBlade is the Schema for the chaosblades API
      properties:
        apiVersion:
          description: 'APIVersion defines the versioned schema of this representation
            of an object. Servers should convert recognized schemas to the latest
            internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
          type: string
        kind:
          description: 'Kind is a string value representing the REST resource this
            object represents. Servers may infer this from the endpoint the client
            submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
          type: string
        metadata:
          type: object
        spec:
          description: ChaosBladeSpec defines the desired state of ChaosBlade
          properties:
            experiments:
              description: 'INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
                Important: Run "operator-sdk generate k8s" to regenerate code after
                modifying this file Add custom validation using kubebuilder tags:
                https://book-v1.book.kubebuilder.io/beyond_basics/generating_crd.html'
              items:
                properties:
                  action:
                    description: Action is the experiment scenario of the target,
                      such as delay, load
                    type: string
                  desc:
                    description: Desc is the experiment description
                    type: string
                  matchers:
                    description: Matchers is the experiment rules
                    items:
                      properties:
                        name:
                          description: Name is the name of flag
                          type: string
                        value:
                          description: 'TODO: Temporarily defined as an array for
                            all flags Value is the value of flag'
                          items:
                            type: string
                          type: array
                      required:
                      - name
                      - value
                      type: object
                    type: array
                  scope:
                    description: Scope is the area of the experiments, currently support
                      node, pod and container
                    type: string
                  target:
                    description: Target is the experiment target, such as cpu, network
                    type: string
                required:
                - action
                - scope
                - target
                type: object
              type: array
          required:
          - experiments
          type: object
        status:
          description: ChaosBladeStatus defines the observed state of ChaosBlade
          properties:
            expStatuses:
              description: 'Important: Run "operator-sdk generate k8s" to regenerate
                code after modifying this file Add custom validation using kubebuilder
                tags: https://book-v1.book.kubebuilder.io/beyond_basics/generating_crd.html'
              items:
                properties:
                  action:
                    type: string
                  error:
                    type: string
                  resStatuses:
                    description: ResStatuses is the details of the experiment
                    items:
                      properties:
                        error:
                          description: experiment error
                          type: string
                        id:
                          description: experiment uid in chaosblade
                          type: string
                        identifier:
                          description: 'Resource identifier, rules as following: container:
                            Namespace/NodeName/PodName/ContainerName pod： Namespace/NodeName/PodName'
                          type: string
                        kind:
                          description: Kind
                          type: string
                        state:
                          description: experiment state
                          type: string
                        success:
                          description: success
                          type: boolean
                      required:
                      - kind
                      - state
                      - success
                      type: object
                    type: array
                  scope:
                    description: experiment scope for cache
                    type: string
                  state:
                    description: State is used to describe the experiment result
                    type: string
                  success:
                    description: Success is used to judge the experiment result
                    type: boolean
                  target:
                    type: string
                required:
                - action
                - scope
                - state
                - success
                - target
                type: object
              type: array
            phase:
              description: Phase indicates the state of the experiment   Initial ->
                Running -> Updating -> Destroying -> Destroyed
              type: string
          required:
          - expStatuses
          type: object
      type: object
  version: v1alpha1
  versions:
  - name: v1alpha1
    served: true
    storage: true


```

可以对比看下错误的情况：

```html

apiVersion: chaosblade.io/v1alpha1
kind: ChaosBlade
metadata:
  name: delay-pod-network-by-names
spec:
  experiments:
  - scope: pod
    target: network
    action: delay
    desc: "delay pod network by names"
    matchers:
    - name: namesss  // 故意错误
      value:
      - "java-springboottest-65c7798879-q5gvg"
    - name: namespace
      value:
      - "default"
    - name: interface
      value: ["eth0"]
    - name: time
      value: ["200"]
    - name: offset
      value: ["10"]


看下结果

"status": {
        "expStatuses": [
            {
                "action": "delay",
                "error": "less parameter: `evict-count|evict-percent|labels|names`",
                "resStatuses": [
                    {
                        "code": 45000,
                        "error": "less parameter: `evict-count|evict-percent|labels|names`",
                        "id": "delay-pod-network-by-names",
                        "kind": "", // 内层无信息
                        "state": "", // 内层无信息
                        "success": false
                    }
                ],
                "scope": "pod",
                "state": "Error",  // 外层有error
                "success": false,  //
                "target": "network"
            }
        ],
        "phase": "Error"
    }


```

这是最简单的一个示例，还有比如加timeout的，会在指定时间后消失创建的blade资源

```html

apiVersion: chaosblade.io/v1alpha1
kind: ChaosBlade
metadata:
  name: loss-pod-network-by-names
spec:
  experiments:
  - scope: pod
    target: network
    action: loss
    desc: "loss pod network by names"
    matchers:
    - name: names
      value:
      - "java-springboottest-65c7798879-q5gvg"
    - name: namespace
      value:
      - "default"
    - name: interface
      value: ["eth0"]
    - name: percent
      value: ["100"]
    - name: timeout   // 注意，创建blade资源后120秒自动删除
      value: ["120"]
#    - name: destination-ip
#      value: ["10"]


```

对于container的实验，可以这样

```html

// 获取信息

kubectl get pod  java-springboottest-65c7798879-q5gvg -o custom-columns=CONTAINER:.status.containerStatuses[0].name,ID:.status.containerStatuses[0].containerID

CONTAINER     ID
istio-proxy   docker://e370c1fc19d10f5f1daf72c3a8dc0b81ed7881f69f8a814add191cca287e2c82

// 替换下面的值

apiVersion: chaosblade.io/v1alpha1
kind: ChaosBlade
metadata:
  name: remove-container-by-id
spec:
  experiments:
  - scope: container
    target: container
    action: remove
    desc: "remove container by id"
    matchers:
    - name: container-ids
      value: ["c6cdcf60b82b854bc4bab64308b466102245259d23e14e449590a8ed816403ed"]
      # pod name
    - name: names
      value: ["guestbook-7b87b7459f-cqkq2"]
    - name: namespace
      value: ["chaosblade"]
```

不同场景的混沌实验中的参数与操作方式有些类似。其实对于这些在不同场景，比如 Pod、Node 和 Container 中进行混沌实验的实现是一致的，都是基于 `blade 这个 CLI 工具`，只对对其在不同场景进行了不同的封装


还有个非常重要的是

```html


{
        "ChaosBladeContainerName":"chaosblade-tool",
        "ChaosBladeNamespace":"chaosblade",
        "ChaosBladePodName":"chaosblade-tool-xdtgl",
        "Code":0,
        "Command":"/opt/chaosblade/blade create docker network loss  --percent=100 --interface=eth0 --image-repo hub.cloud.xxxx.com/chaosblade/chaosblade-tool --image-version 1.3.0 --container-id 40e991f7508a",
        "ContainerId":"40e991f7508a",
        "ContainerName":"app",
        "Error":"",
        "Id":"",
        "Namespace":"pro",
        "NodeName":"svr35612inxxxx",
        "PodIP":"10.109.18.87",
        "PodName":"rxxxx-91006940-2k5qj"
}


```

其实在生成执行器的时候有个判断，是不是网络的故障，并且--daemonset-enable=true

因为这样会走到docker的执行器中，也就是上面 blade create docker这种模式的，那么chaosblade-tool就有作用了，在目标pod中启动一个容器，并且运行tc，因为是共享网络的，所以也能达到效果

这边1.3的更新日志中有写

```html

增强安装时配置参数 daemonset.enable

daemonset.enable 是控制是否部署 chaosblade-tool daemonset，此 daemonset 目前进展 docker 容器下起作用，作用如下：

1、解决演练的目标容器中没有 tc 命令问题
2、支持节点演练

如果没有以上需求，可以将其设置为 false，即在使用 helm 安装时配置 daemonset.enable=false 此版本增强了此参数，在 1.3.0 版本之前，chaosblade-tool 的部署是通过 chaosblade operator 来控制部署，当前版本是单独出一个 yaml 文件部署，方便修改部署参数，支持指定节点部署。


还有就是 1.3和1.4的区别，1.3在上面使用 --daemonset-enable=true 就能使用blade create docker的命令。但是1.4改了代码，yaml要这样写才能进入sidecar的方式

apiVersion: chaosblade.io/v1alpha1
kind: ChaosBlade
metadata:
  name: delay-pod-network-by-names-14
spec:
  experiments:
  - scope: pod
    target: network
    action: delay
    desc: "delay pod network by names"
    matchers:
    - name: names
      value:
      - "java-springboottest-2-6b98cb5bb7-xmbh5"
    - name: namespace
      value:
      - "default"
    - name: is-docker-network
      value:
      - "true"
    - name: use-sidecar-container-network
      value:
      - "true"
    - name: interface
      value: ["eth0"]
    - name: time
      value: ["200"]
    - name: offset
      value: ["10"]


其执行命令却是 /opt/chaosblade/blade create cri network delay  --interface=eth0 --time=200 --offset=10 --image-repo chaosbladeio/chaosblade-tool --image-version 1.4.0 --container-id 535fe14d5536 --container-runtime

因为是1.4.0了，进入了cri的创建

```

#### chaosblade-exec-jvm

Chaosblade-exec-jvm通过JavaAgent attach方式来实现类的transform注入故障，底层使用jvm-sandbox实现，通过插件的可拔插设计来扩展对不同java应用的支持，可以很方便的扩展插件

下面是一些基础java类的介绍

模块管理

SandboxModule

作为Sandbox（chaosblade）的模块、所有的Sandbox事件，如Agent挂载（模块加载）、Agent卸载（模块卸载）、模块激活、模块冻结等都会在此触发，Sandbox内置jetty容器，访问api回调到注解为@Http("/xx")的方法，来实现故障能力。

StatusManager

blade create 命令在StatusManager注册状态、并管理整个实验的状态，包含攻击次数、攻击的百分比、命令参数、攻击方式（Action）等。

ModelSpecManager

管理插件的ModelSpec，ModelSpec的注册、卸载。

ListenerManager

管理插件的生命周期，插件的加载、卸载。

RequestHandler

Sandbox内置jetty容器，访问api回调到注解为@Http("/xx")的方法，由事件分发器（DispatchService）将事件分到RequestHandler处理，RequestHandler分为如下表格（表格中的【一定条件】可以参考下面的plugin加载方式）：


命令	RequestHandler
blade create	CreateHandler创建一个实验，StatusManager注册状态，满足一定条件的插件加载。
blade status	StatusHandler去StatusManager查询实验状态。
blade destroy	DestroyHandlerr销毁实验，满足一定条件的插件卸载。

以JVM的注入为例说明：

```html

./blade p jvm --pid 888

```

该命令下发后，将在目标jvm进程挂在Agent，触发SandboxModule onLoad()事件，初始化PluginLifecycleListener来管理插件的生命周期。

Plugin加载方式

加载方式	                                                 加载条件
SandboxModule onActive()事件	             Pointcut、ClassMatcher、MethodMatcher都不为空
blade create命令CreateHandler	             ModelSpect为PreCreateInjectionModelHandler类型，且ActionFlag 不为DirectlyInjectionAction类型（这是要加载的，下面有没有加载的，是直接注入的）

SandboxModule onActive()事件，会注册ModelSpec；Plugin加载时（add），创建事件监听器SandboxEnhancerFactory.createAfterEventListener(plugin)，监听器会监听感兴趣的事件，如BeforeAdvice、AfterAdvice等，具体实现如下：

```html

// 加载插件
public void add(PluginBean plugin) {
        PointCut pointCut = plugin.getPointCut();
        if (pointCut == null) {
            return;
        }
        String enhancerName = plugin.getEnhancer().getClass().getSimpleName();
  			// 创建filter PointCut匹配
        Filter filter = SandboxEnhancerFactory.createFilter(enhancerName, pointCut);

        if (plugin.isAfterEvent()) {
          	// 事件监听
            int watcherId = moduleEventWatcher.watch(filter, SandboxEnhancerFactory.createAfterEventListener(plugin),
                Type.BEFORE, Type.RETURN);
            watchIds.put(PluginUtil.getIdentifierForAfterEvent(plugin), watcherId);
        } else {
            int watcherId = moduleEventWatcher.watch(
                filter, SandboxEnhancerFactory.createBeforeEventListener(plugin), Event.Type.BEFORE);
            watchIds.put(PluginUtil.getIdentifier(plugin), watcherId);
        }
    }

```

PointCut匹配

SandboxModule onActive()事件触发Plugin加载后，SandboxEnhancerFactory创建filter，filter内部通过PointCut的ClassMatcher和MethodMatcher过滤。

Enhancer

如果已经加载插件，此时目标应用匹配能匹配到filter后，EventListener已经可以被触发，但是chaosblade-exec-jvm内部通过StatusManager管理状态，所以故障能力不会被触发。


例如BeforeEventListener触发调用BeforeEnhancer的beforeAdvice方法，在ManagerFactory.getStatusManager().expExists(targetName)判断时候被中断，具体的实现如下：


```html

com.alibaba.chaosblade.exec.common.aop.BeforeEnhancer

public void beforeAdvice(String targetName,
            ClassLoader classLoader,
            String className,
            Object object,
            Method method,
            Object[] methodArguments) throws Exception {
  			// StatusManager
        if (!ManagerFactory.getStatusManager().expExists(targetName)) {
            return;
        }
        EnhancerModel model = doBeforeAdvice(classLoader, className, object, method, methodArguments);
        if (model == null) {
            return;
        }
        model.setTarget(targetName).setMethod(method).setObject(object).setMethodArguments(methodArguments);
        Injector.inject(model);
    }

```

创建混沌实验

```html

./blade create servlet --requestpath=/topic delay --time=3000

```

该命令下发后，触发SandboxModule @Http("/create")注解标记的方法，将事件分发给com.alibaba.chaosblade.exec.service.handler.CreateHandler处理 在判断必要的uid、target、action、model参数后调用handleInjection，handleInjection通过状态管理器注册本次实验，如果插件类型是PreCreateInjectionModelHandler的类型，将预处理一些东西。同时如果Action类型是DirectlyInjectionAction，那么将直接进行故障能力注入，如jvm oom等，如果不是那么将加载插件。

// delay不属于DirectlyInjectionAction


ModelSpec

PreCreateInjectionModelHandler	预创建
PreDestroyInjectionModelHandler	预销毁

DirectlyInjectionAction

如果ModelSpec是PreCreateInjectionModelHandler类型，且ActionSpec的类型是DirectlyInjectionAction类型，将直接进行故障能力注入，比如JvmOom故障能力，ActionSpec的类型不是DirectlyInjectionAction类型，将加载插件。

```html

DirectlyInjectionAction	Not DirectlyInjectionAction
PreCreateInjectionModelHandler（ModelSpec）	直接进行故障能力注入	加载插件
PreDestroyInjectionModelHandler（ModelSpec）	停止故障能力注入	卸载插件

```

```html

private Response handleInjection(String suid, Model model, ModelSpec modelSpec) {
 				// 注册
        RegisterResult result = this.statusManager.registerExp(suid, model);
        if (result.isSuccess()) {
            // handle injection
            try {
                applyPreInjectionModelHandler(suid, modelSpec, model);
            } catch (ExperimentException ex) {
                this.statusManager.removeExp(suid);
                return Response.ofFailure(Response.Code.SERVER_ERROR, ex.getMessage());
            }

            return Response.ofSuccess(model.toString());
        }
        return Response.ofFailure(Response.Code.DUPLICATE_INJECTION, "the experiment exists");
    }

```

注册成功后返回uid，如果本阶段直接进行故障能力注入了，或者自定义Enhancer advice返回null，那么后不通过Inject类触发故障。


故障能力注入

故障能力注入的方式，最终都是调用ActionExecutor执行故障能力。

1、通过Inject注入。
2、DirectlyInjectionAction直接注入，直接注入不进过Inject类调用阶段，如jvm oom等。


DirectlyInjectionAction直接注入不经过Enhancer参数包装匹配直接到故障触发ActionExecutor执行阶段，如果是Inject注入此时因为StatusManager已经注册了实验，当事件再次出发后ManagerFactory.getStatusManager().expExists(targetName)的判断不会被中断，继续往下走，到了自定义的Enhancer，在自定义的Enhancer里面可以拿到原方法的参数、类型等，甚至可以反射调原类型的其他方法，这样做风险较大，一般在这里往往是取一些成员变量或者get方法等，用户后续参数匹配。

匹配参数包装

自定义的Enhancer，如ServletEnhancer，把一些需要与命令行匹配的参数 包装在MatcherModel里面，然后包装EnhancerModel返回，比如 --requestpath = /index，那么requestpath等于requestURI去除contextPath。参数匹配在 Injector.inject(model)阶段判断。

```html

public class ServletEnhancer extends BeforeEnhancer {

    private static final Logger LOOGER = LoggerFactory.getLogger(ServletEnhancer.class);

    @Override
    public EnhancerModel doBeforeAdvice(ClassLoader classLoader, String className, Object object,
                                        Method method, Object[] methodArguments,String targetName)
        throws Exception {
      	// 获取原方法的一些参数
        Object request = methodArguments[0];
        String queryString = ReflectUtil.invokeMethod(request, "getQueryString", new Object[] {}, false);
        String contextPath = ReflectUtil.invokeMethod(request, "getContextPath", new Object[] {}, false);
        String requestURI = ReflectUtil.invokeMethod(request, "getRequestURI", new Object[] {}, false);
        String requestMethod = ReflectUtil.invokeMethod(request, "getMethod", new Object[] {}, false);

        String requestPath = StringUtils.isBlank(contextPath) ? requestURI : requestURI.replaceFirst(contextPath, "");

      	//
        MatcherModel matcherModel = new MatcherModel();
        matcherModel.add(ServletConstant.QUERY_STRING_KEY, queryString);
        matcherModel.add(ServletConstant.METHOD_KEY, requestMethod);
        matcherModel.add(ServletConstant.REQUEST_PATH_KEY, requestPath);
        return new EnhancerModel(classLoader, matcherModel);
    }
}

```

参数匹配和能力注入（Inject调用）

inject阶段首先获取StatusManager注册的实验，compare(model, enhancerModel)经常参数比对，失败后return，limitAndIncrease(statusMetric)判断 --effect-count --effect-percent来控制影响的次数和百分比

```html

public static void inject(EnhancerModel enhancerModel) throws InterruptProcessException {
        String target = enhancerModel.getTarget();
        List<StatusMetric> statusMetrics = ManagerFactory.getStatusManager().getExpByTarget(
            target);
        for (StatusMetric statusMetric : statusMetrics) {
            Model model = statusMetric.getModel();
            if (!compare(model, enhancerModel)) {
                continue;
            }
            try {
                boolean pass = limitAndIncrease(statusMetric);
                if (!pass) {
                    LOGGER.info("Limited by: {}", JSON.toJSONString(model));
                    break;
                }
                LOGGER.info("Match rule: {}", JSON.toJSONString(model));
                enhancerModel.merge(model);
                ModelSpec modelSpec = ManagerFactory.getModelSpecManager().getModelSpec(target);
                ActionSpec actionSpec = modelSpec.getActionSpec(model.getActionName());
                actionSpec.getActionExecutor().run(enhancerModel);
            } catch (InterruptProcessException e) {
                throw e;
            } catch (UnsupportedReturnTypeException e) {
                LOGGER.warn("unsupported return type for return experiment", e);
                statusMetric.decrease();
            } catch (Throwable e) {
                LOGGER.warn("inject exception", e);          
                statusMetric.decrease();
            }
            break;
        }
    }

```

故障触发

由Inject触发，或者由DirectlyInjectionAction直接触发，最后调用自定义的ActionExecutor生成故障，如 DefaultDelayExecutor，此时故障能力已经生效了。

```html

public void run(EnhancerModel enhancerModel) throws Exception {
    String time = enhancerModel.getActionFlag(timeFlagSpec.getName());
    Integer sleepTimeInMillis = Integer.valueOf(time);
    int offset = 0;
    String offsetTime = enhancerModel.getActionFlag(timeOffsetFlagSpec.getName());
    if (!StringUtil.isBlank(offsetTime)) {
        offset = Integer.valueOf(offsetTime);
    }
    TimeoutExecutor timeoutExecutor = enhancerModel.getTimeoutExecutor();
    if (timeoutExecutor != null) {
        long timeoutInMillis = timeoutExecutor.getTimeoutInMillis();
        if (timeoutInMillis > 0 && timeoutInMillis < sleepTimeInMillis) {
            sleep(timeoutInMillis, 0);
            timeoutExecutor.run(enhancerModel);
            return;
        }
    }
    sleep(sleepTimeInMillis, offset);
}

public void sleep(long timeInMillis, int offsetInMillis) {
        Random random = new Random();
        int offset = 0;
        if (offsetInMillis > 0) {
            offset = random.nextInt(offsetInMillis);
        }
        if (offset % 2 == 0) {
            timeInMillis = timeInMillis + offset;
        } else {
            timeInMillis = timeInMillis - offset;
        }
        if (timeInMillis <= 0) {
            timeInMillis = offsetInMillis;
        }
        try {
          	// 触发延迟
            TimeUnit.MILLISECONDS.sleep(timeInMillis);
        } catch (InterruptedException e) {
            LOGGER.error("running delay action interrupted", e);
        }
    }

```

销毁实验

```html

./blade destroy 52a27bafc252beee

```

该命令下发后，触发SandboxModule @Http("/destory")注解标记的方法，将事件分发给com.alibaba.chaosblade.exec.service.handler.DestroyHandler处理。注销本次故障的状态。

如果插件的ModelSpec是PreDestroyInjectionModelHandler类型，且ActionSpec的类型是DirectlyInjectionAction类型，停止故障能力注入，ActionSpec的类型不是DirectlyInjectionAction类型，将卸载插件。

```html

public Response handle(Request request) {
        String uid = request.getParam("suid");
        String target = request.getParam("target");
        String action = request.getParam("action");
        if (StringUtil.isBlank(uid)) {
            if (StringUtil.isBlank(target) || StringUtil.isBlank(action)) {
                return Response.ofFailure(Code.ILLEGAL_PARAMETER, "less necessary parameters, such as uid, target and"
                    + " action");
            }
            // 注销status
            return destroy(target, action);
        }
        return destroy(uid);
    }

```

卸载Agent

```html

./blade revoke 98e792c9a9a5dfea

```

该命令下发后，触发SandboxModule unload()事件，同时插件卸载。

```html

public void onUnload() throws Throwable {
    LOGGER.info("unload chaosblade module");
    dispatchService.unload();
    ManagerFactory.unload();
    watchIds.clear();
    LOGGER.info("unload chaosblade module successfully");
}

```


如何去扩展一个插件

1、在chaosblade-exec-plugin模块下新建子模块，如chaosblade-exec-plugin-servlet

2、自定义Enhancer

例如ServletEnhancer，获取ContextPath、RequestURI、Method等，将获取到的参数放到MatcherModel，返回EnhancerModel，Inject阶段会与输入的参数做比对。


```html

public class ServletEnhancer extends BeforeEnhancer {

    @Override
    public EnhancerModel doBeforeAdvice(ClassLoader classLoader,
                                        String className,
                                        Object object,
                                        Method method,
                                        Object[] methodArguments
                                        ,String targetName) throws Exception {
        Object request = methodArguments[0];
        // 执行被增强类的方法，获取一些需要的值
        String queryString = ReflectUtil.invokeMethod(request, "getQueryString", new Object[] {}, false);
        String contextPath = ReflectUtil.invokeMethod(request, "getContextPath", new Object[] {}, false);
        String requestURI = ReflectUtil.invokeMethod(request, "getRequestURI", new Object[] {}, false);
        String requestMethod = ReflectUtil.invokeMethod(request, "getMethod", new Object[] {}, false);

        String requestPath = StringUtils.isBlank(contextPath) ? requestURI : requestURI.replaceFirst(contextPath, "");

        MatcherModel matcherModel = new MatcherModel();
        matcherModel.add(ServletConstant.QUERY_STRING_KEY, queryString);
        matcherModel.add(ServletConstant.METHOD_KEY, requestMethod);
        matcherModel.add(ServletConstant.REQUEST_PATH_KEY, requestPath);

        return new EnhancerModel(classLoader, matcherModel);
    }
}

```

需不同的通知可继承不同的类

beforeAdvice：继承com.alibaba.chaosblade.exec.common.aop.BeforeEnhancer
afterAdvice：继承com.alibaba.chaosblade.exec.common.aop.AfterEnhancer

3、自定义PointCut

例如ServletPointCut拦截类：spring的FrameworkServlet、webx的WebxFrameworkFilter、及父类为HttpServletBean或HttpServlet的子类。

拦截方法：doGet、doPost、doDelete、doPut、doFilter

```html

public class ServletPointCut implements PointCut {

    public static final String SPRING_FRAMEWORK_SERVLET = "org.springframework.web.servlet.FrameworkServlet";
    public static final String ALIBABA_WEBX_FRAMEWORK_FILTER = "com.alibaba.citrus.webx.servlet.WebxFrameworkFilter";
    public static final String SPRING_HTTP_SERVLET_BEAN = "org.springframework.web.servlet.HttpServletBean";
    public static final String HTTP_SERVLET = "javax.servlet.http.HttpServlet";

    public static Set<String> enhanceMethodSet = new HashSet<String>();
    public static Set<String> enhanceMethodFilterSet = new HashSet<String>();

    static {
        enhanceMethodSet.add("doGet");
        enhanceMethodSet.add("doPost");
        enhanceMethodSet.add("doDelete");
        enhanceMethodSet.add("doPut");
        enhanceMethodFilterSet.add("doFilter");
    }

    @Override
    public ClassMatcher getClassMatcher() {
        OrClassMatcher orClassMatcher = new OrClassMatcher();
        orClassMatcher.or(new NameClassMatcher(SPRING_FRAMEWORK_SERVLET)).or(
            new NameClassMatcher(ALIBABA_WEBX_FRAMEWORK_FILTER)).or(
            new SuperClassMatcher(SPRING_HTTP_SERVLET_BEAN, HTTP_SERVLET));
        return orClassMatcher;
    }

    @Override
    public MethodMatcher getMethodMatcher() {
        AndMethodMatcher andMethodMatcher = new AndMethodMatcher();
        OrMethodMatcher orMethodMatcher = new OrMethodMatcher();
        orMethodMatcher.or(new ManyNameMethodMatcher(enhanceMethodSet)).or(new ManyNameMethodMatcher
            (enhanceMethodFilterSet));
        andMethodMatcher.and(orMethodMatcher).and(new ParameterMethodMatcher(1, ParameterMethodMatcher.GREAT_THAN));
        return andMethodMatcher;
    }
}

```

继承com.alibaba.chaosblade.exec.common.aop.PointCut
getClassMatcher：类匹配
SuperClassMatcher：父类名称匹配
OrClassMatcher：多个匹配
NameClassMatcher：类名匹配
getMethodMatcher：类方法匹配
ManyNameMethodMatcher：方法名集合
NameMethodMatcher：方法名称匹配
OrMethodMatcher：多个方法匹配
AndMethodMatcher：多条件匹配
ParameterMethodMatcher：参数匹配

4、自定义Spec

举个例子：命令[./blade create servlet delay --time=3000] 对于命令而言主要分为phases、target、action、flag，phases相对插件而言不需要很强的灵活性，因此由chaosblade-exec-service模块管理，对于自定义插件只需要扩展ModelSpec(target)、action、 flag。

ModelSpec

ModelSpec的getTarget()方法对于命令中target部分的名称，如servlet、dubbo等，createNewMatcherSpecs()方法添加ModelSpec下的FlagSpec，例如ServletModelSpec的getTarget()返回servlet，createNewMatcherSpecs()包含很多flagSpec，那么ModelSpec支持的命令如下：

./blade create servlet --method=post --requestpath=/index --contextpath=/shop

--参数可任意组合。

```html

public class ServletModelSpec extends FrameworkModelSpec {

    @Override
    public String getTarget() {
        return "servlet";
    }

    @Override
    public String getShortDesc() {
        return "java servlet experiment";
    }

    @Override
    public String getLongDesc() {
        return "Java servlet experiment, support path, query string, context path and request method matcher";
    }

    @Override
    public String getExample() {
        return "servlet --requestpath /hello --method post";
    }

    @Override
    protected List<MatcherSpec> createNewMatcherSpecs() {
        ArrayList<MatcherSpec> matcherSpecs = new ArrayList<MatcherSpec>();
        matcherSpecs.add(new ServletContextPathMatcherSpec());
        matcherSpecs.add(new ServletQueryStringMatcherSpec());
        matcherSpecs.add(new ServletMethodMatcherSpec());
        matcherSpecs.add(new ServletRequestPathMatcherSpec());
        return matcherSpecs;
    }
}

```

ModelSpec的实现方式如下：

实现com.alibaba.chaosblade.exec.common.model.ModelSpec
继承BaseModelSpec，实现了对CreateHandler阶段的输入参数的校验
继承FrameworkModelSpec包含DelayActionSpec、ThrowCustomExceptionActionSpec，默认实现了不同target的延迟侵入和异常侵入。


ActionSpec

例如DelayActionSpec，支持参数 --time=xx --offset=xx

./blade create servlet --method=post delay --time=3000 ----offset=10

```html
public class DelayActionSpec extends BaseActionSpec {

    private static TimeFlagSpec timeFlag = new TimeFlagSpec();
    private static TimeOffsetFlagSpec offsetFlag = new TimeOffsetFlagSpec();

    public DelayActionSpec() {
      	//添加 actionExecutor
        super(new DefaultDelayExecutor(timeFlag, offsetFlag));
    }

    @Override
    public String getName() {
        return "delay";
    }

    @Override
    public String[] getAliases() {
        return new String[0];
    }

    @Override
    public String getShortDesc() {
        return "delay time";
    }

    @Override
    public String getLongDesc() {
        return "delay time...";
    }

    @Override
    public List<FlagSpec> getActionFlags() {
        return Arrays.asList(timeFlag, offsetFlag);
    }

    @Override
    public PredicateResult predicate(ActionModel actionModel) {
        if (StringUtil.isBlank(actionModel.getFlag(timeFlag.getName()))){
            return PredicateResult.fail("less time argument");
        }
        return PredicateResult.success();
    }
}


```

ActionSpec的getName()方法对应命令中action部分的名称，如delay、throwCustomExceptionde等，ActionSpec由ModelSpec的addActionSpec()方法添加，可以有以下方式实现：

实现com.alibaba.chaosblade.exec.common.model.action.ActionSpec

继承BaseActionSpec，实现了对CreateHandler阶段的输入参数的校验

FlagSpec

例如TimeOffsetFlagSpec， 支持--offset=10 的参数

```html

public class TimeOffsetFlagSpec implements FlagSpec {
    @Override
    public String getName() {
        return "offset";
    }

    @Override
    public String getDesc() {
        return "delay offset for the time";
    }

    @Override
    public boolean noArgs() {
        return false;
    }

    @Override
    public boolean required() {
        return false;
    }
}

```

FlagSpec的getName()方法对应命令中flag部分的名称，如--time等

实现com.alibaba.chaosblade.exec.common.model.FlagSpec，由ActionSpec的getFlagSpec方法添加

继承com.alibaba.chaosblade.exec.common.model.matcher.MatcherSpec，由ActionSpec的addActionSpec添加，CreateHandler阶段会做参数校验

继承com.alibaba.chaosblade.exec.common.model.matcher.BasePredicateMatcherSpec


ActionExecutor

ActionExecutor执行器作为BaseActionSpec的构造参数，ActionExecutor可以自定义一些增强业务的操作，如修改方法的参数、篡改方法的返回值等。

```html

public interface ActionExecutor {

    /**
     * Run executor
     *
     * @param enhancerModel
     * @throws Exception
     */
    void run(EnhancerModel enhancerModel) throws Exception;
}

```

实现ActionExecutor的接口，EnhancerModel里面可以拿到命令行输入的参数以及原始方法的参数，类型，返回值、异常，做一些增强业务操作。

```html

// 延迟多少毫秒
Long time = Long.valueOf(enhancerModel.getActionFlag("time"));
TimeUnit.MILLISECONDS.sleep(time);

```


5、自定义Plugin

继承com.alibaba.chaosblade.exec.common.aop.Plugin，自定义target名称，添加Enhancer、PointCut、ModelSpec即可，实现类需要全路径名复制到 : resources/META-INF/services/com.alibaba.chaosblade.exec.common.aop.Plugin 挂载Agent，模块激活后plugin自动加载。

```html

public class ServletPlugin implements Plugin {

    @Override
    public String getName() {
        return "servlet";
    }

    @Override
    public ModelSpec getModelSpec() {
        return new ServletModelSpec();
    }

    @Override
    public PointCut getPointCut() {
        return new ServletPointCut();
    }

    @Override
    public Enhancer getEnhancer() {
        return new ServletEnhancer();
    }
}

```



```html

// 挂载agent： -p 3356 是被攻击应用的jvm进程号   // 这个可以查看版本啊，启动端口等等
cd ./build-target/chaosblade-0.7.0/lib/sandbox/bin/ && ./sandbox.sh -p 3356

// 激活模块
./sandbox.sh -p 3356 -a chaosblade

```

此阶段可反复侵入和销毁，suid为一次实验的上下文唯一， 一个简单的例子，对servlet容器，api接口延迟3秒


```html

创建混沌实验

curl -X post http://127.0.0.1:52971/sandbox/default/module/http/chaosblade/create -H 'Content-Type:application/json' -d '{"action":"delay","target":"servlet","suid":"110","time":"3000","requestpath":"/hello"}'

销毁

curl -X post http://127.0.0.1:52971/sandbox/default/module/http/chaosblade/destroy -H 'Content-Type:application/json' -d '{"action":"delay","target":"servlet","suid":"110"}'


```

```html

./sandbox.sh -p 3356 -S   // 这个是进行agent的卸载

```

有一个问题就是

```html

./blade status --type prepare

./blade status --type create

agent和注入都还在运行，为什么不生效？因为可能tomcat已经重启啊等等操作导致运行的假象，telnet存储的端口
cat /home/deploy/.sandbox.token
chaosblade;234817233629;localhost;39107

比如这个39107，还在不在了，不在了说明都没有挂载了

```

6、关于切面的问题

```html

当匹配到match的类和method后就会流转到jvm程序中，但是匿名内部类（数字按照声明先后排序，纯数字）和 局部内部类（可能重名，会添加数字）会带数字。。

一旦别人的代码发生了变化，声明的顺序变了，那么数字就变了。。。

javap -v Class$SubClass1 这样去查看内部类的字节码定义 就能看见问题

```
