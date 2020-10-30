---
layout: post
title: 微服务及相关框架
author-id: "zqmalyssa"
tags: [java, microservice]
---

微服务用了这么长时间，也确实要总结一下，还有其相关技术等方方面面

#### 微服务基本内容

微服务应该没有具体的定义，主要还是够小，够自治，但是使不使用微服务，有什么优缺点需要说下

优点：

1. 易于部署
2. 与组织结构对齐
3. 可组合性
4. 可替代性

缺点：

有代价的

1. 分布式系统的复杂性，用TCC啊，2PC啊
2. 开发，测试等诸多研发过程中的复杂性
3. 部署，监控等诸多运维复杂性

还要说下云原生：CNCF说 云原生技术有利于各组织在公有云，私有云和混合云等新型动态环境中，构建和运行可弹性扩展的应用，主要是提升效率

Spring中对云原生应用有要求，要有DEVOPS，CONTINUOUS DELIVERY，MICROSERVICES，CONTAINERS等

DEVOPS：开发与运维一同致力于交付高品质的软件服务于客户
CONTINUOUS DELIVERY：软件的构建、测试和发布，要更快，更频繁，更稳定
MICROSERVICES：以一组小型服务的形式来部署应用
CONTAINERS：提供比传统虚拟机更高的效率

CNCF(Cloud Native Computing Foundation)旗下有kubernetes，Prometheus，Envoy，CoreDNS，Container等

12 Factors

1. 一份基准代码，多份部署

SVN， GIT

2. 显示声明依赖

Maven

3. 严格分离构建和运行

构建有代码扫描，自动化测试，安全扫描

发布平台进行推送，别scp

4. 以一个或多个无状态进程进行应用

任何需要持久化的数据都要存储在后端服务内

5. 快速的启动和优雅终止可最大化健壮性

进程应当追去最小启动时间

6. 尽可能的保持开发，预发布，线上环境相同

#### Spirng Cloud

让你不要关注服务间的东西，tracing，注册，发现等，交给Spring Cloud，更多关注自身的业务上

1. 服务发现

Eureka不建议在AWS上使用eureka，而且2.0版本也不再开源，作为学习参考就行，生产上

EurekaClient和DiscoveryClient中对应的getNextServerFromEureka()和getInstances()可以获得服务地址，建议后者，还有loadbalancerClient（如何做loadbalancer，其实是对RestTemplate进行了增强）

Spring Cloud Commons提供的抽象：
1、服务注册抽象
  - 提供了ServiceRegistry抽象
2、客户发现抽象
  - 提供了DiscoveryClient抽象
    - @EnableDiscoveryClient
  - 提供了LoadBalancerClient抽象

自动向Eureka服务端注册

ServiceRegistry
  - EurakaServiceRegistry
  - EurekaRegistration

这部分看看源码，ServiceRegistry接口里有register和deregister两个方法，是一个lifecycle的管理

自动配置
  - EurekaClientAutoConfiguration
  - EurekaAutoServiceRegistration
    - SmartLifecycle

**补充：ZooKeeper也可以做注册中心**

ZooKeeper是一个分布式的协调服务，最早是Hadoop中的一个组件，现在apache的顶级项目，其设计目标

简单，多副本，有序，快

如果要用ZooKeeper作为注册中心，就要引入以下starter

spring-cloud-starter-zookeeper-discovery

简单配置

spring.cloud.zookeeper.connect-string=localhost:2181

有两篇文章

《阿里巴巴为什么不用ZooKeeper做服务发现》
《Eureka! Why You Shouldn't Use ZooKeeper for Service Discovery》 这是Netflix写的

核心思想

在实践中，注册中心不能因为自身的任何原因破坏服务之间本身的可连通性

注册中心需要AP，而ZooKeeper是CP

CAP--一致性，可用性，分区容忍性，看看分布式事务和分布式锁那篇文章，BASE的理论，要用ZooKeeper，尽量在同一个机房或者同一个可用区里面

如果要跨机房，就是多个Zookeeper注册中心

**Consul也可以作为注册中心**

consul是分布式的，高可用的，可感知数据中心的

