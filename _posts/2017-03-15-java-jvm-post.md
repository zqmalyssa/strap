---
layout: post
title: Java虚拟机
tags: [code, java]
author-id: zqmalyssa
---

JVM还是要知道的，要与实践结合起来才行

reference: 《深入理解JAVA虚拟机》

#### 理解清楚内存模型

Java把内存控制的权利交给了Java虚拟机，一旦出现内存泄漏或者内存溢出的问题，如果不了解虚拟机是怎样使用内存的，那么排查错误将是一项非常困难的工作，内存中的几块区域

1. 程序计数器，当做线程所执行的字节码的行号指示器。线程私有。
2. 虚拟机栈，就是俗称的栈(Stack)，也是线程私有的。每个方法在执行的时候会创建一个栈帧(Stack Frame)，用于存储局部变量表，操作数栈，动态链接，方法出口等信息。堆栈的区分其实是比较粗糙的
	- 局部变量表存放了编译期可知的各种基本数据类型、对象引用，64位长度的long和double会占据2个局部变量空间(Slot)，其余数据类型只占用一个
	- -Xss来指定其大小
	- 线程私有
3. 本地方法栈，类似虚拟机栈，只不过是虚拟机栈为虚拟机执行java方法，也就是字节码的服务，而本地方法栈则为虚拟机使用到的native方法服务
4. Java堆，就是俗称的堆(heap)，是Java虚拟机中所管理的内存中最大的一块，Java堆是所有线程共享的一块内存区域，此内存区域的唯一目的就是存放对象实例，多有的对象实例和数组都要在堆上分配，**Java堆是垃圾收集器管理的主要区域**，目前收集器**基本采用分代算法**，所以Java堆中还可以细分为新生代和老年代，再细致一点有Eden空间，From Survivor空间(S0)，To Survivor空间(S1)等，线程共享的Java堆中可能划分出多个线程私有的分配缓冲区(Thread Local Allocation Buffer，TLAB)，**这货就是ThreadLocal吧？？**，旧生代，用于存放新生代中经过多次垃圾回收依然存活的对象，例如缓存对象，新建的对象也有可能直接在旧生代分配内存
	- -Xms和-Xmx设置堆的最小和最大值，-Xmn设置新生代的值，-XX:SurvivorRatio设置新生代中Eden和Survivor的比例
	- 另外，不同的GC方式会以不同的方式按SurvivorRatio的值去划分Eden Space和Survivor Space，有些GC方式还会根据运行状况来动态的调整Eden，S0，S1的大小
	- PretenureSizeThreshold的值代表对象超过多大就不能在新生代分配，此参数在新生代采用Parallel Scavenge GC时无效
	- 该区域是所有线程共享的
	- TLAB是新创建的线程在新生代的Eden Space上分配的一块独立空间，由JVM计算，TLABWasteTargetPercent是设置TLAB所占用的Eden Space的百分比，默认是1%，在TLAB上分配内存时不需要加锁，因此给线程中的对象分配内存时尽量在TLAB上分配
5. 方法区，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息，常量，静态变量，即时编译器编译后的代码等数据，有的时候也将方法区描述为堆的一个逻辑部分，很多人愿意把方法区称为永久代
	- 可以通过-XX:PermSize和-XX:MaxPermSize来指定最小值和最大值
	- 该区域是全局共享的
6. 运行时常量池是方法区的一部分，Class文件中除了有类的版本，字段，方法，接口等描述信息外，还有一项信息是常量池(Constant Pool Table)，用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放

对象在内存中存储的布局

1. 对象头，用于存储对象自身运行时数据，如哈希码(HashCode)，GC分代年龄，锁状态标志，线程持有的锁等
2. 实例数据(Instance Data)，是对象真正存储的有效信息，也是在程序代码中所定义的各种类型的字段内容
3. 对齐填充(Padding)，对齐填充并不是必然存在的，也没有特别含义，仅仅起着占位符的作用

对象的访问定位

1. Java没有定义一个reference应该通过何种方式去定位，访问堆中的对象的具体位置，所以对象访问方式也是取决于虚拟机实现而定的，目前主流的访问方式有使用句柄和直接指针两种：
	- 句柄，在堆中有句柄池，reference存储的就是对象的句柄地址
	- 直接访问，在堆中有到对象类型数据的指针，reference中存储的直接就是对象地址


#### 类编译加载

JVM负责装载class文件并执行，将源码编译成class文件的实现取决于各个JVM的实现或各种源码编译器，class文件通常由类加载器(ClassLoader)来完成加载。

**JAVA源码编译机制**

SunJDK中就是用javac将源码编译成class，步骤是1分析和输入到符号集，2注解处理，3语义分析和生成class文件

1、分析和输入到符号集(Parse and Enter)

Parse过程所做的为词法和语法分析，词法分析com.sun.tools.javac.parser.Scanner要完成的是将代码字符串转变为token序列，语法分析com.sun.tools.javac.parser.Parser完成的是根据语法由token序列生成抽象语法树

Enter(com.sun.tools.javac.comp.Enter)过程为将符号输入到符号表，通常包括确定类的超类型和接口，根据需要添加默认构造器，将类中出现的符号输入类自身的符号表中等

2、注解处理(Annotation Processing)

该步骤主要用于处理用户自定义的annotation，可能带来的好处是基于annotation来生成附加的代码或进行一些特殊的检查，从而节省一些共用的代码的编写，比如Lombok，用注解后，编译完成再通过javap查看class文件可看到自动生成了需要的代码

3、语义分析和生成class文件(Analyse and Generate)

Analyse步骤基于抽象语法树进行一系列的语义分析，包括将语法树中的名字、表达式等元素与变量、方法、类型等联系到一起，检查变量使用前是否已声明，推导泛型方法的类型参数，检查类型匹配性，进行常量折叠，检查所有语句都可达，检查所有checked exception都被捕获或抛出等

完成语义分析后，开始生成class文件，一系列操作后最终从符号表生成class文件，class文件中并不仅仅存放了字节码，还存放了很多辅助jvm来执行class的附加信息，一个class文件包含了以下信息，

结构信息

包括class文件格式版本号及各部分的数量与大小的信息

元数据

简单来说，可以认为元数据对应的就是java源码中"声明"与"常量"的信息，主要有：类/继承的超类和实现的接口的声明信息，域(Field)与方法声明信息和常量池

方法信息

简单来说，可以认为方法信息对应的就是Java源码中"语句"与"表达式"对应的信息，主要有：字节码，异常处理器表，求值栈与局部变量区大小，求值栈的类型记录，调试用符号信息

```java
package com.qiming.test.classcompileandload;

/**
 * 这个例子用来说明class文件格式
 *
 * javac -g Foo.java (加-g是为了生成所有的调试信息，包括局部变量名及行号信息，不加-g默认只生成行号信息)
 *
 * 之后通过javap -c -s -l -verbose Foo来查看编译后的class文件
 *
 */
public class Foo {

  private static final int MAX_COUNT = 1000;
  private static int count = 0;
  public int bar() throws Exception {
    if (++count >= MAX_COUNT) {
      count = 0;
      throw new Exception("count overflow");
    }
    return count;
  }

}

```
看下javap后的关键内容

