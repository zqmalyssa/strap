---
layout: post
title: jvm-sandbox
tags: [jvm-sandbox]
author-id: zqmalyssa
---

jvm-sandbox的一些分析

#### 分析

下面有一个比较详尽的分析

启动分析

[启动分析](https://developer.aliyun.com/article/716768)

启动时加载模块

[启动时加载模块](https://developer.aliyun.com/article/717553)

增强目标类

[增强目标类](https://developer.aliyun.com/article/717913)

模块刷新和卸载

(模块刷新和卸载)[https://developer.aliyun.com/article/717965]

沙箱事件

在JVM-Sandbox的世界观中，任何一个Java方法的调用都可以分解为BEFORE、RETURN和THROWS三个环节，由此在三个环节上引申出对应环节的事件探测和流程控制机制

1、BEFORE事件：执行方法体之前被调用
2、RETURN事件：执行方法体返回之前被调用
3、THROWS事件：执行方法体抛出异常之前被调用

为了记录代码调用行记录，增加了一个LineEvent

4、LINE事件：方法行被执行后调用，目前仅记录行号

CALL事件系列是从GREYS中衍生过来的事件，它描述了一个方法内部，调用其他方法的过程。整个过程可以被描述成为三个阶段

5、CALL_BEFORE事件：一个方法被调用之前
6、CALL_RETURN事件：一个方法被调用正常返回之后
7、CALL_THROWS事件：一个方法被调用抛出异常之后

```html

void foo(){
	// BEFORE-EVENT
	try {

   		/*
   	 	* do something...
   	 	*/
    	try{
    	    //LINE-EVENT
    	    //CALL_BEFORE-EVENT
    		a();
    		//CALL_RETURN-EVENT
    	} catch (Throwable cause) {
    		// CALL_THROWS-EVENT
		}
		//LiNE-EVENT
    	// RETURN-EVENT
    	return;

	} catch (Throwable cause) {
    	// THROWS-EVENT
	}
}

```

严格意义上，IMMEDIATELY_RETURN和IMMEDIATELY_THROWS不是事件，他们是流程控制机制，由com.alibaba.jvm.sandbox.api.ProcessControlException的throwReturnImmediately(Object)和throwThrowsImmediately(Throwable)触发，完成对方法的流程控制

IMMEDIATELY_RETURN：立即调用:RETURN事件

IMMEDIATELY_THROWS：立即调用:THROWS事件

```html

                                      +-------+
                                      |        |
+========+  <return>             +========+    | <return immediately>
|        |  <return immediately> |        |    |
| BEFORE |---------------------->| RETURN |<---+
|        |                       |        |
+========+                       +========+
|                                  |    ^
|         <throws immediately>     |    |
|                                  |    |   <return immediately>
|                                  v    |
|                                +========+
|                                |        |
+--------------------------->    | THROWS |<---+
              <throws>           |        |    |
       <throws immediately>      +========+    | <throws immediately>
                                       |       |
                                       +-------+

```


模块生命周期

模块生命周期
模块生命周期类型有模块加载、模块卸载、模块激活、模块冻结、模块加载完成五个状态。

模块加载：创建ClassLoader，完成模块的加载
模块卸载：模块增强的类会重新load，去掉增强的字节码
模块激活：模块被激活后，模块所增强的类将会被激活，所有com.alibaba.jvm.sandbox.api.listener.EventListener将开始收到对应的事件
模块冻结：模块被冻结后，模块所持有的所有com.alibaba.jvm.sandbox.api.listener.EventListener将被静默，无法收到对应的事件。需要注意的是，模块冻结后虽然不再收到相关事件，但沙箱给对应类织入的增强代码仍然还在。
模块加载完成：模块加载已经完成，这个状态是为了做日志处理，本身不会影响模块变更行为
模块可以通过实现com.alibaba.jvm.sandbox.api.ModuleLifecycle接口，对模块生命周期进行控制，接口中的方法：

onLoad：模块开始加载之前调用
onUnload：模块开始卸载之前调用
onActive：模块被激活之前调用，抛出异常将会是阻止模块被激活的唯一方式
onFrozen：模块被冻结之前调用，抛出异常将会是阻止模块被冻结的唯一方式


```html

<parent>
    <groupId>com.alibaba.jvm.sandbox</groupId>
    <artifactId>sandbox-module-starter</artifactId>
    <version>1.2.0</version>
</parent>

package com.alibaba.jvm.sandbox.demo;

import com.alibaba.jvm.sandbox.api.Information;
import com.alibaba.jvm.sandbox.api.Module;
import com.alibaba.jvm.sandbox.api.ProcessController;
import com.alibaba.jvm.sandbox.api.annotation.Command;
import com.alibaba.jvm.sandbox.api.listener.ext.Advice;
import com.alibaba.jvm.sandbox.api.listener.ext.AdviceListener;
import com.alibaba.jvm.sandbox.api.listener.ext.EventWatchBuilder;
import com.alibaba.jvm.sandbox.api.resource.ModuleEventWatcher;
import org.kohsuke.MetaInfServices;

import javax.annotation.Resource;

@MetaInfServices(Module.class)
@Information(id = "broken-clock-tinker")
public class BrokenClockTinkerModule implements Module {

    @Resource
    private ModuleEventWatcher moduleEventWatcher;

    @Command("repairCheckState")
    public void repairCheckState() {

        new EventWatchBuilder(moduleEventWatcher)
                .onClass("com.taobao.demo.Clock")
                .onBehavior("checkState")
                .onWatch(new AdviceListener() {

                    /**
                     * 拦截{@code com.taobao.demo.Clock#checkState()}方法，当这个方法抛出异常时将会被
                     * AdviceListener#afterThrowing()所拦截
                     */
                    @Override
                    protected void afterThrowing(Advice advice) throws Throwable {

                        // 在此，你可以通过ProcessController来改变原有方法的执行流程
                        // 这里的代码意义是：改变原方法抛出异常的行为，变更为立即返回；void返回值用null表示
                        ProcessController.returnImmediately(null);
                    }
                });

    }

}


1、运行命令完成打包

mvn clean package

2、将打好的包复制到用户模块目录下

cp target/clock-tinker-1.0-SNAPSHOT-jar-with-dependencies.jar ~/.sandbox-module/

3、下载并安装最新版本沙箱

下载地址：https://ompc.oss.aliyuncs.com/jvm-sandbox/release/sandbox-stable-bin.zip

执行安装

unzip sandbox-stable-bin.zip
cd sandbox

4、启动沙箱

假设目标进程号：64229

./sandbox.sh -p 64229
                  NAMESPACE : default
                    VERSION : 1.2.0
                       MODE : ATTACH
                SERVER_ADDR : 0.0.0.0
                SERVER_PORT : 56854
             UNSAFE_SUPPORT : ENABLE
               SANDBOX_HOME : /Users/vlinux/opt/sandbox
          SYSTEM_MODULE_LIB : /Users/vlinux/opt/sandbox/module
            USER_MODULE_LIB : ~/.sandbox-module;
        SYSTEM_PROVIDER_LIB : /Users/vlinux/opt/sandbox/provider
         EVENT_POOL_SUPPORT : DISABLE


./sandbox.sh -p 64229 -l
broken-clock-tinker ACTIVE  LOADED  0  0  UNKNOW_VERSION  UNKNOW_AUTHOR
sandbox-info        ACTIVE  LOADED  0  0  0.0.4           luanjia@taobao.com
sandbox-module-mgr  ACTIVE  LOADED  0  0  0.0.2           luanjia@taobao.com
sandbox-control     ACTIVE  LOADED  0  0  0.0.3           luanjia@taobao.com
total=4

触发broken-clock-tinker模块的repairCheckState()，让修复逻辑生效！

执行命令：触发BrokenClockTinkerModule#repairCheckState()方法执行

./sandbox.sh -p 64229 -d 'broken-clock-tinker/repairCheckState'

当你卸载掉JVM-SANDBOX时候，你就会发现原本已经被修复好的钟，又开始继续报错了。原因是原来通过clock-tinker模块修复的checkState()方法随着沙箱的卸载又恢复成原来错误的代码流程。

卸载沙箱

./sandbox.sh -p 64229 -S
jvm-sandbox[default] shutdown finished.

在这个教程中给大家演示了如何利用沙箱的模块改变了原有方法的执行流程，这里涉及到了沙箱最核心的类ModuleEventWatcher，这个类的实现可以通过@Resource注释注入进来。

```

#### 一些point

1、agent方式启动的好处

AGENT方式启动、有些时候我们需要沙箱工作在应用代码加载之前，或者一次性渲染大量的类、加载大量的模块，此时如果用ATTACH方式加载，可能会引起目标JVM的卡顿或停顿（GC），这就需要启用到AGENT的启动方式。

假设SANDBOX被安装在了/Users/luanjia/pe/sandbox，需要在JVM启动参数中增加上

2、模块常用操作

-F：强制刷新用户模块

沙箱容器在强制刷新的时候，首先会卸载当前所有已经被加载的用户模块，然后再重新对用户模块进行加载

a、首先卸载掉所有已加载的用户模块，然后再重新进行加载

b、模块卸载时将会释放掉沙箱为模块所有开启的资源

模块打开的HTTP链接
模块打开的WEBSOCKET链接
模块打开所在的ModuleClassLoader
模块进行的事件插桩

c、当任何一个模块加载失败时，忽略该模块，继续加载其他可加载的模块

-f：刷新用户模块

刷新用户模块，与强制刷新用户模块不同的地方是，普通刷新会遍历用户模块下所有发生改变的模块文件，当且仅对发生变化的文件进行重新加载操作。

-R：沙箱模块重置

沙箱模块重置的时候将会强制刷新所有的模块，包括用户模块和系统模块。

同样的是，sandbox.properties不会被重新加载

-u：卸载指定模块

卸载指定模块，支持通配符表达式子。卸载模块不会区分系统模块和用户模块，所有模块都可以通过这个参数完成卸载，所以切记不要轻易卸载module-mgr，否则你将失去模块管理功能，不然就只能-R来恢复了。

-a：激活模块

模块激活后才能受到沙箱事件

-A：冻结模块

模块冻结后将感知不到任何沙箱事件，但对应的代码插桩还在。

-d：模块自定义命令

在模块中可以通过对方法标记@Command注释，让sandbox.sh可以将自定义命令传递给被标记的方法。

此时对应过来的-d命令参数为：-d sandbox-info/version即可指定到这个方法。



#### sandbox破坏了双亲委派机制

1、什么是双亲委派机制

首先，我们知道，虚拟机在加载类的过程中需要使用类加载器进行加载，而在Java中，类加载器有很多，那么当JVM想要加载一个.class文件的时候，到底应该由哪个类加载器加载呢？

这就不得不提到”双亲委派机制”。

首先，我们需要知道的是，Java语言系统中支持以下4种类加载器：

Bootstrap ClassLoader 启动类加载器
Extention ClassLoader 标准扩展类加载器
Application ClassLoader 应用类加载器
User ClassLoader 用户自定义类加载器

这四种类加载器之间，是存在着一种层次关系的，如下

Bootstrap ClassLoader > Extention ClassLoader > Application ClassLoader > User ClassLoader

一般认为上一层加载器是下一层加载器的父加载器，那么，除了BootstrapClassLoader之外，所有的加载器都是有父加载器的。

那么，所谓的双亲委派机制，指的就是：当一个类加载器收到了类加载的请求的时候，他不会直接去加载指定的类，而是把这个请求委托给自己的父加载器去加载。只有父加载器无法加载这个类的时候，才会由当前这个加载器来负责类的加载。

那么，什么情况下父加载器会无法加载某一个类呢？

其实，Java中提供的这四种类型的加载器，是有各自的职责的：

Bootstrap ClassLoader ，主要负责加载Java核心类库，%JRE_HOME%\lib下的rt.jar、resources.jar、charsets.jar和class等。
Extention ClassLoader，主要负责加载目录%JRE_HOME%\lib\ext目录下的jar包和class文件。
Application ClassLoader ，主要负责加载当前应用的classpath下的所有类
User ClassLoader ， 用户自定义的类加载器,可加载指定路径的class文件

那么也就是说，一个用户自定义的类，如com.hollis.ClassHollis 是无论如何也不会被Bootstrap和Extention加载器加载的。


2、为什么需要双亲委派？


如上面我们提到的，因为类加载器之间有严格的层次关系，那么也就使得Java类也随之具备了层次关系。

或者说这种层次关系是优先级。

比如一个定义在java.lang包下的类，因为它被存放在rt.jar之中，所以在被加载过程汇总，会被一直委托到Bootstrap ClassLoader，最终由Bootstrap ClassLoader所加载。

而一个用户自定义的com.hollis.ClassHollis类，他也会被一直委托到Bootstrap ClassLoader，但是因为Bootstrap ClassLoader不负责加载该类，那么会在由Extention ClassLoader尝试加载，而Extention ClassLoader也不负责这个类的加载，最终才会被Application ClassLoader加载。

这种机制有几个好处

首先，通过委派的方式，可以避免类的重复加载，当父加载器已经加载过某一个类时，子加载器就不会再重新加载这个类。

另外，通过双亲委派的方式，还保证了安全性。因为Bootstrap ClassLoader在加载的时候，只会加载JAVA_HOME中的jar包里面的类，如java.lang.Integer，那么这个类是不会被随意替换的，除非有人跑到你的机器上， 破坏你的JDK。

那么，就可以避免有人自定义一个有破坏功能的java.lang.Integer被加载。这样可以有效的防止核心Java API被篡改。

3、“父子加载器”之间的关系是继承吗？

很多人看到父加载器、子加载器这样的名字，就会认为Java中的类加载器之间存在着继承关系。

甚至网上很多文章也会有类似的错误观点。

这里需要明确一下，双亲委派模型中，类加载器之间的父子关系一般不会以继承（Inheritance）的关系来实现，而是都使用组合（Composition）关系来复用父加载器的代码的。

如下为ClassLoader中父加载器的定义：

```html

public abstract class ClassLoader {
    // The parent class loader for delegation
    private final ClassLoader parent;
}

```

4、双亲委派是怎么实现的？

双亲委派模型对于保证Java程序的稳定运作很重要，但它的实现并不复杂。

实现双亲委派的代码都集中在java.lang.ClassLoader的loadClass()方法之中：

```html

protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }


```

代码不难理解，主要就是以下几个步骤：

1)先检查类是否已经被加载过
2)若没有加载则调用父加载器的loadClass()方法进行加载
3)若父加载器为空则默认使用启动类加载器作为父加载器。
4)如果父类加载失败，抛出ClassNotFoundException异常后，再调用自己的findClass()方法进行加载

