---
layout: post
title: Spring的那些事
tags: [code, java, spring]
author-id: zqmalyssa
---

生活就像一盒巧克力，你永远不知道打开来是什么，都知道谁说的，每个人都有每个人的体会，这篇体会一下著名的Spring，是甜还是苦

### 概念介绍

1. 一站式框架：管理项目中的对象。spring框架性质是容器（对象容器）

2. 核心是控制反转（IOC）和面向切面（AOP）

    - IOC：反转控制--将创建对象的方式反转

        - 自己创建、维护对象-->由spring完成创建、注入

        - 反转控制就是反转了对象的创建方式，从自己创建反转给了程序

    - DI：依赖注入--实现IOC需要DI做支持

        - 注入方式：set、构造方法、字段 注入

        - 注入类型：值类型（8大数据类型）、引用类型（对象） 注入


### 简单实现


创建一个对象bean

```java
package com.dic.bean;

public class User {

    private String name;
    private String age;


    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public String getAge() {
        return age;
    }
    public void setAge(String age) {
        this.age = age;
    }

}
```

配置核心xml文件，applicationContext.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://www.springframework.org/schema/beans"
xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.2.xsd ">
    <!-- 将user对象交给spring -->
    <!--
        name:调用时用的名字
        class：路径
    -->
       <bean name="user" class="com.dic.bean.User"></bean>   
</beans>
```

junit测试以下代码，控制台打印不为空且无报错即为成功

```java
package com.dic.text;

import org.junit.Test;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

import com.dic.bean.User;

public class Demo1 {

    @Test
    public void fun1(){

        //1 创建容器对象
        ApplicationContext ac = new ClassPathXmlApplicationContext("applicationContext.xml");
        //2 向容器"要"user对象
        User u = (User) ac.getBean("user");
        //3 打印user对象
        System.out.println(u);


    }

}
```

### Spring解决循环依赖

我将会从获取 bean 的方法getBean(String)开始，把整个调用过程梳理一遍。

什么是循环依赖呢，看下面的代码和配置信息

```java
public class BeanB {
    private BeanA beanA;
    // 省略 getter/setter
}

public class BeanA {
    private BeanB beanB;
}

//配置
<bean id="beanA" class="xyz.coolblog.BeanA">
    <property name="beanB" ref="beanB"/>
</bean>
<bean id="beanB" class="xyz.coolblog.BeanB">
    <property name="beanA" ref="beanA"/>
</bean>
```

IOC 容器在读到上面的配置时，会按照顺序，先去实例化 beanA。然后发现 beanA 依赖于 beanB，接在又去实例化 beanB。实例化 beanB 时，发现 beanB 又依赖于 beanA。如果容器不处理循环依赖的话，容器会无限执行上面的流程，直到内存溢出，程序崩溃。当然，Spring 是不会让这种情况发生的。

在容器再次发现 beanB 依赖于 beanA 时，容器会获取 beanA 对象的一个早期的引用（early reference），并把这个早期引用注入到 beanB 中，让 beanB 先完成实例化。beanA 就可以获取到 beanB 的引用，beanA 随之完成实例化。

源码分析前，看一组缓存的定义，DefaultSingletonBeanRegistry.java 部分代码如下：

```java
/** Cache of singleton objects: bean name --> bean instance */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);

/** Cache of singleton factories: bean name --> ObjectFactory */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);

/** Cache of early singleton objects: bean name --> bean instance */
private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);
```

这些缓存的作用见表格

| 缓存 | 用途 |
| :----: | :----:  |
| singletonObjects | 用于存放完全初始化好的 bean，从该缓存中取出的 bean 可以直接使用 |
| earlySingletonObjects | 存放原始的 bean 对象（尚未填充属性），用于解决循环依赖 |
| singletonFactories | 存放 bean 工厂对象，用于解决循环依赖 |

上面提到了”早期引用“，所谓的”早期引用“是指向原始对象的引用。所谓的原始对象是指刚创建好的对象，但还未填充属性。这样讲大家不知道大家听明白了没，不过没听明白也不要紧。简单做个实验就知道了，这里我们先定义一个对象 Room：

```java
/** Room 包含了一些电器 */
public class Room {
    private String television;
    private String airConditioner;
    private String refrigerator;
    private String washer;
    // 省略 getter/setter
}


//配置
<bean id="room" class="xyz.coolblog.demo.Room">
    <property name="television" value="Xiaomi"/>
    <property name="airConditioner" value="Gree"/>
    <property name="refrigerator" value="Haier"/>
    <property name="washer" value="Siemens"/>
</bean>
```

配置好的bean属性全部注入了，没配置好的，属性值还都是null，这就是一个原始bean，它们在debug的时候是指向的同一个对象

再看看从SpringIOC容器中获取bean实例的流程（简化版）

![getbean]({{ "/assets/img/spring/getbean.png" | relative_url}})


开始流程图中只有一条执行路径，在条件 sharedInstance != null 这里出现了岔路，形成了绿色和红色两条路径。在上图中，读取/添加缓存的方法我用蓝色的框和☆标注了出来。至于虚线的箭头，和虚线框里的路径，这个下面会说到。

我来按照上面的图，分析一下整个流程的执行顺序。这个流程从 getBean 方法开始，getBean 是个空壳方法，所有逻辑都在 doGetBean 方法中。doGetBean 首先会调用 getSingleton(beanName) 方法获取 sharedInstance，sharedInstance 可能是完全实例化好的 bean，也可能是一个原始的 bean，当然也有可能是 null。如果不为 null，则走绿色的那条路径。再经 getObjectForBeanInstance 这一步处理后，绿色的这条执行路径就结束了。

我们再来看一下红色的那条执行路径，也就是 sharedInstance = null 的情况。在第一次获取某个 bean 的时候，缓存中是没有记录的，所以这个时候要走创建逻辑。上图中的 getSingleton(beanName,new ObjectFactory<Object>() {...}) 方法会创建一个 bean 实例，上图虚线路径指的是 getSingleton 方法内部调用的两个方法，其逻辑如下：

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    // 省略部分代码
    singletonObject = singletonFactory.getObject();
    // ...
    addSingleton(beanName, singletonObject);
}
```

如上所示，getSingleton 会在内部先调用 getObject 方法创建 singletonObject，然后再调用 addSingleton 将 singletonObject 放入缓存中。getObject 在内部调用了 createBean 方法，createBean 方法基本上也属于空壳方法，更多的逻辑是写在 doCreateBean 方法中的。doCreateBean 方法中的逻辑很多，其首先调用了 createBeanInstance 方法创建了一个原始的 bean 对象，随后调用 addSingletonFactory 方法向缓存中添加单例 bean 工厂，从该工厂可以获取原始对象的引用，也就是所谓的“早期引用”。再之后，继续调用 populateBean 方法向原始 bean 对象中填充属性，并解析依赖。getObject 执行完成后，会返回完全实例化好的 bean。紧接着再调用 addSingleton 把完全实例化好的 bean 对象放入缓存中。到这里，红色执行路径差不多也就要结束的。

源码看下分析

```java
protected <T> T doGetBean(
            final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
            throws BeansException {

    // ......

    // 从缓存中获取 bean 实例
    Object sharedInstance = getSingleton(beanName);

    // ......
}

public Object getSingleton(String beanName) {
    return getSingleton(beanName, true);
}

protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 从 singletonObjects 获取实例，singletonObjects 中的实例都是准备好的 bean 实例，可以直接使用
    Object singletonObject = this.singletonObjects.get(beanName);
    // 判断 beanName 对应的 bean 是否正在创建中
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            // 从 earlySingletonObjects 中获取提前曝光的 bean
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                // 获取相应的 bean 工厂
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    // 提前曝光 bean 实例（raw bean），用于解决循环依赖
                    singletonObject = singletonFactory.getObject();

                    // 将 singletonObject 放入缓存中，并将 singletonFactory 从缓存中移除
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return (singletonObject != NULL_OBJECT ? singletonObject : null);
```

上面的源码中，doGetBean 所调用的方法 getSingleton(String) 是一个空壳方法，其主要逻辑在 getSingleton(String, boolean) 中。该方法逻辑比较简单，首先从 singletonObjects 缓存中获取 bean 实例。若未命中，再去 earlySingletonObjects 缓存中获取原始 bean 实例。如果仍未命中，则从 singletonFactory 缓存中获取 ObjectFactory 对象，然后再调用 getObject 方法获取原始 bean 实例的引用，也就是早期引用。获取成功后，将该实例放入 earlySingletonObjects 缓存中，并将 ObjectFactory 对象从 singletonFactories 移除。看完这个方法，我们再来看看 getSingleton(String, ObjectFactory) 方法，这个方法也是在 doGetBean 中被调用的。这次我会把 doGetBean 的代码多贴一点出来，如下：

