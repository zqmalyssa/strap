---
layout: post
title: Netty相关
tags: [code, java, netty]
author-id: zqmalyssa
---

Netty相关内容在这

#### Java的IO演进

要先了解Linux网络I/O模型，linux内核将所有外部设备都看作一个文件来操作，对一个文件的读写操作会调用内核提供的系统命令，返回一个file descriptor（fd，文件描述符），而对一个socket的读写也会有相应的描述符，称为socketfd（socket描述符），描述符就是一个数字，指向内存中的一个结构体（文件路径，数据区等一些属性）

根据UNIX网络编程对I/O模型的分类，UNIX提供了5种I/O模型，分别如下

1、阻塞I/O模型，最常用的I/O模型就是阻塞的，在进程空间中调用recvfrom，其系统调用直到数据包到达且被复制到应用程序的缓冲区中或者发生错误时才返回，在此期间其一直会等待，进程在从recvfrom开始到它返回的整段时间内都是被阻塞的，因此被称为阻塞I/O模型

2、非阻塞I/O模型：recvfrom从应用层到内核的时候，如果该缓冲区没有数据的话，就直接返回一个EWOULDBLOCK错误，一般对非阻塞I/O模型进行轮询检查这个状态，看内核是不是有数据到来

![netty_nio1]({{ "/assets/img/netty/netty_nio1.png" | relative_url}})

3、I/O复用模型，Linux提供select/poll，进程通过将一个或多个fd传递给select或者poll系统调用，阻塞在select操作上，这样select/poll可以帮我们侦测多个fd是否处于就绪状态，select/poll是顺序扫描fd是否就绪，而且支持的fd数量有限，因此它的使用受到了一些限制，linux还提供了一个epoll系统调用，epoll使用基于事件驱动方式代替顺序扫描，因此性能更高，还要当fd就绪时，立即回调函数rollback。

4、信号驱动I/O模型：首先开启套接口信号驱动I/O功能，并通过系统调用sigaction执行一个信号处理函数（此系统调用立即返回，进程继续工作，它是非阻塞的）当数据准备就绪时，就为该进程生成一个SIGIO信号，通过信号回调通知应用程序调用recvfrom来读取数据，并通知主循环函数处理数据

![netty_nio2]({{ "/assets/img/netty/netty_nio2.png" | relative_url}})

5、异步I/O，告知内核启动某个操作，并让内核再整个操作完成后（包括将数据从内核复制到用户自己的缓冲区）通知我们，这种模型与信号驱动模型的主要区别是：信号驱动I/O由内核通知我们何时可以开始一个I/O操作，异步I/O模型由内核通知我们I/O操作何时已经完成

![netty_nio3]({{ "/assets/img/netty/netty_nio3.png" | relative_url}})

Java的主要看下I/O多路复用技术，因为JavaNIO的核心类库多路复用器Selector就是epoll多路复用的实现

在I/O编程中，当需要同时处理多个客户端接入请求时，可以利用多线程或者I/O多路复用技术来进行处理，I/O多路复用技术通过把多个I/O的阻塞复用到同一个select的阻塞上，从而使得系统在单线程的情况下可以同时处理多个客户端请求，与传统的多线程多进程相比，I/O多路复用技术的最大优势是系统开销小，系统不需要额外创建进程或者线程，也不需要维护它们的运行，降低了系统的维护工作量，其典型应用场景是：

1、服务器需要同时处理多个处于监听状态或者多个连续状态的套接字
2、服务器需要同时处理多种网络协议的套接字

支持I/O多路复用的系统调用有select，pselect，poll和epoll，epoll克服了select的缺点，总结：

1、支持一个进程打开的socket描述符（FD）不受限制（仅受限于操作系统的最大文件句柄数）
2、I/O效率不会随着FD数目的增加而线性下降
3、使用mmap加速内核与用户空间的消息传递
4、epoll的API更加简单

Java的I/O演进就是

JavaNIO之前，基于Java的所有Socket通信采用的都是同步阻塞模式（BIO），这简化了上层开发，但是性能方面却有着巨大的瓶颈，因此很长一段时间，大型的应用服务器都采用C或者C++语言开发，因为他们可以直接使用操作系统提供的异步I/O或者AIO能力，

JDK1.4才出了NIO，加了很多的东西（Pipe，Channel，Buffer，Selector），JDK1.7（2011）又将NIO进行了生机，被称为NIO2.0（提供了AIO）

注意因为有了网络间的通信，才需要网络编程，才有NIO对BIO的改进

传统的BIO的方式，看下面的例子，用到了普通IO

服务端

