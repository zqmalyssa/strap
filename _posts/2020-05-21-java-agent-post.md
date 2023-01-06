---
layout: post
title: Java的Agent
tags: [code, java]
author-id: zqmalyssa
---

Java Agent，也是最近做项目才发现的东西。。

#### Java Agent介绍

下面这些技术都使用了Java Agent 技术，看一下你就知道为什么了。

各个 Java IDE 的调试功能，例如 eclipse、IntelliJ ；
热部署功能，例如 JRebel、XRebel、 spring-loaded；
各种线上诊断工具，例如 Btrace、Greys，还有阿里的 Arthas；
各种性能分析工具，例如 Visual VM、JConsole 等；

Java Agent 直译过来叫做 Java 代理，还有另一种称呼叫做 Java 探针。首先说 Java Agent 是一个 jar 包，只不过这个 jar 包不能独立运行，它需要依附到我们的目标 JVM 进程中。我们来理解一下这两种叫法。

代理：比方说我们需要了解目标 JVM 的一些运行指标，我们可以通过 Java Agent 来实现，这样看来它就是一个代理的效果，我们最后拿到的指标是目标 JVM ,但是我们是通过 Java Agent 来获取的，对于目标 JVM 来说，它就像是一个代理；

探针：这个说法我感觉非常形象，JVM 一旦跑起来，对于外界来说，它就是一个黑盒。而 Java Agent 可以像一支针一样插到 JVM 内部，探到我们想要的东西，并且可以注入东西进去。

拿上面的几个我们平时会用到的技术举例子。拿 IDEA 调试器来说吧，当开启调试功能后，在 debugger 面板中可以看到当前上下文变量的结构和内容，还可以在 watches 面板中运行一些简单的代码，比如取值赋值等操作。还有 Btrace、Arthas 这些线上排查问题的工具，比方说有接口没有按预期的返回结果，但日志又没有错误，这时，我们只要清楚方法的所在包名、类名、方法名等，不用修改部署服务，就能查到调用的参数、返回值、异常等信息。


在方法中插入代码主要是用到了字节码修改技术，字节码修改技术主要有 javassist、ASM，以及 ASM 的高级封装可扩展 cglib，这个例子中用的是 javassist。所以需要引入相关的 maven 包。

```java
<dependency>
   <groupId>javassist</groupId>
   <artifactId>javassist</artifactId>
   <version>3.12.1.GA</version>
</dependency>
```

mark，这边javasist是有坑的，先跑个小栗子

首先写java agent

```java
package com.qiming.test.javaagent;

import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;
import java.io.ByteArrayInputStream;
import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.lang.instrument.Instrumentation;
import java.security.ProtectionDomain;

/**
 * Created by qmzhang on 2020/05/21
 */
public class MyTransformer implements ClassFileTransformer {

    public static void premain(String options, Instrumentation ins) {
        if (options != null) {
            System.out.printf("  I've been called with options: \"%s\"\n",
                options);
        } else
            System.out.println("  I've been called with no options.");
        ins.addTransformer(new MyTransformer());
    }

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws
        IllegalClassFormatException {
        System.out.println("正在加载类："+ className);
        if (!"com/qiming/test/ctrip/javaagent/Person".equals(className)){
            return classfileBuffer;
        }
        String line="";
        for(int i=0;i<classfileBuffer.length;i++){
            line +=Byte.toString(classfileBuffer[i])+" ";
            if(line.length()>60){
                System.out.println(line);
                line="";
            }
            //这边就是替换字节码了
            if(classfileBuffer[i]==(byte)'2')
                classfileBuffer[i]=(byte)'7';
        }
        System.out.println(line);
        System.out.println("The number of bytes in Person: "+classfileBuffer.length);

        // 打开这边的注释进行

//        CtClass cl = null;
//        try {
//            System.out.println(classfileBuffer.toString());
//            System.out.println("Am I here Now 1??");
//            ClassPool classPool = ClassPool.getDefault();
//            cl = classPool.makeClass(new ByteArrayInputStream(classfileBuffer));
////            cl = classPool.get(className);
//            System.out.println(cl);
//            CtMethod ctMethod = cl.getDeclaredMethod("test");
//            System.out.println("Am I here Now 2??");
//            System.out.println(ctMethod);
//            System.out.println("获取方法名称："+ ctMethod.getName());
//            ctMethod.insertBefore("System.out.println(\" 动态插入的打印语句 \");");
//            ctMethod.insertAfter("System.out.println($_);");
//            byte[] transformed = cl.toBytecode();
//            return transformed;
//        }catch (Exception e){
//            System.out.println("Am I here Now 3??");
//            e.printStackTrace();
//        }
        return classfileBuffer;
    }

}
```
配置项目pom

