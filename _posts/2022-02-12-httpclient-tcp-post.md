---
layout: post
title: HttpClient相关
tags: [httpclient]
author-id: zqmalyssa
---

关于HttpClient的相关使用和与Tcp的交互及复用问题

#### httpclient

目前测试使用的是4.x版本

EntityUtils.toString()方法会将响应体的输入流关闭，相当于消耗了响应体，
此时连接会回到httpclient中的连接管理器的连接池中，如果下次访问的路由
是一样的（如第一次访问https://www.jianshu.com/,第二次访问则此连接可以被复用。

System.out.println("响应体内容：" + EntityUtils.toString(responseEntity));
如果关闭了httpEntity的inputStream，httpEntity长度应该为0，而且再次请求相同路由的连接可以共用一个连接。
可以通过设置连接管理器最大连接为1来验证。

```java


CloseableHttpClient httpClient = HttpClients.createDefault();

//        HttpGet httpGet = new HttpGet("http://cmonkey.ops.ctripcorp.com/api/hello");

        HttpGet httpGet = new HttpGet("http://10.6.10.232:8080/api/hello");

        CloseableHttpResponse response = null;
        try {
            response = httpClient.execute(httpGet);
            System.out.println(EntityUtils.toString(response.getEntity()));
            // EntityUtils.toString输入流关闭，消耗了响应体，连接会回到httpclient的连接管理器的连接池中，
            // 下次访问相同路由，连接可以复用
        } catch (IOException e) {
            e.printStackTrace();
}

System.out.println("finish");


```

这段代码请求完，断点打在finish，会发生什么，正常的 TCP建连，Http请求，Http响应，TCP的ack后，等了约莫1min后，服务端主动发来了[FIN, ACK]，客户端回了[ACK]，然后客户端进入了CLOSE_WAIT，服务端进入了FIN_WAIT2，这相当于是服务端主动发起的挥手，不一样

这段代码正常一次请求，没有出现挥手？？直接客户端发起了[RST, ACK]，来了解下

RST：（Reset the connection）用于复位因某种原因引起出现的错误连接，也用来拒绝非法数据和请求。如果接收到RST位时候，通常发生了某些错误；

发送RST包关闭连接时，不必等缓冲区的包都发出去，直接就丢弃缓冲区中的包，发送RST；接收端收到RST包后，也不必发送ACK包来确认。

“Connection reset”的原因是服务器关闭了Connection[调用了Socket.close()方法]。大家可能有疑问了：服务器关闭了Connection为什么会返回“RST”而不是返回“FIN”标志。原因在于Socket.close()方法的语义和TCP的“FIN”标志语义不一样：发送TCP的“FIN”标志表示我不再发送数据了，而Socket.close()表示我不在发送也不接受数据了。问题就出在“我不接受数据” 上，如果此时客户端还往服务器发送数据，服务器内核接收到数据，但是发现此时Socket已经close了，则会返回“RST”标志给客户端。当然，此时客户端就会提示：“Connection reset”。顺带看下如何在不shutdown进程的情况下关闭一个连接