```java
package com.qiming.test.netty.bio;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.Date;

/**
 * BIO的服务端，接收客户端发来的请求，用到普通IO
 *
 */
public class TimeServer {

  public static void main(String[] args) {

    int port = 8080;
    if (args != null && args.length > 0) {
      port = Integer.valueOf(args[0]);
    }

    ServerSocket server = null;
    try {
      server = new ServerSocket(port);
      System.out.println("The time server is start in port: " + port);
      Socket socket = null;
      while (true) {
        socket = server.accept();
        new Thread(new TimeServerHandler(socket)).start();
      }
    } catch (IOException e) {
      e.printStackTrace();
    } finally {
      if (server != null) {
        System.out.println("The time server close");
        try {
          server.close();
          server = null;
        } catch (IOException e) {
          e.printStackTrace();
        }
      }
    }


  }

}

class TimeServerHandler implements Runnable {

  private Socket socket;

  public TimeServerHandler(Socket socket) {
    this.socket = socket;
  }

  @Override
  public void run() {

    BufferedReader in = null;
    PrintWriter out = null;

    try {
      in = new BufferedReader(new InputStreamReader(this.socket.getInputStream()));
      out = new PrintWriter(this.socket.getOutputStream(), true);

      String currentTime = null;
      String body = null;

      while (true) {
        body = in.readLine();
        if (body == null) {
          break;
        }
        System.out.println("The time server receive order: " + body);
        currentTime = "QUERY TIME ORDER".equalsIgnoreCase(body) ? new Date(System.currentTimeMillis()).toString() : "BAD ORDER";
        out.println(currentTime);
      }

    } catch (IOException e) {
      e.printStackTrace();
    } finally {
      //注意in的close是要try catch的，out不需要，还有细节关闭的顺序
      if (in != null) {
        try {
          in.close();
        } catch (IOException e) {
          e.printStackTrace();
        }
      }

      if (out != null) {
        out.close();
        out = null;
      }

      if (this.socket != null) {
        try {
          this.socket.close();
        } catch (IOException e) {
          e.printStackTrace();
        }
        this.socket = null;
      }
    }


  }
}
```

客户端

```java
package com.qiming.test.netty.bio;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.Socket;

/**
 * 客户端向服务端发送"QUERY TIME ORDER"指令获取当前时间，用到普通IO
 *
 */
public class TimeClient {

  public static void main(String[] args) {

    int port = 8080;

    if (args != null && args.length > 0) {
      port = Integer.valueOf(args[0]);
    }

    Socket socket = null;
    BufferedReader in = null;
    PrintWriter out = null;

    try {
      socket = new Socket("127.0.0.1", port);
      in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
      out = new PrintWriter(socket.getOutputStream(), true);
      out.println("QUERY TIME ORDER"); //注意是println哦
      System.out.println("Send order 2 server succeed.");
      String resp = in.readLine();
      System.out.println("Now is : " + resp);
    } catch (IOException e) {
      e.printStackTrace();
    } finally {
      //注意in的close是要try catch的，out不需要，还有细节关闭的顺序

      if (out != null) {
        out.close();
        out = null;
      }

      if (in != null) {
        try {
          in.close();
        } catch (IOException e) {
          e.printStackTrace();
        }
      }

      if (socket != null) {
        try {
          socket.close();
        } catch (IOException e) {
          e.printStackTrace();
        }
        socket = null;
      }
    }

  }

}

```

模拟客户端向服务端拿时间的例子，当有新的客户请求接入时，服务端必须创建一个新的线程处理新接入的客户端链路。一个线程只能处理一个客户端连接。

伪异步IO通过线程池来处理线程会耗尽的问题，线程池就用原生的线程池就行了

但是改造后，还是读写都是同步阻塞的

到了NIO，NIO之前也提过，但有点模糊的是这个N到底是什么，官方其实就是New IO，但是人们为了体现它的特点，也可以称呼它为Non-block IO，NIO的内容还是看看之前的文章（Buffer，Channel，Selector），NIO的编程特别复杂，那使用它必然是有很多好处的：

1、客户端发起的连接操作时异步的，可以通过在多路复用注册器注册OP_CONNECT等待后续结果，不需要像之前的客户端那样被同步阻塞
2、SocketChannel的读写操作时异步的，如果没有可读写的数据它不会同步等待，直接返回
3、线程模型的优化，epoll实现，没有连接句柄的限制，Selector线程可以同时处理成千上万个客户端连接

NIO2.0里面刚才说又有AIO，引入了异步通道的概念，并提供异步文件通道和异步套接字通道的实现，异步通信提供以下两种方式获取操作结果

1、java.util.concurrent.Future类来表示异步操作的结果
2、在执行异步操作的时候传入一个java.nio.channels

异步套接字通道是真正的异步非阻塞I/O。对应UNIX的AIO，没有了Selector

开发高质量的NIO程序并不是容易的事，调试定位问题也非常麻烦，java原生这个NIO一个是API太复杂了，需要多线程和网络编程的知识（Reactor模式），epoll的bug等，为了项目好维护，选择Netty：

1、API简单，开发门槛低
2、功能强大，且定制功能强大，
3、性能高
4、成熟稳定
5、社区活跃
6、经历过大型商业考验

我们看下用netty怎么改上面的代码，netty的pom版本有两种，这边选用netty-all的，不要两个一起引入，会有冲突的，代码也不一样

```java
<!-- 用于测试Netty的 -->
    <dependency>
      <groupId>io.netty</groupId>
      <artifactId>netty-all</artifactId>
      <version>5.0.0.Alpha1</version>
    </dependency>

    <!--<dependency>-->
      <!--<groupId>io.netty</groupId>-->
      <!--<artifactId>netty</artifactId>-->
    <!--</dependency>-->
```

看看更新后的服务端