```java
Classfile /e:/tmpFile/code_test/Foo.class
  Last modified 2020-2-8; size 679 bytes
  MD5 checksum 1db50c5bc6288b0fe93151d20ffbe1b3
  Compiled from "Foo.java"
public class com.qiming.test.classcompileandload.Foo
  minor version: 0
	//class文件格式版本号，52是JDK8,51是JDK7,50是JDK6
	major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
	//常量池，存放了所有的field名称，方法名，方法签名，类型名，代码及class文件中的常量值
Constant pool:
   #1 = Methodref          #7.#27         // java/lang/Object."<init>":()V
   #2 = Fieldref           #3.#28         // com/qiming/test/classcompileandload/Foo.count:I
   #3 = Class              #29            // com/qiming/test/classcompileandload/Foo
   #4 = Class              #30            // java/lang/Exception
   #5 = String             #31            // count overflow
   #6 = Methodref          #4.#32         // java/lang/Exception."<init>":(Ljava/lang/String;)V
   #7 = Class              #33            // java/lang/Object
   #8 = Utf8               MAX_COUNT
   #9 = Utf8               I
  #10 = Utf8               ConstantValue
  #11 = Integer            1000
  #12 = Utf8               count
  #13 = Utf8               <init>
  #14 = Utf8               ()V
  #15 = Utf8               Code
  #16 = Utf8               LineNumberTable
  #17 = Utf8               LocalVariableTable
  #18 = Utf8               this
  #19 = Utf8               Lcom/qiming/test/classcompileandload/Foo;
  #20 = Utf8               bar
  #21 = Utf8               ()I
  #22 = Utf8               StackMapTable
  #23 = Utf8               Exceptions
  #24 = Utf8               <clinit>
  #25 = Utf8               SourceFile
  #26 = Utf8               Foo.java
  #27 = NameAndType        #13:#14        // "<init>":()V
  #28 = NameAndType        #12:#9         // count:I
  #29 = Utf8               com/qiming/test/classcompileandload/Foo
  #30 = Utf8               java/lang/Exception
  #31 = Utf8               count overflow
  #32 = NameAndType        #13:#34        // "<init>":(Ljava/lang/String;)V
  #33 = Utf8               java/lang/Object
  #34 = Utf8               (Ljava/lang/String;)V
{
	//将符号输入到符号表时生成的默认构造器方法
  public com.qiming.test.classcompileandload.Foo();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/qiming/test/classcompileandload/Foo;

	//bar方法的元数据信息
  public int bar() throws java.lang.Exception;
    descriptor: ()I
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=1, args_size=1
         0: getstatic     #2                  // Field count:I
         3: iconst_1
         4: iadd
         5: dup
         6: putstatic     #2                  // Field count:I
         9: sipush        1000
        12: if_icmplt     29
        15: iconst_0
        16: putstatic     #2                  // Field count:I
        19: new           #4                  // class java/lang/Exception
        22: dup
        23: ldc           #5                  // String count overflow
        25: invokespecial #6                  // Method java/lang/Exception."<init>":(Ljava/lang/String;)V
        28: athrow
        29: getstatic     #2                  // Field count:I
        32: ireturn
			//对应字节码的源码行号信息，可在编译的时候通过-g:none去掉行号信息，行号信息对于查找问题而言至关重要，因此最好还是保留
			LineNumberTable:
        line 8: 0
        line 9: 15
        line 10: 19
        line 12: 29
			//局部变量信息，如生成的class文件中无局部变量信息，则无法知道局部变量的名称，并且局部变量信息是和和方法绑定的，接口是没有方法体的，所以ASM之类的在获取接口方法时，是拿不到方法中参数的信息的
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      33     0  this   Lcom/qiming/test/classcompileandload/Foo;
      StackMapTable: number_of_entries = 1
        frame_type = 29 /* same */
				//异常处理表
		Exceptions:
      throws java.lang.Exception

  static {};
    descriptor: ()V
    flags: ACC_STATIC
    Code:
      stack=1, locals=0, args_size=0
         0: iconst_0
         1: putstatic     #2                  // Field count:I
         4: return
      LineNumberTable:
        line 6: 0
}
SourceFile: "Foo.java"
```
class是一个完整的自描述文件，字节码在其中只占了很小的部分，源码编译为class文件后，即可放入jvm中执行，执行时jvm首先要做的是装载class文件，这个机制通常称为类加载机制

**类加载机制**

类加载机制是指.class文件加载到JVM中，并形成Class对象的机制，之后应用就可对Class对象进行实例化并调用，类加载机制可在运行时动态加载外部的类，远程网络下载过来的class文件等，除了该动态化的优点外，还可通过JVM的类加载机制来达到类隔离的效果，例如Application Server中通常要避免两个应用的类互相干扰

JVM将类加载过程划分为三个步骤，装载，链接和初始化，装载和链接过程完成后，即将二进制的字节码转换为Class对象，初始化过程不是加载类时必须触发的，但最迟必须在初次主动使用对象前执行，其所作的动作给静态变量赋值，调用<clinit>()等

1、装载(Load)

装载过程负责找到二进制字节码并加载至JVM中，JVM通过类的全限定名及类加载器完成类的加载，同样，也采用以上两个元素来表示一个被加载了的类：类的全限定名+ClassLoader实例ID

对于接口或非数组型的类，其名称为类名，此种类型的类由所在的ClassLoader负责加载，对于数组型的类，其名称为"["+(基本类型或L+引用类型类名)，例如，byte[] bytes = new byte[512]，该bytes的类名为 [B; Object[] objects = new Object[10]，object的类名则为：[Ljava.lang.Object;，数组型类中的元素类型由所在的ClassLoader负责加载，但数组类则由JVM直接创建

2、链接(Link)

链接过程负责对二进制字节码的格式进行校验，初始化装载类中的静态变量及解析类中调用的接口、类

二进制字节码的格式校验遵循Java Class File Format规范，如果格式不符合，则抛出VerifyError，校验过程中如果碰到要引用到其他的接口和类，也会进行加载，如果加载过程失败，则会抛出NoClassDefFoundError

在完成了校验后，JVM初始化类中的静态变量，并将其赋值为默认值

最后对类中的所有属性、方法进行验证，以确保其调用的属性，方法存在，以及具备相应的权限，如果这个阶段失败，可能造成NoSuchMethodError，NoSuchFieldError等错误信息

3、初始化(Initialize)

初始化过程即执行类中的静态初始化代码，构造器代码及静态属性的初始化，以下四种情况下初始化过程会被触发执行
1)调用new
2)反射调用了类中的方法
3)子类调用了初始化
4)JVM启动过程中指定的初始化类

在执行初始化过程之前，首先必须完成链接过程中的校验和准备阶段，解析阶段则不强制

JVM的类加载通过ClassLoader及其子类来完成，分为Bootstrap ClassLoader、Extension ClassLoader、System ClassLoader、User-Defined ClassLoader，这4种ClassLoader的关系如下图

![classloader]({{ "/assets/img/jvm/classloader.png" | relative_url}})

1)Bootstrap ClassLoader，SunJDK采用C++实现了此类，此类并非ClassLoader的子类，在代码中没有办法拿到这个对象，SunJDK启动时会初始化此ClassLoader，并由ClassLoader完成$JAVA_HOME中jre/lib/rt.jar里所有class文件的加载，jar中包含了java规范定义的所有接口及实现
2)Extension ClassLoader，JVM用此ClassLoader来加载扩展功能的一些jar包，例如SunJDK中目录下有dns工具jar包等，在SunJDK中ClassLoader对应的类名为ExtClassLoader
3)System ClassLoader，JVM用此ClassLoader来加载启动参数中指定的Classpath中的jar包及目录，在SunJDK中ClassLoader对应的类名为AppClassLoader

```java
package com.qiming.test.classcompileandload;

/**
 * 看JDK中的ClassLoader的继承关系
 *
 * 可以看见System后Extension，但是没有Bootstrap，因为它并不是Java中的ClassLoader
 */
public class ClassLoaderDemo {

  public static void main(String[] args) {

    System.out.println(ClassLoaderDemo.class.getClassLoader());
    System.out.println(ClassLoaderDemo.class.getClassLoader().getParent());
    System.out.println(ClassLoaderDemo.class.getClassLoader().getParent().getParent());


  }

}

```

4)User-Defined ClassLoader

User-Defined ClassLoader是java并发人员继承ClassLoader抽象类自行实现的ClassLoader，基于自定义的ClassLoader可用于加载非Classpath中（例如从网络上下载的jar或二进制）的jar及目录，还可以在加载之前对class文件做一些动作，例如解密等