```java
protected <T> T doGetBean(
        final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
        throws BeansException {

    // ......
    Object bean;

    // 从缓存中获取 bean 实例
    Object sharedInstance = getSingleton(beanName);

    // 这里先忽略 args == null 这个条件
    if (sharedInstance != null && args == null) {
        // 进行后续的处理
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    } else {
        // ......

        // mbd.isSingleton() 用于判断 bean 是否是单例模式
        if (mbd.isSingleton()) {
            // 再次获取 bean 实例
            sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
                @Override
                public Object getObject() throws BeansException {
                    try {
                        // 创建 bean 实例，createBean 返回的 bean 是完全实例化好的
                        return createBean(beanName, mbd, args);
                    } catch (BeansException ex) {
                        destroySingleton(beanName);
                        throw ex;
                    }
                }
            });
            // 进行后续的处理
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        }

        // ......
    }

    // ......

    // 返回 bean
    return (T) bean;
}
```

这里的代码逻辑和我在回顾获取 bean 的过程 一节的最后贴的主流程图已经很接近了，对照那张图和代码中的注释，大家应该可以理解 doGetBean 方法了。继续往下看：

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    synchronized (this.singletonObjects) {

        // ......

        // 调用 getObject 方法创建 bean 实例
        singletonObject = singletonFactory.getObject();
        newSingleton = true;

        if (newSingleton) {
            // 添加 bean 到 singletonObjects 缓存中，并从其他集合中将 bean 相关记录移除
            addSingleton(beanName, singletonObject);
        }

        // ......

        // 返回 singletonObject
        return (singletonObject != NULL_OBJECT ? singletonObject : null);
    }
}

protected void addSingleton(String beanName, Object singletonObject) {
    synchronized (this.singletonObjects) {
        // 将 <beanName, singletonObject> 映射存入 singletonObjects 中
        this.singletonObjects.put(beanName, (singletonObject != null ? singletonObject : NULL_OBJECT));

        // 从其他缓存中移除 beanName 相关映射
        this.singletonFactories.remove(beanName);
        this.earlySingletonObjects.remove(beanName);
        this.registeredSingletons.add(beanName);
    }
}
```

上面的代码中包含两步操作，第一步操作是调用 getObject 创建 bean 实例，第二步是调用 addSingleton 方法将创建好的 bean 放入缓存中。代码逻辑并不复杂，相信大家都能看懂。那么接下来我们继续往下看，这次分析的是 doCreateBean 中的一些逻辑。如下

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
        throws BeanCreationException {

    BeanWrapper instanceWrapper = null;

    // ......

    // ☆ 创建 bean 对象，并将 bean 对象包裹在 BeanWrapper 对象中返回
    instanceWrapper = createBeanInstance(beanName, mbd, args);

    // 从 BeanWrapper 对象中获取 bean 对象，这里的 bean 指向的是一个原始的对象
    final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);

    /*
     * earlySingletonExposure 用于表示是否”提前暴露“原始对象的引用，用于解决循环依赖。
     * 对于单例 bean，该变量一般为 true。更详细的解释可以参考我之前的文章
     */
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        // ☆ 添加 bean 工厂对象到 singletonFactories 缓存中
        addSingletonFactory(beanName, new ObjectFactory<Object>() {
            @Override
            public Object getObject() throws BeansException {
                /*
                 * 获取原始对象的早期引用，在 getEarlyBeanReference 方法中，会执行 AOP
                 * 相关逻辑。若 bean 未被 AOP 拦截，getEarlyBeanReference 原样返回
                 * bean，所以大家可以把
                 *      return getEarlyBeanReference(beanName, mbd, bean)
                 * 等价于：
                 *      return bean;
                 */
                return getEarlyBeanReference(beanName, mbd, bean);
            }
        });
    }

    Object exposedObject = bean;

    // ......

    // ☆ 填充属性，解析依赖
    populateBean(beanName, mbd, instanceWrapper);

    // ......

    // 返回 bean 实例
    return exposedObject;
}

protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
    synchronized (this.singletonObjects) {
        if (!this.singletonObjects.containsKey(beanName)) {
            // 将 singletonFactory 添加到 singletonFactories 缓存中
            this.singletonFactories.put(beanName, singletonFactory);

            // 从其他缓存中移除相关记录，即使没有
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.add(beanName);
        }
    }
}
```

上面的代码简化了不少，不过看起来仍有点复杂。好在，上面代码的主线逻辑比较简单，由三个方法组成。如下：

```java
1. 创建原始 bean 实例 → createBeanInstance(beanName, mbd, args)
2. 添加原始对象工厂对象到 singletonFactories 缓存中
        → addSingletonFactory(beanName, new ObjectFactory<Object>{...})
3. 填充属性，解析依赖 → populateBean(beanName, mbd, instanceWrapper)
```

到这里，本节涉及到的源码就分析完了。可是看完源码后，我们似乎仍然不知道这些源码是如何解决循环依赖问题的。难道本篇文章就到这里了吗？答案是否。下面我来解答这个问题，这里我还是以 BeanA 和 BeanB 两个类相互依赖为例。在上面的方法调用中，有几个关键的地方，下面一一列举出来：

1、创建原始 bean 对象

```java
instanceWrapper = createBeanInstance(beanName, mbd, args);
final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
```
假设 beanA 先被创建，创建后的原始对象为 BeanA@1234，上面代码中的 bean 变量指向就是这个对象。

2、暴露早期引用

```java
addSingletonFactory(beanName, new ObjectFactory<Object>() {
    @Override
    public Object getObject() throws BeansException {
        return getEarlyBeanReference(beanName, mbd, bean);
    }
});
```
beanA 指向的原始对象创建好后，就开始把指向原始对象的引用通过 ObjectFactory 暴露出去。getEarlyBeanReference 方法的第三个参数 bean 指向的正是 createBeanInstance 方法创建出原始 bean 对象 BeanA@1234。

3、解析依赖

```java
populateBean(beanName, mbd, instanceWrapper);
```
populateBean 用于向 beanA 这个原始对象中填充属性，当它检测到 beanA 依赖于 beanB 时，会首先去实例化 beanB。beanB 在此方法处也会解析自己的依赖，当它检测到 beanA 这个依赖，于是调用 BeanFactry.getBean("beanA") 这个方法，从容器中获取 beanA。

4、获取早期引用

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            // ☆ 从缓存中获取早期引用
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    // ☆ 从 SingletonFactory 中获取早期引用
                    singletonObject = singletonFactory.getObject();

                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```

接着上面的步骤讲，populateBean 调用 BeanFactry.getBean("beanA") 以获取 beanB 的依赖。getBean("beanA") 会先调用 getSingleton("beanA")，尝试从缓存中获取 beanA。此时由于 beanA 还没完全实例化好，于是 this.singletonObjects.get("beanA") 返回 null。接着 this.earlySingletonObjects.get("beanA") 也返回空，因为 beanA 早期引用还没放入到这个缓存中。最后调用 singletonFactory.getObject() 返回 singletonObject，此时 singletonObject != null。singletonObject 指向 BeanA@1234，也就是 createBeanInstance 创建的原始对象。此时 beanB 获取到了这个原始对象的引用，beanB 就能顺利完成实例化。beanB 完成实例化后，beanA 就能获取到 beanB 所指向的实例，beanA 随之也完成了实例化工作。由于 beanB.beanA 和 beanA 指向的是同一个对象 BeanA@1234，所以 beanB 中的 beanA 此时也处于可用状态了。可以看下方的流程图

![beanAbeanB]({{ "/assets/img/spring/beanAbeanB.png" | relative_url}})

但是也有无法解决的情况，网上一般给出三种

1、构造器参数循环依赖

表示通过构造器注入构成的循环依赖，此依赖是无法解决的，只能抛出BeanCurrentlyIn CreationException异常表示循环依赖。

下面三个类A-B-C-A这样的模式

```java
public class StudentA {

 private StudentB studentB ;

 public void setStudentB(StudentB studentB) {
 this.studentB = studentB;
 }

 public StudentA() {
 }

 public StudentA(StudentB studentB) {
 this.studentB = studentB;
 }
}
```

```java
public class StudentB {

 private StudentC studentC ;

 public void setStudentC(StudentC studentC) {
 this.studentC = studentC;
 }

 public StudentB() {
 }

 public StudentB(StudentC studentC) {
 this.studentC = studentC;
 }
}
```

```java
public class StudentC {

 private StudentA studentA ;

 public void setStudentA(StudentA studentA) {
 this.studentA = studentA;
 }

 public StudentC() {
 }