```java
package com.qiming.pom.netty;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.buffer.Unpooled;
import io.netty.channel.Channel;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelHandlerAdapter;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.buffer.ByteBuf;
import java.util.Date;

public class TimeServer {

  public void bind(int port) {
    //配置服务端的NIO线程组
    //NioEventLoopGroup是个线程组，它包含一组NIO线程，专门用于网络事件的处理，实际上它们就是Reactor线程组，这里创建两个的原因是一个用于服务端接受客户端的连接，另一个用于进行SocketChannel网络
    //读写，ServerBootstrap对象，它是Netty用于启动NIO服务端的辅助启动类，目的是降低服务端的开发复杂度，调用group方法，创建NioServerSocketChannel的Channel，它的功能跟JDK的NIO库中的
    //ServerSocketChannel类似，然后配置NioServerSocketChannel的TCP参数，此处将它的backlog设置为1024，最后绑定I/O事件的处理类ChildChannelHandler，它的作用类似于Reactor模式中的Handler类
    //主要用于处理网络I/O事件，例如记录日志，对消息进行编解码等
    EventLoopGroup bossGroup = new NioEventLoopGroup();
    EventLoopGroup workerGroup = new NioEventLoopGroup();

    try {
      ServerBootstrap b = new ServerBootstrap();
      b.group(bossGroup, workerGroup).channel(NioServerSocketChannel.class).option(
          ChannelOption.SO_BACKLOG, 1024).childHandler(new ChildChannelHandler());
      //绑定端口，调动bing方法绑定监听端口，随后调用它的同步阻塞方法sync等待绑定操作的完成，完成之后Netty会返回一个ChannelFuture，主要用于异步操作的通知回调
      ChannelFuture f = b.bind(port).sync();

      //等到服务器端监听端口关闭，使用方法进行阻塞，等待服务器链路关闭之后main函数才退出
      f.channel().closeFuture().sync();
    } catch (Exception e) {
      e.printStackTrace();
    } finally {
      //优雅退出，释放线程池资源
      bossGroup.shutdownGracefully();
      workerGroup.shutdownGracefully();
    }


  }

  private class ChildChannelHandler extends ChannelInitializer {

    @Override
    protected void initChannel(Channel channel) throws Exception {
      channel.pipeline().addLast(new TimeServerHandler());
    }
  }

  public static void main(String[] args) {
    int port = 8080;
    if (args != null && args.length > 0) {
      port = Integer.valueOf(args[0]);
    }
    new TimeServer().bind(port);
  }

}

/**
 * TimeServerHandler继承自ChannelHandlerAdapter，它用于对网络事件进行读写操作，关注channelRead和exceptionCaught方法
 *
 */
class TimeServerHandler extends ChannelHandlerAdapter {

  @Override
  public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception{

    //ByteBuf，类似JDK的ByteBuffer
    ByteBuf buf = (ByteBuf)msg;
    byte[] req = new byte[buf.readableBytes()];
    buf.readBytes(req);
    String body = new String(req, "UTF-8");
    System.out.println("The time server receive order : " + body);
    String currentTime = "QUERY TIME ORDER".equalsIgnoreCase(body) ? new Date(System.currentTimeMillis()).toString() : "BAD ORDER";
    ByteBuf resp = Unpooled.copiedBuffer(currentTime.getBytes());
    //write方法并不直接将消息写入SocketChannel中，调用write方法只是把待发送的消息放到发送缓冲数组，再通过调用flush方法，将发送缓冲区中的消息全部写到SocketChannel
    ctx.write(resp);

  }

  @Override
  public void channelReadComplete(ChannelHandlerContext ctx) {
    //将消息发送队列中的消息写入到SocketChannel中发送给对方
    ctx.flush();
  }

  @Override
  public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
    //发生异常时，关闭ChannelHandlerContext，释放和ChannelHandlerContext相关联的句柄等资源
    ctx.close();
  }

}
```

看看更新后的客户端

```java
package com.qiming.pom.netty;

import io.netty.bootstrap.Bootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelHandlerAdapter;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import java.util.logging.Logger;

public class TimeClient {

  public void connect(int port, String host) {
    //配置客户端NIO线程组
    EventLoopGroup group = new NioEventLoopGroup();

    try {
      Bootstrap b = new Bootstrap();
      //跟服务端不同的是，这边是NioSocketChannel，然后为其添加handler
      b.group(group).channel(NioSocketChannel.class).option(ChannelOption.TCP_NODELAY, true).handler(
          new ChannelInitializer<SocketChannel>() {
            @Override
            protected void initChannel(SocketChannel socketChannel) throws Exception {
              socketChannel.pipeline().addLast(new TimeClientHandler());
            }
          });
      //发起异步连接操作
      ChannelFuture f = b.connect(host, port).sync();

      //等待客户端链路关闭
      f.channel().closeFuture().sync();
    } catch (InterruptedException e) {
      e.printStackTrace();
    } finally {
      group.shutdownGracefully();
    }
  }

  public static void main(String[] args) {
    int port = 8080;
    if (args != null && args.length > 0) {
      port = Integer.valueOf(args[0]);
    }
    new TimeClient().connect(port, "127.0.0.1");
  }

}


class TimeClientHandler extends ChannelHandlerAdapter {

  private static final Logger logger = Logger.getLogger(TimeClientHandler.class.getName());

  private final ByteBuf firstMessage;

  public TimeClientHandler() {
    byte[] req = "QUERY TIME ORDER".getBytes();
    this.firstMessage = Unpooled.buffer(req.length);
    firstMessage.writeBytes(req);
  }

  @Override
  public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception{

    ByteBuf buf = (ByteBuf)msg;
    byte[] req = new byte[buf.readableBytes()];
    buf.readBytes(req);
    String body = new String(req, "UTF-8");
    System.out.println("Now is : " + body);

  }

  /**
   * 当客户端和服务端TCP链路成功后，Netty的NIO线程会调用channelActive方法，发送查询时间的指令给服务端
   * @param ctx
   */
  @Override
  public void channelActive(ChannelHandlerContext ctx) {
    //将请求消息发送给服务端
    ctx.writeAndFlush(firstMessage);
  }

  @Override
  public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
    //释放资源
    logger.warning("Unexpected exception from downstream : " + cause.getMessage());
    ctx.close();
  }
}
```