5、如何主动破坏双亲委派机制？

知道了双亲委派模型的实现，那么想要破坏双亲委派机制就很简单了。

因为他的双亲委派过程都是在loadClass方法中实现的，那么想要破坏这种机制，那么就自定义一个类加载器，重写其中的loadClass方法，使其不进行双亲委派即可。

6、loadClass（）、findClass（）、defineClass（）区别

ClassLoader中和类加载有关的方法有很多，前面提到了loadClass，除此之外，还有findClass和defineClass等，那么这几个方法有什么区别呢？

loadClass()

就是主要进行类加载的方法，默认的双亲委派机制就实现在这个方法中。

findClass()
根据名称或位置加载.class字节码

defineclass()
把字节码转化为Class

这里面需要展开讲一下loadClass和findClass，我们前面说过，当我们想要自定义一个类加载器的时候，并且像破坏双亲委派原则时，我们会重写loadClass方法。

那么，如果我们想定义一个类加载器，但是不想破坏双亲委派模型的时候呢？

这时候，就可以继承ClassLoader，并且重写findClass方法。findClass()方法是JDK1.2之后的ClassLoader新添加的一个方法。


```html

/**
* @since  1.2
*/
protected Class<?> findClass(String name) throws ClassNotFoundException {
   throw new ClassNotFoundException(name);
}

```