 public StudentC(StudentA studentA) {
 this.studentA = studentA;
 }
}
```
上面是很基本的3个类，，StudentA有参构造是StudentB。StudentB的有参构造是StudentC，StudentC的有参构造是StudentA ，这样就产生了一个循环依赖的情况，我们都把这三个Bean交给Spring管理，并用有参构造实例化，注意之前讲的循环依赖的解决方法是用的property

```java
<bean id="a" class="com.zfx.student.StudentA">
 <constructor-arg index="0" ref="b"></constructor-arg>
</bean>
<bean id="b" class="com.zfx.student.StudentB">
 <constructor-arg index="0" ref="c"></constructor-arg>
</bean>
<bean id="c" class="com.zfx.student.StudentC">
 <constructor-arg index="0" ref="a"></constructor-arg>
</bean>
```
测试一下

```java
public class Test {
 public static void main(String[] args) {
 ApplicationContext context = new ClassPathXmlApplicationContext("com/zfx/student/applicationContext.xml");
 System.out.println(context.getBean("a", StudentA.class));
 }
}
```

执行结果报错信息为：

```java
Caused by: org.springframework.beans.factory.BeanCurrentlyInCreationException:  
    Error creating bean with name 'a': Requested bean is currently in creation: Is there an unresolvable circular reference?
```

2、setter方式单例，默认的方式（也就是上面分析的方式）

```java
<!--scope="singleton"(默认就是单例方式) -->
<bean id="a" class="com.zfx.student.StudentA" scope="singleton">
 <property name="studentB" ref="b"></property>
</bean>
<bean id="b" class="com.zfx.student.StudentB" scope="singleton">
 <property name="studentC" ref="c"></property>
</bean>
<bean id="c" class="com.zfx.student.StudentC" scope="singleton">
 <property name="studentA" ref="a"></property>
</bean>
```

测试是没有毛病的

3、setter方式原型，prototype

对于"prototype"作用域bean，Spring容器无法完成依赖注入，**因为Spring容器不进行缓存"prototype"作用域的bean**，因此无法提前暴露一个创建中的bean。

```java
<!--scope="prototype" -->
<bean id="a" class="com.zfx.student.StudentA" scope="prototype">
 <property name="studentB" ref="b"></property>
</bean>
<bean id="b" class="com.zfx.student.StudentB" scope="prototype">
 <property name="studentC" ref="c"></property>
</bean>
<bean id="c" class="com.zfx.student.StudentC" scope="prototype">
 <property name="studentA" ref="a"></property>
</bean>
```
scope="prototype" 意思是 每次请求都会创建一个实例对象。两者的区别是：有状态的bean都使用Prototype作用域，无状态的一般都使用singleton单例作用域。


### Spring的bean加载过程

Spring 作为 Ioc 框架，实现了依赖注入，由一个中心化的 Bean 工厂来负责各个 Bean 的实例化和依赖管理。各个 Bean 可以不需要关心各自的复杂的创建过程，达到了很好的解耦效果。

我们对 Spring 的工作流进行一个粗略的概括，主要为两大环节：
  - 解析，读 xml 配置，扫描类文件，从配置或者注解中获取 Bean 的定义信息，注册一些扩展功能。
  - 加载，通过解析完的定义信息获取 Bean 实例。

![loadbean]({{ "/assets/img/spring/loadbean.png" | relative_url}})

我们假设所有的配置和扩展类都已经装载到了 ApplicationContext 中，然后具体的分析一下 Bean 的加载流程。思考一个问题，抛开 Spring 框架的实现，假设我们手头上已经有一套完整的 BeanDefinition Map，然后指定一个 beanName 要进行实例化，需要关心什么？即使我们没有 Spring 框架，也需要了解这两方面的知识：

1、作用域。单例作用域或者原型作用域，单例的话需要全局实例化一次，原型每次创建都需要重新实例化。
2、依赖关系。一个 Bean 如果有依赖，我们需要初始化依赖，然后进行关联。如果多个 Bean 之间存在着循环依赖，A 依赖 B，B 依赖 C，C 又依赖 A，需要解这种循环依赖问题。

Spring 进行了抽象和封装，使得作用域和依赖关系的配置对开发者透明，我们只需要知道当初在配置里已经明确指定了它的生命周期和依赖了谁，至于是怎么实现的，依赖如何注入，托付给了 Spring 工厂来管理。

Spring 只暴露了很简单的接口给调用者，比如 getBean

```java
ApplicationContext context = new ClassPathXmlApplicationContext("hello.xml");
HelloBean helloBean = (HelloBean) context.getBean("hello");
helloBean.sayHello();
```
那我们就从 getBean 方法作为入口，去理解 Spring 加载的流程是怎样的，以及内部对创建信息、作用域、依赖关系等等的处理细节。

![loadbeantotal]({{ "/assets/img/spring/loadbeantotal.png" | relative_url}})

上面是跟踪了 getBean 的调用链创建的流程图，为了能够很好地理解 Bean 加载流程，省略一些异常、日志和分支处理和一些特殊条件的判断。

从上面的流程图中，可以看到一个 Bean 加载会经历这么几个阶段（用绿色标记）：

1、获取 BeanName，对传入的 name 进行解析，转化为可以从 Map 中获取到 BeanDefinition 的 bean name。
2、合并 Bean 定义，对父类的定义进行合并和覆盖，如果父类还有父类，会进行递归合并，以获取完整的 Bean 定义信息。
3、实例化，使用构造或者工厂方法创建 Bean 实例。
4、属性填充，寻找并且注入依赖，依赖的 Bean 还会递归调用 getBean 方法获取。
5、初始化，调用自定义的初始化方法。
6、获取最终的 Bean，如果是 FactoryBean 需要调用 getObject 方法，如果需要类型转换调用 TypeConverter 进行转化。

细节一点去分析，

1、转化BeanName

而在我们解析完配置后创建的 Map，使用的是 beanName 作为 key。见 DefaultListableBeanFactory：

```java
/** Map of bean definition objects, keyed by bean name */
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>(256);
```
BeanFactory.getBean 中传入的 name，有可能是这几种情况：
a.bean name，可以直接获取到定义 BeanDefinition。
b.alias name，别名，需要转化。
c.factorybean name, 带 & 前缀，通过它获取 BeanDefinition 的时候需要去除 & 前缀。

为了能够获取到正确的 BeanDefinition，需要先对 name 做一个转换，得到 beanName。

见`AbstractBeanFactory.doGetBean`
```java
protected <T> T doGetBean ... {
    ...

    // 转化工作
    final String beanName = transformedBeanName(name);
    ...
}
```
如果是 alias name，在解析阶段，alias name 和 bean name 的映射关系被注册到 SimpleAliasRegistry 中。从该注册器中取到 beanName。见 SimpleAliasRegistry.canonicalName：

```java
public String canonicalName(String name) {
    ...
    resolvedName = this.aliasMap.get(canonicalName);
    ...
}
```
如果是 factorybean name，表示这是个工厂 bean，有携带前缀修饰符 & 的，直接把前缀去掉。见`BeanFactoryUtils.transformedBeanName`

```java
public static String transformedBeanName(String name) {
    Assert.notNull(name, "'name' must not be null");
    String beanName = name;
    while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
        beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
    }
    return beanName;
}
```

2、合并RootBeanDefinition

我们从配置文件读取到的 BeanDefinition 是 GenericBeanDefinition。它的记录了一些当前类声明的属性或构造参数，但是对于父类只用了一个 parentName 来记录。

```java
public class GenericBeanDefinition extends AbstractBeanDefinition {
    ...
    private String parentName;
    ...
}
```

接下来会发现一个问题，在后续实例化 Bean 的时候，使用的 BeanDefinition 是 RootBeanDefinition 类型而非 GenericBeanDefinition。这是为什么？答案很明显，GenericBeanDefinition 在有继承关系的情况下，定义的信息不足：
a.如果不存在继承关系，GenericBeanDefinition 存储的信息是完整的，可以直接转化为 RootBeanDefinition。
b.如果存在继承关系，GenericBeanDefinition 存储的是 增量信息 而不是 全量信息。

为了能够正确初始化对象，需要完整的信息才行。需要递归合并父类的定义：

见`AbstractBeanFactory.doGetBean`

```java
protected <T> T doGetBean ... {
    ...

    // 合并父类定义
    final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);

    ...

    // 使用合并后的定义进行实例化
    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);

    ...
}
```

在判断`parentName`存在的情况下，说明存在父类定义，启动合并。如果父类还有父类怎么办？递归调用，继续合并。

```java
protected RootBeanDefinition getMergedBeanDefinition(
            String beanName, BeanDefinition bd, BeanDefinition containingBd)
            throws BeanDefinitionStoreException {

        ...

        String parentBeanName = transformedBeanName(bd.getParentName());

        ...

        // 递归调用，继续合并父类定义
        pbd = getMergedBeanDefinition(parentBeanName);

        ...

        // 使用合并后的完整定义，创建 RootBeanDefinition
        mbd = new RootBeanDefinition(pbd);

        // 使用当前定义，对 RootBeanDefinition 进行覆盖
        mbd.overrideFrom(bd);

        ...
        return mbd;

    }