```java
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.qiming.test</groupId>
    <artifactId>javaagent</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>

        <dependency>
            <groupId>javassist</groupId>
            <artifactId>javassist</artifactId>
            <version>3.12.1.GA</version>
        </dependency>

    </dependencies>

    <build>
        <plugins>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <configuration>
                    <archive>
                        <manifestEntries>
                            <Premain-Class>com.qiming.test.javaagent.MyTransformer</Premain-Class>
                            <Agent-Class>com.qiming.test.javaagent.MyTransformer</Agent-Class>
                            <Can-Redefine-Classes>true</Can-Redefine-Classes>
                            <Can-Retransform-Classes>true</Can-Retransform-Classes>
                        </manifestEntries>
                    </archive>
                </configuration>
            </plugin>
        </plugins>
    </build>


</project>
```

注意plugin中premain class和agent class的对应值

然后打成jar包，接着写个客户端，或者说agent所寄生的地方

```java
package com.qiming.test.ctrip.javaagent;

import java.util.Scanner;

/**
 * Created by qmzhang on 2020/05/21
 */
public class RunJVM {

    public static void main(String[] args){
        System.out.println("按数字键 1 调用测试方法");
        while (true) {
            Scanner reader = new Scanner(System.in);
            int number = reader.nextInt();
            if(number==1){
                Person person = new Person();
                person.test();
            }
        }
    }

}

package com.qiming.test.ctrip.javaagent;

import java.util.Scanner;

/**
 * Created by qmzhang on 2020/05/21
 */
public class RunJVM {

    public static void main(String[] args){
        System.out.println("按数字键 1 调用测试方法");
        while (true) {
            Scanner reader = new Scanner(System.in);
            int number = reader.nextInt();
            if(number==1){
                Person person = new Person();
                person.test();
            }
        }
    }

}


package com.qiming.app.fan.web.javaagent;

/**
 * Created by qmzhang on 2022/10/20
 */
public class Person {

  public String test() {
    System.out.println("执行测试方法");
    return "I'm ok";
  }

}


```

启动的时候需要指定参数为之前build好的agent jar

```java
-javaagent:D:\Code\FanTestPom\target\javaagent-1.0-SNAPSHOT.jar
```

然后运行就可以出效果了，在加载类的时候动态的修改了字节码，将改变后的值就行输出

这里补充说明：


Java Agent 最终以 jar 包的形式存在。主要包含两个部分，一部分是实现代码，一部分是配置文件。

```html
                              agentmain   com.sun.tools.attach.VirtualMachine  loadAgent/detach   // 动态attch到目标jvm上
            1、Agent class
                              premain   -javaagent:xxx.jar，在启动的时候加上参数的方式执行，在主进程main方法之前执行

Java agent  


            2、Packaging    META-INF/MANIFEST.MF  Agentmain-Class/Premain-Class/Boot-Class-Path/Can-Redefine-Classes/Can-Retransform-Classes

                            How to avoid packaging in development? -javaagent:<jarfile>[=arguments]
```

配置文件放在 META-INF 目录下，文件名为 MANIFEST.MF 。包括以下配置项：

Manifest-Version: 版本号
 Created-By: 创作者
 Agent-Class: agentmain 方法所在类
 Can-Redefine-Classes: 是否可以实现类的重定义
 Can-Retransform-Classes: 是否可以实现字节码替换
 Premain-Class: premain 方法所在类

