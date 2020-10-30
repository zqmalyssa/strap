---
layout: post
title: JAVA的NIO
tags: [code, java]
author-id: zqmalyssa
---

Java中的NIO，NIO即新的输入输出，这个库是在JDK1.4中才引入的。它在标准java代码中提供了高速的面向块的IO操作。


#### 基本概述

I/O即输入输出，是计算机与外界世界的一个借口。IO操作的实际主题是操作系统。在java编程中，一般使用流的方式来处理IO，所有的IO都被视作是单个字节的移动，通过stream对象一次移动一个字节。流IO负责把对象转换为字节，然后再转换为对象。

NIO即New IO，这个库是在JDK1.4中才引入的。NIO和IO有相同的作用和目的，但实现方式不同，NIO主要用到的是块，所以NIO的效率要比IO高很多。

在Java API中提供了两套NIO，一套是针对标准输入输出NIO，另一套就是网络编程NIO，先看标准NIO，NIO和IO最大的区别是数据打包和传输方式。IO是以流的方式处理数据，而NIO是以块的方式处理数据。

1. 面向流的IO一次一个字节的处理数据，一个输入流产生一个字节，一个输出流就消费一个字节。为流式数据创建过滤器就变得非常容易，链接几个过滤器，以便对数据进行处理非常方便而简单，但是面向流的IO通常处理的很慢。
2. 面向块的IO系统以块的形式处理数据。每一个操作都在一步中产生或消费一个数据块。按块要比按流快的多，但面向块的IO缺少了面向流IO所具有的有雅兴和简单性。


#### NIO基础

Buffer和Channel是标准NIO中的核心对象（网络NIO中还有个Selector核心对象，具体请参考Java NIO网络详解），几乎每一个IO操作中都会用到它们。Channel是对原IO中流的模拟，任何来源和目的数据都必须通过一个Channel对象。一个Buffer实质上是一个容器对象，发给Channel的所有对象都必须先放到Buffer中；同样的，从Channel中读取的任何数据都要读到Buffer中。

关于Buffer，Buffer是一个对象，它包含一些要写入或读出的数据。在NIO中，数据是放入buffer对象的，而在IO中，数据是直接写入或者读到Stream对象的。应用程序不能直接对 Channel 进行读写操作，而必须通过 Buffer 来进行，即 Channel 是通过 Buffer 来读写数据的。在NIO中，所有的数据都是用Buffer处理的，它是NIO读写数据的中转池。Buffer实质上是一个数组，通常是一个字节数据，但也可以是其他类型的数组。但一个缓冲区不仅仅是一个数组，重要的是它提供了对数据的结构化访问，而且还可以跟踪系统的读写进程。

使用 Buffer 读写数据一般遵循以下四个步骤：

1. 写入数据到 Buffer
2. 调用 flip() 方法
3. 从 Buffer 中读取数据
4. 调用 clear() 方法或者 compact() 方法

当向 Buffer 写入数据时，Buffer 会记录下写了多少数据。一旦要读取数据，需要通过 flip() 方法将 Buffer 从写模式切换到读模式。在读模式下，可以读取之前写入到 Buffer 的所有数据。一旦读完了所有的数据，就需要清空缓冲区，让它可以再次被写入。有两种方式能清空缓冲区：调用 clear() 或 compact() 方法。clear() 方法会清空整个缓冲区。compact() 方法只会清除已经读过的数据。任何未读的数据都被移到缓冲区的起始处，新写入的数据将放到缓冲区未读数据的后面。

关于Channel，Channel是一个对象，可以通过它读取和写入数据。可以把它看做IO中的流。但是它和流相比还有一些不同：

1. Channel是双向的，既可以读又可以写，而流是单向的
2. Channel可以进行异步的读写
3. 对Channel的读写必须通过buffer对象

正如上面提到的，所有数据都通过Buffer对象处理，所以，您永远不会将字节直接写入到Channel中，相反，您是将数据写入到Buffer中；同样，您也不会从Channel中读取字节，而是将数据从Channel读入Buffer，再从Buffer获取这个字节。

因为Channel是双向的，所以Channel可以比流更好地反映出底层操作系统的真实情况。特别是在Unix模型中，底层操作系统通常都是双向的。在Java NIO中Channel主要有如下几种类型：