```

每次合并完父类定义后，都会调用 RootBeanDefinition.overrideFrom 对父类的定义进行覆盖，获取到当前类能够正确实例化的 全量信息。

3、准备开始解决循环依赖

Spring 不支持原型模式的任何循环依赖。使用了一个 ThreadLocal 变量 prototypesCurrentlyInCreation 来记录当前线程正在创建中的 Bean 对象，见 AbtractBeanFactory#prototypesCurrentlyInCreation

```java
/** Names of beans that are currently in creation */
private final ThreadLocal<Object> prototypesCurrentlyInCreation =
            new NamedThreadLocal<Object>("Prototype beans currently in creation");
```

在 Bean 创建前进行记录，在 Bean 创建后删除记录。见`AbstractBeanFactory.doGetBean`

```java
if (mbd.isPrototype()) {
    // It's a prototype -> create a new instance.
    Object prototypeInstance = null;
    try {

        // 添加记录
        beforePrototypeCreation(beanName);
        prototypeInstance = createBean(beanName, mbd, args);
    }
    finally {
        // 删除记录
        afterPrototypeCreation(beanName);
    }
    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
}  
```
看下添加和删除记录

```java
protected void beforePrototypeCreation(String beanName) {
        Object curVal = this.prototypesCurrentlyInCreation.get();
        if (curVal == null) {
            this.prototypesCurrentlyInCreation.set(beanName);
        }
        else if (curVal instanceof String) {
            Set<String> beanNameSet = new HashSet<String>(2);
            beanNameSet.add((String) curVal);
            beanNameSet.add(beanName);
            this.prototypesCurrentlyInCreation.set(beanNameSet);
        }
        else {
            Set<String> beanNameSet = (Set<String>) curVal;
            beanNameSet.add(beanName);
        }
    }
```

```java
protected void afterPrototypeCreation(String beanName) {
        Object curVal = this.prototypesCurrentlyInCreation.get();
        if (curVal instanceof String) {
            this.prototypesCurrentlyInCreation.remove();
        }
        else if (curVal instanceof Set) {
            Set<String> beanNameSet = (Set<String>) curVal;
            beanNameSet.remove(beanName);
            if (beanNameSet.isEmpty()) {
                this.prototypesCurrentlyInCreation.remove();
            }
        }
    }