#### Netty线程模型及EventLoop详解

首先什么是线程模型？线程模型指定了线程管理的模型。在进行并发编程的过程中，我们需要小心的处理多个线程之间的同步关系，而一个好的线程模型可以大大减少管理多个线程的成本。

Reactor是一种经典的线程模型，Reactor线程模型分为单线程模型、多线程模型以及主从多线程模型。

1、Reactor单线程模型仅使用一个线程来处理所有的事情，包括客户端的连接和到服务器的连接，以及所有连接产生的读写事件，这种线程模型需要使用异步非阻塞I/O，使得每一个操作都不会发生阻塞，Handler为具体的处理事件的处理器，而Acceptor为连接的接收者，作为服务端接收来自客户端的链接请求。这样的线程模型理论上可以仅仅使用一个线程就完成所有的事件处理，显得线程的利用率非常高，而且因为只有一个线程在工作，所有不会产生在多线程环境下会发生的各种多线程之间的并发问题，架构简单明了，线程模型的简单性决定了线程管理工作的简单性。但是这样的线程模型存在很多不足，比如：
  - 仅利用一个线程来处理事件，对于目前普遍多核心的机器来说太过浪费资源
  - 一个线程同时处理N个连接，管理起来较为复杂，而且性能也无法得到保证，这是以线程管理的简洁换取来的事件管理的复杂性，而且是在性能无 法得到保证的前提下换取的，在大流量的应用场景下根本没有实用性
  - 当处理的这个线程负载过重之后，处理速度会变慢，会有大量的事件堆积，甚至超时，而超时的情况下，客户端往往会重新发送请求，这样的情况下，这个单线程的模型就会成为整个系统的瓶颈
  - 单线程模型的一个致命缺点就是可靠性问题，因为仅有一个线程在工作，如果这个线程出错了无法正常执行任务了，那么整个系统就会停止响应。

2、Reactor多线程模型，接收连接和处理请求作为两部分分离了，Acceptor使用单独的线程来接收请求，做好准备后就交给事件处理的handler来处理，而handler使用了一个线程池来实现，这个线程池可以使用Executor框架实现的线程池来实现。所以，一个连接会交给一个handler线程来负责其上面的所有事件。需要注意，一个连接只会由一个线程来处理，而多个连接可能会由一个handler线程来处理，关键在于一个连接上的所有事件都只会由一个线程来处理，这样的好处就是消除了不必要的并发同步的麻烦。感觉好多了，但仍有缺点
  - 多线程模型下仍然只有一个线程来处理客户端的连接请求，那如果这个线程挂了，那整个系统任然会变为不可用
  - 因为仅仅由一个线程来负责客户端的连接请求，如果连接之后要做一些验证之类复杂耗时操作再提交给handler线程来处理的话，就会出现性能问题

3、Reactor主从多线程模型，解决了Reactor单线程模型和Reactor多线程模型中存在的问题，解决了handler的性能问题，以及Acceptor的安全以及性能问题，Netty就使用了这种线程模型来处理事件

首先，Netty使用EventLoop来处理连接上的读写事件，而一个连接上的所有请求都保证在一个EventLoop中被处理，一个EventLoop中只有一个Thread，所以也就实现了一个连接上的所有事件只会在一个线程中被执行。一个EventLoopGroup包含多个EventLoop，可以把一个EventLoop当做是Reactor线程模型中的一个线程，而一个EventLoopGroup类似于一个ExecutorService，当然，这只是为了更好的理解Netty的线程模型，它们之间是没有等价关系的，后面的分析中会详细讲到。下面的图片展示了Netty的线程模型：

![netty_thread]({{ "/assets/img/netty/netty_thread.png" | relative_url}})

看一个案例：

```java
// Configure the server.
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class)
             .option(ChannelOption.SO_BACKLOG, 100)
             .handler(new LoggingHandler(LogLevel.INFO))
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     p.addLast(your_handler_name, your_handler_instance);
                 }
             });

            // Start the server.
            ChannelFuture f = b.bind(PORT).sync();

            // Wait until the server socket is closed.
            f.channel().closeFuture().sync();
        } finally {
            // Shut down all event loops to terminate all threads.
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
```

Netty的服务端使用了两个EventLoopGroup，而第一个EventLoopGroup通常只有一个EventLoop，通常叫做bossGroup，负责客户端的连接请求，然后打开Channel，交给后面的EventLoopGroup中的一个EventLoop来负责这个Channel上的所有读写事件，一个Channel只会被一个EventLoop处理，而一个EventLoop可能会被分配给多个Channel来负责上面的事件，当然，Netty不仅支持NI/O，还支持OI/O，所以两者的EventLoop分配方式有所区别，下面分别展示了NI/O和OI/O的分配方式：

![EventLoopNio]({{ "/assets/img/netty/EventLoopNio.png" | relative_url}})

