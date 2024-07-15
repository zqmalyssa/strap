---
layout: post
title: tensorflow
tags: [tensorflow]
author-id: zqmalyssa
---

关于机器学习及深度学习的一些总结，代码使用tensorflow

#### 基本使用

```html

import os
os.environ['TF_CPP_MIN_LOG_LEVEL'] = '1'

// 这样可以忽略以下message
This TensorFlow binary is optimized with oneAPI Deep Neural Network Library (oneDNN)to use the following CPU instructions in performance-critical operations:  AVX AVX2

// message说的是不同的硬件会让tensor用不同的代码，支持AVX和AVX2，计算会有额外加速

```


比如tomcat，jetty等

说下tomcat

Tomcat 默认配置的最大请求数是 150，也就是说同时支持 150 个并发，当然了，也可以将其改大。
当某个应用拥有 250 个以上并发的时候，应考虑应用服务器的集群。
具体能承载多少并发，需要看硬件的配置，CPU 越多性能越高，分配给 JVM 的内存越多性能也就越高，但也会加重 GC 的负担。
操作系统对于进程中的线程数有一定的限制：
Windows 每个进程中的线程数不允许超过 2000
Linux 每个进程中的线程数不允许超过 1000
另外，在 Java 中每开启一个线程需要耗用 1MB 的 JVM 内存空间用于作为线程栈之用。
Tomcat的最大并发数是可以配置的，实际运用中，最大并发数与硬件性能和CPU数量都有很大关系的。更好的硬件，更多的处理器都会使Tomcat支持更多的并发。
Tomcat 默认的 HTTP 实现是采用阻塞式的 Socket 通信，每个请求都需要创建一个线程处理。这种模式下的并发量受到线程数的限制，但对于 Tomcat 来说几乎没有 BUG 存在了。
Tomcat 还可以配置 NIO 方式的 Socket 通信，在性能上高于阻塞式的，每个请求也不需要创建一个线程进行处理，并发能力比前者高。但没有阻塞式的成熟。
这个并发能力还与应用的逻辑密切相关，如果逻辑很复杂需要大量的计算，那并发能力势必会下降。如果每个请求都含有很多的数据库操作，那么对于数据库的性能也是非常高的。
对于单台数据库服务器来说，允许客户端的连接数量是有限制的。
并发能力问题涉及整个系统架构和业务逻辑。
系统环境不同，Tomcat版本不同、JDK版本不同、以及修改的设定参数不同。并发量的差异还是蛮大的。

Tomcat接收请求的方式

Tomcat支持三种接收请求的处理方式：BIO、NIO、APR 。  

1>、Bio方式，阻塞式I/O操作即使用的是传统Java I/O操作，Tomcat7以下版本默认情况下是以bio模式运行的，由于每个请求都要创建一个线程来处理，线程开销较大，不能处理高并发的场景，在三种模式中性能也最低

配置如下(tomcat安装目录下的/conf/server.xml)

<Connector port = "8080" protocol="HTTP/1.1"

tomcat启动如下，看到http-bio-8080便是bio模式

2>、Nio方式，是Java SE 1.4及后续版本提供的一种新的I/O操作方式(即java.nio包及其子包)，是一个基于缓冲区、并能提供非阻塞I/O操作的Java API，它拥有比传统I/O操作(bio)更好的并发运行性能。tomcat 8版本及以上默认nio模式（注意，8以上版本）

<Connector port = "8080" protocol="HTTP/1.1"

tomcat启动如下，看到http-nio-8080便是nio模式，如下

31-Jan-2024 17:31:47.802 INFO [main] org.apache.coyote.AbstractProtocol.init Initializing ProtocolHandler ["http-nio-8080"]

3>、apr模式：简单理解，就是从操作系统级别解决异步IO问题，大幅度的提高服务器的处理和响应性能， 也是Tomcat运行高并发应用的首选模式。
启用这种模式稍微麻烦一些，需要安装一些依赖库， 而apr的本质就是使用jni技术调用操作系统底层的IO接口，所以需要提前安装所需要的依赖，首先是需要安装openssl和apr。具体的怎么安装在此就不讲解,想了解的可以百度，可以参考https://www.cnblogs.com/freeweb/p/6430053.html