JVM的ClassLoader采用的是树形结构，除Bootstrap外，其他的ClassLoader都会有parent的ClassLoader

**类执行机制**

在完成将class文件信息加载到JVM并产生Class对象后，就可执行Class对象的静态方法或实例化对象进行调用了，在源码编译阶段将源码编译为JVM字节码，JVM字节码是一种中间代码的方式，要由JVM在运行期对其进行解释并执行，这种方式称为字节码解释执行方式，JVM采用

1、invokestatic方法调用static方法
2、invokevirtual方法调用对象实例方法
3、invokeinterface方法调用接口方法
4、invokespecial方法调用private方法和编译源码后生成的<init>方法

看个例子

```java
package com.qiming.test.classcompileandload;

/**
 * 看JVM如何调用方法
 *
 * javac ExecuteDemo.java
 *
 * javap -c ExecuteDemo查看字节码
 *
 */
public class ExecuteDemo {

  public void execute() {
    A.execute();
    A a = new A();
    a.bar();
    IFoo b = new B();
    b.bar();
  }
}

class A {
  public static int execute() {
    return 1 + 2;
  }

  public int bar() {
    return 1 + 2;
  }
}

class B implements IFoo{
  public int bar() {
    return 1 + 2;
  }
}

interface IFoo {
  public int bar();
}

```

用`javap -c`来看下

```java
public class com.qiming.test.classcompileandload.ExecuteDemo {
  public com.qiming.test.classcompileandload.ExecuteDemo();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public void execute();
    Code:
       0: invokestatic  #2                  // Method com/qiming/test/classcompileandload/A.execute:()I
       3: pop
       4: new           #3                  // class com/qiming/test/classcompileandload/A
       7: dup
       8: invokespecial #4                  // Method com/qiming/test/classcompileandload/A."<init>":()V
      11: astore_1
      12: aload_1
      13: invokevirtual #5                  // Method com/qiming/test/classcompileandload/A.bar:()I
      16: pop
      17: new           #6                  // class com/qiming/test/classcompileandload/B
      20: dup
      21: invokespecial #7                  // Method com/qiming/test/classcompileandload/B."<init>":()V
      24: astore_2
      25: aload_2
      26: invokeinterface #8,  1            // InterfaceMethod com/qiming/test/classcompileandload/IFoo.bar:()I
      31: pop
      32: return
}
```
invokespecial在new完对象后出现

解释执行的效率较低，为提升代码的执行性能，SunJDK提供将字节码编译为机器码的支持，编译在运行时进行，通常称为JIT编译器，SunJDK在执行过程中对执行频率高的代码进行编译，对执行不频繁的代码则继续采用解释的方法，因此SunJDK又称为Hotspot VM，在编译上SunJDK提供了两种模式，client compiler(-client)，就是C1和server compiler(-server)，就是C2

逃逸分析！！

SunJDK之所以未选择在启动时即编译成机器码，有几个方面的原因：

1. 静态编译并不能根据程序的运行情况来优化执行的代码，C2这种方式是根据运行情况来进行动态编译的，例如分支判断，逃逸分析，这些措施会对提升程序执行的性能起到很大的帮助，在静态编译的情况下是无法实现的，C2收集运行数据越长时间，编译出来的代码会越优

2. 解释执行比编译执行更节省内存

3. 启动时解释执行的启动速度比编译再启动更快


#### 垃圾收集器

**判断对象死了吗的算法**

JVM通过GC来回收堆和方法区中的内存，GC的原理首先会找到程序中不再被使用的对象，然后回收这些对象所占用的内存，通常采用收集器的方式实现GC，主要收集器有如下两种：

1. 引用计数算法

给对象中添加一个引用计数器，每当有一个地方引用它时，计数器就加1，当引用失效时，计数器就减1，任何时候计数器为0的对象就是不可能再被使用的，虽然不错，也有案例，但是主流的Java虚拟机里面没有使用

比如循环引用时，引用计数器就不起作用了

2. 可达性分析算法

通过一系列的"GC ROOTS"的对象为起点，从这些节点开始向下搜素，搜索的路径称为引用链，当一个对象到"GC ROOTS"没有任何引用链相连，则证明对象是不可用的。

这边要补充说明引用，JDK1.2后，对引用分为强引用(Strong Reference)，软引用(Soft Reference)，弱引用(Weak Reference)，虚引用(Phantom Reference)，引用的强度依次减弱


永久代(方法区)的垃圾回收主要是回收两部分内容，废弃常量和无用的类，效率不像新生代那样，能一次收回70%到95%的空间

常量的话没有引用就可以回收，类的话复杂一点，满足如下条件
1. 类的所有实例被回收，就是Java堆中不存在该类的任何实例
2. 加载该类的ClassLoad已经被回收
3. 该类对应的java.lang.Class对象没有在任何地方被引用，**无法在任何地方通过反射访问该类的方法**

**在大量使用反射，动态代理，CGLIB等ByteCode框架，动态生成JSP及OSGI这类频繁自定义ClassLoader的场景都需要虚拟机具备类卸载的功能，保证永久代不会溢出**


**垃圾收集算法**

这不同于判断对象存活的问题，主要使用下列算法

1. 标记-清除算法

根据标记将对象回收，因为是直接回收，所以会造成内存碎片

2. 复制算法

两块内存，每次就使用一块，将未标记的复制到另一块后清除一整块，不需要1:1，一个较大的Eden空间和两个较小的survivor空间。每次使用Eden和其中一块survivor空间，回收时一次性复制到另一块survivor空间。如果不够放，通过分配担保机制进入老年代

3. 标记-整理算法

标记后移动存活内存空间，然后再清除，解决标记-清除的问题，但因为要移动，成本也高

4. 分代收集算法

将堆分成新生代和老年代，这样可以根据各个年代的特点，采用最适当的收集算法

新生代中经常采用复制算法，因为每次垃圾收集都会发现大批对象死去，只有少量存活，而老年代中因为对象的存活率高，没有额外空间对他们进行分配担保，就必须使用标记-清理，标记-整理 算法。

在新生代中，复制算法常常把它又划分成一块Eden和两块Survivor，其中一个Survivor就是存放活着的对象的

总结：