```html

1、tcpkill/killcx切断tcp连接的原理
实际上这些工具都是通过模拟发送RST包来强制关闭tcp连接的，RST标示复位、用于关闭异常的tcp连接。
但是发送RST包需要知道SEQ（Sequence Number）和ACK（Acknowledgment Number）号；
这里就体现出了tcpkill和killcx的区别啦~也就是下文要说的tcpkill的局限性所在。

tcpkill的局限性
从“实际使用”中，大家也看到了，tcpkill不会立刻关闭已经建立的连接；而是在连接发生网络交互之后才关闭的。
这就是tcpkill的断流策略，它会监听指定的端口；发现符合条件的连接报文之后，根据报文中得到的SEQ/ACK号模拟RST包然后发送，从而断流。

在本文的使用场景中，这种策略是没有问题的；
但是在实际场景中，遇到的关闭不了的tcp连接，往往是“非活跃”的；这时tcpkill就起不到作用了。

不过，程序员是有办法的~

2、切断非活跃tcp连接的备选工具
现成的答案就是killcx。不同于tcpkill守株待兔的策略，killcx选择主动出击；
killcx伪造了一条tcp请求，发送给服务端；然后另一端会回送一个ACK报文，其中携带了正确的SEQ/ACK号；然后killcx就能顺利断流啦~

ps：不过写这篇博文查资料的时候看到有群众报告killcx没能成功关闭连接；然后一位兄台开创性地提出了tcpkill和killcx一起使用的建议；很有创见。

3、用gdb

使用 lsof 找到进程 45059 打开的所有文件描述符，并找到对应的 Socket 连接。

# lsof -np 45059
COMMAND     PID USER   FD   TYPE             DEVICE SIZE/OFF       NODE NAME
ceph-fuse 45059 root  rtd    DIR                8,2     4096          2 /
ceph-fuse 45059 root  txt    REG                8,2  6694144    1455967 /usr/bin/ceph-fuse
ceph-fuse 45059 root  mem    REG                8,2   510416    2102312 /usr/lib64/libfreeblpriv3.so
...
ceph-fuse 45059 root   12u  IPv4         1377072656      0t0        TCP 1.1.1.1:59950->1.1.1.2:smc-https (ESTABLISHED)

其中 12u 就是上面对应 Socket 连接的文件描述符。

gdb 连接到进程

$ gdb -p 45059

关闭 Socket 连接

(gdb) call close(12u)

Socket 连接就可以关闭了，但是进程 45059 还是好着的。

你可能会问，什么时候会用到这个特性呢？场景还是比较多的，比如你想测试下应用是否会自动重连 MySQL，通过这个办法就可以比较方便的测试了。

```

这边有个细节就是netstat 和 lsof都是分用户的，你用root用户，看不到netstat -p后的pid啥的，lsof看不到别的用户打开的文件信息！！！！lsof -u qiming就是列出具体用户的


socket 与 连接的对应关系，可以上面lsof找fd，也可以/proc/进程ID/fd这个目录找（是对应用户的），然后根据后面的数字去/proc/net/tcp里面找，是16进制的，对应一个连接


看下面这篇可以比较清楚认知time_wait和close_wait

https://www.cnblogs.com/kevingrace/p/9988354.html