这个方法只抛出了一个异常，没有默认实现。

JDK1.2之后已不再提倡用户直接覆盖loadClass()方法，而是建议把自己的类加载逻辑实现到findClass()方法中。

因为在loadClass()方法的逻辑里，如果父类加载器加载失败，则会调用自己的findClass()方法来完成加载。

所以，如果你想定义一个自己的类加载器，并且要遵守双亲委派模型，那么可以继承ClassLoader，并且在findClass中实现你自己的加载逻辑即可。

7、双亲委派被破坏的例子

双亲委派机制的破坏不是什么稀奇的事情，很多框架、容器等都会破坏这种机制来实现某些功能

第一种被破坏的情况是在双亲委派出现之前。

由于双亲委派模型是在JDK1.2之后才被引入的，而在这之前已经有用户自定义类加载器在用了。所以，这些是没有遵守双亲委派原则的。

第二种，是JNDI、JDBC等需要加载SPI接口实现类的情况。

第三种是为了实现热插拔热部署工具。为了让代码动态生效而无需重启，实现方式时把模块连同类加载器一起换掉就实现了代码的热替换。

第四种时tomcat等web容器的出现。

第五种时OSGI、Jigsaw等模块化技术的应用。

8、为什么JNDI，JDBC等需要破坏双亲委派？