在NI/O非阻塞模式下，Netty将负责为每个Channel分配一个EventLoop，一旦一个EventLoop被分配给了一个Channel，那么在它的整个生命周期中都使用这个EventLoop，但是多个Channel将可能共享一个EventLoop，所以和Thread相关的ThreadLocal的使用就要特别注意，因为有多个Channel在使用该Thread来处理读写事件。在阻塞IO模式下，考虑到一个Channel将会阻塞，所以不太可能将一个EventLoop共用于多个Channel之间，所以，每一个Channel都将被分配一个EventLoop，并且反过来也成立，也就是一个EventLoop将只会被绑定到一个Channel上来处理这个Channel上的读写事件。无论是非阻塞模式还是阻塞模式，一个Channel都将会保证一个Channel上的所有读写事件都只会在一个EventLoop上被处理。

![EventLoopOio]({{ "/assets/img/netty/EventLoopOio.png" | relative_url}})

所以在netty中，EventLoop是一个极为重要的组件，它翻译过来称为事件循环，一个EventLoop将被分配给一个Channel，来负责这个Channel的整个生命周期之内的所有事件

![EventLoop]({{ "/assets/img/netty/EventLoop.png" | relative_url}})

从EventLoop的类图中可以发现，其实EventLoop继承了Java的ScheduledExecutorService，也就是调度线程池，所以，EventLoop应当有ScheduledExecutorService提供的所有功能。那为什么需要继承ScheduledExecutorService呢，也就是为什么需要延时调度功能，那是因为，在Netty中，有可能用户线程和Netty的I/O线程同时操作网络资源，而为了减少并发锁竞争，Netty将用户线程的任务包装成Netty的task，然后向Netty的I/O任务一样去执行它们。有些时候我们需要延时执行任务，或者周期性执行任务，那么就需要调度功能。这是Netty在设计上的考虑，为我们极大的简化的编程方法。

EventLoop是一个接口，它在继承了ScheduledExecutorService等多个类的同时，仅仅提供了一个方法parent，这个方法返回它属于哪个EventLoopGroup。本文只分析非阻塞模式，而阻塞模式留到未来某个合适的时候再做分析总结。在上文中展示的服务端启动的代码中我们发现我们使用的EventLoop是一个子类NioEventLoopGroup，下面就来分析一下NioEventLoopGroup这个类。首先展示一下NioEventLoopGroup的类图：

![NioEventLoopGroup]({{ "/assets/img/netty/NioEventLoopGroup.png" | relative_url}})

可以发现，NioEventLoopGroup的实现非常的复杂，但是只要我们清楚了Netty的线程模型，我们就可以有入口去分析它的代码。首先，我们知道每个EventLoop只要一个Thread来处理事件，那我们就来找到那个Thread在什么地方。可以在SingleThreadEventExecutor类中找到thread，它的初始化在doStartThread这个方法中，而这个方法被startThread方法调用，而startThread 这个方法被execute方法调用，也就是提交任务的入口，这个方法是Executor接口的唯一方法。也就是说，所有我们通过EventLoop的execute方法提交的任务都将被这个Thread线程来执行。我们还知道一个事实，EventLoop是一个循环执行来消耗Channel事件的类，那么它必然会有一个类似循环的方法来作为任务，来提交给这个Thread来执行，而这可以在doStartThread方法中被发现

```java
private void doStartThread() {
       assert thread == null;
       executor.execute(new Runnable() {
           @Override
           public void run() {
               thread = Thread.currentThread();
               if (interrupted) {
                   thread.interrupt();
               }

               boolean success = false;
               updateLastExecutionTime();
               try {
                   SingleThreadEventExecutor.this.run();
                   success = true;
               } catch (Throwable t) {
                   logger.warn("Unexpected exception from an event executor: ", t);
               } finally {
                   for (;;) {
                       int oldState = state;
                       if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
                               SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
                           break;
                       }
                   }

                   try {
                       // Run all remaining tasks and shutdown hooks.
                       for (;;) {
                           if (confirmShutdown()) {
                               break;
                           }
                       }
                   } finally {
                       try {
                           cleanup();
                       } finally {
                           STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
                           threadLock.release();
                           terminationFuture.setSuccess(null);
                       }
                   }
               }
           }
       });
   }
```

上面所提到的事件循环就是通过SingleThreadEventExecutor.this.run()这句话来触发的。这个run方法的具体实现在NioEventLoop中，下面展示了它的实现代码：

```java
protected void run() {
        for (;;) {
            try {
                switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                    case SelectStrategy.CONTINUE:
                        continue;
                    case SelectStrategy.SELECT:
                        select(wakenUp.getAndSet(false));
                        if (wakenUp.get()) {
                            selector.wakeup();
                        }
                        // fall through
                    default:
                }

                cancelledKeys = 0;
                needsToSelectAgain = false;
                final int ioRatio = this.ioRatio;
                if (ioRatio == 100) {
                    try {
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        runAllTasks();
                    }
                } else {
                    final long ioStartTime = System.nanoTime();
                    try {
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        final long ioTime = System.nanoTime() - ioStartTime;
                        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
            // Always handle shutdown even if the loop processing threw an exception.
            try {
                if (isShuttingDown()) {
                    closeAll();
                    if (confirmShutdown()) {
                        return;
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
        }
    }
```
首先，我们来分析一下NioEventLoop的相关细节，在一个无限循环里面，只有在遇到shutdown的情况下才会停止循环。然后在循环里会询问是否有事件，如果没有，则继续循环，如果有事件，那么就开始处理时间。上文中我们提到，在事件循环中我们不仅要处理IO事件，还要处理非I/O事件。Netty中可以设置用于I/O操作和非I/O操作的时间占比，默认各位50%，也就是说，如果某次I/O操作的时间花了100ms，那么这次循环中非I/O得任务也可以花费100ms。Netty中的I/O时间处理通过processSelectedKeys方法来进行，而非I/O操作通过runAllTasks反复来进行，首先来看runAllTasks方法，虽然设定了一个可以运行的时间参数，但是实际上Netty并不保证能精确的确保非I/O任务只运行设定的毫秒，下面来看下runAllTasks的代码：

