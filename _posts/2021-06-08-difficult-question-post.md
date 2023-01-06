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

#### chaosblade中卸载javaagent的功能

使用go语言去做了

另外有延迟的功能：异步的延迟

1、每种框架异步延迟的切点在哪

2、如何在请求的时候跳过enhancer

对于1，比如dubbo，底层使用netty，有调用链，最终通过netty.send去发送，响应的时候也在receive的地方，需要找到比较好的切入点

比如异步httpclient，底层是一个reactor模型，跟netty是类似的，也需要在上层找一个比较好的切入点

#### sandbox的问题



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

```

wiershark可以搜索SYN，等flags，tcp.flags.SYN == 1 这样，就是过滤是否建连的，然后tcp.stream eq 0 等等可以看指定 ip:port 到 ip:port的交互

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
