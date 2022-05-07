---
layout: post
title: 疑难杂症
tags: [difficult]
author-id: zqmalyssa
---

#### pj的问题

1、no work available的问题

得是一个workflow的任务，开始在LZ的机器，然后2点半篡位了，app2的master变到了XZ，看日志，正常上报心跳，3点清理不用的worker（就是现在时间到最后一次心跳上报时间晚于1min的实例），然后workflow的任务继续在

LZ上执行的时候就没有worker了


worker连接server（服务发现，隔10秒发一次discovery）用的powerjob的域名是 local to local的，那么根据acquire的逻辑，currentServer（10.xx.xx.xx:10086）到本机的就应该直接返回（问题之前一直是local to local的，但删除的时候各个地址都有的）

心跳上报是隔15秒上报一次，有了currentServer（也是起了一个线程不停的更新），才会上报


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

所以acquire带过来的currentServer是一个

如果当时server是LZ的，那么LZ的请求过来会直接返回，没有选主，如果XZ的过来的话，看数据库里的（也是LZ的），akka端口没问题就返回数据库的，也是LZ的，akka有问题，分布式锁下，然后本机作为server，更新回worker的话currentServer也就变了，那么后续心跳上报的server也变了

akka里的serverPath是 akka://oms-server@10.96.178.40:10086/user/server_actor，心态中的workAddress是 XX.XX.XX.XX:27777

对于出现的问题，一开始主在LZ起的任务，原理就是server只会执行本及关联的app的任务，当时数据库中的server是LZ，那么推任务到时间轮的也是LZ，但是后面换主了，又被清理了worker，那么LZ就找不到worker了

```

SHAXZ机器上的问题

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



SHALZ机器上的问题

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

#### chaosblade中卸载javaagent的功能

使用go语言去做了


#### sandbox的问题