```java
protected boolean runAllTasks(long timeoutNanos) {
       fetchFromScheduledTaskQueue();
       Runnable task = pollTask();
       if (task == null) {
           afterRunningAllTasks();
           return false;
       }

       final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos;
       long runTasks = 0;
       long lastExecutionTime;
       for (;;) {
           safeExecute(task);

           runTasks ++;

           // Check timeout every 64 tasks because nanoTime() is relatively expensive.
           // XXX: Hard-coded value - will make it configurable if it is really a problem.
           if ((runTasks & 0x3F) == 0) {
               lastExecutionTime = ScheduledFutureTask.nanoTime();
               if (lastExecutionTime >= deadline) {
                   break;
               }
           }

           task = pollTask();
           if (task == null) {
               lastExecutionTime = ScheduledFutureTask.nanoTime();
               break;
           }
       }

       afterRunningAllTasks();
       this.lastExecutionTime = lastExecutionTime;
       return true;
   }

   // 将任务运行起来
   protected static void safeExecute(Runnable task) {
       try {
           task.run();
       } catch (Throwable t) {
           logger.warn("A task raised an exception. Task: {}", task, t);
       }
   }
```

可以看到，这个方法是在每运行了64个任务之后再进行比较的，如果超出了设定的运行时间则退出，否则再运行64个任务再比较。所以，Netty强烈要求不要在I/O线程中运行阻塞任务，因为阻塞任务将会阻塞住Netty的事件循环，从而造成事件堆积的现象。现在回头看处理I/O任务的processSelectedKeys方法，跟踪代码之后发现最后实际处理I/O事件的一个方法为processSelectedKey，下面展示了它的代码：

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
        final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
        if (!k.isValid()) {
            final EventLoop eventLoop;
            try {
                eventLoop = ch.eventLoop();
            } catch (Throwable ignored) {
                // If the channel implementation throws an exception because there is no event loop, we ignore this
                // because we are only trying to determine if ch is registered to this event loop and thus has authority
                // to close ch.
                return;
            }
            // Only close ch if ch is still registered to this EventLoop. ch could have deregistered from the event loop
            // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
            // still healthy and should not be closed.
            // See https://github.com/netty/netty/issues/5125
            if (eventLoop != this || eventLoop == null) {
                return;
            }
            // close the channel if the key is not valid anymore
            unsafe.close(unsafe.voidPromise());
            return;
        }

        try {
            int readyOps = k.readyOps();
            // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
            // the NIO JDK channel implementation may throw a NotYetConnectedException.
            if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
                // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
                // See https://github.com/netty/netty/issues/924
                int ops = k.interestOps();
                ops &= ~SelectionKey.OP_CONNECT;
                k.interestOps(ops);

                unsafe.finishConnect();
            }

            // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
            if ((readyOps & SelectionKey.OP_WRITE) != 0) {
                // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
                ch.unsafe().forceFlush();
            }

            // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
            // to a spin loop
            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
            }
        } catch (CancelledKeyException ignored) {
            unsafe.close(unsafe.voidPromise());
        }
    }
```

这个方法运行的流程为：

1、从Channel上获取一个unsafe对象，这个对象 是用来进行NIO操作的一系列系统级API，关于Netty的Channel的深层次分析将在另外的篇章中进行
2、从Channel上获取了eventLoop，而这个eventLoop是什么时候分配给Channel的细节在后文中进行分析
3、根据事件调用底层API来处理事件

下面，我们分析一下是什么时候将一个EventLoop分配给一个Channel的，并且这个EventLoop的那个唯一的Thread是什么时候被赋值的。在这个问题上，服务端的流程和客户端的流程可能不太一样，对于服务端来说，首先需要bind一个端口，然后在进行Accept进来的连接，而客户端需要进行connect到服务端。先来分析一下服务端。

```java
// Start the server.
ChannelFuture f = b.bind(PORT).sync();

```

也就是我们网络编程中的bind操作，这个操作会发生什么呢？追踪代码如下：

```java
-> AbstractBootstrap.bind(port)
 -> AbstractBootstrap.bind(address)
 -> AbstractBootstrap.doBind(final SocketAddress localAddress)
 -> AbstractBootstrap.initAndRegister
 -> AbstractBootstrap.doBind0
 -> SingleThreadEventExecutor.execute
 -> SingleThreadEventExecutor.startThread()
 -> SingleThreadEventExecutor.doStartThread
