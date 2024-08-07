---
layout: post
title: 疑难杂症
tags: [difficult]
author-id: zqmalyssa
---

#### pj的问题

1、no work available的问题(如果有这个问题先看是否在启动wf任务后换主了，并且换主前有多个app共享一个server)

得是一个workflow的任务，开始在LZ的机器，然后2点半篡位了，app2的master变到了XZ，看日志，正常上报心跳，3点清理不用的worker（就是现在时间到最后一次心跳上报时间晚于1min的实例），然后workflow的任务继续在

LZ上执行的时候就没有worker了


a、worker连接server（服务发现，隔10秒发一次discovery）用的powerjob的域名是 local to local的，那么根据acquire的逻辑，currentServer（10.xx.xx.xx:10086）到本机的就应该直接返回（问题之前一直是local to local的，但删除的时候各个地址都有的）

ServerDiscoveryService.discovery()，就是一个http请求（客户端发起的），http://powerjob.xxx.xx.com/server/acquire?appId=%d&currentServer=%s（第一次去拿，拿完就塞到客户端自己的内存中10.6.6.1:10086这样的形式，后续不是Ip去访问，还是域名去拿的）

到达server，无论哪台，就是去查一下数据库里面的currentServer，如果和本机server一样，直接返回server，如果不是，去ping一把数据库中这个currentServer，ok就返回 // 所以基本上数据库是什么server，返回就是什么server了，对所有client一样

到达server，还有个情况 就是没有可用的server，就要选举，先查看有没有别的server选举好了，没有就自己当主，刷新回app_info中并返回给客户端

b、心跳上报是隔15秒上报一次，有了currentServer（也是起了一个线程不停的更新），才会上报，主要currentServer这边就是ip+port了

akka://oms-server@10.6.6.1:10086/user/server_actor

akka到达server端后，server会把心跳的信息保存起来，每个client一个key，有最近一次的时间map，还有对应的指标map，定时3点

c、client调用时候的currentAddress和appId，也是调用域名获取，所以也是从worker通过local to local去call到server的

d、cancel dispatch job due to no worker available, clusterStatus is CAN'T_FIND_ANY_WORKER. // 这个是直接map里面是null，appId2ClusterStatus会每个server每隔15s去清除不是自己管理appId的key（怀疑是这个影响）验证如下

15号11点起的wf任务，16点凌晨的都没成功，报上面的错误，是另一种no work available的问题，关键日志如下：

