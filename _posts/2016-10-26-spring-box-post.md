---
layout: post
title: Spring全家桶
tags: [code, java, spring]
author-id: zqmalyssa
---

Spring，替代EJB的生态圈，在这篇中好好整理一下，参考如下，不定期补充
1. 《Spring实战》
2. 《Spring源码深度解析》

### 基本介绍

Spring的初衷是简化代码的开发，激发POJO的潜能，降低耦合度

DI是怎么实现的？还是要看例子，下面这段，构造函数总自行创建了RescueDameslQuest，使得DamselRescuingKnight紧密的与RescueDameslQuest耦合到一起了

```java
public class DamselRescuingKnight implements Knigth {

	private RescueDameslQuest quest;

	public DamselRescuingKnight() {
		this.quest = new RescueDameslQuest();
	}

   	public void embarkOnQuest() {
   		quest.embark();
	}
}
```

修改一下，如下

```java
public class BraveKnight implements Knigth {

	private Quest quest;

	public BraveKnight(Quest quest) {
		this.quest = quest;
	}

   	public void embarkOnQuest() {
   		quest.embark();
	}
}
```

不同于之前，`BraveKnight`并没有自行创建任务，而是在构造的时候把探险任务作为构造器的参数传入，这是DI(依赖注入)的方式之一，即构造器注入(constructor injection)，传入的Quest是所有任务都必须实现的一个接口，如果一个对象只通过接口(而不是具体的实现或初始化过程)来表明依赖关系，那么这种依赖就能够在对象毫不知情情况下，用不同的具体实现进行替换

1. IOC是通过依赖注入实现的，所谓依赖注入就是容器负责创建对象和维护对象间的依赖关系，而不是通过对象本身负责自己的创建和解决自己的依赖，依赖注入的主要目的是为了解耦，体现的是一种"组合"思想，无论XML配置还是Java配置，都是元数据，Spring容器解析这些配置元数据，进行Bean初始化，配置和管理依赖

2. 声明Bean，@Service，@Component，@Repository，@Controller,注册Bean，@Autowired，@Inject，@Resource

3. Java使用@Configuration和@Bean进行配置，上面2属于注解配置，全局配置使用Java配置，业务Bean使用注解配置

AOP的简单应用

除了骑士类，定义另一个吟游诗人类

```java
public class Minstrel {

	private PrintStream stream;

	public Minstrel(PrintStream stream) {
		this.stream = stream;
	}

   	public void singBeforeQuest() {
   		stream.println("准备吟唱");
	}

	public void singAfterQuest() {
   		stream.println("结束吟唱");
	}
}
```
然后骑士类修改

```java
public class BraveKnight implements Knigth {

	private Quest quest;
	private Minstrel minstrel;

	public BraveKnight(Quest quest, Minstrel minstrel) {
		this.quest = quest;
		this.minstrel = minstrel;
	}

   	public void embarkOnQuest() {
   		minstrel.singBeforeQuest();
		quest.embark();
		minstrel.singAfterQuest();
	}
}
```

这样能实现功能，但感觉不太对，因为吟游诗人并不是骑士需要管理的，因为骑士需要知道吟游诗人，就把它注进来了，这使得`BraveKnight`代码复杂化了，万一吟游诗人是个null呢，还要有逻辑去校验，所以把Minstrel抽象成一个切面，非完全代码，看意思

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<bean id="knight" class="BraveKnight">
	<constructor=arg ref="quest" />
</bean>

<bean id="quest" class="XXXXQuest">
	<constructor=arg value="#{T(System).out}" />
</bean>

<bean id="minstrel" class="Minstrel">
	<constructor=arg value="#{T(System).out}" />
</bean>

<aop:config>
	<aop:aspect ref="minstrel">
		<aop:pointcut id="embark"
			expression="execution(* *.embarkOnQuest(..))" />

		<aop:before pointcut-ref="embark"
			method="singBeforeQuest" />

		<aop:after pointcut-ref="embark"
			method="singAfterQuest" />
	</aop:aspect>
</aop:config>