1. FileChannel：从文件读取数据的
2. DatagramChannel：读写UDP网络协议数据
3. SocketChannel：读写TCP网络协议数据
4. ServerSocketChannel：可以监听TCP连接

IO中的读和写，对应的是数据和Stream，NIO中的读和写，则对应的就是通道和缓冲区。NIO中从通道中读取：创建一个缓冲区，然后让通道读取数据到缓冲区。NIO写入数据到通道：创建一个缓冲区，用数据填充它，然后让通道用这些数据来执行写入。

**从文件中读取**
在NIO系统中，任何时候执行一个读操作，您都是从Channel中读取，而您不是直接从Channel中读取数据，因为所有的数据都必须用Buffer来封装，所以您应该是从Channel读取数据到Buffer，因此，从文件读取数据的话，需要三步：

1. 从FileInputStream获取Channel
2. 创建Buffer
3. 从Channel读取数据到Buffer

**数据写入文件**

1. 从FileOutputStream获取Channel
2. 创建Buffer
3. buffer的数据写入Channel

下面的代码是Sample

```java
package com.qiming.test.nio;

import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

public class NioStandard {

  public static void main(String[] args) {
    String src = "E:\\Code\\NioTestSample\\src.txt";
    String dst = "E:\\Code\\NioTestSample\\dst.txt";
    try {
      copyFileUseNio(src, dst);
    } catch (IOException e) {
      e.printStackTrace();
    }
  }

  public void ReadFromFileByNio() throws IOException {
    //第一步，获取通道
    FileInputStream fin = new FileInputStream("123.txt");
    FileChannel fc = fin.getChannel();

    //第二步，创建缓冲区
    ByteBuffer buffer = ByteBuffer.allocate(1024);

    //第三步，将数据从通道读到缓冲区
    fc.read(buffer);

  }


  public void WriteToFileByNio() throws IOException {
    //第一步，获取通道
    FileOutputStream fout = new FileOutputStream("123.txt");
    FileChannel fc = fout.getChannel();

    //第二步，创建缓冲区，将数据放入缓冲区
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    Byte[] message = new Byte[10];
    for (int i = 0; i < message.length; i++) {
      buffer.put(message[i]);
    }
    buffer.flip();

    //第三步，将缓冲区数据写入通道
    fc.write(buffer);
  }

  public static void copyFileUseNio(String src, String dst) throws IOException{
    //声明源文件和目标文件
    FileInputStream fi = new FileInputStream(new File(src));
    FileOutputStream fo = new FileOutputStream(new File(dst));

    //获取传输通道
    FileChannel inChannel = fi.getChannel();
    FileChannel outChannel = fo.getChannel();

    //获得容器buffer
    ByteBuffer buffer = ByteBuffer.allocate(1024);

    while(true) {
      //判断是否读完文件
      int eof = inChannel.read(buffer);
      if (eof == -1) {
        break;
      }
      //重设一下buffer的position=0，limit=position
      buffer.flip();
      //开始写
      outChannel.write(buffer);
      //写完要重置buffer，重设position=0, limit=capacity
      buffer.clear();
    }
    inChannel.close();
    outChannel.close();
    fi.close();
    fo.close();
  }

}
```

Buffer类的flip和clear方法这边说下，控制buffer状态的三个变量

1. position：跟踪已经写了多少数据或读了多少数据，它指向的是下一个字节来自哪个位置
2. limit：代表还有多少数据可以取出或还有多少空间可以写入，它的值小于等于capacity
3. capacity：代表缓冲区的最大容量，一般新建一个缓冲区的时候，limit的值和capacity的值默认是相等的

flip、clear这两个方法便是用来设置这些值的

`buffer.flip()`方法，这个方法把当前的指针位置position设置成了limit，再将当前指针position指向数据的最开始端，我们现在可以将数据从缓冲区写入通道了。这段区间就是可以写的数据

`buffer.clear()`方法，这个方法在写入数据之后也就是读数据之前调用，重设缓冲区也便接收更多的字节

#### 网络NIO

这里介绍基于网络编程NIO（多路复用IO），最好不要说是异步IO，N是非阻塞的意思，下面所有的异步说法有待商榷，在JDK1.4推出JAVA NIO之前，基于Java的所有Socket通信都采用了同步阻塞模式（BIO），这种一请求一应答的通信模型简化了上层的应用开发，但是在性能和可靠性方面却存在着巨大的瓶颈