```
为了节省内存空间，在单个元素时 prototypesCurrentlyInCreation 只记录 String 对象，在多个依赖元素后改用 Set 集合。这里是 Spring 使用的一个节约内存的小技巧。了解了记录的写入和删除过程好了，再来看看读取以及判断循环的方式。这里要分两种情况讨论。

a.构造函数循环依赖
b.设置循环依赖

这两个地方的实现略有不同。

如果是构造函数依赖的，比如 A 的构造函数依赖了 B，会有这样的情况。实例化 A 的阶段中，匹配到要使用的构造函数，发现构造函数有参数 B，会使用 BeanDefinitionValueResolver 来检索 B 的实例。见 BeanDefinitionValueResolver.resolveReference：

```java
private Object resolveReference(Object argName, RuntimeBeanReference ref) {

    ...
    Object bean = this.beanFactory.getBean(refName);
    ...
}
```
我们发现这里继续调用 beanFactory.getBean 去加载 B。如果是设值循环依赖的的，比如我们这里不提供构造函数，并且使用了 @Autowire 的方式注解依赖（还有其他方式不举例了）：

```java
public class A {
    @Autowired
    private B b;
    ...
}
```

加载过程中，找到无参数构造函数，不需要检索构造参数的引用，实例化成功。接着执行下去，进入到属性填充阶段 AbtractBeanFactory.populateBean ，在这里会进行 B 的依赖注入。为了能够获取到 B 的实例化后的引用，最终会通过检索类 DependencyDescriptor 中去把依赖读取出来，见 DependencyDescriptor.resolveCandidate ：

```java
public Object resolveCandidate(String beanName, Class<?> requiredType, BeanFactory beanFactory)
            throws BeansException {
    return beanFactory.getBean(beanName, requiredType);
}
```
发现 beanFactory.getBean 方法又被调用到了。
在这里，两种循环依赖达成了同一。无论是构造函数的循环依赖还是设置循环依赖，在需要注入依赖的对象时，会继续调用 beanFactory.getBean 去加载对象，形成一个递归操作。（所以上面解决循环依赖的文章中，是@Autowired的设置方式，所以在属性填充判定循环而不是构造器那样的判定。但最终，走向一致）

而每次调用 beanFactory.getBean 进行实例化前后，都使用了 prototypesCurrentlyInCreation 这个变量做记录。按照这里的思路走，整体效果等同于 建立依赖对象的构造链。调用判定的地方在 AbstractBeanFactory.doGetBean 中，所有对象的实例化均会从这里启动。

```java
// Fail if we're already creating this bean instance:
// We're assumably within a circular reference.
if (isPrototypeCurrentlyInCreation(beanName)) {
    throw new BeanCurrentlyInCreationException(beanName);
}
```
判定的实现方法为 AbstractBeanFactory.isPrototypeCurrentlyInCreation ：

```java
protected boolean isPrototypeCurrentlyInCreation(String beanName) {
    Object curVal = this.prototypesCurrentlyInCreation.get();
    return (curVal != null &&
        (curVal.equals(beanName) || (curVal instanceof Set && ((Set<?>) curVal).contains(beanName))));
}
```

所以在原型模式下，构造函数循环依赖和设值循环依赖，本质上使用同一种方式检测出来。Spring 无法解决，直接抛出 BeanCurrentlyInCreationException 异常。

为了能够实现单例的提前暴露。Spring 使用了三级缓存，见 DefaultSingletonBeanRegistry，就是上面解决循环依赖的三个Map

3、创建实例

获取到完整的 RootBeanDefintion 后，就可以拿这份定义信息来实例具体的 Bean。具体实例创建见 AbstractAutowireCapableBeanFactory.createBeanInstance ，返回 Bean 的包装类 BeanWrapper，一共有三种策略：

a.使用工厂方法创建，instantiateUsingFactoryMethod。
b.使用有参构造函数创建，autowireConstructor。
c.使用无参构造函数创建，instantiateBean。

使用工厂方法创建，会先使用 getBean 获取工厂类，然后通过参数找到匹配的工厂方法，调用实例化方法实现实例化，具体见ConstructorResolver.instantiateUsingFactoryMethod ：

```java
public BeanWrapper instantiateUsingFactoryMethod ... (
    ...
    String factoryBeanName = mbd.getFactoryBeanName();
    ...
    factoryBean = this.beanFactory.getBean(factoryBeanName);
    ...
    // 匹配正确的工厂方法
    ...
    beanInstance = this.beanFactory.getInstantiationStrategy().instantiate(...);
    ...
    bw.setBeanInstance(beanInstance);
    return bw;
}
```

使用有参构造函数创建，整个过程比较复杂，涉及到参数和构造器的匹配。为了找到匹配的构造器，Spring 花了大量的工作，见 ConstructorResolver.autowireConstructor ：

```java
public BeanWrapper autowireConstructor ... {
    ...
    Constructor<?> constructorToUse = null;
    ...
    // 匹配构造函数的过程
    ...
    beanInstance = this.beanFactory.getInstantiationStrategy().instantiate(...);
    ...
    bw.setBeanInstance(beanInstance);
    return bw;
}   
```

使用无参构造函数创建是最简单的方式，见 AbstractAutowireCapableBeanFactory.instantiateBean:

```java
protected BeanWrapper instantiateBean ... {
    ...
    beanInstance = getInstantiationStrategy().instantiate(...);
    ...
    BeanWrapper bw = new BeanWrapperImpl(beanInstance);
    initBeanWrapper(bw);
    return bw;
    ...
}
```
我们发现这三个实例化方式，最后都会走 getInstantiationStrategy().instantiate(...)，见实现类 SimpleInstantiationStrategy.instantiate：

```java
public Object instantiate ... {
    if (bd.getMethodOverrides().isEmpty()) {
        ...
        return BeanUtils.instantiateClass(constructorToUse);
    }
    else {
        // Must generate CGLIB subclass.
        return instantiateWithMethodInjection(bd, beanName, owner);
    }
}
```
虽然拿到了构造函数，并没有立即实例化。因为用户使用了 replace 和 lookup 的配置方法，用到了动态代理加入对应的逻辑。如果没有的话，直接使用反射来创建实例。

创建实例后，就可以开始注入属性和初始化等操作。

但这里的 Bean 还不是最终的 Bean。返回给调用方使用时，如果是 FactoryBean 的话需要使用 getObject 方法来创建实例。(看下面FactoryBean和BeanFactory的区别)见 AbstractBeanFactory.getObjectFromBeanInstance ，会执行到 doGetObjectFromFactoryBean ：

```java
private Object doGetObjectFromFactoryBean ... {
    ...
    object = factory.getObject();
    ...
    return object;
}
```

4、注入属性

实例创建完后开始进行属性的注入，如果涉及到外部依赖的实例，会自动检索并关联到该当前实例。Ioc 思想体现出来了。正是有了这一步操作，Spring 降低了各个类之间的耦合。

属性填充的入口方法在AbstractAutowireCapableBeanFactory.populateBean。

```java
protected void populateBean ... {
    PropertyValues pvs = mbd.getPropertyValues();

    ...
    // InstantiationAwareBeanPostProcessor 前处理
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
        if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                continueWithPropertyPopulation = false;
                break;
            }
        }
    }
    ...

    // 根据名称注入
    if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
        autowireByName(beanName, mbd, bw, newPvs);
    }

    // 根据类型注入
    if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
        autowireByType(beanName, mbd, bw, newPvs);
    }

    ...
    // InstantiationAwareBeanPostProcessor 后处理
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
        if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
            if (pvs == null) {
                return;
            }
        }
    }

    ...

    // 应用属性值
    applyPropertyValues(beanName, mbd, bw, pvs);
}
```

可以看到主要的处理环节有：
a.应用 InstantiationAwareBeanPostProcessor 处理器，在属性注入前后进行处理。假设我们使用了 @Autowire 注解，这里会调用到 AutowiredAnnotationBeanPostProcessor 来对依赖的实例进行检索和注入的，它是 InstantiationAwareBeanPostProcessor 的子类。
b.根据名称或者类型进行自动注入，存储结果到 PropertyValues 中。
c.应用 PropertyValues，填充到 BeanWrapper。这里在检索依赖实例的引用的时候，会递归调用 BeanFactory.getBean 来获得。

5、初始化

触发Aware

如果我们的 Bean 需要容器的一些资源该怎么办？比如需要获取到 BeanFactory、ApplicationContext 等等。Spring 提供了 Aware 系列接口来解决这个问题。比如有这样的 Aware：

a.BeanFactoryAware，用来获取 BeanFactory。
b.ApplicationContextAware，用来获取 ApplicationContext。
c.ResourceLoaderAware，用来获取 ResourceLoader
d.ServletContextAware，用来获取 ServletContext。

Spring 在初始化阶段，如果判断 Bean 实现了这几个接口之一，就会往 Bean 中注入它关心的资源。见 `AbstractAutowireCapableBeanFactory.invokeAwareMethos`

```java
private void invokeAwareMethods(final String beanName, final Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware) {
            ((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
        }
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
}
```

触发 BeanPostProcessor

在 Bean 的初始化前或者初始化后，我们如果需要进行一些增强操作怎么办？这些增强操作比如打日志、做校验、属性修改、耗时检测等等。Spring 框架提供了 BeanPostProcessor 来达成这个目标。比如我们使用注解 @Autowire 来声明依赖，就是使用 AutowiredAnnotationBeanPostProcessor 来实现依赖的查询和注入的。接口定义如下：

```java
public interface BeanPostProcessor {

    // 初始化前调用
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

    // 初始化后调用
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;

}
```
实现该接口的 Bean 都会被 Spring 注册到 beanPostProcessors 中，见 AbstractBeanFactory :

```java
/** BeanPostProcessors to apply in createBean */
private final List<BeanPostProcessor> beanPostProcessors = new ArrayList<BeanPostProcessor>();
```
只要 Bean 实现了 BeanPostProcessor 接口，加载的时候会被 Spring 自动识别这些 Bean，自动注册，非常方便。然后在 Bean 实例化前后，Spring 会去调用我们已经注册的 beanPostProcessors 把处理器都执行一遍。
```java
public abstract class AbstractAutowireCapableBeanFactory ... {

    ...

    @Override
    public Object applyBeanPostProcessorsBeforeInitialization ... {

        Object result = existingBean;
        for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
            result = beanProcessor.postProcessBeforeInitialization(result, beanName);
            if (result == null) {
                return result;
            }
        }
        return result;
    }

    @Override
    public Object applyBeanPostProcessorsAfterInitialization ... {

        Object result = existingBean;
        for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
            result = beanProcessor.postProcessAfterInitialization(result, beanName);
            if (result == null) {
                return result;
            }
        }
        return result;
    }

    ...
}
```
这里使用了责任链模式，Bean 会在处理器链中进行传递和处理。当我们调用 BeanFactory.getBean 的后，执行到 Bean 的初始化方法 AbstractAutowireCapableBeanFactory.initializeBean 会启动这些处理器。

```java
protected Object initializeBean ... {   
    ...
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    ...
    // 触发自定义 init 方法
    invokeInitMethods(beanName, wrappedBean, mbd);
    ...
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    ...
}
```

触发自定义 init

自定义初始化有两种方式可以选择：实现 InitializingBean。提供了一个很好的机会，在属性设置完成后再加入自己的初始化逻辑。定义 init 方法。自定义的初始化逻辑。见 AbstractAutowireCapableBeanFactory.invokeInitMethods ：

```java
protected void invokeInitMethods ... {

        boolean isInitializingBean = (bean instanceof InitializingBean);
        if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
            ...

            ((InitializingBean) bean).afterPropertiesSet();
            ...
        }

        if (mbd != null) {
            String initMethodName = mbd.getInitMethodName();
            if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
                    !mbd.isExternallyManagedInitMethod(initMethodName)) {
                invokeCustomInitMethod(beanName, bean, mbd);
            }
        }
    }
```

6、类型转换

Bean 已经加载完毕，属性也填充好了，初始化也完成了。在返回给调用者之前，还留有一个机会对 Bean 实例进行类型的转换。见AbstractBeanFactory.doGetBean ：

```java
protected <T> T doGetBean ... {
    ...
    if (requiredType != null && bean != null && !requiredType.isInstance(bean)) {
        ...
        return getTypeConverter().convertIfNecessary(bean, requiredType);
        ...
    }
    return (T) bean;
}
```

综上所述，抛开一些细节处理和扩展功能，一个 Bean 的创建过程无非是：获取完整定义 -> 实例化 -> 依赖注入 -> 初始化 -> 类型转换。作为一个完善的框架，Spring 需要考虑到各种可能性，还需要考虑到接入的扩展性。所以有了复杂的循环依赖的解决，复杂的有参数构造器的匹配过程，有了 BeanPostProcessor 来对实例化或初始化的 Bean 进行扩展修改。

### Spring中BeanFactory和FactoryBean的区别

共同点应该只有都是接口了吧

区别：
1、BeanFactory以Factory结尾，表示它是一个工厂类，用于管理Bean的一个工厂，在Spring中，所有的Bean都是由BeanFactory(也就是IOC容器)来进行管理的。
2、但对FactoryBean而言，这个Bean不是简单的Bean，而是一个能生产或者修饰对象生成的工厂Bean,它的实现与设计模式中的工厂模式和修饰器模式类似。

BeanFactory定义了IOC容器的最基本形式，并提供了IOC容器应遵守的的最基本的接口，也就是Spring IOC所遵守的最底层和最基本的编程规范。它的职责包括：实例化、定位、配置应用程序中的对象及建立这些对象间的依赖。在Spring代码中，BeanFactory只是个接口，并不是IOC容器的具体实现，但是Spring容器给出了很多种实现，如 DefaultListableBeanFactory、XmlBeanFactory、ApplicationContext等，都是附加了某种功能的实现。

```java
package org.springframework.beans.factory;

import org.springframework.beans.BeansException;
import org.springframework.core.ResolvableType;

public interface BeanFactory {
    String FACTORY_BEAN_PREFIX = "&";

    Object getBean(String var1) throws BeansException;

    <T> T getBean(String var1, Class<T> var2) throws BeansException;

    <T> T getBean(Class<T> var1) throws BeansException;

    Object getBean(String var1, Object... var2) throws BeansException;

    <T> T getBean(Class<T> var1, Object... var2) throws BeansException;

    boolean containsBean(String var1);

    boolean isSingleton(String var1) throws NoSuchBeanDefinitionException;