[官方指引](https://www.consul.io)

关键特性
  - 服务发现
  - 健康检查
  - KV存储
  - 多数据中心支持
  - 安全的服务间通信

好用的功能
  - HTTP API
  - DNS(xxx.service.consul)
  - 与nginx联动，比如ngx_http_consul_backed_module

如果要用Consul作为注册中心，就要引入以下starter

spring-cloud-starter-consul-discovery

简单配置

spring.cloud.consul.host=
spring.cloud.consul.port=

**Nacos也可以作为注册中心**

一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台，https://nacos.io/zh-cn/index.html

阿里巴巴开源，功能：

1、动态服务配置
2、服务发现和管理
3、动态DNS服务

**也可以自己实现**


2. 服务熔断

Hystrix实现服务熔断，先了解下断路器

Martin Fowler提出的在一篇文章里，核心思想

a.在断路器对象中封装受保护的方法调用
b.该对象监控调用和断路情况
c.调用失败触发阈值后，后续调用直接由断路器返回错误，不再执行实际调用

Aspect可以自己实现一个断路器


Hystrix也是实现了一个断路器，但现在跟eureka一样，也不维护了

如何理解熔断的情况（Circuit Breaking）

a.打日志
b.看监控，主动向监控系统埋点，上报熔断情况  提供与熔断相关的endpoint，让第三方系统来拉取信息

熔断的话有dashboard可以看，但是最好还是输出到统一监控系统中，比如普罗米修斯

官方推荐使用Resilience4j来替代hystrix，https://github.com/resilience4j/resilience4j

```java
<dependency>
  <groupId>io.github.resilience4j</groupId>
  <artifactId>resilience4j-spring-boot2</artifactId>
  <version>0.14.1</version>
</dependency>
```

它更轻量，且易于使用的容错库，针对java8的函数式编程

它的实现也是断路器，也就是基于ConcurrentHashMap的内存断路器，有CircuitBreakerRegistry和CircuitBreakerConfig，比如某些API访问先返回出错日志，然后断路器生效，就返回500 internal server error，这样也比较好看

注意**Resilience4j**是可以做限流的，**限流可以理解一个是一次并发下来有多少个请求可以被接受，还有个就是一段时间内有多少个请求可以被接受**

限流的目的是防止下游依赖被并发请求冲击，防止发生连环事故，用法是bulkhead(隔舱)

BulkheadRegistry / BulkheadConfig
@Bulkhead(name = "名称")

配置

BulkheadProperities
resilience4j.bulkhead.backends.名称
max-concurrent-call
max-wait-time

还有个限流的使用是RateLimiter，目的是限制特定时间内的执行次数，用法

RateLimiterRegistry / RateLimiterConfig
@RateLimiter(name = "名称")

配置

RateLimiterProperities
resilience4j.ratelimiter.limiters.名称
limit-for-period
limit-refresh-period-in-millis
timeout-in-millis

配合使用，在customer做并发的bulkhead，在server那边再做一个ratelimit的控制

另外，有Google Guava（有令牌桶限制，缓存机制，如果是JDK6，可以像8一样的集合框架）

3. 配置服务

目的是提供对外配置的HTTP API

依赖Spring-cloud-config
@EnableConfigServer
支持Git/SVN/Vault/JDBC

client要怎么使用呢

spring-cloud-starter-config

也可以做基于Zookeeper的配置中心，依赖

spring-cloud-starter-zookeeper-config

ZK里面有监控，会监控配置中的变化，refresh，Spring Cloud自己做刷新的话要用SpringBus

原理是类似于Spring应用中的environment和propertysource

Spring Cloud Config Client - CompositePropertySource
Zookeeper - ZookeeperPropertySource
Consul - ConsulPropertySource / ConsulFilesPropertySource

nacos也可以做配置中心，还有携程的Apollo

4. 服务安全

5. 服务网关

6. 分布式消息

一款构建消息驱动的微服务应用程序的轻量级框架
  - 声明式编程模型
  - 引入多种概念抽象，发布订阅，消费组，分区
  - 支持多种中间件，rabbitmq，kafka

主要是跟Binder打交道

Binding是应用中生产者、消费者与系统之间的桥梁
@EnableBinding
@Input / SubscribableChannel
@Output / MessageChannel

消费组，对同一个消息，每个组中都会有一个消费者收到消息，每个组里面都会收到消息，然后仅有一个节点处理消息，但是还是要处理好消息的重投，幂等

分区，在一个分区里，把消息看成是有序的，且都是同一个消费者消费，在消费者里面还是要考虑顺序

消费组和分区的概念都是kafka里面的

依赖

Spring Cloud - spring-cloud-starter-stream-rabbit
Spring Boot - spring-boot-starter-amqp

配置

spring.cloud.stream.rabbit.binder.*
spring.cloud.stream.rabbit.bindings.<channelName>.consumer.*
spring.rabbitmq.*

http://tryrabbitmq.com

另一个常用的是kafka，是2011年由linkedin开源的

依赖

spring-cloud-starter-stream-kafka

配置

spring.cloud.stream.kafka.binder.*
spring.cloud.stream.kafka.bindings.<channelName>.consumer.*
spring.kafka.*

补充一下

Spring中的定时任务机制@EnableSchedule
Spring中的事件机制ApplicationEvent，发送事件ApplicationEventPublisherAware，ApplicationEventPublisher.publishEvent()。监听事件ApplicationListener<T>，@EventListener

7. 分布式追踪

链路治理

系统中都有哪些服务，服务之间的依赖关系是什么样的，一个常见请求具体的执行路径是什么样的，请求每个环节的执行是否正常与耗时情况

而埋点一般是通过采样的分析

Google中的Dapper，span，基本的工作单元，trace，由一组span构成的树型结构，annotation，用于及时记录事件cs- Client Sent，sr- Server Received，ss- Server Sent，cr- Client Received

用Spring Cloud Sleuth提供服务治理

依赖

spring-cloud-starter-sleuth
spring-cloud-starter-zipkin

日志输出

[appname,traceId,spanId,exportable]

将埋点日志输出到zipkin，配置

spring.zipkin.base-url=
spirng.zipkin.discovery-client-enable=
spring.zipkin.sender.type=web | rabbit | kafka
spring.zipkin.compression.enabled=
spring.sleuth.sampler.probability=0.1 //采样比例

如果通过MQ进行埋点，需要增加RabbitMQ和Kafka依赖，那么type就是MQ了，这就完整了，一个请求怎么走的，web，包括走到了mq中(broker)

8. 各种云平台支持


#### 补充

SpringSession，在SpringSession中运用到了redis，那么可以解决分布式session中遇到的问题

解决分布式session，其实可以

1. Sticky Sessio
2. Session Replication
3. 集中会话，Centralized Session
  - SpringSession就是集中会话，支持的存储有redis（在redis中存储了session），MongoDB，JDBC，Hazelcast等

实现原理就是定制HttpSession，

通过定制的HttpServletRequest返回定制的HttpSession，比如定制的要createSession就会创建RedisSession，然后有个CommitSession，有save，保存session的信息