```html

所以说这里凭直觉看，TIME_WAIT并不可怕，CLOSE_WAIT才可怕，因为CLOSE_WAIT很多，表示说要么是你的应用程序写的有问题，没有合适的关闭socket；要么是说，你的服务器CPU处理不过来（CPU太忙）或者你的应用程序一直睡眠到其它地方(锁，或者文件I/O等等)，你的应用程序获得不到合适的调度时间，造成你的程序没法真正的执行close操作。

ss -tan state time-wait | wc -l 出来成千上百也不需要太担心

但TIME_WAIT也有调优，则必须理解的几个调优参数:

sysctl -a | grep ipv4.tcp

net.ipv4.tcp_timestamps
RFC 1323 在 TCP Reliability一节里，引入了timestamp的TCP option，两个4字节的时间戳字段，其中第一个4字节字段用来保存发送该数据包的时间，第二个4字节字段用来保存最近一次接收对方发送到数据的时间。有了这两个时间字段，也就有了后续优化的余地。tcp_tw_reuse 和 tcp_tw_recycle就依赖这些时间字段。

net.ipv4.tcp_tw_reuse
从字面意思来看，这个参数是reuse TIME_WAIT状态的连接。时刻记住一条socket连接，就是那个五元组，出现TIME_WAIT状态的连接，一定出现在主动关闭连接的一方。所以，当主动关闭连接的一方，再次向对方发起连接请求的时候（例如，客户端关闭连接，客户端再次连接服务端，此时可以复用了；负载均衡服务器，主动关闭后端的连接，当有新的HTTP请求，负载均衡服务器再次连接后端服务器，此时也可以复用），可以复用TIME_WAIT状态的连接。

通过字面解释以及例子说明，可以看到，tcp_tw_reuse应用的场景：某一方，需要不断的通过“短连接“连接其他服务器，总是自己先关闭连接(TIME_WAIT在自己这方)，关闭后又不断的重新连接对方。

那么，当连接被复用了之后，延迟或者重发的数据包到达，新的连接怎么判断，到达的数据是属于复用后的连接，还是复用前的连接呢？那就需要依赖前面提到的两个时间字段了。复用连接后，这条连接的时间被更新为当前的时间，当延迟的数据达到，延迟数据的时间是小于新连接的时间，所以，内核可以通过时间判断出，延迟的数据可以安全的丢弃掉了。

这个配置，依赖于连接双方，同时对timestamps的支持。同时，这个配置，仅仅影响outbound连接，即做为客户端的角色，连接服务端[connect(dest_ip, dest_port)]时复用TIME_WAIT的socket。

net.ipv4.tcp_tw_recycle
从字面意思来看，这个参数是销毁掉 TIME_WAIT。当开启了这个配置后，内核会快速的回收处于TIME_WAIT状态的socket连接。多快？不再是2MSL，而是一个RTO（retransmission timeout，数据包重传的timeout时间）的时间，这个时间根据RTT动态计算出来，但是远小于2MSL。

有了这个配置，还是需要保障丢失重传或者延迟的数据包，不会被新的连接(注意，这里不再是复用了，而是之前处于TIME_WAIT状态的连接已经被destroy掉了，新的连接，刚好是和某一个被destroy掉的连接使用了相同的五元组而已)所错误的接收。在启用该配置，当一个socket连接进入TIME_WAIT状态后，内核里会记录包括该socket连接对应的五元组中的对方IP等在内的一些统计数据，当然也包括从该对方IP所接收到的最近的一次数据包时间。当有新的数据包到达，只要时间晚于内核记录的这个时间，数据包都会被统统的丢掉。

这个配置，依赖于连接双方对timestamps的支持。同时，这个配置，主要影响到了inbound的连接（对outbound的连接也有影响，但是不是复用），即做为服务端角色，客户端连进来，服务端主动关闭了连接，TIME_WAIT状态的socket处于服务端，服务端快速的回收该状态的连接。


另外，开启tw_recylce和tw_reuse功能, 一定需要timestamps的支持，而且这些配置一般不建议开启，但是对解决TIME_WAIT很多的问题，有很好的用处。
果然, 经过如上配置后, 过了几分钟，再查看TIME_WAIT的数量快速下降了不少，并且后面也没发现哪个用户说有问题了


CLOSE_WAIT的情况，看一些场景把

服务器A是一台爬虫服务器，它使用简单的HttpClient去请求资源服务器B上面的apache获取文件资源，正常情况下，如果请求成功，那么在抓取完资源后，服务器A会主动发出关闭连接的请求，这个时候就是主动关闭连接，服务器A的连接状态我们可以看到是TIME_WAIT。如果一旦发生异常呢？假设请求的资源服务器B上并不存在，那么这个时候就会由服务器B发出关闭连接的请求，服务器A就是被动的关闭了连接，如果服务器A被动关闭连接之后程序员忘了让HttpClient释放连接，那就会造成CLOSE_WAIT的状态了

```


1. HttpClient 的范围

基于HttpCore的客户端HTTP传输库。

基于经典（阻塞）I / O。

内容不可知


2. HttpClient不是什么

HttpClient不是浏览器。它是一个客户端HTTP传输库。HttpClient的目的是传输和接收HTTP消息。HttpClient不会尝试处理内容、执行嵌入在HTML页面中的javascript、尝试猜测内容类型(如果没有显式设置)或重新格式化请求/重写位置uri，或其他与HTTP传输无关的功能。


补充：

