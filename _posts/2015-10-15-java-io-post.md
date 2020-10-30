---
layout: post
title: JAVA的IO
tags: [code, java]
author-id: zqmalyssa
---

#### JAVA IO概述

Java中的IO，即Java 输入输出系统。不管我们编写何种应用，都难免和各种输入输出相关的媒介打交道，其实和媒介进行IO的过程是十分复杂的，这要考虑的因素特别多，比如我们要考虑和哪种媒介进行IO（文件、控制台、网络），我们还要考虑具体和它们的通信方式（顺序、随机、二进制、按字符、按字、按行等等）。Java类库的设计者通过设计大量的类来攻克这些难题，这些类就位于java.io包中

在JDK1.4之后，为了提高Java IO的效率，Java又提供了一套新的IO，Java New IO简称Java NIO。它在标准java代码中提供了高速的面向块的IO操作。下面会有NIO的介绍。

**流**
在Java IO中，流是一个核心的概念。流从概念上来说是一个连续的数据流。你既可以从流中读取数据，也可以往流中写数据。流与数据源或者数据流向的媒介相关联。在Java IO中流既可以是字节流(以字节为单位进行读写)，也可以是字符流(以字符为单位进行读写)。字符流和字节流的区别见下图

![zifuzijie]({{ "/assets/img/java/zifuzijie.png" | relative_url}})

主要是字节流在操作时本身不会用到缓冲区（内存），是文件本身直接操作的，而字符流在操作时使用了缓冲区，通过缓冲区再操作文件。

**IO相关的媒介**
Java的IO包主要关注的是从原始数据源的读取以及输出原始数据到目标媒介。以下是最典型的数据源和目标媒介：
1. 文件
2. 管道
3. 网络连接
4. 内存缓存
5. System.in, System.out, System.error(注：Java标准输入、输出、错误输出)


#### JAVA IO类库框架

虽然java IO类库庞大，但总体来说其框架还是很清楚的。从是读媒介还是写媒介的维度看，Java IO可以分为：
1、输入流：InputStream和Reader
2、输出流：OutputStream和Writer
而从其处理流的类型的维度上看，Java IO又可以分为：
1、字节流：InputStream和OutputStream
2、字符流：Reader和Writer

看下面这张图吧

![javaio]({{ "/assets/img/java/java_io.png" | relative_url}})

可以看下表找到对应的媒介

| 媒体 | Input(字节) | Output(字节) | Input(字符) | Output(字符) |
| :----: | :----:  | :----: | :----: | :----: |
| Basic | InputStream | OutputStream | Reader / InputStreamReader | Writer / OutputStreamWriter |
| Arrays | ByteArrayInputStream | ByteArrayOutputStream | CharArrayReader | CharArrayWriter |
| Files | FileInputStream / RandomAccessFile | FileOutputStream / RandomAccessFile | FileReader | FileWriter |
| Pipes | PipedInputStream | PipedOutputStream | PipedReader | PipedWriter |
| Buffering | BufferedInputStream | BufferedOutputStream | BufferedReader | BufferedWriter |
| Filtering | FilterInputStream | FilterOutputStream | FilterReader | FilterWriter |
| Parsing | PushbackInputStream / StreamTokenizer |  | PushbackReader / LineNumberReader |  |
| Strings |  |  | StringReader | StringWriter |
| Data | DataInputStream | DataOutputStream |  |  |
| Data - Formatted |  | PrintStream |  | PrintWriter |
| Objects | ObjectInputStream | ObjectOutputStream |  |  |
| Utilities | SequenceInputStream |  |  |  |

通过上面的介绍我们已经知道，字节流对应的类应该是InputStream和OutputStream，而在我们实际开发中，我们应该根据不同的媒介类型选用相应的子类来处理。下面我们就用字节流来操作文件媒介：

**用字节流写文件**

```java
/**
   * 用字节流写文件
   * @throws IOException
   */
  public static void writeByteToFile() throws IOException {
    String hello= new String( "hello word!");
    byte[] byteArray= hello.getBytes();
    File file= new File( "d:/test.txt");
    //因为是用字节流来写媒介，所以对应的是OutputStream
    //又因为媒介对象是文件，所以用到子类是FileOutputStream
    OutputStream os= new FileOutputStream(file);
    os.write(byteArray);
    os.close();
  }
```
**用字节流读文件**