```
这样，Minstrel也可以被应用到BraveKnight，但BraveKnight不需要去显示的调用它！

想想之前写过的JDBC连接，业务代码就那几行，一堆连接，try、catch处理，这是所谓的样板代码，如果使用spring的JdbcTemplate，就简单许多，只关注业务逻辑

Spring的应用中，你的应用对象生存于Spring容器中，Spring容器负责创建对象，装配它们，配置它们并管理它们的整个生命周期，从生存到死亡，容器是Spring框架的核心。

看下整个Spring的生态圈

![fruli]({{ "/assets/img/spring/spring_frame.jpg" | relative_url}})

大致可分成6大部分，有些需要知道的

1. 要知道的一些注解@Scope，@Value，@Profile(spring.profiles.active)
2. @EnableAsync开启对异步任务的支持，并通过在实际执行的Bean方法中使用@Async注解来声明其是一个异步任务，可以注解在类上，那么所有都是异步任务
3. @EnableScheduling开启对计划任务的支持，用@Scheduled声明方法是计划任务，使用fixedRate属性每隔固定时间执行，cron属性指定时间执行，类似linux的定时任务
4. @Conditional注解
5. @Enable* 系列的注解里面都有个@Import注解，这些开始的实现其实都是导入一些自动配置的Bean

版本迭代的更新

Spring3.1引入了profile功能，
Spring3.2主要去改善了SpringMVC
Spring4.0支持JAVA8新特性，新的异步实现，立即返回并且允许在操作完成后执行回调
Spring4.0引入了@RestController，用此替换@Controller，Spring将会为该控制器的所有处理方法应用消息转换功能，我们就不必为每个方法都添加@ResponseBody了(不然要去进行转换)


### 装配Bean

Spring从两个角度实现自动化装配：

组件扫描(component scanning):Spring会自动发现应用上下文中所创建的bean
自动装配(autowiring):Spring自动满足bean之间的依赖

1. 用Spring特有的注解@Autowired
2. JavaConfig的方法优于XML的方法，比如<bean id="compactDisc" class="soundsystem.SgtPeppers">，这边的class是字符串，如果类名发生了改变，这边是不会去校验的
3. 对强依赖使用构造器注入，对可选性的依赖使用属性注入
4. 不管使用javaconfig或者xml，通常创建一个根配置，再将其他配置引入进来
5. @Profile("dev")的使用，在spring3.2后，这个不仅可以作用在类上，而且可以作用在方法上
6. @Conditional可以条件化的配置bean
7. @Primary用来作为首选配置，存在歧义时候的首选bean
8. @Qualifier注解是使用限定符的主要方式，直接在bean上设置@Qualifier，别其他类引入，让它没有耦合类名
9. Spring定义了多个作用域，默认是单利(Singleton)，原型(Prototype)，会话(Session)，请求(Request)，用@Scope进行调整
10. Spring提供的两种在运行时求值的方法，a.属性占位符(Property placeholder)，b.Spring表达式语言(SpEL)，可以使用@PropertySource和Environment，使用组件扫描和自动装配的话，就可以用@Value注解

### 面向切面的Spring

横切关注点可以被模块化为特殊的类，这些类被称为切面(aspect)

```java
@Aspect		//定义切面
public class Audience {

