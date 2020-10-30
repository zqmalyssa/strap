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


在方法中插入代码主要是用到了字节码修改技术，字节码修改技术主要有 javassist、ASM，已经 ASM 的高级封装可扩展 cglib，这个例子中用的是 javassist。所以需要引入相关的 maven 包。

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
```

启动的时候需要指定参数为之前build好的agent jar

```java
-javaagent:D:\Code\FanTestPom\target\javaagent-1.0-SNAPSHOT.jar
```

然后运行就可以出效果了，在加载类的时候动态的修改了字节码，将改变后的值就行输出