    boolean isPrototype(String var1) throws NoSuchBeanDefinitionException;

    boolean isTypeMatch(String var1, ResolvableType var2) throws NoSuchBeanDefinitionException;

    boolean isTypeMatch(String var1, Class<?> var2) throws NoSuchBeanDefinitionException;

    Class<?> getType(String var1) throws NoSuchBeanDefinitionException;

    String[] getAliases(String var1);
}
```

而对于FactoryBean，一般情况下，Spring通过反射机制利用<bean>的class属性指定实现类实例化Bean，在某些情况下，实例化Bean过程比较复杂，如果按照传统的方式，则需要在<bean>中提供大量的配置信息。配置方式的灵活性是受限的，这时采用编码的方式可能会得到一个简单的方案。Spring为此提供了一个org.springframework.bean.factory.FactoryBean的工厂类接口，用户可以通过实现该接口定制实例化Bean的逻辑。FactoryBean接口对于Spring框架来说占用重要的地位，Spring自身就提供了70多个FactoryBean的实现。它们隐藏了实例化一些复杂Bean的细节，给上层应用带来了便利。从Spring3.0开始，FactoryBean开始支持泛型，即接口声明改为FactoryBean<T>的形式

```java
package org.springframework.beans.factory;

public interface FactoryBean<T> {
    T getObject() throws Exception;

    Class<?> getObjectType();

    boolean isSingleton();
}
```
该接口定义了3个方法

T getObject()：返回由FactoryBean创建的Bean实例，如果isSingleton()返回true，则该实例会放到Spring容器中单实例缓存池中；
boolean isSingleton()：返回由FactoryBean创建的Bean实例的作用域是singleton还是prototype；
Class<T> getObjectType()：返回FactoryBean创建的Bean类型。当配置文件中<bean>的class属性配置的实现类是FactoryBean时，通过getBean()方法返回的不是FactoryBean本身，而是FactoryBean#getObject()方法所返回的对象，相当于FactoryBean#getObject()代理了getBean()方法。

例：如果使用传统方式配置下面Car的<bean>时，Car的每个属性分别对应一个<property>元素标签。

```java
public   class  Car  {  
    private   int maxSpeed ;  
    private  String brand ;  
    private   double price ;  
    public   int  getMaxSpeed ()   {  
        return   this . maxSpeed ;  
    }  
    public   void  setMaxSpeed ( int  maxSpeed )   {  
        this . maxSpeed  = maxSpeed;  
    }  
    public  String getBrand ()   {  
        return   this . brand ;  
    }  
    public   void  setBrand ( String brand )   {  
        this . brand  = brand;  
    }  
    public   double  getPrice ()   {  
        return   this . price ;  
    }  
    public   void  setPrice ( double  price )   {  
        this . price  = price;  
   }  
}
```
如果用FactoryBean的方式实现就灵活点，下例通过逗号分割符的方式一次性的为Car的所有属性指定配置值：

```java
import  org.springframework.beans.factory.FactoryBean;  
public   class  CarFactoryBean  implements  FactoryBean<Car>  {  
    private  String carInfo ;  
    public  Car getObject ()   throws  Exception  {  
        Car car =  new  Car () ;  
        String []  infos =  carInfo .split ( "," ) ;  
        car.setBrand ( infos [ 0 ]) ;  
        car.setMaxSpeed ( Integer. valueOf ( infos [ 1 ])) ;  
        car.setPrice ( Double. valueOf ( infos [ 2 ])) ;  
        return  car;  
    }  
    public  Class<Car> getObjectType ()   {  
        return  Car. class ;  
    }  
    public   boolean  isSingleton ()   {  
        return   false ;  
    }  
    public  String getCarInfo ()   {  
        return   this . carInfo ;  
    }  

    // 接受逗号分割符设置属性信息  
    public   void  setCarInfo ( String carInfo )   {  
        this . carInfo  = carInfo;  
    }  
}
```
有了这个CarFactoryBean后，就可以在配置文件中使用下面这种自定义的配置方式配置CarBean了：

```java
<bean id="car" class="com.baobaotao.factorybean.CarFactoryBean" P:carInfo="法拉利,400,2000000"/>
```
当调用getBean("car")时，Spring通过反射机制发现CarFactoryBean实现了FactoryBean的接口，这时Spring容器就调用接口方法CarFactoryBean#getObject()方法返回。

如果希望获取CarFactoryBean的实例，则需要在使用getBean(beanName)方法时在beanName前显示的加上"&"前缀：如getBean("&car");

Spring一共有两种bean，一个是工厂bean，一个就是普通bean

### Spring的装配方式

Spring有三种装配方式

创建应用对象之间协作关系的行为通常称为装配（wiring）,这也是依赖注入（Dependence Injection）的本质

1、自动化装配bean

Spring从两个角度来实现自动化装配：

组件扫描（component scanning）：Spring会自动发现应用上下文中所创建的bean
自动装配（autowiring)：Spring自动满足bean之间的依赖

a.创建可被发现的bean，并启用组件扫描

```java
package soundsystem

public interface CompactDisc{
    viod play();
}
```

```java
package soundsystem;

public interface MediaPlayer {
    void play();
}
```
下面是CompactDisc的一个实现类，@Component注解表明该类会作为组件类，并告知Spring要为这个类创建bean。
```java
package soundsystem;

import org.springframework.stereotype.Component;

@Component
public class SgtPeppers implements CompactDisc {

    private String title = "Sgt.Pepper Lonely Hearts Club Band";
    private String artist = "The beatles";

    public void play() {
        System.out.println("Playing "+ title + " by " + artist);
    }

}

```
可被发现的bean已创建好，**但这个bean只是可被发现，要想真正地被发现还需还需启用组件扫描（组件扫描默认是不开启的）**。方法有二：

方法一：创建配置类，通过@ComponentScan启用组件扫描。

```java
package soundsystem;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan(basePackages="soundsystem")  
public class CDPlayerConfig {

}
```
方法二：通过XML文件的配置启用组件扫描。

```java
<context:component-scan base-package="soundsystem"></context:component-scan>
```

b.通过为bean添加注解实现自动装配

在上面的工作中我们已经能够将bean自动添加到Spring容器中，接下来需实现自动装配来满足bean之间的依赖。注意这个@Autowired

```java
package soundsystem;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class CDPlayer implements MediaPlayer {

    private CompactDisc cd;

    @Autowired
    public CDPlayer(CompactDisc cd) {
        this.cd = cd;
    }

    public void play() {
        cd.play();
    }

}

```
2、通过JAVA代码装配Bean

创建配置类，@Configuration注解表明这个类是一个注解类，该类包含了在Spring应用上下文中如何创建bean的细节。与上面不同的是，这里我们移除了@ComponentScan注解，不在用组件扫描的方式加载bean,在这里我们更加关注显示配置

```java
package soundsystem;

import org.springframework.context.annotation.Configuration;

@Configuration
public class CDPlayerConfig2 {

}

```

声明bean
```java
@Bean
public CompactDisc sgtPeppers() {
    return new SgtPeppers();
}

```

借助JavaConfig实现注入（可以有多种方法，在这里只提供其中一种）

```java
@Bean
public CDPlayer cdPlayer(CompactDisc cd) {
    return new CDPlayer(cd);
}
```


3、通过XML装配Bean

声明bean
在基于XML的Spring配置中声明一个bean，我们要使用spring-beans模式中的一个元素：<bean>。我们可按照如下方式声明CompactDisc bean:

```java
<bean id="sgtPepper" class="soundsystem.SgtPepper" />
```
注入bean（方法有二）

其一：构造注入

我们已经声明了SgtPepper bean,并且SgtPepper类实现了CompactDiSC接口，现在我们要做的就是在XML中声明CDPlayer并通过ID引用SgtPepper：

```java
<bean id="cdPlayer" class="soundsystem.CDPlayer">
    <constructor-arg ref="sgtPepper" />
</bean>
```

```java
package soundsystem;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component("mediaPlayer")
public class CDPlayer implements MediaPlayer {

    private CompactDisc cd;

    @Autowired
    public CDPlayer(CompactDisc cd) {
        this.cd = cd;
    }

    public void play() {
        cd.play();
    }

}

```
其二：设值注入

```java
<bean id="cdPlayer" class="soundsystem.CDPlayer">
    <property name="compactDisc" ref="sgtPepper"></property>
</bean>
```
```java
package soundsystem;

import org.springframework.beans.factory.annotation.Autowired;

public class CDPlayer implements MediaPlayer {

    private CompactDisc cd;

    @Autowired
    public void setCd(CompactDisc cd) {
        this.cd = cd;
    }