	@Pointcut("execution(** concert.Performance.perform(..))") //定义切点(何处)
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
主要的业务逻辑是performance

```java
public interface Performance {
	public void perform();
}
```

JavaConfig的配置如下，这边springboot启动的时候不会产生切面，可以写测试或者用controller去验证

```java
@Configuration
@EnableAspectJAutoProxy
@ComponentScan
public class JavaConfig {

  @Bean
  public Audience audience() {
    return new Audience();
  }

}
```

1. AOP有面向方法的拦截，如上面，有可以写面向注解的拦截，自己写的一个注解@interface
2.

### 源码解析

1. 一个类要么是面向继承设计的，要么就是final修饰的
2. 设计中的`模板设计模式`应用到了preProcessXml(root)和postProcessXml(root)方法中
3. bean最终注册的beanDefinitionMap是全局变量，会存在并发问题，所以用synchronized包住，这是一个map缓存
4. 有这样的流程，bean解析(XML或者javaconfig的解析，其实XML就是通过对XML的解析，入Element，Node等，加上一些反射做的)，然后是bean加载，一般情况下Spring通过反射机制利用bean的class属性指定实现类来实例化bean
5. ApplicationContext和BeanFactory两者都是用于加载bean的，但是相比之下，ApplicationContext提供了更多的功能扩展，ApplicationContext包含BeanFactory的所有功能，比BeanFactory优先
6. AOP的实现中有动态代理，源码里有创建代理的部分(JAVA的代理或者CGLIB的)(在获取完所有对应的bean的增强器后，进行代理的创建	)，源码中可以判断Bean上是否存在Aspect注解，反射获取method上是否有pointcut之类的，逻辑中肯定还是会用到反射
7. Spring其实也是用了Proxy和InvocationHandler这两个东西，InvocationHandler中需要重写3个函数，构造函数，将代理的对象接入，invoke方法，此方法实现AOP增强的所有逻辑，getProxy方法，此方法千篇一律，但是必不可少
8. JDBC的封装方法，消除了模板形式的编程
9. Mybatis的使用，pojo类中必须有个无参构造函数，否则查询数据库时，将不能反射构造出实例，DAO层就是操作数据库方法定义的接口层，Spring跟Mybatis整合，配置上倒没少什么，用法上比较简洁，都感觉不到Mybatis的存在

### Spring Cloud Config

使用config进行配置，公司的config为服务中会启动一个config-server

```java
server:
 port: 3901
spring:
  profiles:
    active: native
  cloud:
#    bus:
#      trace:
#        enabled: true
    config:
      server:
        health:
          enabled:  false
        native:
          searchLocations: file:/var/lib/config
          searchPaths: /
```

然后配置下@EnableConfigServer注解

```java
@SpringBootApplication
@EnableConfigServer
@EnableDiscoveryClient
public class  ConfigServerApplication {
  public static void main(String[] args) {

    SpringApplication.run(ConfigServerApplication.class, args);

  }
}
```
当然，我们也可以加载本地的配置文件，使用spring.profiles.active=native就会自动搜索resources目录下的文件，也可以通过设置spring.cloud.config.server.native.searchLocations的方式指定具体位置，这边就是容器中的位置

### Spring Cloud Stream

实现消息队列，通过Binder层与消息服务绑定，编写发送消息与接收消息的服务

### Spring中的事务

传播行为定义了被调用方法的事务边界

| 传播行为 | 意义 |
| :----: | :----:  |
| PROPERGATION_MANDATORY | 表示方法必须运行在一个事务中，如果当前事务不存在，就抛出异常 |
| PROPAGATION_NESTED | 表示如果当前事务存在，则方法应该运行在一个嵌套事务中。否则，它看起来和 PROPAGATION_REQUIRED 看起来没什么俩样 |
| PROPAGATION_NEVER | 表示方法不能运行在一个事务中，否则抛出异常 |
| PROPAGATION_NOT_SUPPORTED |表示方法不能运行在一个事务中，如果当前存在一个事务，则该方法将被挂起 |
| PROPAGATION_REQUIRED | 表示当前方法必须运行在一个事务中，如果当前存在一个事务，那么该方法运行在这个事务中，否则，将创建一个新的事务 |
| PROPAGATION_REQUIRES_NEW | 表示当前方法必须运行在自己的事务中，如果当前存在一个事务，那么这个事务将在该方法运行期间被挂起 |
| PROPAGATION_SUPPORTS | 表示当前方法不需要运行在一个是事务中，但如果有一个事务已经存在，该方法也可以运行在这个事务中 |

隔离级别

在操作数据时可能带来 3 个副作用，分别是**脏读、不可重复读、幻读**。为了避免这 3 中副作用的发生，在标准的 SQL 语句中定义了 4 种隔离级别，分别是未提交读、已提交读、可重复读、可序列化。而在 spring 事务中提供了 5 种隔离级别来对应在 SQL 中定义的 4 种隔离级别，如下

| 隔离界别 | 意义 |
| :----: | :----:  |
| ISOLATION_DEFAULT | 使用后端数据库默认的隔离级别 |
| ISOLATION_READ_UNCOMMITTED | 允许读取未提交的数据（对应未提交读），可能导致脏读、不可重复读、幻读 |
| ISOLATION_READ_COMMITTED | 允许在一个事务中读取另一个已经提交的事务中的数据（对应已提交读）。可以避免脏读，但是无法避免不可重复读和幻读 |
| ISOLATION_REPEATABLE_READ | 一个事务不可能更新由另一个事务修改但尚未提交（回滚）的数据（对应可重复读）。可以避免脏读和不可重复读，但无法避免幻读 |
| ISOLATION_SERIALIZABLE | 	
这种隔离级别是所有的事务都在一个执行队列中，依次顺序执行，而不是并行（对应可序列化）。可以避免脏读、不可重复读、幻读。但是这种隔离级别效率很低，因此，除非必须，否则不建议使用 |

**只读**
如果在一个事务中所有关于数据库的操作都是只读的，也就是说，这些操作只读取数据库中的数据，而并不更新数据，那么应将事务设为只读模式（ READ_ONLY_MARKER ） , 这样更有利于数据库进行优化 。

因为只读的优化措施是事务启动后由数据库实施的，因此，只有将那些具有可能启动新事务的传播行为 (PROPAGATION_NESTED 、 PROPAGATION_REQUIRED 、 PROPAGATION_REQUIRED_NEW) 的方法的事务标记成只读才有意义。

如果使用 Hibernate 作为持久化机制，那么将事务标记为只读后，会将 Hibernate 的 flush 模式设置为 FULSH_NEVER, 以告诉 Hibernate 避免和数据库之间进行不必要的同步，并将所有更新延迟到事务结束

**事务超时**
如果一个事务长时间运行，这时为了尽量避免浪费系统资源，应为这个事务设置一个有效时间，使其等待数秒后自动回滚。与设

置“只读”属性一样，事务有效属性也需要给那些具有可能启动新事物的传播行为的方法的事务标记成只读才有意义