2023-05-15 15:10:31.506 [http-nio-8080-exec-29] WARN  c.g.k.p.s.s.h.ServerSelectService - [ServerSelectService] server(10.97.182.70:10086) was down.
2023-05-15 15:10:33.219 [http-nio-8080-exec-29] INFO  c.g.k.p.s.s.h.ServerSelectService - [ServerSelectService] this server(10.112.13.82:10086) become the new server for app(appId=2).
2023-05-15 15:10:33.645 [oms-server-akka.actor.default-dispatcher-143208] WARN  c.g.k.p.s.a.a.ServerTroubleshootingActor - [ServerTroubleshootingActor] receive DeadLetter: DeadLetter(AskResponse(success=true, data=[45, 49], message=null),Actor[akka://oms-server@10.97.182.70:10086/user/friend_actor#1073795699],Actor[akka://oms-server/temp/$JCBd])


这其中有三个app，c1 对应10.97.182.70:10086 c2 也对应 10.97.182.70:10086 c3 对应 10.112.89.121:10086，15号11点起的任务扔到了10.97.182.70:10086（是c1这个app的），另有一个server 10.112.13.82:10086之前一直没有接app，但是下午3点10分检测到10.97.182.70:10086 down了，自己变成c1的主了，数据库里c1的currentServer变成了10.112.13.82:10086

但是10.97.182.70:10086 还对应了c2这个app，代码里面每隔15s就会粗暴的remove不是我这个server对应的app，也就是10.97.182.70这台承接任务的server把c1这个app的key给干掉了。。也就拿不到worker信息，所以问题还是在换了主

为什么10.112.13.82:10086 认为 10.97.182.70:10086 down了，因为检测的方式是akka的ping类型，貌似是1s的阈值


```html

验证local to local的方法

curl -X POST -H "Content-Type: application/json" http://powerjob.xxx.xxx.com/appInfo/assert -d '{"appName": "xxx", "password": "xxx"}'

因为serverController的log会被exclude掉，所以用上面这个api进行测试，LZ的请求只在LZ的服务端被接收，XZ的请求只在XZ的服务端被接受

acquire的时候 假设按照作者所说，同一个app归属于一个server

https://cloud.tencent.com/developer/article/1824288

其实这段经历现在回过头来想特别搞笑，也有被自己蠢到。那无数个方案的失败原因其实都是同一个，也就是出发点错了。我一直在尝试让 worker 决定连接哪台 server，却一而再再而三忽略 worker 永远不可能获取 server 真正的存活信息（比如心跳无法传达，可能是 worker 本身的网络故障），因此 worker 不应该决定连接哪台 server，这应该由 server 来决定。worker 能做的，只有服务发现。想明白了这点，具体的方案也就应运而生了。

就像前面说的那样，worker 因为没办法获取 server 的准确状态，所以不能由 worker 来决定连接哪一台 server。因此，worker 需要做的，只是服务发现。即定时使用 HTTP 请求任意一台 server，请求获取当前该分组（appName）对应的 server。

而 server 收到来自 worker 的服务发现请求后，其实就是进行了一场小型的分布式选主：server 依赖的数据库中存在着 server_info 表，其中记录了每一个分组（appName）所对应的 server 信息。如果该 server 发现表中存在记录，那就说明该 worker 集群中已经有别的 worker 事先请求 server 进行选举，那么此时只需要发送 PING 请求检测该 server 是否存活。如果发现该 server 存活，那么直接返回该 server 的信息作为该分组的 server。否则就完成篡位，将自己的信息写入数据库表中，成为该分组的 server。

细心的小伙伴可能又要问了？发送 PING 请求检测该 server 是否存活，不还是有和刚才一样的问题吗？请求不同，发送方和接收方都有可能出问题，凭什么认为是原先的 server 挂了呢？

确实，在这个方案下，依旧没办法解决 server 到底挂没挂这个堪比“真假美猴王”的玄学问题。但是，这还重要吗？我们的目标是某个分组下所有的 worker 都连接到同一台 server，因此，即便产生那种误打误撞篡位的情况，在服务发现机制的加持下，整个集群最终还是会连接到同一台 server，完美实现我们的需求。另外，这里面如何判断Server是否可用？？？就是向这个Server中的Akka服务发送Ping信号，能够得到正确的响应，就说明这个Server是可用的。


解释下：

所以acquire带过来的currentServer是一个（server端只有一个currentSever，即主）

如果当时server是LZ的，那么LZ的请求过来会直接返回，没有选主，如果XZ的过来的话，看数据库里的（也是LZ的），akka端口没问题就返回数据库的，也是LZ的，akka有问题，分布式锁下，然后本机作为server，更新回worker的话currentServer也就变了，那么后续心跳上报的server也变了

akka里的serverPath是 akka://oms-server@10.96.178.40:10086/user/server_actor，心态中的workAddress是 XX.XX.XX.XX:27777

对于出现的问题，一开始主在LZ起的任务，原理就是server只会执行本及关联的app的任务，当时数据库中的server是LZ，那么推任务到时间轮的也是LZ，但是后面换主了，又被清理了worker，那么LZ就找不到worker了

```

SHAXZ机器上的问题 （2点多换主了，心跳都发给新server了）

2022-02-25 02:36:04.629 [omsTimingPool-64] INFO  c.g.k.p.s.s.t.s.OmsScheduleService - [JobScheduleService] cron schedule: 2.198 ms, workflow schedule: 4.829 ms, frequent schedule: 9.258 ms.
2022-02-25 02:36:04.732 [omsTimingPool-63] INFO  c.g.k.p.s.s.t.InstanceStatusCheckService - [InstanceStatusChecker] status check using 119.2 ms.
2022-02-25 02:36:14.727 [omsTimingPool-63] INFO  c.g.k.p.s.s.t.InstanceStatusCheckService - [InstanceStatusChecker] status check using 114.7 ms.
2022-02-25 02:36:14.959 [http-nio-8080-exec-13] WARN  c.g.k.p.s.s.h.ServerSelectService - [ServerSelectService] server(10.112.62.2:10086) was down.
2022-02-25 02:36:14.966 [http-nio-8080-exec-13] INFO  c.g.k.p.s.s.h.ServerSelectService - [ServerSelectService] this server(10.96.178.40:10086) become the new server for app(appId=2).
2022-02-25 02:36:15.433 [oms-server-akka.actor.default-dispatcher-224275] WARN  c.g.k.p.s.a.a.ServerTroubleshootingActor - [ServerTroubleshootingActor] receive DeadLetter: DeadLetter(AskResponse(success=true, data=[51], message=null),Actor[akka://oms-server@10.112.62.2:10086/user/friend_actor#1993468795],Actor[akka://oms-server/temp/$JhOd])
2022-02-25 02:36:19.616 [omsTimingPool-63] INFO  c.g.k.p.s.s.t.s.OmsScheduleService - [JobScheduleService] cron schedule: 1.893 ms, workflow schedule: 516.2 μs, frequent schedule: 894.2 μs.
2022-02-25 02:36:24.727 [omsTimingPool-63] INFO  c.g.k.p.s.s.t.InstanceStatusCheckService - [InstanceStatusChecker] status check using 114.8 ms.
2022-02-25 02:36:34.616 [omsTimingPool-64] INFO  c.g.k.p.s.s.t.s.OmsScheduleService - [JobScheduleService] cron schedule: 1.753 ms, workflow schedule: 436.6 μs, frequent schedule: 688.0 μs.
2022-02-25 02:36:34.729 [omsTimingPool-63] INFO  c.g.k.p.s.s.t.InstanceStatusCheckService - [InstanceStatusChecker] status check using 116.6 ms.
2022-02-25 02:36:44.726 [omsTimingPool-63] INFO  c.g.k.p.s.s.t.InstanceStatusCheckService - [InstanceStatusChecker] status check using 113.6 ms.
2022-02-25 02:36:49.616 [omsTimingPool-63] INFO  c.g.k.p.s.s.t.s.OmsScheduleService - [JobScheduleService] cron schedule: 1.664 ms, workflow schedule: 481.8 μs, frequent schedule: 766.7 μs.
2022-02-25 02:36:54.728 [omsTimingPool-63] INFO  c.g.k.p.s.s.t.InstanceStatusCheckService - [InstanceStatusChecker] status check using 115.5 ms.
2022-02-25 02:37:04.616 [omsTimingPool-64] INFO  c.g.k.p.s.s.t.s.OmsScheduleService - [JobScheduleService] cron schedule: 1.591 ms, workflow schedule: 429.2 μs, frequent schedule: 897.0 μs.
2022-02-25 02:37:04.729 [omsTimingPool-63] INFO  c.g.k.p.s.s.t.InstanceStatusCheckService - [InstanceStatusChecker] status check using 115.9 ms.
2022-02-25 02:37:14.728 [omsTimingPool-63] INFO  c.g.k.p.s.s.t.InstanceStatusCheckService - [InstanceStatusChecker] status check using 114.9 ms.
2022-02-25 02:37:19.616 [omsTimingPool-63] INFO  c.g.k.p.s.s.t.s.OmsScheduleService - [JobScheduleService] cron schedule: 1.789 ms, workflow schedule: 470.3 μs, frequent schedule: 700.9 μs.



SHALZ机器上的问题 （3点旧的server因为没有心跳发送过来了，导致最近激活的时间超过60s，将worker全部清除了，后面任务也法发了）（每天凌晨3点，omsTimingPool线程池中的线程会去触发timingClean，这个线程池在server端，做1、释放本地缓存 2、）

2022-02-25 03:00:00.000 [omsTimingPool-64] INFO  c.g.k.p.s.s.h.ClusterStatusHolder - [ClusterStatusHolder-monkey] clean the containerInfos, listDeployedContainer service may down about 1min~
2022-02-25 03:00:00.000 [omsTimingPool-64] INFO  c.g.k.p.s.s.h.ClusterStatusHolder - [ClusterStatusHolder-monkey] detective timeout workers([10.112.54.96:27777, 10.60.41.201:27777, 10.60.117.206:27777, 10.112.54.95:27777]), try to release their infos.
2022-02-25 03:00:00.001 [omsTimingPool-64] INFO  c.g.k.p.s.s.t.CleanService - [CleanService] won't clean up /home/deploy/powerjob-server/online_log/ because of offset day <= 0.
2022-02-25 03:00:00.001 [omsTimingPool-64] INFO  c.g.k.p.s.s.t.CleanService - [CleanService] won't clean up /home/deploy/powerjob-server/container/ because of offset day <= 0.
2022-02-25 03:00:00.001 [omsTimingPool-64] INFO  c.g.k.p.s.s.t.CleanService - [CleanService] won't clean up bucket(log) because of offset day <= 0.
2022-02-25 03:00:00.001 [omsTimingPool-64] INFO  c.g.k.p.s.s.t.CleanService - [CleanService] won't clean up bucket(container) because of offset day <= 0.
2022-02-25 03:00:08.163 [omsTimingPool-64] INFO  c.g.k.p.s.s.t.InstanceStatusCheckService - [InstanceStatusChecker] current server has no app's job to check


2022-02-25 04:00:13.985 [HashedWheelTimer-Executor-5] INFO  c.g.k.p.s.s.DispatchService - [Dispatcher-23|377306224615491328] start to dispatch job: JobInfoDO(id=23, jobName=廾@妾K任佊¡(任佊¡潊¶彀~A置为RUNNING)-pro, jobDescription=null
l, appId=2, jobParams=null, timeExpressionType=5, timeExpression=null, executeType=1, processorType=1, processorInfo=com.ctrip.ops.app.monkey.taskjob.powerjob.TaskStartProcessor, maxInstanceNum=50, concurrency=1, instanceTimeLimit=0, instanceRetryNum=0, taskRetryNum=0, status=1, nextTriggerTime=null, minCpuCores=0.0, minMemorySpace=0.0, minDiskSpace=0.0, designatedWorkers=, maxWorkerCount=0, notifyUserIds=null, gmtCreate=2021-01-20 17:17:21.835, gmtModified=2021-01-20 17:17:21.835);instancePrams: {"-1":"{\"taskId\":1999}"}.
2022-02-25 04:00:13.993 [HashedWheelTimer-Executor-5] WARN  c.g.k.p.s.s.DispatchService - [Dispatcher-23|377306224615491328] cancel dispatch job due to no worker available, clusterStatus is appName:monkey,clusterStatus:{}.
2022-02-25 04:00:14.001 [HashedWheelTimer-Executor-5] INFO  c.g.k.p.s.s.i.InstanceManager - [Instance-377306224615491328] process finished, final status is FAILED.
2022-02-25 04:00:14.007 [HashedWheelTimer-Executor-5] INFO  c.g.k.p.s.s.w.WorkflowInstanceManager - [Workflow-1|377049566677042944] node(jobId=23,instanceId=377306224615491328) finished in workflowInstance, status=FAILED,result=no worker available
2022-02-25 04:00:14.007 [HashedWheelTimer-Executor-5] WARN  c.g.k.p.s.s.w.WorkflowInstanceManager - [Workflow-1|377049566677042944] workflow instance process failed because middle task(instanceId=377306224615491328) failed

一个是LZ出问题，XY

关于连接的问题，应用层都会有自定义的心跳，tcp有自己的心跳

```html

// 注意看两边的端口，netstat信息是C/S同步的，双方都有

[root@xxxxxx1 ~]# netstat -anp | grep 27777
tcp        0      0 10.5.29.21:27777        0.0.0.0:*               LISTEN      -                   
tcp        0      0 10.5.29.21:27777        10.4.110.194:56880      ESTABLISHED -

[root@xxxxxx2 ~]# netstat -anp | grep 27777
tcp        0      0 10.4.110.194:56880      10.5.29.21:27777        ESTABLISHED -

[root@xxxxxx1 ~]# netstat -anp | grep 10086
tcp        0      0 10.5.29.21:40780        10.4.110.194:10086      ESTABLISHED -                   
tcp        0      0 10.5.29.21:40782        10.4.110.194:10086      ESTABLISHED -

[root@xxxxxx2 ~]# netstat -anp | grep 10086
tcp        0      0 10.4.110.194:10086      0.0.0.0:*               LISTEN      -                   
tcp        0      0 10.4.110.194:10086      10.5.29.21:40780        ESTABLISHED -                   
tcp        0      0 10.4.110.194:10086      10.5.29.21:40782        ESTABLISHED -

// 2022-08-05 18:55:29，过几天再来看看，变化嘛，短连接 连完后会开随机端口
// server端口当然可以被client多个端口连接了
// web服务短连接，数据库服务长连接

// 过了几天后，还是连的 40780 和 40782 端口
// 为什么worker会开两个端口 连server的10086，应该是心跳一个，日志一个，discoverServer是http的，不是akka的tell

[root@xxxxxx1 ~]# netstat -anp | grep 27777
tcp        0      0 10.5.29.21:27777        0.0.0.0:*               LISTEN      -                   
tcp        0      0 10.5.29.21:27777        10.4.110.194:45884      ESTABLISHED -

27777口的端口变了56880 -> 45884 -> 33652，server到27777口都是实际任务的交互

```

2、no server available的问题（PJ Instance的问题）

有流量的切换，之后开始出现no server available，其实就是 http 请求server有问题，报错 connect timeout，网络的问题，看下连接，到server的VIP，有不少是CLOSE-WAIT的状态，引起注意，因为有几个这样的状态 ss -4 state CLOSE-WAIT

重启下应用把，重置连接

后续还是会有报错，有点费解，然后发现其实有个应用层的bug，每两分钟会去各个环境去调用 HPA相关任务，这些任务在间隔2min的情况下同时并发runJob，那么就会有个connect timeout的问题，应用层做一定优化，减少runJob的实例！！！

#### chaosblade中卸载javaagent的功能

使用go语言去做了

另外有延迟的功能：异步的延迟

1、每种框架异步延迟的切点在哪

2、如何在请求的时候跳过enhancer

对于1，比如dubbo，底层使用netty，有调用链，最终通过netty.send去发送，响应的时候也在receive的地方，需要找到比较好的切入点

比如异步httpclient，底层是一个reactor模型，跟netty是类似的，也需要在上层找一个比较好的切入点

#### chaos相关的问题

1、数据库层面的切入点

是在mysql-connector驱动这边，在初始化连接的时候会有多次数据传输？（要确认，跟db之间的交互，肯定是应用层发起的），连接有了之后，每次固定n秒，一段时间后，又会有多次传输，时间会变大

```html

可以用arthas去调一下，挂两个agent，也是ok的，先走chaos的逻辑，再接到arthas的trace中，是能够看见trace的

[arthas@49]$ trace com.mysql.jdbc.MysqlIO sqlQueryDirect
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 1393 ms, listenerId: 1
`---ts=2023-10-19 16:42:16;thread_name=http-nio-8080-exec-14;id=62;is_daemon=true;priority=5;TCCL=org.apache.catalina.loader.ParallelWebappClassLoader@cf676ec
    `---[3005.48691ms] com.mysql.jdbc.MysqlIO:sqlQueryDirect()
        +---[0.00% 0.087657ms ] com.mysql.jdbc.MysqlIO:invokeStatementInterceptorsPre() #2580
        +---[0.00% 0.020428ms ] com.mysql.jdbc.MySQLConnection:getStatementComment() #2587
        +---[0.00% 0.029363ms ] com.mysql.jdbc.MySQLConnection:getIncludeThreadNamesAsStatementComment() #2589
        +---[0.03% 0.981993ms ] com.mysql.jdbc.MysqlIO:sendCommand() #2675
        +---[0.02% 0.661688ms ] com.mysql.jdbc.MysqlIO:readAllResults() #2725
        `---[0.00% 0.026265ms ] com.mysql.jdbc.MysqlIO:invokeStatementInterceptorsPost() #2774


而在有问题的机器上，看见了jacocoInit

`---ts=2023-10-19 16:56:08;thread_name=domain-executor-2;id=22c;is_daemon=false;priority=5;TCCL=org.apache.catalina.loader.WebappClassLoader@3045a112
    `---[0.848607ms] com.mysql.jdbc.MysqlIO:sqlQueryDirect()
        +---[1.38% 0.011737ms ] com.mysql.jdbc.MysqlIO:$jacocoInit()
        +---[1.39% 0.011776ms ] com.mysql.jdbc.MysqlIO:invokeStatementInterceptorsPre() #2580
        +---[1.25% 0.010573ms ] com.mysql.jdbc.MySQLConnection:getStatementComment() #2587
        +---[0.99% 0.008439ms ] com.mysql.jdbc.MySQLConnection:getIncludeThreadNamesAsStatementComment() #2589
        +---[0.99% 0.008374ms ] com.mysql.jdbc.Buffer:clear() #2611
        +---[0.97% 0.008253ms ] com.mysql.jdbc.Buffer:writeByte() #2614
        +---[2.34% 0.019821ms ] com.mysql.jdbc.StringUtils:startsWithIgnoreCaseAndWs() #2627
        +---[1.29% 0.010919ms ] com.mysql.jdbc.MySQLConnection:getServerCharset() #2630
        +---[0.98% 0.008325ms ] com.mysql.jdbc.MySQLConnection:parserKnowsUnicode() #2630
        +---[1.38% 0.011734ms ] com.mysql.jdbc.Buffer:writeStringNoNull() #2630
        +---[44.90% 0.381018ms ] com.mysql.jdbc.MysqlIO:sendCommand() #2675
        +---[15.28% 0.129654ms ] com.mysql.jdbc.MysqlIO:readAllResults() #2725
        `---[1.92% 0.016327ms ] com.mysql.jdbc.MysqlIO:invokeStatementInterceptorsPost() #2774

`---ts=2023-10-19 16:56:08;thread_name=domain-executor-2;id=22c;is_daemon=false;priority=5;TCCL=org.apache.catalina.loader.WebappClassLoader@3045a112
    `---[0.966727ms] com.mysql.jdbc.MysqlIO:sqlQueryDirect()
        +---[1.16% 0.011186ms ] com.mysql.jdbc.MysqlIO:$jacocoInit()
        +---[1.23% 0.011893ms ] com.mysql.jdbc.MysqlIO:invokeStatementInterceptorsPre() #2580
        +---[0.96% 0.009253ms ] com.mysql.jdbc.MySQLConnection:getStatementComment() #2587
        +---[1.00% 0.009701ms ] com.mysql.jdbc.MySQLConnection:getIncludeThreadNamesAsStatementComment() #2589
        +---[64.70% 0.625504ms ] com.mysql.jdbc.MysqlIO:sendCommand() #2675
        +---[14.14% 0.1367ms ] com.mysql.jdbc.MysqlIO:readAllResults() #2725
        `---[1.42% 0.01375ms ] com.mysql.jdbc.MysqlIO:invokeStatementInterceptorsPost() #2774

那就有点头绪了，有别的agent干扰了，查下

/opt/tomcat/bin/setenv.sh
/opt/tomcat/bin/extraenv.sh

所以就符合现象描述了，应用本地调试能走到切点，但是线上就没了，线上环境干扰（切点根本就没有进去，被jacoco干扰了），也和是否shard的db没有关系，最后总得走到驱动

把启动中的agent去掉后就恢复了，这边arthas帮了不少忙


```



#### sandbox的问题

接上面的javagent，javaagent是可以挂载多个的agent静态模式，当sandbox触发的动态attach触发的时候，rasp和已有的又会跑一遍？


也是接上面的问题：挂载失败的问题：

动态attach模式排障的一些规律：

开始对同一个实例反复注入，卸载，会导致MetaSpace的OOM，因为sandbox的原因，导致加载的类无法完全释放（这边日志中是有OOM日志的）

然后还有情况是当cpu特别高的时候也会影响挂载，之前以为是rasp等作用引起，触发了降级操作，可见当时cpu飙升

但有的时候应用重启过，监控状态下，cpu，mem等也基本正常，还是失败，后面发现两种静态挂载，一个rasp，一个其他sandbox-agent，都会在动态挂载的时候触发？？！！（需要再验证一下）

sandbox中会有多余日志，rasp中也有日志，是不是多余的静态挂载影响到，删除之前的一些agent后测试

#### SF问题

1、一个字串的list，如果是奇数删除，并保留最大的那个，要求在这个list中操作



#### 用tc阻断数据库（mongodb）

对于一个端口进行delay

```html

网络上的参考

tc qdisc add dev eth0 root handle 1: prio
tc qdisc add dev eth0 parent 1:3 handle 30: netem delay 5ms
tc filter add dev eth0 protocol ip parent 1:0 u32 match ip sport 34001 0xffff flowid 1:3

chaosblade中显示

tc qdisc show dev eth0

qdisc prio 1: root refcnt 2 bands 4 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
qdisc netem 40: parent 1:4 limit 1000 delay 200.0ms

tc filter show dev eth0

filter parent 1: protocol ip pref 4 u32 chain 0
filter parent 1: protocol ip pref 4 u32 chain 0 fh 800: ht divisor 1
filter parent 1: protocol ip pref 4 u32 chain 0 fh 800::800 order 2048 key ht 800 bkt 0 flowid 1:4 not_in_hw
  match 0000d747/0000ffff at 20


blade create cri network delay --image-version=1.5.0 --image-repo=XXX.XXX/chaosblade-tool --remote-port=55111 --time=200 --interface=eth0 --container-id=c04d31cf4fce // 这是目标容器的id

tc的命令被封装在chaos_tcnetwork中

build_tcnetwork: exec/bin/tcnetwork/tcnetwork.go
	$(GO) build $(GO_FLAGS) -o $(BUILD_TARGET_BIN)/chaos_tcnetwork $<

所以看下tcnetwork.go这个文件，如果就是interface，就是简单的delay，loss等等，如果带上port啥的，就先创建qdisc

tc qdisc add dev eth0 root handle 1: prio bands 4  （handle是句柄，bands波段，还是看这个把https://cloud.tencent.com/developer/article/1409664，PRIO，QDisc不能限制带宽，因为属于不同类别的数据包是顺序离队的。使用PRIO QDisc可以很容易对流量进行优先级管理，只有属于高优先级类别的数据包全部发送完毕，才会发送属于低优先级类别的数据包。为了方便管理，需要使用iptables或者ipchains处理数据包的服务类型(Type Of Service,ToS)）


// Add class rule to 1,2,3 band, exclude port and exclude ip are added to 4 band

func buildNetemToDefaultBandsArgs(netInterface, classRule string) string {
	args := fmt.Sprintf(
		`qdisc add dev %s parent 1:1 %s && \
			tc qdisc add dev %s parent 1:2 %s && \
			tc qdisc add dev %s parent 1:3 %s && \
			tc qdisc add dev %s parent 1:4 handle 40: prio`,
		netInterface, classRule, netInterface, classRule, netInterface, classRule, netInterface)
	return args
}

func buildExcludeFilterToNewBand(netInterface string, excludePorts []string, excludeIp string) string {
	var args string
	excludeIpRules := getIpRules(excludeIp)
	for _, rule := range excludeIpRules {
		args = fmt.Sprintf(
			`%s && \
			tc filter add dev %s parent 1: prio 4 protocol ip u32 %s flowid 1:4`,
			args, netInterface, rule)
	}

	for _, port := range excludePorts {
		if strings.TrimSpace(port) == "" {
			continue
		}
		args = fmt.Sprintf(
			`%s && \
			tc filter add dev %s parent 1: prio 4 protocol ip u32 match ip dport %s 0xffff flowid 1:4 && \,
			tc filter add dev %s parent 1: prio 4 protocol ip u32 match ip sport %s 0xffff flowid 1:4`,
			args, netInterface, port, netInterface, port)
	}
	return args
}


tc qdisc add dev eth0 parent 1:1 netem delay 20ms && tc qdisc add dev eth0 parent 1:2 netem delay 20ms && tc qdisc add dev eth0 parent 1:3 netem delay 20ms
&& tc qdisc add dev eth0 parent 1:4 handle 40: prio

```

发现超时会被放大，因为是在建立连接后，传输数据的时候，那就要抓包了，tcpdump，然后在wireshark中看，也可以直接tcpdump中看一些ascii码的东西

```html
比如set session transaction的交互
比如set autocommit=0的交互
比如query的交互

每一次交互都会被delay

这边可以借助redis的ping-pong传输来看，设置了redis的ip和port的delay是2000ms，就是2s，那么看下ping-pong的情况

10:27:51秒发送的ping，过了2s才抓到包

10:27:53.202437 IP (tos 0x10, ttl 64, id 3625, offset 0, flags [DF], proto TCP (6), length 58)
    10.128.191.182.50870 > 10.4.73.209.6379: Flags [P.], cksum 0x1e38 (incorrect -> 0x51d4), seq 17955598:17955604, ack 2277199122, win 229, options [nop,nop,TS val 520050858 ecr 3484885535], length 6: RESP "ping"
10:27:53.202636 IP (tos 0x0, ttl 58, id 22188, offset 0, flags [DF], proto TCP (6), length 52)
    10.4.73.209.6379 > 10.128.191.182.50870: Flags [.], cksum 0x06a6 (correct), seq 2277199122, ack 17955604, win 510, options [nop,nop,TS val 3484964886 ecr 520050858], length 0
10:27:53.202654 IP (tos 0x0, ttl 58, id 22189, offset 0, flags [DF], proto TCP (6), length 59)
    10.4.73.209.6379 > 10.128.191.182.50870: Flags [P.], cksum 0x3aeb (correct), seq 2277199122:2277199129, ack 17955604, win 510, options [nop,nop,TS val 3484964886 ecr 520050858], length 7: RESP "PONG"
10:27:55.202715 IP (tos 0x10, ttl 64, id 3626, offset 0, flags [DF], proto TCP (6), length 52)
    10.128.191.182.50870 > 10.4.73.209.6379: Flags [.], cksum 0x1e32 (incorrect -> 0xffe7), seq 17955604, ack 2277199129, win 229, options [nop,nop,TS val 520052858 ecr 3484964886], length 0


看回包很快没有问题，最后一次的ack，也过了2s

那其实tc的作用效果是对【出包】产生的的影响！！试下其他redis的命令，比如mget啥的，跟ping-pong是一样的逻辑，只是发的命令不一样


再看下telnet的例子，telnet会去进行三次握手，注意，第一次请求，不论什么协议，第一次出包（SYN或者PSH），也是会被delay的，只是抓包看不太出

10:36:40.940982 IP (tos 0x10, ttl 64, id 55133, offset 0, flags [DF], proto TCP (6), length 60)
    10.128.191.182.50876 > 10.4.73.209.6379: Flags [S], cksum 0x1e3a (incorrect -> 0x254b), seq 2085400953, win 29200, options [mss 1460,sackOK,TS val 520578596 ecr 0,nop,wscale 7], length 0
10:36:40.941162 IP (tos 0x0, ttl 58, id 0, offset 0, flags [DF], proto TCP (6), length 60)
    10.4.73.209.6379 > 10.128.191.182.50876: Flags [S.], cksum 0x187f (correct), seq 4159066889, ack 2085400954, win 65160, options [mss 1460,sackOK,TS val 3485492625 ecr 520578596,nop,wscale 7], length 0
10:36:41.942332 IP (tos 0x10, ttl 64, id 55134, offset 0, flags [DF], proto TCP (6), length 60)
    10.128.191.182.50876 > 10.4.73.209.6379: Flags [S], cksum 0x1e3a (incorrect -> 0x2161), seq 2085400953, win 29200, options [mss 1460,sackOK,TS val 520579598 ecr 0,nop,wscale 7], length 0
10:36:41.942503 IP (tos 0x0, ttl 58, id 0, offset 0, flags [DF], proto TCP (6), length 60)
    10.4.73.209.6379 > 10.128.191.182.50876: Flags [S.], cksum 0x1495 (correct), seq 4159066889, ack 2085400954, win 65160, options [mss 1460,sackOK,TS val 3485493627 ecr 520578596,nop,wscale 7], length 0
10:36:42.941223 IP (tos 0x10, ttl 64, id 55135, offset 0, flags [DF], proto TCP (6), length 52)
    10.128.191.182.50876 > 10.4.73.209.6379: Flags [.], cksum 0x1e32 (incorrect -> 0x3d1f), seq 2085400954, ack 4159066890, win 229, options [nop,nop,TS val 520580596 ecr 3485492625], length 0
10:36:43.942561 IP (tos 0x10, ttl 64, id 55136, offset 0, flags [DF], proto TCP (6), length 52)
    10.128.191.182.50876 > 10.4.73.209.6379: Flags [.], cksum 0x1e32 (incorrect -> 0x3935), seq 2085400954, ack 4159066890, win 229, options [nop,nop,TS val 520581598 ecr 3485492625], length 0


C->S，S->C，C->S

第一次的发送SYNC就被延迟了2s，然后最后的ACK也被延迟了2s，也符合之前的验证，延迟了2s，感觉SYN被重传了？？！！

如果2s的延迟，那么看上面第一的重传时间是+1s，发生了，所以在某个节点确实触发了重传，如果延迟更大点，是不是触发的重传会更多，调整成5000ms

14:05:34.027817 IP (tos 0x10, ttl 64, id 16604, offset 0, flags [DF], proto TCP (6), length 60)
    10.128.191.182.51044 > 10.4.73.209.6379: Flags [S], cksum 0x1e3a (incorrect -> 0x9cf7), seq 2884884247, win 29200, options [mss 1460,sackOK,TS val 533108683 ecr 0,nop,wscale 7], length 0
14:05:34.028021 IP (tos 0x0, ttl 58, id 0, offset 0, flags [DF], proto TCP (6), length 60)
    10.4.73.209.6379 > 10.128.191.182.51044: Flags [S.], cksum 0xb2e0 (correct), seq 826980572, ack 2884884248, win 65160, options [mss 1460,sackOK,TS val 3498025702 ecr 533108683,nop,wscale 7], length 0
14:05:35.028381 IP (tos 0x10, ttl 64, id 16605, offset 0, flags [DF], proto TCP (6), length 60)
    10.128.191.182.51044 > 10.4.73.209.6379: Flags [S], cksum 0x1e3a (incorrect -> 0x990e), seq 2884884247, win 29200, options [mss 1460,sackOK,TS val 533109684 ecr 0,nop,wscale 7], length 0
14:05:35.028538 IP (tos 0x0, ttl 58, id 0, offset 0, flags [DF], proto TCP (6), length 60)
    10.4.73.209.6379 > 10.128.191.182.51044: Flags [S.], cksum 0xaef7 (correct), seq 826980572, ack 2884884248, win 65160, options [mss 1460,sackOK,TS val 3498026703 ecr 533108683,nop,wscale 7], length 0
14:05:36.085083 IP (tos 0x0, ttl 58, id 0, offset 0, flags [DF], proto TCP (6), length 60)
    10.4.73.209.6379 > 10.128.191.182.51044: Flags [S.], cksum 0xaad7 (correct), seq 826980572, ack 2884884248, win 65160, options [mss 1460,sackOK,TS val 3498027759 ecr 533108683,nop,wscale 7], length 0
14:05:37.032391 IP (tos 0x10, ttl 64, id 16606, offset 0, flags [DF], proto TCP (6), length 60)
    10.128.191.182.51044 > 10.4.73.209.6379: Flags [S], cksum 0x1e3a (incorrect -> 0x913a), seq 2884884247, win 29200, options [mss 1460,sackOK,TS val 533111688 ecr 0,nop,wscale 7], length 0
14:05:37.032566 IP (tos 0x0, ttl 58, id 0, offset 0, flags [DF], proto TCP (6), length 60)
    10.4.73.209.6379 > 10.128.191.182.51044: Flags [S.], cksum 0xa723 (correct), seq 826980572, ack 2884884248, win 65160, options [mss 1460,sackOK,TS val 3498028707 ecr 533108683,nop,wscale 7], length 0
14:05:39.028081 IP (tos 0x10, ttl 64, id 16607, offset 0, flags [DF], proto TCP (6), length 52)
    10.128.191.182.51044 > 10.4.73.209.6379: Flags [.], cksum 0x1e32 (incorrect -> 0xcbc8), seq 2884884248, ack 826980573, win 229, options [nop,nop,TS val 533113683 ecr 3498025702], length 0
14:05:40.028607 IP (tos 0x10, ttl 64, id 16608, offset 0, flags [DF], proto TCP (6), length 52)
    10.128.191.182.51044 > 10.4.73.209.6379: Flags [.], cksum 0x1e32 (incorrect -> 0xc7df), seq 2884884248, ack 826980573, win 229, options [nop,nop,TS val 533114684 ecr 3498025702], length 0
14:05:41.085175 IP (tos 0x10, ttl 64, id 16609, offset 0, flags [DF], proto TCP (6), length 52)
    10.128.191.182.51044 > 10.4.73.209.6379: Flags [.], cksum 0x1e32 (incorrect -> 0xc3bf), seq 2884884248, ack 826980573, win 229, options [nop,nop,TS val 533115740 ecr 3498025702], length 0
14:05:42.032633 IP (tos 0x10, ttl 64, id 16610, offset 0, flags [DF], proto TCP (6), length 52)
    10.128.191.182.51044 > 10.4.73.209.6379: Flags [.], cksum 0x1e32 (incorrect -> 0xc00b), seq 2884884248, ack 826980573, win 229, options [nop,nop,TS val 533116688 ecr 3498025702], length 0


又多出了一次SYNC，说明还是结合了下面的重传机制的

```

顺带就说下重传

```html

看内核的参数

/usr/sbin/sysctl -a

// 跟tcp相关的
/usr/sbin/sysctl -a | grep ipv4.tcp_syn

net.ipv4.tcp_syn_retries = 6   // sync得不到响应重试的次数，RTO时间是递增往上的，表示当没有收到服务器端的SYN+ACK包时，客户端重发SYN握手包的次数
net.ipv4.tcp_synack_retries = 5   // 表示回应第二个握手包（SYN+ACK包）给客户端IP后，如果收不到第三次握手包（ACK包），进行重试的次数（默认为5）
net.ipv4.tcp_syncookies = 1    // 防止 sync_flood攻击，dos（拒绝服务型） 和 ddos（分布式拒绝服务型）


net.ipv4.tcp_syn_retries = 6 看下抓包，time telnet 10.10.10.10 3000（随便看个没有的ip+port）

13:42:31.810899 IP (tos 0x10, ttl 64, id 9810, offset 0, flags [DF], proto TCP (6), length 60)
    10.128.191.182.44480 > 10.10.10.10.30000: Flags [S], cksum 0xde78 (incorrect -> 0x9040), seq 3730813528, win 29200, options [mss 1460,sackOK,TS val 531731466 ecr 0,nop,wscale 7], length 0
13:42:32.812314 IP (tos 0x10, ttl 64, id 9811, offset 0, flags [DF], proto TCP (6), length 60)
    10.128.191.182.44480 > 10.10.10.10.30000: Flags [S], cksum 0xde78 (incorrect -> 0x8c56), seq 3730813528, win 29200, options [mss 1460,sackOK,TS val 531732468 ecr 0,nop,wscale 7], length 0
13:42:34.816317 IP (tos 0x10, ttl 64, id 9812, offset 0, flags [DF], proto TCP (6), length 60)
    10.128.191.182.44480 > 10.10.10.10.30000: Flags [S], cksum 0xde78 (incorrect -> 0x8482), seq 3730813528, win 29200, options [mss 1460,sackOK,TS val 531734472 ecr 0,nop,wscale 7], length 0
13:42:38.824304 IP (tos 0x10, ttl 64, id 9813, offset 0, flags [DF], proto TCP (6), length 60)
    10.128.191.182.44480 > 10.10.10.10.30000: Flags [S], cksum 0xde78 (incorrect -> 0x74da), seq 3730813528, win 29200, options [mss 1460,sackOK,TS val 531738480 ecr 0,nop,wscale 7], length 0
13:42:46.840315 IP (tos 0x10, ttl 64, id 9814, offset 0, flags [DF], proto TCP (6), length 60)
    10.128.191.182.44480 > 10.10.10.10.30000: Flags [S], cksum 0xde78 (incorrect -> 0x558a), seq 3730813528, win 29200, options [mss 1460,sackOK,TS val 531746496 ecr 0,nop,wscale 7], length 0
13:43:02.872302 IP (tos 0x10, ttl 64, id 9815, offset 0, flags [DF], proto TCP (6), length 60)
    10.128.191.182.44480 > 10.10.10.10.30000: Flags [S], cksum 0xde78 (incorrect -> 0x16ea), seq 3730813528, win 29200, options [mss 1460,sackOK,TS val 531762528 ecr 0,nop,wscale 7], length 0
13:43:34.904321 IP (tos 0x10, ttl 64, id 9816, offset 0, flags [DF], proto TCP (6), length 60)
    10.128.191.182.44480 > 10.10.10.10.30000: Flags [S], cksum 0xde78 (incorrect -> 0x99c9), seq 3730813528, win 29200, options [mss 1460,sackOK,TS val 531794560 ecr 0,nop,wscale 7], length 0

一共7次SYNC，其中6次是重试。每次的间隔是2的n-1次方，相当于上一次增加，最后time是2min7s，也就是127秒（第六次在1min后就发出了）

个人理解是就行等待64秒（下一次的间隔），发的时候发现到达阈值，就取消了

```


wiershark可以搜索SYN，等flags，tcp.flags.SYN == 1 这样，就是过滤是否建连的，然后tcp.stream eq 0 等等可以看指定 ip:port 到 ip:port的交互


另外补充 loss 100%的时候，表现的效果是

```html

{"timestamp":"2023-12-14T07:41:20.747+0000","status":500,"error":"Internal Server Error","message":"com.xxx.platform.xxx.exceptions.XxxException: Can not get connection from DB chaos-0-master-10.139.185.25","path":"/api/chaos/test1"}

看着是直接连接拿不到了（有db的中间件）


```

另外，正常的tc注入比如

```html

tc qdisc add dev eth0 root netem loss100% 或者 delay 100ms 的时候

这边对于delay来讲，也是【出包】的影响，linux对出包控制的好，进包不好控制

那么loss 100% 的时候，尼玛 其实也是对于【出包】的！！！！！，利用

nohup tcpdump -nn -vvv -i eth0 host 10.10.10.10 > data & 后台执行

从远端ping机器，尼玛，是有入包的，loss掉了出包

15:57:33.769246 IP (tos 0x0, ttl 59, id 47033, offset 0, flags [DF], proto ICMP (1), length 84)
    10.6.57.58 > 10.4.128.166: ICMP echo request, id 1015, seq 2, length 64
15:57:34.793259 IP (tos 0x0, ttl 59, id 47528, offset 0, flags [DF], proto ICMP (1), length 84)
    10.6.57.58 > 10.4.128.166: ICMP echo request, id 1015, seq 3, length 64
15:57:35.816241 IP (tos 0x0, ttl 59, id 47923, offset 0, flags [DF], proto ICMP (1), length 84)
    10.6.57.58 > 10.4.128.166: ICMP echo request, id 1015, seq 4, length 64

抓包是只有单向的，反向没有的

```

#### jpa的问题

jpa中某张表的值被莫名的更新了，只是用了下set，查出来的其实就是托管状态了（这部份之前还真没注意过）

```html

public void saveTest5() {

    // 自动提交机制，
    // 如果数据库中有数据，就是托管状态，会被自动提交到数据库，不想这样，就可以进行bean之间的转换set，也就是PO转成VO，再update，而不是直接update这个PO
    DataEntity dataEntity = dataRepository.findByKey("321");
    dataEntity.setType("test11");


    // 还是要进行save的，实体对象处于托管状态，会被自动更新到数据表中
    IssueEntity issueEntity = issueRepository.findByKey("654");
    issueEntity.setType("test00");
    issueRepository.save(issueEntity);

}

// 参考一些save时候的坑

https://www.modb.pro/db/51252
https://www.modb.pro/db/51253
https://www.modb.pro/db/51254
https://codeantenna.com/a/bcoH277bsB

// delete是不是也有这个问题

```


#### 高可用切换的问题

1、keepalived是做主备的，VIP会绑定到某一个实例上，数据库连接，一个VIP就是一台实例，master实例，因为有状态的，其实是不做负载均衡的，keepalived只负责切换到备

这其中 一个vip只能绑定到一个实例，而bgp宣告出来的，一个vip可以对应多个ip

vip可以用来做域名解析，python脚本连接域名用的，比如dig xxx.mysql.db.fat.qa.nt.xxx.com出来的是VIP 10.6.53.128

而接了框架的db，那么连接实际是10.6.53.116(master)/10.6.53.117(slave)

#### equal和hashcode的一个例子

两个obj，如果equals()相等，hashCode()一定相等。equals()相等的两个对象，hashcode()一定相等。

两个obj，如果hashCode()相等，equals()不一定相等（Hash散列值有冲突的情况，虽然概率很低）。hashcode()不等，一定能推出equals()也不等。

改写equals时总是要改写hashcode

比如改写一个类的equals方法，让它去比较内容而不是引用是否相同，又不去改写hashcode方法，那么就会出现一个问题，就是我new两个内容一样的对象，但是它们的地址是不同的，equals方法得到true，但是hashcode方法依赖与地址可能就会得到false，这显然不行，所以我们必须改写hashcode方法使他根据内容判断（equals相同，hashcode必须相同）。


一个例子，当用某个Object作为Key的时候，做分类

```html

public class TestActionFlag extends ActionFlag {

  private String classRule;
  private List<String> destinationIp;
  private List<String> remotePort;
  private Boolean idcSpec;

  public String getClassRule() {
    return classRule;
  }

  public void setClassRule(String classRule) {
    this.classRule = classRule;
  }

  public List<String> getDestinationIp() {
    return destinationIp;
  }

  public void setDestinationIp(List<String> destinationIp) {
    this.destinationIp = destinationIp;
  }

  public List<String> getRemotePort() {
    return remotePort;
  }

  public void setRemotePort(List<String> remotePort) {
    this.remotePort = remotePort;
  }

  public Boolean getIdcSpec() {
    return idcSpec;
  }

  public void setIdcSpec(Boolean idcSpec) {
    this.idcSpec = idcSpec;
  }

  @Override
  public boolean equals(Object obj) {
    if (this == obj) {
      return true;
    }

    if (obj == null) {
      return false;
    }

    if (!(obj instanceof TestActionFlag)) {
      return false;
    }

    TestActionFlag other = (TestActionFlag) obj;
    return Objects.equals(this.classRule, other.classRule) && Objects.equals(this.destinationIp, other.destinationIp)
        && Objects.equals(this.remotePort, other.remotePort) && Objects.equals(this.idcSpec, other.idcSpec);
  }

  @Override
  public int hashCode() {
    int result = 1;
    result = 31 * result + (this.classRule == null ? 0 : this.classRule.hashCode());
    result = 31 * result + (this.destinationIp == null ? 0 : this.destinationIp.hashCode());
    result = 31 * result + (this.remotePort == null ? 0 : this.remotePort.hashCode());
    result = 31 * result + (this.idcSpec == null ? 0 : this.idcSpec.hashCode());
    return result;
  }
}

```

要重写下equal和hashcode才能完整实现其作为Key的价值，如果不重写，就不能通过不同的参数去做不同的key，因为后面用了 TestActionFlag 去做groupBy的key


#### 设计模式



#### 扩容问题

一个Web服务，容器类型的，单机cpu利用率150%~200%（cpu util per core 每20s统计一次，一般稳定在27-60%），在时间段内有大量请求，峰值达到了每分钟 19470 次请求 / min，325qps，机器是8C / 24GB / 40GB，分析requestUri，分析源IP

cpu的throttled_time从0min飙升到10-12min（每20s统计一次），纯粹是CPU被榨干了，JVM相关参数没有发生太大的变化

源头上控制，同时扩容，扩容一台机器还是报错，如果是就近访问，发现XZ和LZ的访问量是不一样的，XZ才有尖峰，说明client在XZ部署多，手动先调整为 5 : 5

cpu.util_percent 除以分配的核心数量. 正常情况下 cpu.util_percent.per_core这个值应该在0~100%的区间内. 但是k8s允许需让要更多CPU资源的容器抢占一部分闲置的CPU用来紧急情况下burst. 目前允许最大burst到两倍的申请的CPU. 所以在图上会看到超过100%的现象. 一方面是整体平台集群的CPU利用率低, 另一方面也可以缓解throttled的问题. 当这个值接近或者超过100%的时候, 应用应该排查性能问题或者考虑扩容。


docker容器下所有的task被限制的总时间。我们知道在Docker中对CPU的限制方式有几种，可以通过--cpu-shares，--cpu-period和--cpu-quota，--cpuset-cpus来配置，具体细节这里不赘述。现在使用最多的方式是--cpu-period和--cpu-quota结合的方式，这时候CPU使用率的上限由两者共同决定，比如说A容器配置的--cpu-period=100000 --cpu-quota=50000，那么A容器就可以最多使用50%个CPU资源，如果配置的--cpu-quota=200000，那就可以使用200%个CPU资源。所有对采集到的CPU used的绝对值没有意义，还需要参考上限。还是这个例子--cpu-period=100000 --cpu-quota=50000，如果容器试图在0.1秒内使用超过0.05秒，则throttled就会触发，所有throttled的count和time是衡量CPU是否达到瓶颈的最直观指标。


另外补充：

上面Pod Spec 里的 Request CPU和Limit CPU的值，最后会通过 CPU Cgroup 的配置，来实现控制容器CPU 资源的作用。
CPU Cgroup 中三个重要的参数：

cpu.cfs_quota_us
cpu.cfs_period_us
cpu.shares

https://blog.csdn.net/u012271526/article/details/118094324?spm=1001.2014.3001.5502

https://blog.csdn.net/u012271526/article/details/118462257

当用户程序开始运行时，对应着第一个 ‘us’ 框，‘us’ 是 user 的缩写，代表 Linux 的用户态 CPU Usage，普通用户程序代码中，只要不是调用系统调用（System Call），这些代码指令消耗的CPU 都属于‘us’。

当这个用户程序代码中调用了系统调用，比如说 read（）去读取一个文件，这个时候用户进程就会从用户态切换到内核态。

内核态read（）系统调用在读到真正 disk 上的文件前，就会进行一些文件系统层的操作，那么这些代码指令的消耗就属于‘sy’，这里就对应上面图中的第二个框 ’sy‘ 是 system的缩写，代表内核态CPU 使用。

然后 read()系统调用会向Linux 的Block Layer 发出一个 I/O Request，触发一个真正的磁盘读取操作。

这个时候进程一般会被置为 TASK_UNINTERRUPTIBLE，Linux会把这段时间标示为 wa，对应图中第三个框，wa 是 iowait 的缩写，代表等待Disk I/O 的时间。

当磁盘返回数据时，进程在内核态拿到数据，这里仍然是内核态的CPU 使用的"sy"，也就是图中的第四个框。

然后进程再从内核态切换回用户态，在内核态得到文件数据，这里进程又回到用户态的使用 us，对应图中的第五个框。

假设到这，进程就没事做了，CPU上也没有其他的进程需要运行。 系统就会进入到 id 这个步骤，也就是第六个框，id是 idle 的缩写。 代表系统处于空闲状态。

--

如果这个时候机器的网络收到一个网络数据包，网卡就会发出一个中断（interrupt），响应的，CPU 会中断响应，然后进入中断服务程序。

这时 CPU 就会进入hi ，hi 是 hardware irq 的缩写，代表 CPU 处理硬中断的开销，由于中断服务处理需要关闭中断，所以这个硬中断不能时间太长。

中断后的工作必须要完成的，如果这些工作比较耗时怎么办呢？ Linux 中有一个软中断的概念（softirq） ，它可以完成这些耗时比较长的工作。

软中断：从网卡收到数据包的大部分工作，都是通过软中断来处理的。
CPU 会进入到第八个框，si， 这里的si 是softirq的缩写，代表CPU 处理软中断的开销。

hi，si 本身处理的时候不属于任何一个进程，它们的CPU 时间都不会进入到进程的CPU 时间。

ni 是 nice 的缩写，
PRI ：进程优先权，代表这个进程可被执行的优先级，其值越小，优先级就越高，越早被执行;
NI ：进程Nice值，代表这个进程的优先值;
%nice ：改变过优先级的进程的占用CPU的百分比。

PRI是比较好理解的，即进程的优先级，或者通俗点说就是程序被CPU执行的先后顺序，此值越小进程的优先级别越高。那NI呢？就是我们所要说的nice值了，其表示进程可被执行的优先级的修正数值。如前面所说，PRI值越小越快被执行，那么加入nice值后，将会使得PRI变为：PRI(new)=PRI(old)+nice。由此看出，PR是根据NICE排序的，规则是NICE越小PR越前（小，优先权更大），即其优先级会变高，则其越快被执行。如果NICE相同则进程uid是root的优先权更大。

另外一个是st，st 是steal的缩写。是虚拟机里用的一个CPU 使用类型，表示有多少时间是被同一个宿主机上的其他虚拟机抢走的。

us      User，用户态CPU时间，不包括低优先级进程的用户态时间
sys     Syetem，内核态CPU时间
ni      Nice，nice值1-19的进程用户态CPU时间
id      Idle，系统空闲CPU时间
wa      Iowait，系统等待I/O的CPU时间，这个时间不计入进程CPU时间
hi      Hardware irq，处理硬中断的时间，这个时间不计入进程CPU时间
si      Softirq，处理软中断的时间，这个时间不计入进程CPU时间
st      Steal，表示同一个宿主机上的其他虚拟机抢走的CPU时间

Cgroups 是对指定进程做计算机资源限制的，CPU Cgroup 是Cgroups 其中的一个 Cgroups 子系统，它是用来限制进程的CPU 使用的。

cpu包括：

用户态：us和ni
内核态：sy

wa，hi，si 这些 I/O 或者中断相关的CPU 使用，CPU Cgroup 不会去做限制。

每个Cgroups 子系统都是通过一个虚拟文件系统挂载点的方式，挂到一个缺省的目录下，CPU Cgroup 一般在Linux 发行版里会放在 /sys/fs/cfroup/cpu 这个目录下。

在这个子系统的目录下，每个控制组（Control Group）都是一个子目录，各个控制组之间的关系就是一个树状的层级关系（hierarchy

cpu.cfs_quota_us 和 cpu.cfs_period_us 这两个值决定了每个控制组中所有进程的可使用CPU资源的最大值
cpu.shares 决定了CPU Cgroup 子系统下控制组可用CPU的相对比例。只有当系统上CPU完全被占满的时候，这个比例才会在各个控制组间起作用。


Limit CPU 是容器可用的上限值：
容器CPU的上限值是由 cpu.cfs_quota_us 除以cpu.cfs_period_us 得出的值决定的 操作系统中 cpu.cfs_period_us 的值一般是个固定值，k8s不会去修改它，所以只修改 cpu.cfs_quota_us

Request CPU ：整个节点CPU都占满的情况，容器可以保证的需要的CPU数目。
是靠 cpu.shares这个参数 在CPU Cgroup中cpu.shares == 1024 表示1个CPU的比例，那么request CPU的值就是n，给cpu.shares的赋值对应就是n*1024


重点总结！！！

每个进程的CPU Usage 只包含了用户态(us或ni)和内核态（sy）两部分，其他的系统CPU开销并不包含在进程的CPU使用中，而CPU Cgroup 只是对进程的CPU 使用做了限制

CPU Cgroup中的三个参数：
cpu.cfs_quota_us（一个调度周期里这个控制组被允许的运行时间） 除以 cpu.cfs_period_us （调度周期）得到的值决定了 CPU Cgroup 每个控制组中CPU 使用的上限值。

cpu.shares 决定了CPU Cgroup 子系统下控制组可用CPU的相对比例，当系统上CPU 完全被占满的时候，这个比例才会在各个控制组间起效。

Limit CPU 就是容器所在Cgroup 控制组中的CPU 上限值，Request CPU 的值就是控制组中的 cpu.shares的值。

#### DNS

内网DNS的TTL是5s

外网DNS的TTL是15min，可能更短

这边融合下长短连接：

短连接：请求完就关连接，TIME_WAIT后等2MSL的时间，在linux系统中，是下面这个时间，等60s后连接结束

查看默认的MSL值(60s)：

[root@DanCentOS65var]# cat /proc/sys/net/ipv4/tcp_fin_timeout

60

修改默认60为120：

[root@DanCentOS65var]# echo 120 > /proc/sys/net/ipv4/tcp_fin_timeout

修改完成后，重新加载配置文件：

[root@DanCentOS65var]# sysctl -p /etc/sysctl.conf

查看是否已经生效：

[root@DanCentOS65var]# sysctl -a | grep fin

net.ipv4.tcp_fin_timeout= 120

长连接：如果用curl的话能curl出长连接嘛。带着 -H "Connection: keep-alive"，搞不出来的

keepalive只是告诉服务器本次连接是长连接，在连接超时（一般服务器可以配置该参数，类似于Kepalive.timeout的参数）前服务器不会主动关闭该连接，也就是说客户端可以重用该连接。

你不明白的地方应该是如果第二次发送消息，clientA没有重新connect的话，使用的还是一个socket fd(1)，否则就是建立新的socket fd(2)了。也就是说，是否重用keepalive的tcp连接主动权在client端。你可以自己通过抓包(tcpdump等）看下重连与不重连时的发送情况。

下面简单地使用curl来做个试验就知道了。

1.连续两次执行两条curl命令，很明显每次都会建立连接    // 抓包完全两次独立，还有断开连接

curl -v -o /dev/null http://www.baidu.com

curl -v -o /dev/null http://www.baidu.com

通过抓包也会发现三次握手每次都会发生，即连接建立了两次。

2.一次性请求两次同一个服务器的资源    //  抓包确实只有一次连接，[S] [S.] [.]

curl -v -o /dev/null http://www.baidu.com -o /dev/null http://www.baidu.com

抓包会发现只有一次三次握手，即连接建立了1次。另外curl也会给出重用连接的提示
Re-using existing connection! (#0) with host www.baidu.com

curl执行完，肯定是要断掉连接的

#### 数据层的一些问题

抛错的问题：

DataIntegrityViolationException 是 DuplicateKeyException的父类

编译器允许catch时候子类在前，父类在后，但貌似不允许父类在前，子类在后

这个DataIntegrityViolationException涵盖饿很多东西，都是约束，比如not null，这个优先级？？！！

1、有唯一键，not null，先抛出not null，补全not null后抛duplicate
2、Spring所有完整性的违反都是DataIntegrityViolationException，DuplicateKeyException并不起到什么作用（无法捕获）
3、data too long 也会被DataIntegrityViolationException捕获，优先于duplicate

catch (DataIntegrityViolationException e) {

  if (e.getRootCause().getMessage().contains("Duplicate entry")) {
      logger.error("duplicated entry , mark as success.");
  } else {
      throw e;
  }

}

捕获时候还是根据message看下把

脏读的问题


#### 一次cpu，内存都爆

问题：出现CPU异常告警，且每个实例轮番告警

现象：CPU打满，内存打满，请求进不来，健康检测flapping。。挂了，尝试水平扩容，垂直扩容，重启，迁移宿主机，看日志没有发现问题

解决：

```html

1、ps -mp 36 -o THREAD,tid,time  // 找到占用CPU最高的tid
2、printf "%x\n" 943  // tid转换成16进制
3、jstack 36 | grep 3af -A60 // 输出对应的堆栈信息，需要发布应用的账户（deploy）


1、程序启动前加载缓存数据

2、代码在不停的运行这一行，吃掉CPU，也吃掉内存

3、整个实例挂掉

if (sourceXXXMap.containsKey(key)) {
//        List<XXXDependency> oldList = sourceXXXMap.get(key);
//        List<String> existOp = oldList.stream().map(CdbOpDependency::getTargetOp).collect(Collectors.toList());
//        if (!existOp.contains(x.getTargetOp())) {

// 去除此缓存后ok

```

原因：脏数据 + 代码要优化，有一个应用间依赖包含了 250w+ 的脏数据

#### 性能优化

同步时间过长，调用多个API，再比对数据，原来以为是比对数据插入慢，后来发现是逻辑慢

先用stopwatch算一算时间，再结合arthas的trace在线上看看耗时，arth的trace在lamdba模式下不是很友好，for循环能统计里面的函数，但是foreach看不见目前

```html

unzip arthas-bin.zip

./as.sh  // 然后选择pid

```


```html

-----------------------------------------
ms     %     Task name
-----------------------------------------
28857  001%  xxx1 api call time
00563  000%  xxx1 logic time
144618  007%  xxx2 api call time
00892  000%  xxx2 logic time
01329  000%  zzz1 api call time
1763608  091%  zzz1 logic time 1
00088  000%  zzz1 logic time 2
00176  000%  zzz1 logic time 3



-----------------------------------------
ms     %     Task name
-----------------------------------------
171720  059%  data compare time   // 比较耗时
80030  028%  update data time  // 更新耗时
38171  013%  delete data time  // 删除耗时


StopWatch stopWatch = new StopWatch();
stopWatch.start("xxx1 api call time");
List<XXX1Server> xxx1Servers = xxx1ServerService.getAllServers();
stopWatch.stop();

stopWatch.start("xxx1 logic time");
stopWatch.stop();

logger.info("xxx job1 statistics " + stopWatch.prettyPrint());



trace com.test.Try runMethod

```

后面就不直接发布去生产看了，还是利用上面的 jstack 去一个运行任务的实例中去查，定位到代码，以下命令也可以去查找进程中的线程

```html

top -pH 36  // 这个也能查找出cpu高的线程

IpAddressUtil.findMatchCidr，可以查出来这个方法，然后进行优化

// 去除重复逻辑，修正ip的cidr范围，优化最终查询（使用排序后的list），速度明显上来了，同步的时间间隔越短，会越快的，如果1个小时间隔，中间断掉一次，还是会增加耗时的

```

#### 服务器上java的调试

1、写相关联的java类，然后package写一个fan这样，以下三个文件的package一样，放在同一个目录比如test下，所以不需要import

CallTest.java ImRequest.java ImResponse.java

2、外部的类在javac编译的时候要用到，那么需要引入jar包

```html

javac -classpath /home/xxx/test/jar/spring-web-5.2.4.RELEASE.jar:/home/xxx/test/jar/spring-core-5.2.4.RELEASE.jar:/home/xxx/test/jar/spring-jcl-5.2.4.RELEASE.jar CallTest.java ImRequest.java ImResponse.java -d .

```

-classpath指定具体的目录，冒号分割，-d指定编译后的文件夹，凡是代码内用到的类都需要放到classpath中

3、运行程序 java fan.CallTest 即可，注意这个时候运行时期的类也都得带上，有的类init的时候需要比编译期更多的类，凡是代码内用到的类都需要放到cp中，比方说gson的序列化需求

```html

java -cp .:/home/xxx/test/jar/spring-web-5.2.4.RELEASE.jar:/home/xxx/test/jar/spring-core-5.2.4.RELEASE.jar:/home/xxx/test/jar/spring-jcl-5.2.4.RELEASE.jar:/home/xxx/test/jar/gson-2.8.6.jar fan.CallTest

```

#### 一些误区

1、只申请4层访问入口8080口，获得一个VIP，后面成员服务器暴露restApi，在8080口上，直接http请求这个VIP，会成功嘛？

是可以成功的，4层只是一个转发，不是http请求前置就一定要个7层的，4层入口的ip也可以配置一个域名

2、vport换成80口呢，那么ip+port也是可以请求成功的，ip后可以带80也可以不带

在dpdk的情况下，看着是client和这个vip进行了建连（抓包看），（一般4层一次建连，7层两次建连），如果创建一个node应用，80->80，也是ok的

3、那么VIP是哪里来的呢

是从vipPool中分一个出来，这个vip会落到一台实体的机器上，但是ip a是看不见的，因为那个vip在用户态，不在内核态，然后具体到哪台server是交换机那边配置路由策略，是多活的

vip可以做到 停止bgp宣告，再开启

4、关于注入和切换

注入：用HttpClient，配置每路由4个连接，每隔5s进行10并发的请求，注入后等一会报错，
call failed. Timeout waiting for connection from pool
call failed. Timeout waiting for connection from pool
call failed. Read timed out
开始几次报上面的错，之前的4个连接变成了TIME_WAIT状态，说明客户端检测到连接异常后主动FIN了
然后再请求又建立新的连接了，请求故障dc，返回502


切换：用HttpClient，配置每路由4个连接，每隔5s进行10并发的请求，那么4个连接是一直在的（此时上面的注入状态也还在），此时进行切换到好的dc，再请求，拿的还是老连接，请求还是失败的502！！！这边说的是域名的切换，所以长连接还是会引起一些问题的
