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