异步 I/O 是一种没有阻塞地读写数据的方法。通常，在代码进行 read() 调用时，代码会阻塞直至有可供读取的数据。同样， write()调用将会阻塞直至数据能够写入，阻塞这种操作就是像多线程一样，用FutureTask去拿数据的时候也会阻塞。

另一方面，异步 I/O 调用不但不会阻塞，相反，您可以注册对特定 I/O 事件诸如数据可读、新连接到来等等，而在发生这样感兴趣的事件时，系统将会告诉您。

异步 I/O 的一个优势在于，它允许您同时根据大量的输入和输出执行 I/O。同步程序常常要求助于轮询，或者创建许许多多的线程以处理大量的连接。使用异步 I/O，您可以监听任何数量的通道上的事件，不用轮询，也不用额外的线程。

除了StandardNio中的Channel和Buffer外，还有一个核心对象是Selector，Selector是一个对象，它可以注册到很多个Channel上，监听各个Channel上发生的事件，并且能够根据事件情况决定Channel读写。这样，通过一个线程管理多个Channel，就可以处理大量网络连接了

采用Selector模式的的好处是，有了Selector，我们就可以利用一个线程来处理所有的channels。线程之间的切换对操作系统来说代价是很高的，并且每个线程也会占用一定的系统资源。所以，对系统来说使用的线程越少越好。也就是说Selector能够处理多个通道Channel

这在其他地方（fbsfwkjylysx）可以理解成I/O多路复用技术，就是通过把多个I/O的阻塞复用到同一个select的阻塞上，从而使得系统在单线程的情况下可以同时处理多个客户端请求，与传统的多线程多进程模型相比，其最大优势就是系统开销小，JDK1.5中用epoll替代了传统的select/poll，极大提高了NIO通信性能，Java自身的Nio被诟病，所以出来了Netty

![selectormodel]({{ "/assets/img/java/selectormodel.png" | relative_url}})

Selector 就是您注册对各种 I/O 事件兴趣的地方，而且当那些事件发生时，就是这个对象告诉您所发生的事件。

```java
Selector selector = Selector.open();
```
然后就需要注册Channel到Selector了，我们需要把Channel注册到Selector上，通过调用channel.register()方法实现注册
```java
channel.configureBlocking(false);
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
```
注册的Channel 必须设置成异步模式 才可以,，否则异步IO就无法工作，这就意味着我们不能把一个FileChannel注册到Selector，因为FileChannel没有异步模式，但是网络编程中的SocketChannel是可以的，需要注意register()方法的第二个参数，它是一个“interest set”,意思是注册的Selector对Channel中的哪些时间感兴趣，事件类型有四种：
1. Connect
2. Accept
3. Read
4. Write

通道触发了一个事件意思是该事件已经 Ready(就绪)。所以，某个Channel成功连接到另一个服务器称为 Connect Ready。一个ServerSocketChannel准备好接收新连接称为 Accept Ready，一个有数据可读的通道可以说是 Read Ready，等待写数据的通道可以说是Write Ready。上面这4种状态对应`SelectionKey`的四个常量：`OP_CONNECT`,`OP_ACCEPT`,`OP_READ`,`OP_WRITE`

如果你对多个事件感兴趣，可以通过or操作符来连接这些常量：

```java
int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
```
请注意对register()的调用的返回值是一个SelectionKey。 SelectionKey 代表这个通道在此 Selector 上的这个注册。当某个 Selector 通知您某个传入事件时，它是通过提供对应于该事件的 SelectionKey 来进行的。SelectionKey 还可以用于取消通道的注册。SelectionKey中包含如下属性：
1. The interest set
2. The ready set
3. The Channel
4. The Selector
5. An attached object (optional)

就像我们在前面讲到的把Channel注册到Selector来监听感兴趣的事件，interest set就是你要选择的感兴趣的事件的集合。你可以通过SelectionKey对象来读写interest set:

```java
int interestSet = selectionKey.interestOps();
boolean isInterestedInAccept  = interestSet & SelectionKey.OP_ACCEPT;
boolean isInterestedInConnect = interestSet & SelectionKey.OP_CONNECT;
boolean isInterestedInRead    = interestSet & SelectionKey.OP_READ;
boolean isInterestedInWrite   = interestSet & SelectionKey.OP_WRITE;   
```
通过上面例子可以看到，我们可以通过用AND 和SelectionKey 中的常量做运算，从SelectionKey中找到我们感兴趣的事件。