HttpClient 一般使用 PoolingHttpClientConnectionManager 实现池化技术，而 BasicHttpClientConnectionManager是单线程的，没有使用到池化技术。池化技术中两个比较关键的参数就是 maxTotal 和 maxPerRoute。

maxTotal 是设置同时间正在使用的最大连接数，默认值是20。（就是指连接）
maxPerRoute 是设置一个 host(ip或域名):port 同时间正在使用的最大连接数，默认值是2。（这边注意，ip或域名是分开的，域名指向对应的ip和ip直接访问是不共享maxPerRoute，没有到Path哦，注意！！！！！）


```java

PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();

cm.setMaxTotal(10);   // 比较重要
cm.setDefaultMaxPerRoute(3);    // 比较重要

RequestConfig requestConfig =
    RequestConfig.custom().setConnectionRequestTimeout(60 * 1000)
        .setSocketTimeout(90 * 1000).setConnectTimeout(30 * 1000).
        build();

HttpClientBuilder httpClientBuilder = HttpClients.custom().setConnectionManager(cm)
    .setDefaultRequestConfig(requestConfig);


CloseableHttpClient httpClient = httpClientBuilder.build();


另外，创建默认的CloseableHttpClient实例中有一个连接管理器，最大连接数为20，每个路由最大连接数为2。因为有了连接管理器对连接的管理，我们可以放心的使用多线程来执行请求，可以有多个HttpClient（我觉得没有很大必要），但是必须将他们设置成同一个连接管理器，才能达到共用连接的目的。


```

3. 相关基础

```java

CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/");
CloseableHttpResponse response = httpclient.execute(httpget);
try {
    HttpEntity entity = response.getEntity();
    if (entity != null) {
        InputStream instream = entity.getContent();
        try {
            // do something useful
        } finally {
            instream.close();   // 关闭内容流
        }
    }
} finally {
    response.close();  // 关闭响应
}

```

以上，关闭内容流和关闭响应之间的区别在于前者将尝试通过使用实体内容来保持底层连接处于活动状态，而后者立即关闭并丢弃连接。

请注意，一旦实体完全写出，还需要HttpEntity#writeTo(OutputStream)方法来确保正确释放系统资源。 如果通过调用HttpEntity#getContent()方法获取了java.io.InputStream的实例，则还应该在finally代码块中关闭该流。

使用流式实体时，可以使用EntityUtils#consume(HttpEntity)方法确保实体内容已完全消耗且基础流已关闭。

然而，可能存在这样的情况：当只需要检索整个响应内容的一小部分并且消耗剩余内容并使连接可重用时的性能损失太高。在这种情况下，可以通过关闭响应来终止内容流。

```java

CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/");
CloseableHttpResponse response = httpclient.execute(httpget);
try {
    HttpEntity entity = response.getEntity();
    if (entity != null) {
        InputStream instream = entity.getContent();
        int byteOne = instream.read();
        int byteTwo = instream.read();
        // Do not need the rest
    }
} finally {
    response.close();  // 直接关闭响应
}

```

连接不会被重用，但它所拥有的所有的级别资源都将被正确释放。

消费实体内容的推荐方法是使用其HttpEntity#getContent()或HttpEntity#writeTo(OutputStream)方法。 HttpClient还附带了EntityUtils类，它公开了几种静态方法，以便更容易地从实体中读取内容或信息。 可以使用此类中的方法检索字符串/字节数组中的整个内容主体，而不是直接读取java.io.InputStream。 但是，强烈建议不要使用EntityUtils，除非响应实体来自可信HTTP服务器并且知晓响应内容长度有限


当不再需要实例CloseableHttpClient并且即将超出使用范围时，必须通过调用CloseableHttpClient#close()方法关闭与其关联的连接管理器。

```java

CloseableHttpClient httpclient = HttpClients.createDefault();
try {
    <...>
} finally {
    httpclient.close();
}

```

4、连接管理

https://www.jianshu.com/p/979ae914612c