```java
/**
   * 用字节流读文件
   * @throws IOException
   */
  public static void readByteFromFile() throws IOException{
    File file= new File( "d:/test.txt");
    byte[] byteArray= new byte[(int)file.length()];
    //因为是用字节流来读媒介，所以对应的是InputStream
    //又因为媒介对象是文件，所以用到子类是FileInputStream
    InputStream is= new FileInputStream(file);
    int size= is.read(byteArray);
    System. out.println( "大小:"+size +";内容:" +new String(byteArray));
    is.close();
  }
```
同样，字符流对应的类应该是Reader和Writer。下面我们就用字符流来操作文件媒介:

**用字符流写文件**

```java
/**
   * 用字符流写文件
   * @throws IOException
   */
  public static void writeCharToFile() throws IOException{
    String hello= new String( "hello word!");
    File file= new File( "d:/test.txt");
    //因为是用字符流来读媒介，所以对应的是Writer，又因为媒介对象是文件，所以用到子类是FileWriter
    Writer os= new FileWriter(file);
    os.write(hello);
    os.close();
  }
```

**用字符流读文件**

```java
/**
   * 用字符流读文件
   * @throws IOException
   */
  public static void readCharFromFile() throws IOException{
    File file= new File( "d:/test.txt");
    //因为是用字符流来读媒介，所以对应的是Reader
    //又因为媒介对象是文件，所以用到子类是FileReader
    Reader reader= new FileReader(file);
    char [] byteArray= new char[(int) file.length()];
    int size= reader.read(byteArray);
    System. out.println( "大小:"+size +";内容:" +new String(byteArray));
    reader.close();
  }
```

字节流可以转换成字符流，java.io包中提供的InputStreamReader类就可以实现，当然从其命名上就可以看出它的作用。其实这涉及到另一个概念，IO流的组合

**字节流转换为字符流**

什么时候使用转换流呢？
1、源或者目的对应的设备是字节流，但是操作的却是文本数据，可以使用转换作为桥梁，提高对文本操作的便捷。　　
2、一旦操作文本涉及到具体的指定编码表时，必须使用转换流。

```java
/**
   * 字节流转换成字符流
   * @throws IOException
   */
  public static void convertByteToChar() throws IOException{
    File file= new File( "d:/test.txt");
    //获得一个字节流
    InputStream is= new FileInputStream(file);
    //把字节流转换为字符流，其实就是把字符流和字节流组合的结果。
    Reader reader= new InputStreamReader(is);
    char [] byteArray= new char[(int) file.length()];
    int size= reader.read(byteArray);
    System. out.println( "大小:"+size +";内容:" +new String(byteArray));
    is.close();
    reader.close();
  }
```

再看下管道媒介的操作，管道主要用来实现同一个虚拟机中的两个线程进行交流。因此，一个管道既可以作为数据源媒介也可作为目标媒介，需要注意的是java中的管道和Unix/Linux中的管道含义并不一样，在Unix/Linux中管道可以作为两个位于不同空间进程通信的媒介，而在java中，管道只能为同一个JVM进程中的不同线程进行通信。和管道相关的IO类为：PipedInputStream和PipedOutputStream

```java
package com.qiming.test.io;

import java.io.IOException;
import java.io.PipedInputStream;
import java.io.PipedOutputStream;

/**
 * 读写管道，只能为同一个JVM进程中的不同线程进行通信。
 */
public class PipeExample {

  public static void main(String[] args) throws IOException {
    final PipedOutputStream output = new PipedOutputStream();
    final PipedInputStream  input  = new PipedInputStream(output);
    Thread thread1 = new Thread(new Runnable() {
      public void run() {
        try {
          output.write("Hello world, pipe!".getBytes());
        } catch (IOException e) {
        }
      }
    });
    Thread thread2 = new Thread(new Runnable() {
      public void run() {
        try {
          int data = input.read();
          while( data != -1){
            System. out.print((char) data);
            data = input.read();
          }
        } catch (IOException e) {
        } finally{
          try {
            input.close();
          } catch (IOException e) {
            e.printStackTrace();
          }
        }
      }
    });
    thread1.start();
    thread2.start();
  }


}
```

还有网络媒介，即Java 网络编程，其核心是Socket，同磁盘操作一样，java网络编程对应着两套API，即Java IO和Java NIO，这部分详聊

BufferedInputStream顾名思义，就是在对流进行写入时提供一个buffer来提高IO效率。在进行磁盘或网络IO时，原始的InputStream对数据读取的过程都是一个字节一个字节操作的，而BufferedInputStream在其内部提供了一个buffer，在读数据时，会一次读取一大块数据到buffer中，这样比单字节的操作效率要高的多，特别是进程磁盘IO和对大量数据进行读写的时候。