我们日常开发中，大多数时候会通过API的方式调用Java提供的那些基础类，这些基础类时被Bootstrap加载的。

但是，调用方式除了API之外，还有一种SPI的方式。

如典型的JDBC服务，我们通常通过以下方式创建数据库连接：

```html

Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/mysql", "root", "1234");

```

在以上代码执行之前，DriverManager会先被类加载器加载，因为java.sql.DriverManager类是位于rt.jar下面的 ，所以他会被根加载器加载。

类加载时，会执行该类的静态方法。其中有一段关键的代码是：

```html

ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);

```

这段代码，会尝试加载classpath下面的所有实现了Driver接口的实现类。

那么，问题就来了。

DriverManager是被根加载器加载的，那么在加载时遇到以上代码，会尝试加载所有Driver的实现类，但是这些实现类基本都是第三方提供的，根据双亲委派原则，第三方的类不能被根加载器加载。

那么，怎么解决这个问题呢？

于是，就在JDBC中通过引入ThreadContextClassLoader（线程上下文加载器，默认情况下是AppClassLoader）的方式破坏了双亲委派原则。

我们深入到ServiceLoader.load方法就可以看到：

```html

public static <S> ServiceLoader<S> load(Class<S> service) {
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
}

```

第一行，获取当前线程的线程上下⽂类加载器 AppClassLoader，⽤于加载 classpath 中的具体实现类。