    public void play() {
        cd.play();
    }

}
```

### Spring的AOP

AOP在SpringBox中也有提交，这边再说一下，总的来说。AOP是能够让我们在不影响原有功能的前提下，为软件横向扩展功能。不会代码侵入，需要知道的AOP概念

切面：拦截器类，其中会定义切点以及通知，也就是@Aspect注解
切点：具体拦截的某个业务点。也就是@Pointcut
通知：切面当中的方法，声明通知方法在目标业务层的执行位置，通知类型如下：
前置通知：@Before 在目标业务方法执行之前执行
后置通知：@After 在目标业务方法执行之后执行
返回通知：@AfterReturning 在目标业务方法返回结果之后执行
异常通知：@AfterThrowing 在目标业务方法抛出异常之后
环绕通知：@Around 功能强大，可代替以上四种通知，还可以控制目标业务方法是否执行以及何时执行

```java
package com.qiming.spring;

import org.aspectj.lang.annotation.AfterReturning;
import org.aspectj.lang.annotation.AfterThrowing;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;

@Aspect //这就是定义的一个切面
public class Audience {
  @Pointcut("execution(* com.qiming.spring.Performance.perform(..))") //定义切点(何处)
  public void performance() {};

  @Before("performance()")		//定义通知(何时，做什么)
  public void silenceCellPhones() {
    System.out.println("Silencing cell phones");
  }

  @Before("performance()")
  public void takeSeats() {
    System.out.println("Taking seats");
  }

  @AfterReturning("performance()")
  public void applause() {
    System.out.println("CLAP CLAP CLAP!!!");
  }

  @AfterThrowing("performance()")
  public void demandRefund() {
    System.out.println("Demanding a refund");
  }
}

```

AOP的底层实现是动态代理，两种代理方式都可以，


### Spring的生命周期管理

生命周期只有4个
1)实例化Instantiation
2)属性赋值Populate
3)初始化Initialization
4)销毁Destruction

是的，Spring Bean的生命周期只有这四个阶段。把这四个阶段和每个阶段对应的扩展点糅合在一起虽然没有问题，但是这样非常凌乱，难以记忆。要彻底搞清楚Spring的生命周期，首先要把这四个阶段牢牢记住。实例化和属性赋值对应构造方法和setter方法的注入，初始化和销毁是用户能自定义扩展的两个阶段。

主要逻辑都在doCreate()方法中，逻辑很清晰，就是顺序调用以下三个方法，这三个方法与三个生命周期阶段一一对应，非常重要，在后续扩展接口分析中也会涉及。

```java
// 忽略了无关代码
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
      throws BeanCreationException {

   // Instantiate the bean.
   BeanWrapper instanceWrapper = null;
   if (instanceWrapper == null) {
       // 实例化阶段！
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }

   // Initialize the bean instance.
   Object exposedObject = bean;
   try {
       // 属性赋值阶段！
      populateBean(beanName, mbd, instanceWrapper);
       // 初始化阶段！
      exposedObject = initializeBean(beanName, exposedObject, mbd);
   }
   }
```

至于销毁，是在容器关闭时调用的，详见ConfigurableApplicationContext#close()

常用的扩展点

Spring生命周期相关的常用扩展点非常多，所以问题不是不知道，而是记不住或者记不牢。其实记不住的根本原因还是不够了解，这里通过源码+分类的方式帮大家记忆。

第一大类：影响多个Bean的接口

实现了这些接口的Bean会切入到多个Bean的生命周期中。正因为如此，这些接口的功能非常强大，Spring内部扩展也经常使用这些接口，例如自动注入以及AOP的实现都和他们有关。

```java
BeanPostProcessor
InstantiationAwareBeanPostProcessor
```

这两兄弟可能是Spring扩展中最重要的两个接口！InstantiationAwareBeanPostProcessor作用于实例化阶段的前后，BeanPostProcessor作用于初始化阶段的前后。正好和第一、第三个生命周期阶段对应。通过图能更好理解：

![springbeanlifecycle]({{ "/assets/img/spring/springbeanlifecycle.png" | relative_url}})

InstantiationAwareBeanPostProcessor实际上继承了BeanPostProcessor接口，严格意义上来看他们不是两兄弟，而是两父子。但是从生命周期角度我们重点关注其特有的对实例化阶段的影响，图中省略了从BeanPostProcessor继承的方法。

InstantiationAwareBeanPostProcessor源码分析：postProcessBeforeInstantiation调用点，忽略无关代码：

```java
@Override
    protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
            throws BeanCreationException {

        try {
            // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
            // postProcessBeforeInstantiation方法调用点，这里就不跟进了，
            // 有兴趣的同学可以自己看下，就是for循环调用所有的InstantiationAwareBeanPostProcessor
            Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
            if (bean != null) {
                return bean;
            }
        }

        try {   
            // 上文提到的doCreateBean方法，可以看到
            // postProcessBeforeInstantiation方法在创建Bean之前调用
            Object beanInstance = doCreateBean(beanName, mbdToUse, args);
            if (logger.isTraceEnabled()) {
                logger.trace("Finished creating instance of bean '" + beanName + "'");
            }
            return beanInstance;
        }

    }
```

可以看到，postProcessBeforeInstantiation在doCreateBean之前调用，也就是在bean实例化之前调用的，英文源码注释解释道该方法的返回值会替换原本的Bean作为代理，这也是Aop等功能实现的关键点。postProcessAfterInstantiation调用点，忽略无关代码：

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {

   // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
   // state of the bean before properties are set. This can be used, for example,
   // to support styles of field injection.
   boolean continueWithPropertyPopulation = true;
    // InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation()
    // 方法作为属性赋值的前置检查条件，在属性赋值之前执行，能够影响是否进行属性赋值！
   if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
               continueWithPropertyPopulation = false;
               break;
            }
         }
      }
   }

   // 忽略后续的属性赋值操作代码
}
```

可以看到该方法在属性赋值方法内，但是在真正执行赋值操作之前。其返回值为boolean，返回false时可以阻断属性赋值阶段（continueWithPropertyPopulation = false;）。

关于BeanPostProcessor执行阶段的源码穿插在下文Aware接口的调用时机分析中，因为部分Aware功能的就是通过他实现的!只需要先记住BeanPostProcessor在初始化前后调用就可以了。

第二大类：只调用一次的接口

这一大类接口的特点是功能丰富，常用于用户自定义扩展。第二大类中又可以分为两类：

1、Aware类型的接口
2、生命周期接口

无所不知的Aware
Aware类型的接口的作用就是让我们能够拿到Spring容器中的一些资源。基本都能够见名知意，Aware之前的名字就是可以拿到什么资源，例如BeanNameAware可以拿到BeanName，以此类推。调用时机需要注意：所有的Aware方法都是在初始化阶段之前调用的！
Aware接口众多，这里同样通过分类的方式帮助大家记忆。
Aware接口具体可以分为两组，至于为什么这么分，详见下面的源码分析。如下排列顺序同样也是Aware接口的执行顺序，能够见名知意的接口不再解释。

Aware Group1

1、BeanNameAware
2、BeanClassLoaderAware
3、BeanFactoryAware

Aware Group2

1、EnvironmentAware
2、EmbeddedValueResolverAware 这个知道的人可能不多，实现该接口能够获取Spring EL解析器，用户的自定义注解需要支持spel表达式的时候可以使用，非常方便。
3、ApplicationContextAware(ResourceLoaderAware\ApplicationEventPublisherAware\MessageSourceAware) 这几个接口可能让人有点懵，实际上这几个接口可以一起记，其返回值实质上都是当前的ApplicationContext对象，因为ApplicationContext是一个复合接口，如下：

```java
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
        MessageSource, ApplicationEventPublisher, ResourcePatternResolver {}
```

这里涉及到另一个问题，ApplicationContext和BeanFactory的区别，可以从ApplicationContext继承的这几个接口入手，除去BeanFactory相关的两个接口就是ApplicationContext独有的功能，这里不详细说明。？
Aware调用时机源码分析

详情如下，忽略了部分无关代码。代码位置就是我们上文提到的initializeBean方法详情，这也说明了Aware都是在初始化阶段之前调用的！