ready set 是通道已经准备就绪的操作的集合。在一次选Selection之后，你应该会首先访问这个ready set。Selection将在下一小节进行解释。可以这样访问ready集合：

```java
int readySet = selectionKey.readyOps();
```
可以用像检测interest集合那样的方法，来检测Channel中什么事件或操作已经就绪。但是，也可以使用以下四个方法，它们都会返回一个布尔类型：
```java
selectionKey.isAcceptable();
selectionKey.isConnectable();
selectionKey.isReadable();
selectionKey.isWritable();
```
Channel和Selector意思是我们可以通过SelectionKey获得Selector和注册的Channel：

```java
Channel  channel  = selectionKey.channel();
Selector selector = selectionKey.selector();
```

attach是将一个对象或者更多信息attach 到SelectionKey上，这样就能方便的识别某个给定的通道。例如，可以附加 与通道一起使用的Buffer，或是包含聚集数据的某个对象。使用方法如下：

```java
selectionKey.attach(theObject);
Object attachedObj = selectionKey.attachment();
```

**通过Selector选择通道**

一旦向Selector注册了一或多个通道，就可以调用几个重载的select()方法。这些方法返回你所感兴趣的事件（如连接、接受、读或写）已经准备就绪的那些通道。换句话说，如果你对“Read Ready”的通道感兴趣，select()方法会返回读事件已经就绪的那些通道：

1. int select()： 阻塞到至少有一个通道在你注册的事件上就绪
2. int select(long timeout)：select()一样，除了最长会阻塞timeout毫秒(参数)
3. int selectNow()： 不会阻塞，不管什么通道就绪都立刻返回，此方法执行非阻塞的选择操作。如果自从前一次选择操作后，没有通道变成可选择的，则此方法直接返回零

select()方法返回的int值表示有多少通道已经就绪。亦即，**自上次调用select()方法后有多少通道变成就绪状态**。如果调用select()方法，因为有一个通道变成就绪状态，返回了1，若再次调用select()方法，如果另一个通道就绪了，它会再次返回1。如果对第一个就绪的channel没有做任何操作，现在就有两个就绪的通道，但在每次select()方法调用之间，只有一个通道处于就绪状态。

一旦调用了select()方法，它就会返回一个数值，表示一个或多个通道已经就绪，然后你就可以通过调用selector.selectedKeys()方法返回的SelectionKey集合来获得就绪的Channel。当你通过Selector注册一个Channel时，channel.register()方法会返回一个SelectionKey对象，这个对象就代表了你注册的Channel。这些对象可以通过selectedKeys()方法获得。你可以通过迭代这些selected key来获得就绪的Channel，下面是演示代码：

```java
Set<SelectionKey> selectedKeys = selector.selectedKeys();
Iterator<SelectionKey> keyIterator = selectedKeys.iterator();
while(keyIterator.hasNext()) {
SelectionKey key = keyIterator.next();
if(key.isAcceptable()) {
    // a connection was accepted by a ServerSocketChannel.
} else if (key.isConnectable()) {
    // a connection was established with a remote server.
} else if (key.isReadable()) {
    // a channel is ready for reading
} else if (key.isWritable()) {
    // a channel is ready for writing
}
keyIterator.remove();
}
```
这个循环遍历selected key的集合中的每个key，并对每个key做测试来判断哪个Channel已经就绪。请注意循环中最后的keyIterator.remove()方法。Selector对象并不会从自己的selected key集合中自动移除SelectionKey实例。我们需要在处理完一个Channel的时候自己去移除。当下一次Channel就绪的时候，Selector会再次把它添加到selected key集合中。SelectionKey.channel()方法返回的Channel需要转换成你具体要处理的类型，比如是ServerSocketChannel或者SocketChannel等等。

`WakeUp()`和`Close()`是某个线程调用select()方法后阻塞了，即使没有通道就绪，也有办法让其从select()方法返回。只要让其它线程在第一个线程调用select()方法的那个对象上调用Selector.wakeup()方法即可。阻塞在select()方法上的线程会立马返回。如果有其它线程调用了wakeup()方法，但当前没有线程阻塞在select()方法上，下个调用select()方法的线程会立即“醒来（wake up）”，当用完Selector后应调用close()方法，它将关闭Selector并且使注册到该Selector上的所有SelectionKey实例无效。通道本身并不会关闭。下面看个例子