入口类实现 agentmain 和 premain 两个方法即可，方法要实现什么功能就由你的需求决定了。

两个方法。这两个方法的运行时机不一样。这要从 Java Agent 的使用方式来说了，Java Agent 有两种启动方式，一种是以 JVM 启动参数 -javaagent:xxx.jar 的形式随着 JVM 一起启动，这种情况下，会调用 premain方法，并且是在主进程的 main方法之前执行。另外一种是以 loadAgent 方法动态 attach 到目标 JVM 上，这种情况下，会执行 agentmain方法

premain的可以在idea的VM options中加入参数 -javaagent

动态attach的方式，需要先启动项目，找到pid，ps或者jps都行，动态 attach 的方式是需要代码实现的，实现代码如下：

```java

public class AttachAgent {

  public static void main(String[] args) throws Exception {
    VirtualMachine vm = VirtualMachine.attch("pid(进程号)");
    vm.loadAgent("agent路径/test-jar.jar");
  }

}

```

这种就是动态挂载了，本质上还是需要一个jar包的

所以打包成jar包的方式有

```java

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-assembly-plugin</artifactId>
            <configuration>
                <archive>
                    <manifestFile>src/main/resources/META-INF/MANIFEST.MF</manifestFile>
                </archive>
                <descriptorRefs>
                    <descriptorRef>jar-with-dependencies</descriptorRef>
                </descriptorRefs>
            </configuration>
        </plugin>
    </plugins>
</build>


```

用的是 maven 的 maven-assembly-plugin 插件，注意其中要用 manifestFile 指定 MANIFEST.MF 所在路径，然后指定 jar-with-dependencies ，将依赖包打进去。

上面这是一种打包方式，需要单独的 MANIFEST.MF 配合，还有一种方式，不需要在项目中单独的添加 MANIFEST.MF 配置文件，完全在 pom 文件中配置上即可。如下

```html

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-assembly-plugin</artifactId>
            <executions>
                <execution>
                    <goals>
                        <goal>attached</goal>
                    </goals>
                    <phase>package</phase>
                    <configuration>
                        <descriptorRefs>
                            <descriptorRef>jar-with-dependencies</descriptorRef>
                        </descriptorRefs>
                        <archive>
                            <manifestEntries>
                                <Premain-Class>kite.agent.vmargsmethod.MyAgent</Premain-Class>
                                <Agent-Class>kite.agent.vmargsmethod.MyAgent</Agent-Class>
                                <Can-Redefine-Classes>true</Can-Redefine-Classes>
                                <Can-Retransform-Classes>true</Can-Retransform-Classes>
                            </manifestEntries>
                        </archive>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>

```

这种方式是将 MANIFEST.MF 的内容全部写作 pom 配置中，打包的时候就会自动将配置信息生成 MANIFEST.MF 配置文件打进包里。

接下来就简单了，执行一条 maven 命令即可。

```html

mvn assembly:assembly

```

最后打出来的 jar 包默认是以「项目名称-版本号-jar-with-dependencies.jar」这样的格式生成到 target 目录下。


平时用过 visualVM 或者 JConsole 之类的工具，其实，它们就是用了 management-agent.jar 这个Java Agent 来实现的。如果我们希望 Java 服务允许远程查看 JVM 信息，往往会配置上一下这些参数：

```java

-Dcom.sun.management.jmxremote
-Djava.rmi.server.hostname=192.168.1.1
-Dcom.sun.management.jmxremote.port=9999
-Dcom.sun.management.jmxremote.rmi.port=9999
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false

```
这些参数都是 management-agent.jar 定义的。我们进到 management-agent.jar 包下，看到只有一个 MANIFEST.MF 配置文件，配置内容为：


```java

Manifest-Version: 1.0
Created-By: 1.7.0_07 (Oracle Corporation)
Agent-Class: sun.management.Agent
Premain-Class: sun.management.Agent

```

可以看到入口 class 为 sun.management.Agent，进到这个类里面可以找到 agentmain 和 premain，并可以看到它们的逻辑。在这个类的开始，能看到我们前面对服务开启远程 JVM 监控需要开启的那些参数定义。