9、为什么Tomcat要破坏双亲委派

我们知道，Tomcat是web容器，那么一个web容器可能需要部署多个应用程序。

不同的应用程序可能会依赖同一个第三方类库的不同版本，但是不同版本的类库中某一个类的全路径名可能是一样的。

如多个应用都要依赖hollis.jar，但是A应用需要依赖1.0.0版本，但是B应用需要依赖1.0.1版本。这两个版本中都有一个类是com.hollis.Test.class。

如果采用默认的双亲委派类加载机制，那么是无法加载多个相同的类。（不会重复加载的特性）

所以，Tomcat破坏双亲委派原则，提供隔离的机制，为每个web容器单独提供一个WebAppClassLoader加载器。（dump中能够看见）

Tomcat的类加载机制：为了实现隔离性，优先加载 Web 应用自己定义的类，所以没有遵照双亲委派的约定，每一个应用自己的类加载器——WebAppClassLoader负责加载本身的目录下的class文件，加载不到时再交给CommonClassLoader加载，这和双亲委派刚好相反。

10、模块化技术与类加载机制

近几年模块化技术已经很成熟了，在JDK 9中已经应用了模块化的技术。

其实早在JDK 9之前，OSGI这种框架已经是模块化的了，而OSGI之所以能够实现模块热插拔和模块内部可见性的精准控制都归结于其特殊的类加载机制，加载器之间的关系不再是双亲委派模型的树状结构，而是发展成复杂的网状结构。

在JDK9之前，JVM的基础类以前都是在rt.jar这个包里，这个包也是JRE运行的基石。

这不仅是违反了单一职责原则，同样程序在编译的时候会将很多无用的类也一并打包，造成臃肿。

在JDK9中，整个JDK都基于模块化进行构建，以前的rt.jar, tool.jar被拆分成数十个模块，编译的时候只编译实际用到的模块，同时各个类加载器各司其职，只加载自己负责的模块。