```java
package com.qiming.test.nio;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;

public class NioNetwork {

  private int ports[];
  private ByteBuffer echoBuffer = ByteBuffer.allocate(1024);

  public static void main(String[] args1) throws Exception {
    String args[]={"9001","9002","9003"};
    if (args.length <= 0) {
      System.err.println("Usage: java MultiPortEcho port [port port ...]");
      System.exit(1);
    }
    int ports[] = new int[args.length];
    for (int i = 0; i < args.length; ++i) {
      ports[i] = Integer.parseInt(args[i]);
    }
    new NioNetwork(ports);
  }

  public NioNetwork(int ports[]) throws IOException {
    this.ports = ports;
    go();
  }
  private void go() throws IOException {
    // 1. 创建一个selector，select是NIO中的核心对象
    // 它用来监听各种感兴趣的IO事件
    Selector selector = Selector.open();
    // 为每个端口打开一个监听, 并把这些监听注册到selector中
    for (int i = 0; i < ports.length; ++i) {
      //2. 打开一个ServerSocketChannel
      //其实我们没监听一个端口就需要一个channel
      ServerSocketChannel ssc = ServerSocketChannel.open();
      ssc.configureBlocking(false);//设置为非阻塞
      ServerSocket ss = ssc.socket();
      InetSocketAddress address = new InetSocketAddress(ports[i]);
      ss.bind(address);//监听一个端口
      //3. 注册到selector
      //register的第一个参数永远都是selector
      //第二个参数是我们要监听的事件
      //OP_ACCEPT是新建立连接的事件
      //也是适用于ServerSocketChannel的唯一事件类型
      SelectionKey key = ssc.register(selector, SelectionKey.OP_ACCEPT);
      System.out.println("Going to listen on " + ports[i]);
    }
    //4. 开始循环，我们已经注册了一些IO兴趣事件
    while (true) {
      //这个方法会阻塞，直到至少有一个已注册的事件发生。当一个或者更多的事件发生时
      // select() 方法将返回所发生的事件的数量。
      int num = selector.select();
      //返回发生了事件的 SelectionKey 对象的一个 集合
      Set selectedKeys = selector.selectedKeys();
      //我们通过迭代 SelectionKeys 并依次处理每个 SelectionKey 来处理事件
      //对于每一个 SelectionKey，您必须确定发生的是什么 I/O 事件，以及这个事件影响哪些 I/O 对象。
      Iterator it = selectedKeys.iterator();
      while (it.hasNext()) {
        SelectionKey key = (SelectionKey) it.next();
        //5. 监听新连接。程序执行到这里，我们仅注册了 ServerSocketChannel
        //并且仅注册它们“接收”事件。为确认这一点
        //我们对 SelectionKey 调用 readyOps() 方法，并检查发生了什么类型的事件
        if ((key.readyOps() & SelectionKey.OP_ACCEPT) == SelectionKey.OP_ACCEPT) {
          //6. 接收了一个新连接。因为我们知道这个服务器套接字上有一个传入连接在等待
          //所以可以安全地接受它；也就是说，不用担心 accept() 操作会阻塞
          ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
          SocketChannel sc = ssc.accept();
          sc.configureBlocking(false);
          // 7. 讲新连接注册到selector。将新连接的 SocketChannel 配置为非阻塞的
          //而且由于接受这个连接的目的是为了读取来自套接字的数据，所以我们还必须将 SocketChannel 注册到 Selector上
          SelectionKey newKey = sc.register(selector,SelectionKey.OP_READ);
          it.remove();
          System.out.println("Got connection from " + sc);
        } else if ((key.readyOps() & SelectionKey.OP_READ) == SelectionKey.OP_READ) {
          // Read the data
          SocketChannel sc = (SocketChannel) key.channel();
          // Echo data
          int bytesEchoed = 0;
          while (true) {
            echoBuffer.clear();
            int r = sc.read(echoBuffer);
            if (r <= 0) {
              break;
            }
            echoBuffer.flip();
            sc.write(echoBuffer);
            bytesEchoed += r;
          }
          System.out.println("Echoed " + bytesEchoed + " from " + sc);
          it.remove();
        }
      }
      // System.out.println( "going to clear" );
      // selectedKeys.clear();
      // System.out.println( "cleared" );
    }
  }

}
```