```
EventLoop在AbstractBootstrap.initAndRegister中获得了一个新的Channel，然后在AbstractBootstrap.doBind0 方法里面调用接下来的方法来初始化EventLoop的Thread的工作，并且将EventLoop的时间循环打开了，可以开始接收客户端的连接请求了。下面来分析一下客户端的流程。

```java
// Configure the client.
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();
            b.group(group)
             .channel(NioSocketChannel.class)
             .option(ChannelOption.TCP_NODELAY, true)
             .handler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     if (sslCtx != null) {
                         p.addLast(sslCtx.newHandler(ch.alloc(), HOST, PORT));
                     }
                     //p.addLast(new LoggingHandler(LogLevel.INFO));
                     p.addLast(new EchoClientHandler());
                 }
             });

            // Start the client.
            ChannelFuture f = b.connect(HOST, PORT).sync();

            // Wait until the connection is closed.
            f.channel().closeFuture().sync();
        } finally {
            // Shut down the event loop to terminate all threads.
            group.shutdownGracefully();
        }
```
其中关键代码行

```java
// Start the client.
 ChannelFuture f = b.connect(HOST, PORT).sync();
```

下面是connect的调用流程：

```java
-> Bootstrap.doResolveAndConnect
 -> AbstractBootstrap.initAndRegister
 -> Bootstrap.doResolveAndConnect0
 -> Bootstrap.doConnect
 -> SingleThreadEventExecutor.execute
 -> SingleThreadEventExecutor.startThread()
 -> SingleThreadEventExecutor.doStartThread
```
后半部分和服务端的启动过程是一致的，而区别在于服务端是通过bind操作来启动的，而客户端是通过connect操作来启动的。执行到此，客户端和服务端的EventLoop都已经启动起来，服务端可以接受客户端的连接并且处理Channel上的读写事件，而客户端可以去连接远程服务端来请求数据。

还有个问题就是EventLoop是如何被分配给一个Channel的，阻塞IO和非阻塞IO是不一样的，只看NIO

上文中我们分析了EventLoop被启动的过程，我们肯定，EventLoop是在分配之后启动的，因为对于服务端而言，bind是一个最开始的网络操作，对于客户端来说，connect也是最开始的网络操作，在这之前是没有关于网络I/O的操作的，所以，EventLoop的分配和启动是在这两个过程或者之后的流程中进行的，但是EventLoop的分配肯定是在启动之前的，但是EventLoop的分配和启动在bind和connect中进行，那么我们可以肯定，EventLoop的分配也是在这两个方法中进行的。为了证明这个假设，回头再看一下服务端的EventLoop的启动过程，其中有一个方法值得我们注意：AbstractBootstrap.initAndRegister，我们进行了init部分的分析，而register部分我们还没有分析，下面就对服务端来进行register部分的分析，下面展示了register的调用链路：

```java
-> Bootstrap.doResolveAndConnect
 -> AbstractBootstrap.initAndRegister
 -> EventLoopGroup.register
 -> MultithreadEventLoopGroup.register
 -> SingleThreadEventLoop.register
 -> Channel.register
 -> AbstractUnsafe.register

         public final void register(EventLoop eventLoop, final ChannelPromise promise) {
            if (eventLoop == null) {
                throw new NullPointerException("eventLoop");
            }
            if (isRegistered()) {
                promise.setFailure(new IllegalStateException("registered to an event loop already"));
                return;
            }
            if (!isCompatible(eventLoop)) {
                promise.setFailure(
                        new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
                return;
            }

            AbstractChannel.this.eventLoop = eventLoop;

            if (eventLoop.inEventLoop()) {
                register0(promise);
            } else {
                try {
                    eventLoop.execute(new Runnable() {
                        @Override
                        public void run() {
                            register0(promise);
                        }
                    });
                } catch (Throwable t) {
                    logger.warn(
                            "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                            AbstractChannel.this, t);
                    closeForcibly();
                    closeFuture.setClosed();
                    safeSetFailure(promise, t);
                }
            }
        }
```

最后展示了AbstractUnsafe.register这个方法，在这里初始化了一个EventLoop，需要记住的一点是，EventLoopGroup中的是EventLoop，不然在追踪代码的时候会迷失。现在来正式看一下NioEventLoopGroup这个类，它的它继承了MultithreadEventExecutorGroup这个类，而我们在初始化EventLoopGroup的时候传递进去的参数，也就是我们希望这个EventLoopGroup拥有的EventLoop数量，会在MultithreadEventExecutorGroup这个类中初始化，并且是在构造函数中初始化的，如果在new EventLoopGroup的时候没有任何参数，那么默认的EventLoop的数量是机器CPU数量的两倍。现在我们来看一下MultithreadEventExecutorGroup这个类的一个重要的构造函数，这个构造函数初始化了EventLoopGroup的EventLoop。

```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
        if (nThreads <= 0) {
            throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
        }

        if (executor == null) {
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }

        children = new EventExecutor[nThreads];

        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                children[i] = newChild(executor, args);
                success = true;
            } catch (Exception e) {
                // TODO: Think about if this is a good exception type
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
                if (!success) {
                    for (int j = 0; j < i; j ++) {
                        children[j].shutdownGracefully();
                    }

                    for (int j = 0; j < i; j ++) {
                        EventExecutor e = children[j];
                        try {
                            while (!e.isTerminated()) {
                                e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                            }
                        } catch (InterruptedException interrupted) {
                            // Let the caller handle the interruption.
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                }
            }
        }

        chooser = chooserFactory.newChooser(children);

        final FutureListener<Object> terminationListener = new FutureListener<Object>() {
            @Override
            public void operationComplete(Future<Object> future) throws Exception {
                if (terminatedChildren.incrementAndGet() == children.length) {
                    terminationFuture.setSuccess(null);
                }
            }
        };

        for (EventExecutor e: children) {
            e.terminationFuture().addListener(terminationListener);
        }

        Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
        Collections.addAll(childrenSet, children);
        readonlyChildren = Collections.unmodifiableSet(childrenSet);
    }
```

一个较为重要的方法为newChild，这是初始化一个EventLoop的方法，下面是它的具体实现，假设我们使用NioEventLoop：

```java
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
       return new NioEventLoop(this, executor, (SelectorProvider) args[0],
           ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2]);
   }