```java
// 见名知意，初始化阶段调用的方法
    protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {

        // 这里调用的是Group1中的三个Bean开头的Aware
        invokeAwareMethods(beanName, bean);

        Object wrappedBean = bean;

        // 这里调用的是Group2中的几个Aware，
        // 而实质上这里就是前面所说的BeanPostProcessor的调用点！
        // 也就是说与Group1中的Aware不同，这里是通过BeanPostProcessor（ApplicationContextAwareProcessor）实现的。
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
        // 下文即将介绍的InitializingBean调用点
        invokeInitMethods(beanName, wrappedBean, mbd);
        // BeanPostProcessor的另一个调用点
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);

        return wrappedBean;
    }
```
可以看到并不是所有的Aware接口都使用同样的方式调用。Bean××Aware都是在代码中直接调用的，而ApplicationContext相关的Aware都是通过BeanPostProcessor#postProcessBeforeInitialization()实现的。感兴趣的可以自己看一下ApplicationContextAwareProcessor这个类的源码，就是判断当前创建的Bean是否实现了相关的Aware方法，如果实现了会调用回调方法将资源传递给Bean。
至于Spring为什么这么实现，应该没什么特殊的考量。也许和Spring的版本升级有关。基于对修改关闭，对扩展开放的原则，Spring对一些新的Aware采用了扩展的方式添加。

BeanPostProcessor的调用时机也能在这里体现，包围住invokeInitMethods方法，也就说明了在初始化阶段的前后执行。

关于Aware接口的执行顺序，其实只需要记住第一组在第二组执行之前就行了。每组中各个Aware方法的调用顺序其实没有必要记，有需要的时候点进源码一看便知。

**简单的两个生命周期接口**

至于剩下的两个生命周期接口就很简单了，实例化和属性赋值都是Spring帮助我们做的，能够自己实现的有初始化和销毁两个生命周期阶段。
1、InitializingBean 对应生命周期的初始化阶段，在上面源码的invokeInitMethods(beanName, wrappedBean, mbd);方法中调用。有一点需要注意，因为Aware方法都是执行在初始化方法之前，所以可以在初始化方法中放心大胆的使用Aware接口获取的资源，这也是我们自定义扩展Spring的常用方式。
除了实现InitializingBean接口之外还能通过注解或者xml配置的方式指定初始化方法，至于这几种定义方式的调用顺序其实没有必要记。因为这几个方法对应的都是同一个生命周期，只是实现方式不同，我们一般只采用其中一种方式。

2、DisposableBean 类似于InitializingBean，对应生命周期的销毁阶段，以ConfigurableApplicationContext#close()方法作为入口，实现是通过循环取所有实现了DisposableBean接口的Bean然后调用其destroy()方法 。感兴趣的可以自行跟一下源码。

扩展阅读: BeanPostProcessor 注册时机与执行顺序

注册时机

我们知道BeanPostProcessor也会注册为Bean，那么Spring是如何保证BeanPostProcessor在我们的业务Bean之前初始化完成呢？
请看我们熟悉的refresh()方法的源码，省略部分无关代码：

```java
@Override
    public void refresh() throws BeansException, IllegalStateException {
        synchronized (this.startupShutdownMonitor) {

            try {
                // Allows post-processing of the bean factory in context subclasses.
                postProcessBeanFactory(beanFactory);

                // Invoke factory processors registered as beans in the context.
                invokeBeanFactoryPostProcessors(beanFactory);

                // Register bean processors that intercept bean creation.
                // 所有BeanPostProcesser初始化的调用点
                registerBeanPostProcessors(beanFactory);

                // Initialize message source for this context.
                initMessageSource();

                // Initialize event multicaster for this context.
                initApplicationEventMulticaster();

                // Initialize other special beans in specific context subclasses.
                onRefresh();

                // Check for listener beans and register them.
                registerListeners();

                // Instantiate all remaining (non-lazy-init) singletons.
                // 所有单例非懒加载Bean的调用点
                finishBeanFactoryInitialization(beanFactory);

                // Last step: publish corresponding event.
                finishRefresh();
            }

    }
```

可以看出，Spring是先执行registerBeanPostProcessors()进行BeanPostProcessors的注册，然后再执行finishBeanFactoryInitialization初始化我们的单例非懒加载的Bean。

执行顺序

BeanPostProcessor有很多个，而且每个BeanPostProcessor都影响多个Bean，其执行顺序至关重要，必须能够控制其执行顺序才行。关于执行顺序这里需要引入两个排序相关的接口：PriorityOrdered、Ordered

1、PriorityOrdered是一等公民，首先被执行，PriorityOrdered公民之间通过接口返回值排序
2、Ordered是二等公民，然后执行，Ordered公民之间通过接口返回值排序
3、都没有实现是三等公民，最后执行

在以下源码中，可以很清晰的看到Spring注册各种类型BeanPostProcessor的逻辑，根据实现不同排序接口进行分组。优先级高的先加入，优先级低的后加入。

```java
// First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
// 首先，加入实现了PriorityOrdered接口的BeanPostProcessors，顺便根据PriorityOrdered排了序
            String[] postProcessorNames =
                    beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            for (String ppName : postProcessorNames) {
                if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                }
            }
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            registryProcessors.addAll(currentRegistryProcessors);
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            currentRegistryProcessors.clear();

            // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
// 然后，加入实现了Ordered接口的BeanPostProcessors，顺便根据Ordered排了序
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            for (String ppName : postProcessorNames) {
                if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                }
            }
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            registryProcessors.addAll(currentRegistryProcessors);
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            currentRegistryProcessors.clear();

            // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
// 最后加入其他常规的BeanPostProcessors
            boolean reiterate = true;
            while (reiterate) {
                reiterate = false;
                postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
                for (String ppName : postProcessorNames) {
                    if (!processedBeans.contains(ppName)) {
                        currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                        processedBeans.add(ppName);
                        reiterate = true;
                    }
                }
                sortPostProcessors(currentRegistryProcessors, beanFactory);
                registryProcessors.addAll(currentRegistryProcessors);
                invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
                currentRegistryProcessors.clear();
            }
```
根据排序接口返回值排序，默认升序排序，返回值越低优先级越高。

```java
/**
 * Useful constant for the highest precedence value.
 * @see java.lang.Integer#MIN_VALUE
 */
int HIGHEST_PRECEDENCE = Integer.MIN_VALUE;

/**
 * Useful constant for the lowest precedence value.
 * @see java.lang.Integer#MAX_VALUE
 */
int LOWEST_PRECEDENCE = Integer.MAX_VALUE;
```
PriorityOrdered、Ordered接口作为Spring整个框架通用的排序接口，在Spring中应用广泛，也是非常重要的接口。


### Spring中的线程池

1、定时线程池中scheduleWithFixedDelay和scheduleAtFixedRate的区别

scheduleAtFixedRate

没有什么歧义，很容易理解，就是每隔多少时间，固定执行任务。

scheduleWithFixedDelay

貌似也是推迟一段时间执行任务，当结束前一个执行后延迟的时间，scheduleWithFixedDelay 比如当前一个任务结束的时刻，开始结算间隔时间，如0秒开始执行第一次任务，任务耗时5秒，任务间隔时间3秒，那么第二次任务执行的时间是在第8秒开始


### Spring中的BeanUtils

1、Spring中的BeanUtils使用要谨慎，看业务场景，具体规则如下

```html

Hotel1 h11 = Hotel1.buildAll();
Hotel1 h12 = Hotel1.buildWithNull();


// 同一个类中，可以递归的copy
BeanUtils.copyProperties(h11, h12);

Hotel2 h22 = Hotel2.buildWithNull();

// 不同的类中，没有共用相同的类，无法递归的copy
BeanUtils.copyProperties(h11, h22);

SimpleCopy1 s1 = SimpleCopy1.build();
SimpleCopy2 s2 = SimpleCopy2.build();

// 不同的类中，原生类型可以copy
BeanUtils.copyProperties(s1, s2);

SimpleCopy1 sd1 = SimpleCopy1.buildDeep();
SimpleCopy2 sd2 = SimpleCopy2.buildDeep();

// 不同的类中，共用了相同的类，可以递归copy
BeanUtils.copyProperties(sd1, sd2);
```


### 常见问题

1、exchange，post等的问题

no suitable HttpMessageConverter found for request type [org.springframework.util.LinkedMultiValueMap] and content type [application/x-www-form-urlencoded;charset=UTF-8]

报上面的错误的话，看看注入的restTemplate的配置，实在不行new个新的

```java
MultiValueMap<String, Object> postParameters = new LinkedMultiValueMap<>();
postParameters.add("grant_type", "password");
postParameters.add("scope", "write");
postParameters.add("username", username);
postParameters.add("password", password);
HttpHeaders requestHeaders = new HttpHeaders();
requestHeaders.add("Content-Type", "application/x-www-form-urlencoded");
RestTemplate template = new RestTemplate();
  ResponseEntity<AwxGetTokenResult> result =
      template.exchange(this.AWX_HOST_URL + this.AWX_TOKEN_URL, HttpMethod.POST,
          new HttpEntity<>(postParameters, requestHeaders), new ParameterizedTypeReference<AwxGetTokenResult>() {});
  return result.getBody();
```
