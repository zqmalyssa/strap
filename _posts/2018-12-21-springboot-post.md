---
layout: post
title: Springboot相关
tags: [code, java, springboot]
author-id: zqmalyssa
---

Springboot相关介绍及使用

### 概念介绍

Springboot简化了配置，内置了tomcat这样的web服务器，自然集成了Spring的内容，和spring cloud也有联系


### Springboot的自动配置

其实现原理就是：基于添加的jar依赖自动对Springboot应用程序进行配置

spring-boot-autoconfiguration这个jar包里面有自动配置的相关代码

开启自动配置需要添加注解@EnableAutoConfiguration，而一般我们不会显示的使用是因为工程上的@SpringBootApplication已经包含了@EnableAutoConfiguration注解，但是如果我不想自动配置某些类，那么就用exclude = Class<?>[]进行去除(排除写的Class名字其实就是下面说的factories文件里面key(属性)包含的名字)

@EnableAutoConfiguration干了什么事呢？

就是AutoConfigurationImportSelector，它去加载了META-INF/spring.factories文件里面key(属性)为org.springframework.boot.autoconfigure.EnableAutoConfiguration的，有一堆，data类的，jdbc的，web的等等

自动配置的实现原理其实是条件注解

@Conditional
@ConditionalOnClass
@ConditionalOnBean
@ConditionalOnMissingBean
@ConditionalOnProperty

比如说key对应的一个值为org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration，那其实在autoconfiguration的jar包中就能找到这个类文件

```java
@Configuration
@ConditionalOnClass({DataSource.class, EmbeddedDatabaseType.class})
@EnableConfigurationProperties({DataSourceProperties.class})
@Import({Registrar.class, DataSourcePoolMetadataProvidersConfiguration.class})
public class DataSourceAutoConfiguration {
}
```
当这个类的classpath上有DataSource类和EmbeddedDatabaseType类时，这个配置才会生效，当然还有一些无条件配置的

了解自动配置的情况，用`--debug`，可以看出自动配置的结果

这里要补充知道是可以**自己写一个自动配置**，除各种条件注解外，还有执行顺序@AutoConfigureBefore，@AutoConfigureAfter，@AutoConfigureOrder

1、编写java config
@Configuration

比方说我有个FantasyRunner的类，什么都没有，一个无参构造一个带参数构造，参数是一个名字，主要将它做成maven依赖

那么重点是自动配置的写法，首先pom里面依赖一下auto

```java
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-autoconfigure</artifactId>
</dependency>
```
自动配置类

```java
@Configuration
@ConditionalOnClass(FantasyRunner.class)  //只要在有上面提到的类时，这个自动配置才生效
public class FantasyAutoConfiguration {

  @Bean
  @ConditionalOnMissingBean(FantasyRunner.class)
  @ConditionalOnProperty(name = "fantasy.enabled", havingValue= "true", matchIfMissing= true) //如果配置了fantasy.enabled属性，值为true才生效，但是matchIfMissing说了，如果不配，默认就当它是true
  public FantasyRunner getFantasyRunner() {
    return new FantasyRunner();
  }

}
```

2、添加条件
@Conditional

看上面加的条件

3、定位自动配置
META-INF/spring.factories

在上面这个全路径限定名添加到spring.factories里面的key(属性中)

如果在程序中自己已经配了一个FantasyRunner的bean，那么上面的自动配置是不生效的，注意是条件，而且如果fantasy.enabled为false，也不会自动配置

Springboot还提供了一个错误分析机制FailureAnalysis

还有补充的是如果在Spring3.X的老系统中引入自动配置要怎么搞，因为conditional注解是4才出来的，且项目不会引入springboot，核心的解决思路是

通过BeanFactoryPostProcessor进行判断，编写java config类，引入配置类，通过component-scan或者通过XML文件import。来看下Spring的两个扩展点

a.BeanPostProcessor
  - 针对bean实例
  - 在bean创建后提供定制逻辑回调
b.BeanFactoryPostProcessor
  - 针对bean定义
  - 在容器创建bena前获取配置元数据
  - Java config中需要定义为static方法

还有关于bean的一些定制，两个部分

a.Lifecycle Callback
  - InitializingBean / @PostConstruct / init-method //创建
  - DisposableBean / @PreDestory / destory-method   //销毁

b.XxxAware接口
  - ApplicationContextAware
  - BeanFactoryAware
  - BeanNameAware


-------------------------类似的一次debug--------------------------------


工程测试环境下老是报 NoClassDefFoundError（运行时），是okhttp3的，还有一个ClassNotFoundException（编译时就能发现的，注意区别），NoClassDefFoundError是rootcase，第一层是InfluxDbAutoConfiguration自动装配的失败

因为他需要okhttp3，那么首先，@SpringBootTest执行了自动装配，它扫描了上一层的@SpringBootApplication，执行相应的操作，装配的条件是@ConditionalOnClass(InfluxDB.class)，说明有InfluxDB，