```

我们现在知道了EventLoopGroup管理着很多的EventLoop，上文中我们仅仅分析了分配的流程，但是分配的策略还没有分析，现在来分析一下EventLoopGroup是如何分配EventLoop给Channel的，我们仅分析非阻塞I/O下的分配策略，阻塞模式下的分配策略可以参考非阻塞下的分配策略。

在MultithreadEventLoopGroup.register方法中，调用了next()方法，我们来看一下这个流程：

```java
-> MultithreadEventExecutorGroup.next()

    public EventExecutor next() {
        return chooser.next();
    }
```

chooser是什么东西？

```java
private final EventExecutorChooserFactory.EventExecutorChooser chooser;

```

它是怎么初始化的呢？

```java
public EventExecutorChooser newChooser(EventExecutor[] executors) {
        if (isPowerOfTwo(executors.length)) {
            return new PowerOfTwoEventExecutorChooser(executors);
        } else {
            return new GenericEventExecutorChooser(executors);
        }
    }

```

这是它初始化最后调用的方法，这个方法在DefaultEventExecutorChooserFactory中被实现，这个参数是MultithreadEventExecutorGroup类中的children，也就是EventLoopGroup中的所有EventLoop，那这个newChooser得分配方法就是如果EventLoop的数量是2的n次方，那么就使用PowerOfTwoEventExecutorChooser来分配，否则使用GenericEventExecutorChooser来分配。这两个策略类的分配方法实现分别如下：

```java
1、PowerOfTwoEventExecutorChooser
    public EventExecutor next() {
        return executors[idx.getAndIncrement() & executors.length - 1];
    }

2、GenericEventExecutorChooser
    public EventExecutor next() {
        return executors[Math.abs(idx.getAndIncrement() % executors.length)];
    }
```

所以，到此为止，我们可以解决为什么一个EventLoop会被分配给多个Channel的疑惑。本文到此也就结束了。


#### Tcp粘包/拆包问题的解决

熟悉TCP编程的话，无论是服务器端还是客户端，当我们读取或者发送消息的时候，都需要考虑TCP底层的粘包/拆包机制

什么是TCP的粘包/拆包呢，TCP是一个"流"协议，就是没有界限的一串数据，TCP底层并不知道上层业务数据的具体含义，它会根据TCP缓冲区的实际情况进行包的划分，所以在业务上认为的一个完整的包，可能会被TCP拆成多个包进行发送，也有可能把多个小的包封装成一个大的数据包发送。这就是所谓的TCP的粘包/拆包

问题产生的原因有三个：

1、应用程序write写入的字节大小大于套接口发送缓冲区大小
2、进行MSS大小的TCP分段
3、以太网帧的payload大于MTU进行IP分片

粘包的解决方法：

1、消息定长，例如每个报文的大小为固定长度200字节，如果不够，用空格步
2、在包尾增加回车换行符进行分割，例如FTP协议
3、将消息分为消息头和消息体。消息头中包含表示消息总长度（或者消息体长度）的字段，通常设计思路为消息头的第一个字段用int32位来表示消息的总长度
4、更复杂的应用层协议

消息次数上的错误案例

Netty是如何解决的

为了解决半包读写问题，Netty默认提供了多种编解码器用于处理半包，只要能熟练掌握这些类库的使用，很轻松能解决，这是其他NIO框架或者JAVA原生所不能解决的

使用LineBasedFrameDecoder和StringDecoder进行解决，而且只是加了这两个解码器

除了这两个解码器，netty还对上面4种解决方案进行了抽象，提供了其他解码器

DelimiterBasedFrameDecoder和FixedLengthFrameDecoder

看名字应该就知道他们对应的是哪种解决方法了吧


#### 编解码技术

Java序列化的缺点，从1.1版本就支持了java.io.Serializable并生成序列ID即可，但是在远程服务调用（PRC）时，很少直接使用java序列化进行消息的编解码和传输，为什么呢？

1、无法跨语言

对用户完全黑盒的，让C++怎么去解决反序列化的问题

2、序列化后的码流太大

采用JDK序列化机制编码后的二进制数组大小竟然是二进制编码的5.29倍

3、序列化性能太低

性能较低，也就是耗时很长

所以业界主流的编解码框架有：

1、Google的Protobuf（因为是二进制的编码，所以性能上有优势），看上去像是最好的

2、Facebook的Thrift，语言支持的很多

3、JBOSS Marshalling，修正了JDK自带的序列化包的很多问题，但又保持跟java.io.Serializable接口兼容

MessagePack编解码器，提供多种语言的支持，API使用非常简单

Protobuf编解码器，相比于传统的序列化工具，它更小，更快，更简单