新生代可用的GC：串行GC(Serial Copying)，并行回收GC(Parallel Scavenge)，并行GC(ParNew)
	- SunJDK认为新生代的对象通常存活时间较短，因此选择基于复制算法对新生代对象进行回收，所以有了Eden和S0，S1
	- 串行GC，Minor GC，完整的根集合为SunJDK认为的根集合加上remember set中标记的对象，在对象的引用关系上，除了强引用外，还提供了软引用，弱引用和虚引用，强引用A a = new A()，只有主动释放了引用才会被GC，软引用，当JVM内存不足时会被回收，因此SoftReference适合用于实现缓存，弱引用，采用弱引用建立引用的对象没有强引用后，GC时即会被自动释放，虚引用，采用PhantomReference来实现，采用虚引用可跟踪到对象是否已从内存中被删除，当扫描引用关系时，GC会对上面三种类型的引用进行不同的处理
	- 并行回收GC，多了一个参数InitialSurvivorRatio，此值默认是8，对应的大小是新生代大小/survivor space，也可以用SurvivorRatio，将此值加2给InitialSurvivorRatio，对于并行回收而言，初始大小分配这样，但是会根据MinorGC的频率，动态调整Eden，S0，S1的大小，可通过UseAdaptiveSizePolicy来固定Eden，S0和S1的大小，PS GC不是根据PretenureSizeThreshold来决定是否在旧生代上直接分配，而是当需要给对象分配内存时，eden space空间不够的情况下，如果此对象大小大于等于eden space一半的大小，就直接在旧生代上分配，例子如下，并行回收采用的也是Copying算法，但其在扫描和复制时均采用了多线程方式进行并且并行回收GC为大的新生代回收做了很多优化，例如上面提到的动态调整eden，S0，S1空间大小，在多CPU机器上回收时间耗费会比串行方式短。默认线程数根据CPU核数(# cat /proc/cpuinfo | grep "physical id" | wc -l)(也就是processor的个数吧)计算，当核数小于等于8时，并行的线程数就是CPU的核数，多于8时，则为3+(CPU核数*5)/8，也可采用-XX:ParallelGCThreads = 4来强制指定线程数
	```java
	package com.qiming.test.classcompileandload;

/**
 * 示例，当新生代用PS GC时，如何直接去老年代分配内存
 *
 * -Xmx 20M -Xms 20M -Xmn 10M -XX:SurvivorRatio=8 -XX:+UseParallelGC
 */
public class PSGCDirectOldDemo {

  public static void main(String[] args) throws Exception {
    byte[] byte1 = new byte[1024*1024*2]; //2M
    byte[] byte2 = new byte[1024*1024*2]; //2M
    byte[] byte3 = new byte[1024*1024*2]; //2M

    System.out.println("Ready to direct allocate to old");
    Thread.sleep(3000);
    byte[] byte4 = new byte[1024*1024*4]; //4M
    Thread.sleep(3000);
  }

}
	```
	- 并行GC，在基于Survivor划分空间的方式上和串行GC是一样的，并行GC和并行回收GC区别在于并行GC须配合旧生代使用CMS GC，CMS GC在进行旧生代GC时，有些过程是并发进行的，如此时发生MinorGC，需要进行相应的处理，而并行回收GC是没有做这些处理的，也正因为这些特殊处理，ParNew GC不可与并行的旧生代GC同时使用，在配置为使用CMS GC的情况下，新生代默认采用并行GC方式，也可以通过-XX:+UseParNewGC来强制指定


旧生代可用的GC：串行GC(Serial MSC)，并行GC(Parallel MSC)，并发GC(CMS)
	- 串行基于Mark-Sweep-Compact实现，它结合了Mark-Sweep和Mark-Compact做了一些改进，采用串行方式时，旧生代的内存分配方式和新生代采用的串行方式相同，串行执行的整个过程需要暂停应用，且采用单线程方式，要耗费很长时间，可通过-XX:+PrintGCApplicationStoppedTime来查看GC造成的应用暂停时间
	- 并行采用mark-compact实现，在内存分配方式上则和串行相同，并行大部分时间是多线程同步操作的，通过UseParallelGC或者UseParallelOldGC来强制指定
	- 并发CMS(Concurrent Mark-Sweep GC)，Mark-Sweep要对整个空间中的对象进行扫描并标记，这个过程会造成较长时间的应用暂停，所以提供了CMS，GC的大部分动作均与应用并发进行，因此大大缩短了GC造成的暂停时间，CMS会有内存碎片，而源于MinorGC的分配内存，会使得MinorGC速度下降，需要了解它的步骤，1、第一次标记(Initial Marking)，从根开始扫描，2、并发标记(Concurrent Marking)，在初始化标记完成后，CMS恢复所有应用的线程，同时开始并发对之前着色过的对象进行轮询，以标记这些对象可访问的对象 3、重新标记(Final Marking(remark))，该步需要暂停整个应用，因为Concurrent Marking可能会修改对象的引用关系和创建新的对象，所以要对这些变化进行再扫描 4、并发收集(Concurrent Sweeping)，将没有标记的对象进行回收了。所以CMS在Initial和remark需要暂停整个应用，其他动作并发进行，但是并发嘛就就会跟应用的线程竞争资源，与并行相比，CMS会执行三次Mark，因此完整的一次GC执行时间会比并行GC长，对于关注GC总耗应用来说，CMS GC并不是合适的选择。默认情况下CMS不开启，用UseConcMarkSweepGC来开启，其默认的回收线程数是(并行GC线程数+3)/4，可通过ParallelCMSThreads=10来强制指定。触发条件是CMSInitiatingOccupancyFraction百分比，默认是68%，GC就开始了，另一种触发方式是JVM自动触发，基于之前的GC频率及旧生代的增长趋势来的

持久代的GC也可采用CMS方式，CMSPermGenSweepingEnabled CMSClassUnloadingEnabled

**垃圾收集器**

内存是回收的方法论，那么垃圾收集器就是内存回收的具体实现了，有7种不同分代的收集器

CMS收集器，是一种以获取最短回收停顿时间为目标的收集器，基于标记-清除算法实现的

G1收集器，是一款面向服务端应用的垃圾收集器，在JDK7中出现，吸收了增量收集器和CMS GC策略的优点，G1将整个heap划分成固定大小的region，对region进行mark，尽量短时间的暂停应用，首先回收的是垃圾对象所占空间最多的region，因此被称为Garbage Firest。尽管G1将heap划分成多个region，但其**默认采用的还是分代的**方式，只是名称改为年轻代和非年轻代。同时也支持一种叫作pure G1的方式，也就是不进行代的划分

如何查看当前JVM垃圾收集器

```html
java  -XX:+PrintCommandLineFlags  -version
```
或者

```html
jmap -heap [pid]
```

下面一个有比较详细的值

**内存分配和回收策略**

Eden区没有足够空间进行分配时，虚拟机将发起一次Minor GC

**Minor GC**和**Full GC**到底有什么区别

新生代GC(Minor GC)，指发生在新生代的垃圾收集动作，因为java对象大多都具备朝生汐灭的特性，所以Minor GC非常频繁，一般回收速度也比较快

老年代GC(Major GC / Full GC)，指发生在老年代的GC，出现了Major GC，经常会伴随一次Minor GC(但非绝对)，Major GC的速度一般会比Minor GC慢10倍以上

**触发Full GC**执行的情况有大致4种：

1. 老年代空间不足
	- 创建了大对象和大数组
	- 调优就是让对象在MinorGC阶段就被回收，让对象在新生代多存活一段时间及不要创建过大的对象及数组

2. Permanent Generation空间满
	- 存放的为一些class信息，当系统中要加载的类，反射的类，调用的方法较多时，Permanent Generation可能被沾满
	- 可采用增大Perm Gen空间或转为使用CMS GC

3. CMS GC时出现promotion failed和concurrent mode failure
	- 注意GC日志是否有promotion failed和concurrent mode failure两种情况，会触发full gc，promotion failed是在Minor GC的时候，survivor放不下，对象只能放入老年代，而老年代也放不下，concurrent mode failure是在执行CMS GC的过程中同时有对象要放入老年代，空间不足导致
	- 应对方法是增大survivor空间，增大老年代空间

4. 统计得到的Minor GC晋升到老年代的平均大小大于老年代的剩余空间
	- 如果统计所得到的Minor GC晋升到老年代的平均大小大于老年代的剩余空间，那么久触发Full GC

下面就要举一些gc的例子

1、触发MinorGC，看代码详细介绍

不止是GC的打印，可以配合jstat来常看

```
package com.qiming.test.classcompileandload;

/**
 * MinorGC的一个例子
 *
 * -Xmx40M -Xms40M -Xmn16M -verbose:gc -XX:+PrintGCDetails
 *
 * 所以eden是12MB，S0和S1是2M，旧生代是24MB
 *
 * 第一次，eden11MB了，再分配1M就不够了，发生GC，完成后，eden留下新分配的1M，object被扔到了S区，
 *
 * [GC (Allocation Failure) [PSYoungGen: 11507K->1880K(14336K)] 11507K->1888K(38912K), 0.0407404 secs] [Times: user=0.00 sys=0.00, real=0.05 secs]
 *
 * PS说明使用了PS GC，11507K->1880K(14336K)表示在GC前，新生代使用空间11507K，回收后新生代使用空间1880K，新生代总共可用空间是14336K
 *
 * 11507K->1888K(38912K)表示在GC前，堆使用空间11507K，回收后堆使用空间为1888K，总共可用空间为38912K，0.0407404 secs表示了此次minorGC的时间
 *
 * [Times: user=0.00 sys=0.00, real=0.05 secs]表示MinorGC占用cpu user和sys的百分比，以及消耗的总时间
 *
 * 第二次也是Eden空间不足触发，这次可看见S的另一个空间有使用率了，说明S0和S1进行了交换，即From变成To，To变成From
 *
 */
public class MinorGCDemo {

  public static void main(String[] args) throws Exception{
    //object是不会被回收的，放在了S0中
    MemoryObject object = new MemoryObject(1024*1024);
    for (int i = 0; i < 2; i++) {
      happenMinorGC(11);
      Thread.sleep(2000);
    }
  }


  private static void happenMinorGC(int happenMinorGCIndex) throws Exception{
    for (int i = 0; i < happenMinorGCIndex; i++) {
      //if成立前已经分了11MB在eden中了，再分配就不行了，因此触发了MinorGC
      if (i == happenMinorGCIndex - 1) {
        Thread.sleep(2000);
        System.out.println("minor gc should happen");
      }
      new MemoryObject(1024*1024);
    }
  }

}

class MemoryObject {
  private byte[] bytes;
  public MemoryObject(int objectSize) {
    this.bytes = new byte[objectSize];
  }
}
```

2、MinorGC时survivor空间不足，对象直接进入旧生代

```java
public static void main(String[] args) throws Exception{
	MemoryObject object = new MemoryObject(1024*1024);
	MemoryObject m2object = new MemoryObject(1024*1024*2);
	happenMinorGC(9);
	Thread.sleep(2000);
}
```

MinorGC后需要放入survivor空间的有object和m2object，显然2M的空间不够放3M，那么m2object就会被扔进旧生代，jstat就可以看见旧生代的使用率是8.33%

3、不同GC的日志

如果是-XX:+UseSerialGC，就要调整SurvivorRatio，那么GC打印的时候就会显示DefNew，表明是串行GC方式

4、FullGC的例子后面补充

为了避免选择哪种GC而头疼，SunJDK提供了两种简单的方式来帮助选择GC

1. 吞吐量优先，总共运行100min，其中GC执行占用1min，那么吞吐量就是99%，吞吐量优先让JVM自行处理，可用GCTimeRatio=n来使用此策略
2. 暂停时间优先，以暂停时间为指标，JVM自行选择相应GC策略，保证每次停顿时间在可控范围内，MaxGCPauseMillis=n来使用

上面两者互斥，为了减少GC的暂停时间，就要接受吞吐量的下降

还有补充说一下G1收集器，是JDK1.9的默认收集器

其目的是尽量减少GC所导致的应用暂停时间，同时保持JVM堆空间的利用率。必须要多核CPU和较大的内存，另一方面是要接受GC吞吐量的稍微降低，对于响应要求高的系统来说，这点是可以接收的

G1将整个JVM堆划分为多个固定大小的region，用并发marking算法对region进行mark，回收时根据region中活跃对象的bytes进行排序，首先回收活跃对象bytes小及回收耗时短的region，回收的方法将此region中的活跃对象复制到另外的region中，这种回收策略首先回收的是垃圾对象所占空间最多的region，因此称为Garbage First

尽管G1将heap划分成多个region，但其默认采用的仍然是分代的方式，只是名称改为年轻代和非年轻代，这也是G1仍然坚信大多数新创建的对象是不需要长的生命周期的。

G1的步骤和CMS非常相似，只是G1将JVM划分成更小粒度的regions，并在回收时进行了估算，同时能够带来更大内存空间的regions，从而缩短了每次回收所需消耗的时间

#### JVM内存查看和分析工具

推荐图形化的，GC的日志也是必要的

PrintGC，PrintGCDetails，PrintGCTimeStamps，PrintGCApplicationStoppedTime分别输出GC的简要信息，GC的详细信息，GC的时间信息及GC造成的应用暂停时间

如果想要输出指定文件中，-Xloggc:gc.log输出到gc.log中，jps可以看当前运行的进程Id

1. JConsole

JDK的bin目录下自带，直接点开，根据pid去选择跑的java程序

2. JMAP

JMap是JDK自带的分析JVM内存状况的工具，使用JMap可以查看目前JVM中各个代的内存情况，JVM中的各个对象的占用情况以及导出整个JVM中的内存信息

jmap -heap [pid]看JVM中内存的使用情况
jmap -histo [pid]查看对象的详细占用情况，输出内容按照占用空间大小排序，有[C，[B这样代表各种类型

3. JSTAT

jstat -gcutil [pid] [interval]可以查看S0，S1的使用率，Eden，Old的使用率，YoungGC和FullGC的次数和时间

可以查看JVM中各代的空间的占用情况，minor GC的次数，消耗的时间，fullGC的次数及消耗的时间

JDK8中多了Metaspace和CSS，具体如下

S0: Survivor space 0 utilization as a percentage of the space's current capacity. 幸存者区0

S1: Survivor space 1 utilization as a percentage of the space's current capacity. 幸存者区1

E: Eden space utilization as a percentage of the space's current capacity. 伊甸园区

O: Old space utilization as a percentage of the space's current capacity. 老年代

M: Metaspace utilization as a percentage of the space's current capacity. 元空间

CCS: Compressed class space utilization as a percentage. 压缩类空间利用率为百分比。

YGC: Number of young generation GC events. 年轻一代GC事件的数量。

YGCT: Young generation garbage collection time. 年轻一代垃圾收集时间

FGC: Number of full GC events. 完整的GC事件的数量。

FGCT: Full garbage collection time. 完全垃圾收集时间。

GCT: Total garbage collection time. 垃圾回收总时间。

```html
//每个3秒钟显示一次，显示10次
jstat -gcutil 1 3000 10
```

4. JProfiler

应该是商用版本，JProfiler是由ej-technologies GmbH公司开发的一款性能瓶颈分析工具(该公司还开发部署工具)。其特点:

1. 使用方便
2. 界面操作友好
3. 对被分析的应用影响小
4. CPU,Thread,Memory分析功能尤其强大
5. 支持对jdbc,noSql, jsp, servlet, socket等进行分析
6. 支持多种模式(离线，在线)的分析
7. 跨平台

5. JStack

jstack [pid] 可以查看jvm线程的运行情况，包括锁的等待，线程是否在运行等

这边补充一个案例，运行前可以先睡一会，保证可以使用jstack和JVisualVM。需要了解

```java
package com.qiming.concurrent;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 一个场景，两个线程数数，同时启动两个线程，线程A数1、2、3，然后线程B数4、5、6，最后线程A数7、8、9，程序结束，
 * 这涉及到线程之间的通信，但是
 * 下面这段场景有一定的概率无法执行下去
 */

public class ConditionTest {

  static class NumberWrapper {
    public int value = 1;
  }

  public static void main(String[] args) {
    try {
      Thread.sleep(15000);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }

    //初始化可重入锁
    final Lock lock = new ReentrantLock();

    //第一个条件当屏幕上输出到3
    final Condition reachThreeCondition = lock.newCondition();

    //第一个条件当屏幕上输出到6
    final Condition reachSixCondition = lock.newCondition();

    //NumberWrapper只是为了封装一个数字，一边可以将数字对象共享，并可以设置为final :)
    //注意这里不要用Integer, Integer 是不可变对象
    final NumberWrapper num = new NumberWrapper();
    //初始化A线程
    Thread threadA = new Thread(
        new Runnable() {
          @Override
          public void run() {
            //需要先获得锁
            lock.lock();
            System.out.println("ThreadA获得lock");
            try {
              System.out.println("ThreadA start write");
              //A线程先输出前3个数
              while (num.value <= 3) {
                System.out.println(num.value);
                num.value++;
              }
              //输出到3的时候要signal，告诉B线程可以开始了
              reachThreeCondition.signal();
            } catch (Exception e) {
              e.printStackTrace();
            } finally {
              lock.unlock();
              System.out.println("ThreadA释放了lock");
            }

            lock.lock();
            try {
              //等到输出6的条件
              System.out.println("ThreadA获得lock");
              reachSixCondition.await();
              System.out.println("ThreadA start write");
              //输出剩余的数字
              while (num.value <= 9) {
                System.out.println(num.value);
                num.value++;
              }
            } catch (InterruptedException e) {
              e.printStackTrace();
            } finally {
              lock.unlock();
              System.out.println("ThreadA释放了lock");
            }
          }
        }
    );

    Thread threadB = new Thread(new Runnable() {
      @Override
      public void run() {
        try {
          lock.lock();
          System.out.println("ThreadB获得lock");
          Thread.sleep(5000);
          while (num.value <= 3) {
            //等待3输出完毕的信号
            reachThreeCondition.await();
          }
        } catch (InterruptedException e) {
          e.printStackTrace();
        } finally {
          lock.unlock();
          System.out.println("ThreadB释放lock");
        }

        try {
          lock.lock();
          System.out.println("ThreadB获得lock");
          //已经收到信号，开始输出4/5/6
          System.out.println("threadB start write");
          while (num.value <= 6) {
            System.out.println(num.value);
            num.value++;
          }
          //4/5/6输出完毕，告诉A线程6输出完了
          reachSixCondition.signal();
        } catch (Exception e) {
          e.printStackTrace();
        } finally {
          lock.unlock();
          System.out.println("ThreadB释放lock");
        }
      }
      });

    //启动两个线程
    threadA.setName("threadA");
    threadB.setName("threadB");
    threadA.start();
    threadB.start();

    }
}

```
在jstack中发现ThreadA是处于WAITING状态

```html
"threadA" #12 prio=5 os_prio=0 tid=0x000000001e7c8800 nid=0x8880 waiting on condition [0x00000000204bf000]
   java.lang.Thread.State: WAITING (parking)
```
在visualvm中也显示其在驻留状态（也就是WAITING(parking)中），所以可见threadA把自己给搞了，分析代码，打日志

```html
情况1  

ThreadA获得lock
ThreadA start write
1
2
3
ThreadA释放了lock
ThreadB获得lock
ThreadB释放lock
ThreadA获得lock
ThreadB获得lock
threadB start write
4
5
6
ThreadB释放lock
ThreadA start write
7
8
9
ThreadA释放了lock

------------------------------------------  

情况2

ThreadA获得lock
ThreadA start write
1
2
3
ThreadA释放了lock
ThreadA获得lock
ThreadB获得lock
ThreadB释放lock
ThreadB获得lock
threadB start write
4
5
6
ThreadB释放lock
ThreadA start write
7
8
9
ThreadA释放了lock

------------------------------------------  

情况3

ThreadA获得lock
ThreadA start write
1
2
3
ThreadA释放了lock
ThreadB获得lock
ThreadB释放lock
ThreadB获得lock
threadB start write
4
5
6
ThreadB释放lock   //这边signal之后，threadA还没有进第二次呢
ThreadA获得lock   //这时候threadA才进第二次，直接await了

------------------------------------------

修改后

ThreadA获得lock
ThreadA start write
1
2
3
ThreadA释放了lock
ThreadB获得lock
ThreadB释放lock
ThreadB获得lock
threadB start write
4
5
6
ThreadB释放lock
ThreadA获得lock
ThreadA start write
7
8
9
ThreadA释放了lock  //改造完成后运行ok
```
发现了出问题的地方了，进行await之前判断一下，修改如下

```java
package com.qiming.concurrent;

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 一个场景，两个线程数数，同时启动两个线程，线程A数1、2、3，然后线程B数4、5、6，最后线程A数7、8、9，程序结束，
 * 这涉及到线程之间的通信，但是
 * 下面这段场景有一定的概率无法执行下去
 */

public class ConditionTest {

  static class NumberWrapper {
    public int value = 1;
  }

  public static void main(String[] args) {
//    try {
//      Thread.sleep(10000);
//    } catch (InterruptedException e) {
//      e.printStackTrace();
//    }

    //初始化可重入锁
    final Lock lock = new ReentrantLock();

    //第一个条件当屏幕上输出到3
    final Condition reachThreeCondition = lock.newCondition();

    //第一个条件当屏幕上输出到6
    final Condition reachSixCondition = lock.newCondition();

    //NumberWrapper只是为了封装一个数字，一边可以将数字对象共享，并可以设置为final :)
    //注意这里不要用Integer, Integer 是不可变对象
    final NumberWrapper num = new NumberWrapper();
    //初始化A线程
    Thread threadA = new Thread(
        new Runnable() {
          @Override
          public void run() {
            //需要先获得锁
            lock.lock();
            System.out.println("ThreadA获得lock");
            try {
              System.out.println("ThreadA start write");
              //A线程先输出前3个数
              while (num.value <= 3) {
                System.out.println(num.value);
                num.value++;
              }
              //输出到3的时候要signal，告诉B线程可以开始了
              reachThreeCondition.signal();
            } catch (Exception e) {
              e.printStackTrace();
            } finally {
              lock.unlock();
              System.out.println("ThreadA释放了lock");
            }

            lock.lock();
            try {
              //等到输出6的条件
              System.out.println("ThreadA获得lock");
              while (num.value <= 6) {
                reachSixCondition.await();
              }
              System.out.println("ThreadA start write");
              //输出剩余的数字
              while (num.value <= 9) {
                System.out.println(num.value);
                num.value++;
              }
            } catch (InterruptedException e) {
              e.printStackTrace();
            } finally {
              lock.unlock();
              System.out.println("ThreadA释放了lock");
            }
          }
        }
    );

    Thread threadB = new Thread(new Runnable() {
      @Override
      public void run() {
        try {
          lock.lock();
          System.out.println("ThreadB获得lock");
          Thread.sleep(5000);  //停顿时间
          while (num.value <= 3) {
            //等待3输出完毕的信号
            reachThreeCondition.await();
          }
        } catch (InterruptedException e) {
          e.printStackTrace();
        } finally {
          lock.unlock();
          System.out.println("ThreadB释放lock");
        }

        try {
          lock.lock();
          System.out.println("ThreadB获得lock");
          //已经收到信号，开始输出4/5/6
          System.out.println("threadB start write");
          while (num.value <= 6) {
            System.out.println(num.value);
            num.value++;
          }
          //4/5/6输出完毕，告诉A线程6输出完了
          reachSixCondition.signal();
        } catch (Exception e) {
          e.printStackTrace();
        } finally {
          lock.unlock();
          System.out.println("ThreadB释放lock");
        }
      }
      });

    //启动两个线程
    threadA.setName("threadA");
    threadB.setName("threadB");
    threadA.start();
    threadB.start();

    }
}

```

6. JVisualVM

VisualVM是Netbeans的profile子项目，已在JDK6.0 update 7中自带，能够监控线程，内存情况，查看方法的CPU时间和内存中的对象，已被GC的对象

双击启动 jvisualvm.exe，启动起来后和jconsole 一样同样可以选择本地和远程，如果需要监控远程同样需要配置相关参数，不要打开没有的Pid，否则容易挂掉，可以安装插件，如visual GC
1、概述中主要是JVM的参数和系统属性
2、监视中主要有CPU，内存，类，线程查看，还有执行垃圾回收和堆dump的操作
3、安装的插件，VisualGC中，可以直观的看见分配情况，看见年轻代，老年代的内存变化，以及GC频率，GC时间等

一次内存泄漏（Java中的内存泄露，广义并通俗的说，就是：不再会被使用的对象的内存不能被回收，就是内存泄露）的排查的过程，首先在visualvm中打开跑着的应用，然后点开，看visualgc中的变化，将某段时间的堆dump出来，过一段时间可以再dump出堆，然后两个堆进行比较，比较后就可以看见这段时间的变化，通过实例的视图可以看到引用的地方，定位问题

如果长生命周期的对象持有短生命周期的引用，就很可能会出现内存泄露。我们举一个简单的例子：

```java
public class Simple {
    Object object;
    public void method1(){
        object = new Object();
        //...其他代码
    }
}
```
这里的object实例，其实我们期望它只作用于method1()方法中，且其他地方不会再用到它，但是，当method1()方法执行完成后，object对象所分配的内存不会马上被认为是可以被释放的对象，只有在Simple类创建的对象被释放后才会被释放，严格的说，这就是一种内存泄露。解决方法就是将object作为method1()方法中的局部变量。当然，如果一定要这么写，可以改为这样：

```java
public class Simple {
    Object object;
    public void method1(){
        object = new Object();
        //...其他代码
        object = null;
    }
}
```
这样，之前“new Object()”分配的内存，就可以被GC回收，解决的原则是

1、尽量减小对象的作用域
2、赋值null，可以查看LinkedList源码，有很多这样的操作

**JVisualVM远程监控Tomcat**

修改远程tomcat的catalina.sh配置文件，在其中增加：

```java
JAVA_OPTS="$JAVA_OPTS
-Djava.rmi.server.hostname=192.168.122.128
-Dcom.sun.management.jmxremote.port=18999
-Dcom.sun.management.jmxremote.ssl=false
-Dcom.sun.management.jmxremote.authenticate=false"
```
然后再[远程]中添加远程主机，输入IP和端口就可以了

#### java性能调优

调优过程是一个相当复杂的过程，涉及到很多方面：硬件，操作系统，软件，环境本身

方法是设定一个目标，分析出代码瓶颈（结合一些工具），以下是一些常用方法

寻找性能瓶颈

1. CPU消耗分析
	- 上下文切换
	- 运行队列
	- 利用率
	- top命令进行查看，shift+h切换到线程的
	- pidstat，可能需要安装SYSSTAT，pidstat 1 2每隔1秒输出目前活动进程CPU消耗，共输出2次，pidstat -p 952 -t 1 5，查找对应Pid中线程的CPU消耗情况，TID就是线程
	- vmstat，可以使用vmstat 1查看CPU的上下文切换，运行队列，利用率的具体信息

CPU的消耗严重主要体现在us和sy两个值上，us过高，应用消耗了大部分CPU，找到消耗CPU的线程执行的代码，首先是要定位到线程，jstack dump或者kill -3 [javapid]可以，us高只要是线程一直处于（Runnable）状态，通常这些线程执行无阻塞，循环，正则，或纯粹的计算等动作造成，另一个原因是频繁的GC，如每次请求都要分配较多的内存，当访问量上去后要不断的GC，对于频繁GC的情况要通过分析JVM内存的消耗来查找原因

例子：CPU一直在使用，us高，CPU一直在上下文切换，线程竞争锁这种，则sy高

2. 文件IO消耗分析
	- pid -d -t -p [pid] l 100类似的命令可查看线程IO消耗情况
	- pidstat要内核版本高于2.6.20，没有则可用iostat，但无法追踪到进程级别，iostat -x xvda 3 5这样定时采样查看IO的消耗状况

关注IOwait这个值，说明应用在频繁的进行IO操作，可能在写大量日志，也可能本来文件系统就很慢，对应用的线程dump出来进行分析

3. 网络IO消耗分析
	- sar进行分析，sar -n FULL 1 2，主要关心tcpsck和udpsck

分布式Java中需要知道网卡终端，cat /proc/interrupts，无法分析到线程，只能对应dump文件，java应用一般不会造成网络IO的消耗严重

4. 内存消耗分析
	- 主要是JVM内存的消耗情况
	- pidstat， pidstat -r -p [pid] [interval] [times]

程序执行慢的原因：

1. 锁竞争激励

2. 未充分使用硬件资源

3. 数据量增大


可以通过JProfiler分析程序的执行速度

**调优**

1. JVM调优

主要是内存管理的调优，包括各个代的大小和GC策略等

GC策略的调优

CMS GC因为是和程序并发进行，速度确实会比ParallelGC快一点，JVM的调优其实在JVM已经做的不错了，多数情况下只需选择GC策略，并设置JVM Heap的大小，调整了内存管理后应通过-XX:+PrintGCDetails、-XX:+PrintGCTimeStamps、-XX:+PrintGCApplicationStoppedTime及用jstat和visualvm等方式观察调整后的GC状况即可，基本每次JDK的发布都会带来性能上的优化

2. 程序调优
	- CPU消耗严重，线程一直在运行，也无挂起动作，造成线程饿死的现象，常见的加个Thread.sleep，释放CPU的执行权，降低CPU的消耗
	- 使用协程，框架是Kilim，比线程更轻量，不会有线程等待的性能下降及频繁的切换线程，但kilim没有商用
	- 文件IO消耗，用异步写文件，可以使用log4j的AsyncAppender，批量读写，频繁的读写操作对IO消耗会很严重，批量操作将大幅提升IO操作的性能，限流，将IO消耗控制到一个能接受的范围。还有限制文件大小
	- 网络IO消耗
	- 内存消耗严重，释放不必要的引用，在复用线程的情况下使用了ThreadLocal，由于是线程复用，ThreadLocal中存放的对象如未主动释放的话则不会被GC，ThreadLocal的set方法把对象清除。还有使用对象缓存池，使用对象缓存池一定程度上可降低JVM Heap内存的使用，使用HashMap的缓存cache，存大对象。还有的方法是采用合理的缓存失效方法，要合理的设置上面缓存池的大小，否则消耗大量内存，一直持有对象引用，引发Full GC，用FIFO，LRU，LFU等策略，LinkedHashMap可以支持FIFO，LRU策略的CachePool。还有合理使用软引用和弱引用，缓存对象就可以使用
	- 锁竞争激励引起的程序执行缓慢，解决方法使用并发包下的类，减少多线程情况下资源的锁竞争，最小化锁，没必要synchronized(this)，可以synchronized(map)，还可以拆分锁，读写锁拆分，ConcurrentHashMap中默认拆分为16把锁，这要根据业务场景来。还有去除读写操作的互斥锁。

3. 未充分使用硬件资源

	- 未充分使用CPU，更改代码充分使用CPU，启用多线程，JDK7还出来一个fork-join
	- 未充分使用内存

举一个性能调优的例子

参考[文章](https://segmentfault.com/a/1190000005174819)

1、用jmap排除应用程序的内存使用问题
2、排除cache内容过多的问题，用visualvm中的visualgc插件
3、调整GC时间点，加CMS的参数，GC频率增加，但是成功率并没有增加
4、调整对象在年轻代内存中驻留的时间（效果不明显）
5、CMS-Remark之前强制进行年轻代的GC
	-	CMS中两个步骤是STW，只是通过两次短暂停来替代串行标记整理算法的长暂停

通过GC日志和成功率下降的时间点进行比对发现并不是每一次老年代GC都会导致成功率的下降，但是从中发现了一个规律：前两次GC CMS-Remark过程在4s左右造成了成功率的下降，但是第三次GC并没有对成功率造成明显的影响,CMS-Remark只有0.18s。Java HTTP 服务是通过Nginx进行反向代理的，nginx设置的超时时间是3s，所以如果GC卡顿在3s以内就不会对成功率造成太大的影响

从GC日志中又发现一个信息：

在文档和相关资料中没有找到蓝色部分的含义，猜测是remark处理的内存量，处理的越多就越慢。添加下面两个参数强制在remark阶段和FULL GC阶段之前先在进行一次年轻代的GC，这样需要进行处理的内存量就不会太多

```java
-XX:+ScavengeBeforeFullGC
-XX:+CMSScavengeBeforeRemark
```
结论：

1、在CMS-remark阶段需要对堆中所有的内存对象进行处理，如果在这个阶段之前强制执行一次年轻代的GC会大量减少remark需要处理的内存数量，进而降低JVM卡顿对成功率的影响
2、对于Java HTTP服务，JVM的卡顿时间应该小于HTTP客户端的调用超时时间，否则JVM卡顿会对成功率造成影响

还有调优的例子

镜像服务的三个接口导致的，一开始觉得是环境的问题，没有考虑现网的数据量及环境信息，初步觉得是环境的问题，然后就修复环境，修改PodIP，发现没有效果，后来意识到不对，就在本地进行了一些测试，服务器上写定时任务去模拟环境，跑了一段时间，不停在容器中观察GC情况

```html
//查看所有正在运行的java程序
jps -l
//运行时进程参数和JVM参数
jinfo -flags [PID]
//按一定间隔时间查看实时的GC信息
jstat -gcutil 1 2000 50
//导出dump文件
jmap -dump:format=b,file=heap.hprof [PID]
```
这个[网站](http://heaphero.io/index.jsp)可以分析不太大的
[MAT](https://www.eclipse.org/mat/)可以分析大文件

1、首页中的【Leak Suspects】能推测出问题所在
2、点击【Create a histogram from an arbitrary set of objects】查到所有对象的数量
3、右键点击某个对象【Merge Shortest Paths to GC Roots】-> 【exclude all phantom/weak/soft etc. references】能查询到大量数量的某个对象是哪个GC ROOT引用的

如果老年代几乎已满，表明即使使用Full GC，老年代也在不断增长，这是内存泄漏的明显迹象。

#### 与JVM有关的问题

**1.synchronized关键字在jvm中的实现原理**

先来看下利用synchronized实现同步的基础：Java中的每一个对象都可以作为锁。
具体表现为以下3种形式。
	- 对于普通同步方法，锁是当前实例对象。
 	- 对于静态同步方法，锁是当前类的Class对象。
  - 对于同步方法块，锁是Synchonized括号里配置的对象。
当一个线程试图访问同步代码块时，它首先必须得到锁，退出或抛出异常时必须释放锁。

如果一个对象被锁了，锁的信息是会存储在这个对象的对象头中的，什么是对象头呢？每个java对象都有对象头，对象的对象头中存储着这个对象的信息，以32位jvm为例，一个对象的对象头是下图的结构的：
普通对象

![objectheader1]({{ "/assets/img/jvm/ObjectHeaderNormal.png" | relative_url}})

数组对象

![objectheader2]({{ "/assets/img/jvm/ObjectHeaderArray.png" | relative_url}})

我们的锁信息就存储在图中的Mark Word中，Mark Word的结构如下图所示

![markword]({{ "/assets/img/jvm/markword.png" | relative_url}})

由此可见，jvm根据一个对象的对象头中的Mark Work来判断对象是否加锁，但是问题又来了，从上图中我们看到锁的状态可不止一种，而是足足有四种之多，其中的无锁状态不用多说，那么其它三种状态又是什么呢？

我们使用synchronized关键字是为资源加一个锁，但是实际上synchronized会根据不同情况为资源自动选择不同的锁，在Java SE1.6中锁一共有4种状态，级别从低到高分别是无锁、偏向锁、轻量级锁、重量级锁，这几种状态会随着竞争情况逐渐升级，而且只可以升级但是不能降级降回来，这是为了提高获得锁和释放锁的效率。

1. 偏向锁
HotSpot虚拟机的作者经过研究发现，大多数情况下，锁不存在多线程竞争，而且总是由同一线程多次获得，如果在运行过程中，同步锁只有一个线程访问，不存在多线程争用的情况，则线程是不需要同步的，这种情况下，就会给线程加一个偏向锁，如果在运行过程中，遇到了其他线程抢占锁，则持有偏向锁的线程会被挂起，JVM会消除它身上的偏向锁，将锁恢复到标准的轻量级锁，当一个线程访问锁并获取锁的时候，会在对象头和栈帧中的锁记录中存储偏向锁偏向的线程的ID，以后该线程进出同步代码块的时候不需要进行CAS加锁解锁，只需要去看一眼加偏向锁的对象的Mark Word中存储的偏向线程ID是否等于当前访问的线程的ID。如果是，证明线程已经获得了锁；如果不是，先要去对象的MarkWord中看看当前是不是偏向锁，如果不是，就使用CAS竞争锁，如果是，就尝试使用CAS将对象的偏向锁偏向的线程ID改为当前线程的ID

2. 轻量级锁
在轻量级锁加锁的时候，一条线程在执行同步代码块的之前，会先在自己的线程帧中创建存储锁记录的空间，然后将对象的Mark Word复制到锁记录中，接着会尝试使用CAS操作将锁住的对象的Mark Word替换为指向这条线程的锁记录的指针。如果替换成功了，当前线程获得轻量级锁，如果失败了，就会一次次地重复尝试获取锁，**这叫做自旋**，当自旋了很多次依然没有成功的时候，锁就会膨胀，膨胀为重量级锁，此时如果依然没有争夺到资源，线程就会进入阻塞状态。
当释放一个轻量级锁的时候，会使用CAS操作把之前在线程锁记录中存储的Mark Word替换回对象头，如果成功了，表示没有竞争发生，如果失败了，则表示当前锁存在竞争，于是此线程在释放轻量级锁的时候就会唤醒等待的线程。

所以，我们简简单单地使用了一个synchronized关键字，但是背后jvm使用了锁膨胀策略来优化，因为使用重量级锁是一种比较耗费性能的事。

![lock]({{ "/assets/img/jvm/lockwithsync.png" | relative_url}})

补充下CAS，在Java并发应用中通常指CompareAndSwap或CompareAndSet，即比较并交换。

CAS是一个原子操作，它比较一个内存位置的值并且只有相等时修改这个内存位置的值为新的值，保证了新的值总是基于最新的信息计算的，如果有其他线程在这期间修改了这个值则CAS失败。CAS返回是否成功或者内存位置原来的值用于判断是否CAS成功。

执行函数：CAS(V,E,N)
其包含3个参数:
V表示要更新的变量
E表示预期值
N表示新值
如果V值等于E值，则将V的值设为N。若V值和E值不同，则说明已经有其他线程做了更新，则当前线程什么都不做。通俗的理解就是CAS操作需要我们提供一个期望值，当期望值与当前线程的变量值相同时，说明还没线程修改该值，当前线程可以进行修改，也就是执行CAS操作，但如果期望值与当前线程不符，则说明该值已被其他线程修改，此时不执行更新操作，但可以选择重新读取该变量再尝试再次修改该变量，也可以放弃操作.

JVM中的CAS操作是利用了处理器提供的CMPXCHG指令实现的。优点是竞争不大的时候系统开销小。缺点是性能代价高

1、循环时间长开销大。CAS长时间自旋不成功，给CPU带来很大的性能开销。解决方法：JVM能支持pause指令，效率会有一定的提升
2、只能保证一个共享变量的原子操作。对多个共享变量操作时，不能保证原子性。 解决方法：加锁；共享变量合并成一个共享变量
3、ABA的问题。解决方法就是：增加版本号，每次使用的时候版本号+1，每次变量更新的时候版本号+1。java提供AtomicStampzedReference来解决ABA问题

Java中锁的实现是使用的队列同步器，就是一个双端队列

独占锁和共享锁的区分：

独占式：有且只有一个线程能获取到锁，如：ReentrantLock
	- 每个节点自旋观察自己的前一节点是不是Header节点（同步器），如果是，就去尝试获取锁
共享式：可以多个线程同时获取到锁，如：CountDownLatch

ConcurrentHashMap使用的锁分段技术。首先将数据分成一段一段地存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问