项目比较简单，哪里用到了InfluxDB呢，看项目引用的common，common里有

<dependency>
            <groupId>xxxx.tc.xconfig</groupId>
            <artifactId>xconfig-client</artifactId>
            <exclusions>
                <exclusion>
                    <artifactId>hibernate-validator</artifactId>
                    <groupId>org.hibernate</groupId>
                </exclusion>
                <exclusion>
                    <artifactId>jcl-over-slf4j</artifactId>
                    <groupId>org.slf4j</groupId>
                </exclusion>
            </exclusions>
        </dependency>

这里面引了

xxxxwall-sdk（去dependency里面搜一下，查pom），它的pom如下

<dependency>
            <groupId>org.influxdb</groupId>
            <artifactId>influxdb-java</artifactId>
            <version>2.15</version>
            <exclusions>
                <exclusion>
                    <groupId>com.squareup.okio</groupId>
                    <artifactId>okio</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>com.squareup.okhttp3</groupId>
                    <artifactId>okhttp</artifactId>
                </exclusion>
            </exclusions>
        </dependency>


有influxdb-java，但是排除了okhttp？？这个就是我们有InfluxDB类，但是没有okhttp包，这？？

xconfig-client引自哪里，版本号是1.100.37，看着是继承过来的，继承的哪里呢。目前没找到上推的方法，应该是framexxx-bom中的一个


### Springboot的起步依赖

在很久以前，依赖是怎么管理的？

你能记得多少maven依赖
要实现一个功能，需要引入哪些依赖
多个依赖项目之间是否会有兼容问题

maven也做了一些事，如mvn-dependency:tree 和 IDEA Maven Helper插件

排除特定依赖的话用exclusion

统一管理依赖
dependencyManagement  //在主pom中定义，子pom中只要去拿groupId和artifactId就可以了
Bill of Materials - bom

而Starter Dependencies是直接面向功能，一站获得所有相关依赖，不在复制黏贴，官方的Starters

```java
spring-boot-starter-*
```

怎么定制自己的起步依赖呢？

主要内容就是

1、autoconfigure模块，包含自动配置代码 (不是必须的)
2、starter模块，包含指向自动配置模块的依赖及其他相关依赖

命名方式(一般加一个前缀，和官方的做区别)

1、xxx-spring-boot-autoconfigure
2、xxx-spring-boot-starter

不要使用spring-boot作为依赖的前缀
不要使用spring-boot的配置命名空间
starter仅添加必要的依赖
声明对spring-boot-starter的依赖

一个工程`fantasy-spring-boot-starter`，看下它的pom，pom的名字就是`fantasy-spring-boot-starter`

```java
<dependencies>
  <!-- autoconfigure模块  -->
  <dependency>
    <groupId>fantasy.spring.hello</groupId>
    <artifactId>fantasy-spring-boot-autoconfigure</artifactId>
    <version>0.0.1-SNAPSHOT</version>
  </dependency>
  <!-- 实际依赖  -->
  <dependency>
    <groupId>fantasy.spring.hello</groupId>
    <artifactId>fantasy</artifactId>
    <version>0.0.1-SNAPSHOT</version>
  </dependency>
</dependencies>
```

### Springboot加载配置

外化配置加载顺序

1、开启DevTools时，~/spring-boot-devtools/properties
2、测试类上的@TestPropertySource注解
3、@SpringBootTest#properties属性
4、命令行参数(--server.port=9000)
5、SPRING_APPLICATION_JSON中的属性

还有

ServletConfig初始化参数
ServletContext初始化参数
java:comp/env中的JNDI属性
System.getProperties()  //就是-d属性参数
操作系统环境变量
random.*涉及到的RandomValuePropertySource

还有，顺序是从上往下，一次覆盖的

jar包外部的application-{profile}.properties或.yml
jar包内部的application-{profile}.properties或.yml
jar包外部的application.properties或.yml
jar包内部的application.properties或.yml

最后，轮到

@Configuration类上的@PropertySource
SpringApplication.setDefaultProperties()设置的默认属性

application.properties的位置一般在./config或者在./中，然后是CLASSPATH中的/config或者CLASSPATH中的/，当然可以修改配置的名字和路径

```java
spring.config.name
spring.config.location
spring.config.additional-location
```

这些配置都会被抽象成PropertySource，添加PropertySource

<context:property-placeholder>
PropertySourcesPlaceholderConfigurer
  - PropertyPlaceholderConfigurer
@PropertySource
@PropertySources

这个也可以自己定制

### Springboot的运维

Actuator

目的是监控并管理应用程序

访问方式
1、HTTP
  - /actuator/<id>
2、JMX
依赖
spring-boot-starter-actuator

一些常用的endpoint
beans，caches，conditions，configprops，env，health，httptrace，info，metrics，mappings等