使用BufferedInputStream十分简单，只要把普通的输入流和BufferedInputStream组合到一起即可。我们把上面的例子改造成用BufferedInputStream进行读文件，请看下面例子：

**用缓冲字节流读文件**

```java
/**
   * 用缓冲字节流读文件
   * @throws IOException
   */
  public static void readByBufferedInputStream() throws IOException {
    File file = new File( "d:/test.txt");
    byte[] byteArray = new byte[(int) file.length()];
    //可以在构造参数中传入buffer大小
    InputStream is = new BufferedInputStream(new FileInputStream(file), 2*1024);
    int size = is.read(byteArray);
    System.out.println( "大小:" + size + ";内容:" + new String(byteArray));
    is.close();
  }
```
关于如何设置buffer的大小，我们应根据我们的硬件状况来确定。对于磁盘IO来说，如果硬盘每次读取4KB大小的文件块，那么我们最好设置成这个大小的整数倍。因为磁盘对于顺序读的效率是特别高的，所以如果buffer再设置的大些可能会带来更好的效率，比如设置成4*4KB或8*4KB，还需要注意一点的就是磁盘本身就会有缓存，在这种情况下，BufferedInputStream会一次读取磁盘缓存大小的数据，而不是分多次的去读。所以要想得到一个最优的buffer值，我们必须得知道磁盘每次读的块大小和其缓存大小，然后根据多次试验的结果来得到最佳的buffer大小。

**用缓冲字符流读文件**

```java
/**
   * 用缓冲字符流读文件
   * @throws IOException
   */
  public static void readByBufferedReader() throws IOException {
    File file = new File( "d:/test.txt");
    // 在字符流基础上用buffer流包装，也可以指定buffer的大小
    Reader reader = new BufferedReader(new FileReader(file),2*1024);
    char[] byteArray = new char[(int) file.length()];
    int size = reader.read(byteArray);
    System. out.println( "大小:" + size + ";内容:" + new String(byteArray));
    reader.close();
  }
```
学了设计模式就知道IO里其实用到了装饰者模式，这个设计模式。比如`FIleInputStream`是被装饰者组件，与之类似的还有图中的`StringBufferInputStream`，`ByteArrayInputStream`等等，它们提供了基本的字节读取，然后在图中`Wrapper`下面的，比如`BufferedInputStream`就是具体的装饰者，它加入了两种行为，利用缓冲输入来改造性能，用一个`readLine()`方法来增强接口，而`LineNumberInputStream`也是一个装饰者，它加上了计算行数的能力，这两个装饰者都扩展自`FilterInputStream`，这其实就是那个抽象类了。自己实现一个装饰者

```java
package com.qiming.designpattern.decorator;

import java.io.FilterInputStream;
import java.io.IOException;
import java.io.InputStream;

/**
 * 自己实现一个JAVA_IO的装饰者
 *
 * 增强的功能，将所有大写改成小写
 */

public class LowerCaseInputStream extends FilterInputStream {

  public LowerCaseInputStream(InputStream in) {
    super(in);
  }

  /**
   * 针对字节的
   * @return
   * @throws IOException
   */
  @Override
  public int read() throws IOException {
    int c = super.read();
    return (c == -1 ? c : Character.toLowerCase((char)c));
  }

  /**
   * 针对字节数组的
   * @param b
   * @param off
   * @param len
   * @return
   * @throws IOException
   */
  @Override
  public int read(byte[] b, int off, int len) throws IOException {
    int result = super.read(b, off, len);
    for (int i = off; i < off + result; i++) {
      b[i] = (byte)Character.toLowerCase((char)b[i]);
    }
    return result;
  }
}

```

进行测试

```java
package com.qiming.designpattern.decorator;

import java.io.BufferedInputStream;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.InputStream;

/**
 * 自己实现的装饰者，将大写文本转换成小写
 */
public class InputTest {

  public static void main(String[] args) {
    try {
      int c;
      InputStream in = new LowerCaseInputStream(new BufferedInputStream(new FileInputStream("E:\\Code\\TestDataSample\\word1.txt")));
      while ((c = in.read()) >= 0) {
        System.out.print((char) c);
      }
      in.close();
    } catch (IOException e) {
      e.printStackTrace();
    }
  }

}

```