除了Web，JConsole中也可以访问这些信息[bean]标签

### Springboot的启动流程

![springbootrun]({{ "/assets/img/springboot/springbootrun.png" | relative_url}})

上面是springboot启动的整个框架图，我们发现启动流程主要分为三个部分，第一部分进行SpringApplication的初始化模块，配置一些基本的环境变量、资源、构造器、监听器，第二部分实现了应用具体的启动方案，包括启动流程的监听模块、加载配置环境模块、及核心的创建上下文环境模块，第三部分是自动化配置模块，该模块作为springboot自动配置核心，在后面的分析中会详细讨论。在下面的启动程序中我们会串联起结构中的主要功能。

每个SpringBoot程序都有一个主入口，也就是main方法，main里面调用SpringApplication.run()启动整个spring-boot程序，该方法所在类需要使用@SpringBootApplication注解，以及@ImportResource注解(if need)，@SpringBootApplication包括三个注解，功能如下：
@EnableAutoConfiguration：SpringBoot根据应用所声明的依赖来对Spring框架进行自动配置
@SpringBootConfiguration(内部为@Configuration)：被标注的类等于在spring的XML配置文件中(applicationContext.xml)，装配所有bean事务，提供了一个spring的上下文环境
@ComponentScan：组件扫描，可自动发现和装配Bean，默认扫描SpringApplication的run方法里的Booter.class所在的包路径下文件，所以最好将该启动类放到根包路径下

run方法中去创建了一个SpringApplication实例，在该构造方法内，我们可以发现其调用了一个初始化的initialize方法
这里主要是为SpringApplication对象赋一些初值。构造函数执行完毕后，我们回到run方法，该方法中实现了如下几个关键步骤：

1、创建了应用的监听器SpringApplicationRunListeners并开始监听

2、加载SpringBoot配置环境(ConfigurableEnvironment)，如果是通过web容器发布，会加载StandardEnvironment，其最终也是继承了ConfigurableEnvironment
可以看出，*Environment最终都实现了PropertyResolver接口，我们平时通过environment对象获取配置文件中指定Key对应的value方法时，就是调用了propertyResolver接口的getProperty方法

3、配置环境(Environment)加入到监听器对象中(SpringApplicationRunListeners)

4、创建run方法的返回对象：ConfigurableApplicationContext(应用配置上下文)，我们可以看一下

方法会先获取显式设置的应用上下文(applicationContextClass)，如果不存在，再加载默认的环境配置（通过是否是web environment判断），默认选择AnnotationConfigApplicationContext注解上下文（通过扫描所有注解类来加载bean），最后通过BeanUtils实例化上下文对象，并返回

ConfigurableApplicationContext类图主要有两个继承方向

LifeCycle：生命周期类，定义了start启动、stop结束、isRunning是否运行中等生命周期空值方法
ApplicationContext：应用上下文类，其主要继承了beanFactory(bean的工厂类)

5、回到run方法内，prepareContext方法将listeners、environment、applicationArguments、banner等重要组件与上下文对象关联

6、接下来的refreshContext(context)方法(初始化方法如下)将是实现spring-boot-starter-*(mybatis、redis等)自动化配置的关键，包括spring.factories的加载，bean的实例化等核心工作

配置结束后，Springboot做了一些基本的收尾工作，返回了应用环境上下文。回顾整体流程，Springboot的启动，主要创建了配置环境(environment)、事件监听(listeners)、应用上下文(applicationContext)，并基于以上条件，在容器中开始实例化我们需要的Bean，至此，通过SpringBoot启动的程序已经构造完成

### Springboot的RestTemplate

RestTemplate在什么情况下抛异常，在4XX 和 5XX 的情况下会抛异常，3XX 和 2XX 是不会的

### Springboot的一些异常

1、启动报错

```html

Exception in thread "main" java.lang.NoClassDefFoundError: org/springframework/core/metrics/ApplicationStartup
at org.springframework.boot.SpringApplication.<init>(SpringApplication.java:251)
at org.springframework.boot.SpringApplication.<init>(SpringApplication.java:264)
at com.amway.commerce.commodity.Application.main(Application.java:27)
Caused by: java.lang.ClassNotFoundException: org.springframework.core.metrics.ApplicationStartup
at java.net.URLClassLoader.findClass(URLClassLoader.java:381)
at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:338)
at java.lang.ClassLoader.loadClass(ClassLoader.java:357)

Spring Boot从2.4开始使用了Spring Framework 5.3，可以从Spring boot 2.3升级到2.4找到此说明。其中org/springframework/core/metrics/ApplicationStartup也是Spring Framework5.3新增的类。

```

解决方法

如果确定要升级，那Spring Framwork 也要升级到对应的版本，2.4.11对应的版本是Spring Framework 5.3.3。否在降级Spring boot版本。

有一说是parent中的版本要大于等于本pom的版本，但其实不是这个问题，做过测试
