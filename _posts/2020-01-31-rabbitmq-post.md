---
layout: post
title: RabbitMQ相关
author-id: "zqmalyssa"
tags: [java, rabbitmq]
---
RabbitMQ相关介绍，使用和运维

#### 1.RabbitMQ基本介绍

基本概念介绍和名词，看下下面的结构图

![rabbitmq_frame]({{ "/assets/img/rabbitmq/rabbitmq_frame.png" | relative_url}})

RabbitMQ的主要流程:生产者将消息发送到交换机(exchange)，交换机根据不同路由规则将消息路由到已经绑定到该交换机且符合路由规则的队列(queue)中去，消费者通过监听队列来获取消息。注:交换机不存储消息，默认情况下消息如果没有被正确路由到相应队列，该消息将会被丢弃

消息是服务器与应用程序之间传送的数据，由Properties(还是label)和Payload(Body)组成，Message具有如下属性

| 属性名 | 属性描述 |
| :----: | :----:  |
| Routing Key | Routing key |
| Delivery mode | 是否持久化 Persistent持久化 Non-persistent不持久化 |
| Headers | 头信息，是由一个或多个键值对组成的，是AMQP协议留给AMQP实现做扩展使用的 |
| Properties | AMQP提供的部分属性 |
| Payload | 消息体 |

队列是消息的载体，每个消息都应该被投入一个或多个队列，Queue具有如下属性

| 属性名 | 属性描述 |
| :----: | :----:  |
| Virtual | 虚拟主机 |
| Name | 队列名称，同一个Virtual host下不能有相同的Name |
| Durability | 是否持久化，Durable:是 Transient:否 |
| Auto delete | 如果该队列没有任何订阅的消费者的话，该队列会被自动删除 |
| Arguments | 参数，是AMQP协议留给AMQP实现做扩展使用的 |

RabbitMQ中的绑定通常是指交换机与队列的绑定关系(交换机与交换机绑定极少使用)，Binding有如下属性

| 属性名 | 属性描述 |
| :----: | :----:  |
| To queue / To exchange | 队列名称 / 交换机名称 |
| Routing key | Routing key |
| Arguments | 路由参数(只有Headers Exchange是根据参数路由的，故只有Headers Exchange需要设置该参数) |

注意
1、Default Exchange不能进行Binding也不需要进行Binding
2、除了Default Exchange之外其他任何Exchange都需要和Queue进行Binding，否则无法进行消息路由
3、Direct Exchange、Topic Exchange进行Binding的时候需要指定Routing key
4、Fanout Exchange、Headers Exchange进行Binding的时候不需要指定Routing key

exchange看下面

交换机的作用是接收生产者的消息，并将消息路由到已经绑定到该交换机且符合路由规则的队列中去。RabbitMQ定义了如下4种交换机:[直连交换机(Direct)、扇形交换机(Fanout)、主题交换机(Topic)、首部交换机(headers)]，交换机有如下属性:

| 属性名 | 属性描述 |
| :----: | :----:  |
| Virtual host | 虚拟主机 |
| Name | 交换机名称，同一个Virtual host下不能有相同的Name |
| Type | 交换机类型 |
| Durability | 是否持久化，Durable:是 Transient:否 |
| Auto delete	 | 当最后一个绑定被删除后，该交换机将被删除 |
| Internal | 是否是内部专用exchange，是的话就意味着我们不能往exchange里面发送消息 |
| Arguments | 参数，是AMQP协议留给AMQP实现做扩展使用的 |

RabbitMQ内置一个名称为空字符串的默认交换机，它根据Routing key将消息路由到与队列名与Routing key完全相等的队列中

扇形交换机

![fanout_exchange]({{ "/assets/img/rabbitmq/fanout_exchange.png" | relative_url}})

扇形交换机(Fanout Exchange)会把能接受到的消息全部发送给绑定到自己身上的队列，扇形交换机在路由转发的时候忽略Routing Key。因为广播不需要思考所以扇形交换机处理消息的速度是所有交换机类型里面最快的

直连交换机

![direct_exchange]({{ "/assets/img/rabbitmq/direct_exchange.png" | relative_url}})

直连交换机(Direct Exchange)是一种带路由功能的交换机，它将消息中的Routing key与该交换机关联的所有Binding中的Routing key进行比较，如果完全相等则将消息发送到Binding对应的队列中，适用场景:根据任务的优先级把消息发送到对应的队列中，分配更多资源处理优先级高的队列

主题交换机

![topic_exchange]({{ "/assets/img/rabbitmq/topic_exchange.png" | relative_url}})

主题交换机(Topic Exchange)与直连交换机相比主题交换机的Routing key支持通配符，它将消息中的Routing key与该交换机关联的所有Binding中的Routing key进行比较，如果匹配上对应的匹配规则则将消息发送到Binding对应的队列中

```html
*:匹配一个单词
#:匹配0个或多个单词
```
如果Binding中的Routing key不包含*，#，则表示相等转发，类似于Direct Exchange
如果Binding中的Routing key为#或者#.#，则表示全部转发，类似于Fanout Exchange

首部交换机

首部交换机(Headers Exchange)[不常用 了解即可]在进行路由转发的时候会忽略Routing Key，它将消息中的Headers与该交换机关联的所有Binding中的参数进行匹配，如果匹配上则将消息发送到Binding对应的队列中。它的匹配规则有下列两种类型

```html
匹配规则
x-match = any则表示只要有键值对匹配就能转发消息
x-match = all则表示所有的键值对都匹配才能转发消息
```
Binding的时候至少需要指定两个参数，其中一个是x-match = all/any

#### 2.RabbitMQ简单示例

项目中加个依赖

```java
<dependency>
    <groupId>com.rabbitmq</groupId>
    <artifactId>amqp-client</artifactId>
    <version>5.0.0</version>
</dependency>
```
工具类
```java
package com.qiming.pom.rabbitmq.utils;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import java.util.HashMap;
import java.util.Map;

public class ChannelUtils {

  public static Channel getChannelInstance(String connectionDescription) {
    try {
      ConnectionFactory connectionFactory = getConnectionFactory();
      Connection connection = connectionFactory.newConnection(connectionDescription);
      return connection.createChannel();
    } catch (Exception e) {
      throw new RuntimeException("获取Channel连接失败");
    }
  }

  private static ConnectionFactory getConnectionFactory() {
    ConnectionFactory connectionFactory = new ConnectionFactory();

    // 配置连接信息
    connectionFactory.setHost("10.144.91.121");
    //注意端口号不是web端口号
    connectionFactory.setPort(30133);
    connectionFactory.setVirtualHost("qwer-rabbitmq");
    connectionFactory.setUsername("qwer");
    connectionFactory.setPassword("qwer1234");

    // 网络异常自动连接恢复
    connectionFactory.setAutomaticRecoveryEnabled(true);
    // 每10秒尝试重试连接一次
    connectionFactory.setNetworkRecoveryInterval(10000);

    // 设置ConnectionFactory属性信息
    Map<String, Object> connectionFactoryPropertiesMap = new HashMap();
    connectionFactoryPropertiesMap.put("principal", "Qiming");
    connectionFactoryPropertiesMap.put("description", "RabbitMQ测试系统");
    connectionFactoryPropertiesMap.put("emailAddress", "yzzqm@hotmail.com");
    connectionFactory.setClientProperties(connectionFactoryPropertiesMap);

    return connectionFactory;
  }

}

```
消费者
```java
package com.qiming.pom.rabbitmq.consumer;

import com.qiming.pom.rabbitmq.utils.ChannelUtils;
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;
import java.io.IOException;
import java.util.HashMap;
import java.util.concurrent.TimeoutException;

public class MessageConsumer {

  public static void main(String[] args) throws IOException, TimeoutException {
    Channel channel = ChannelUtils.getChannelInstance("WOW消费者");

    // 声明队列 (队列名, 是否持久化, 是否排他, 是否自动删除, 队列属性);
    AMQP.Queue.DeclareOk declareOk = channel.queueDeclare("wow.order.add", true, false, false, new HashMap<>());

    // 声明交换机 (交换机名, 交换机类型, 是否持久化, 是否自动删除, 是否是内部交换机, 交换机属性);
    channel.exchangeDeclare("wow.order", BuiltinExchangeType.DIRECT, true, false, false, new HashMap<>());

    // 将队列Binding到交换机上 (队列名, 交换机名, Routing key, 绑定属性);
    channel.queueBind(declareOk.getQueue(), "wow.order", "add", new HashMap<>());

    // 消费者订阅消息 监听如上声明的队列 (队列名, 是否自动应答(与消息可靠有关 后续会介绍), 消费者标签, 消费者)
    channel.basicConsume(declareOk.getQueue(), true, "WOW副本信息add处理逻辑消费者", new DefaultConsumer(channel) {
      @Override
      public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
        System.out.println(consumerTag);
        System.out.println(envelope.toString());
        System.out.println(properties.toString());
        System.out.println("消息内容:" + new String(body));
      }
    });
  }

}

```
生产者

```java
package com.qiming.pom.rabbitmq.producer;

import com.qiming.pom.rabbitmq.utils.ChannelUtils;
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import java.io.IOException;
import java.util.HashMap;
import java.util.concurrent.TimeoutException;

public class MessageProducer {

  public static void main(String[] args) throws IOException, TimeoutException {
    Channel channel = ChannelUtils.getChannelInstance("WOW生产者");

    // 声明交换机 (交换机名, 交换机类型, 是否持久化, 是否自动删除, 是否是内部交换机, 交换机属性);
    channel.exchangeDeclare("wow.order", BuiltinExchangeType.DIRECT, true, false, false, new HashMap<>());

    // 设置消息属性 发布消息 (交换机名, Routing key, 可靠消息相关属性 后续会介绍, 消息属性, 消息体);
    AMQP.BasicProperties basicProperties = new AMQP.BasicProperties().builder().deliveryMode(2).contentType("UTF-8").build();
    channel.basicPublish("wow.order", "add", false, basicProperties, "副本信息".getBytes());
  }

}

```
注意:初次运行时一定要先运行消费者后运行生产者，因为初次运行时还没有声明队列和交换机的绑定关系(在消费者代码里)，如果先启动生产者会导致消息无法被正确路由而被丢弃

可以去rabbitmq的web界面上查看信息，比如连接信息，交换机信息，队列信息

#### 3.Spring整合RabbitMQ

Spring AMQP是对AMQP协议的抽象和封装，从官方网站上得知它是由两个项目组成的(spring-amqp和spring-rabbit)。在使用Spring整合RabbitMQ时我们主要关注三个核心接口(MessageListenerContainer、RabbitAdmin以及RabbitTemplate)

```html
RabbitAdmin: 用于声明交换机 队列 绑定等
RabbitTemplate: 用于RabbitMQ消息的发送和接收
MessageListenerContainer: 监听容器 为消息入队提供异步处理
```
使用Spring整合RabbitMQ需要导入如下依赖

```java
<dependency>
    <groupId>org.springframework.amqp</groupId>
    <artifactId>spring-rabbit</artifactId>
    <version>2.0.2.RELEASE</version>
</dependency>
```
写配置类，消费，生产者，运行结果和前面所述一致

Spring AMQP还提供了自动声明方式交换机、队列和绑定。我们可以直接把要自动声明的组件纳入Spring容器中管理即可，自动声明发生在RabbitMQ第一次连接创建的时候，自动声明支持单个和批量自动声明。使用自动声明需要符合如下条件:

1、需要有连接产生
2、RabbitAdmin必须交由Spring管理，且autoStartup必须为true(默认)
3、如果ConnectionFactory使用的是CachingConnectionFactory，则cacheMode必须要为CacheMode.CHANNEL
4、所有要声明的组件的shouldDeclare必须为true
5、要声明的Queue名称不能以amq.开头

上诉规则定义在RabbitAdmin的afterPropertiesSet方法中，有兴趣的同学可以自行阅读RabbitAdmin源码

1.创建生产者配置类 将RabbitAdmin、RabbitTemplate纳入Spring管理

```java
@Configuration
public class SpringAMQPProducerConfig {
    @Bean
    public ConnectionFactory connectionFactory() {
        com.rabbitmq.client.ConnectionFactory connectionFactory = new com.rabbitmq.client.ConnectionFactory();

        // 配置连接信息
        connectionFactory.setHost("192.168.56.128");
        connectionFactory.setPort(5672);
        connectionFactory.setVirtualHost("/");
        connectionFactory.setUsername("roberto");
        connectionFactory.setPassword("roberto");

        // 网络异常自动连接恢复
        connectionFactory.setAutomaticRecoveryEnabled(true);
        // 每10秒尝试重试连接一次
        connectionFactory.setNetworkRecoveryInterval(10000);

        // 设置ConnectionFactory属性信息
        Map<String, Object> connectionFactoryPropertiesMap = new HashMap();
        connectionFactoryPropertiesMap.put("principal", "RobertoHuang");
        connectionFactoryPropertiesMap.put("description", "RGP订单系统V2.0");
        connectionFactoryPropertiesMap.put("emailAddress", "RobertoHuang@foxmail.com");
        connectionFactory.setClientProperties(connectionFactoryPropertiesMap);

        CachingConnectionFactory cachingConnectionFactory = new CachingConnectionFactory(connectionFactory);
        return cachingConnectionFactory;
    }

    @Bean
    public RabbitAdmin rabbitAdmin(ConnectionFactory connectionFactory) {
        return new RabbitAdmin(connectionFactory);
    }

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        return new RabbitTemplate(connectionFactory);
    }
}

```

2.创建生产者启动类

```java
@ComponentScan(basePackages = "roberto.growth.process.rabbitmq.spring.amqp.auto.declare.producer")
public class ProducerApplication {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(roberto.growth.process.rabbitmq.spring.amqp.manual.declare.producer.ProducerApplication.class);

        RabbitAdmin rabbitAdmin = context.getBean(RabbitAdmin.class);
        RabbitTemplate rabbitTemplate = context.getBean(RabbitTemplate.class);

        // 声明交换机
        rabbitAdmin.declareExchange(new DirectExchange("roberto.order", true, false, new HashMap<>()));

        // 声明消息 (消息体, 消息属性)
        MessageProperties messageProperties = new MessageProperties();
        messageProperties.setDeliveryMode(MessageDeliveryMode.PERSISTENT);
        messageProperties.setContentType("UTF-8");
        Message message = new Message("订单信息".getBytes(), messageProperties);

        // 发布消息 (交换机名, Routing key, 消息);
        // 发布消息还可以使用rabbitTemplate.convertAndSend(); 其支持消息后置处理
        rabbitTemplate.send("roberto.order", "add", message);
    }
}

```
3.创建消费者配置类 将RabbitAdmin纳入Spring管理，并在MessageListenerContainer类中定义了消息消费的逻辑，并且在该配置类中声明交换机，队列，绑定

```java
@Configuration
public class SpringAMQPConsumerConfig {
    @Bean
    public ConnectionFactory connectionFactory() {
        com.rabbitmq.client.ConnectionFactory connectionFactory = new com.rabbitmq.client.ConnectionFactory();

        // 配置连接信息
        connectionFactory.setHost("192.168.56.128");
        connectionFactory.setPort(5672);
        connectionFactory.setVirtualHost("/");
        connectionFactory.setUsername("roberto");
        connectionFactory.setPassword("roberto");

        // 网络异常自动连接恢复
        connectionFactory.setAutomaticRecoveryEnabled(true);
        // 每10秒尝试重试连接一次
        connectionFactory.setNetworkRecoveryInterval(10000);

        // 设置ConnectionFactory属性信息
        Map<String, Object> connectionFactoryPropertiesMap = new HashMap();
        connectionFactoryPropertiesMap.put("principal", "RobertoHuang");
        connectionFactoryPropertiesMap.put("description", "RGP订单系统V2.0");
        connectionFactoryPropertiesMap.put("emailAddress", "RobertoHuang@foxmail.com");
        connectionFactory.setClientProperties(connectionFactoryPropertiesMap);

        CachingConnectionFactory cachingConnectionFactory = new CachingConnectionFactory(connectionFactory);
        return cachingConnectionFactory;
    }

    @Bean
    public RabbitAdmin rabbitAdmin(ConnectionFactory connectionFactory) {
        return new RabbitAdmin(connectionFactory);
    }

    @Bean
    // 自动声明交换机
    // 如果要一次性声明多个 使用public List<Exchange> listExchange()即可
    public Exchange exchange() {
        return new DirectExchange("roberto.order", true, false, new HashMap<>());
    }

    @Bean
    // 自动声明队列
    // 如果要一次性声明多个 使用public List<Queue> listQueue()即可
    public Queue queue() {
        return new Queue("roberto.order.add", true, false, false, new HashMap<>());
    }

    @Bean
    // 自动声明绑定
    // 如果要一次性声明多个 使用public List<Binding> listBinding()即可
    public Binding binding() {
        return new Binding("roberto.order.add", Binding.DestinationType.QUEUE, "roberto.order", "add", new HashMap<>());
    }

    @Bean
    public MessageListenerContainer messageListenerContainer(ConnectionFactory connectionFactory) {
        SimpleMessageListenerContainer messageListenerContainer = new SimpleMessageListenerContainer();
        messageListenerContainer.setConnectionFactory(connectionFactory);
        messageListenerContainer.setQueueNames("roberto.order.add");

        // 设置消费者线程数
        messageListenerContainer.setConcurrentConsumers(5);
        // 设置最大消费者线程数
        messageListenerContainer.setMaxConcurrentConsumers(10);

        // 设置消费者属性信息
        Map<String, Object> argumentMap = new HashMap();
        messageListenerContainer.setConsumerArguments(argumentMap);

        // 设置消费者标签
        messageListenerContainer.setConsumerTagStrategy(new ConsumerTagStrategy() {
            @Override
            public String createConsumerTag(String s) {
                return "RGP订单系统ADD处理逻辑消费者";
            }
        });

        // 使用setAutoStartup方法可以手动设置消息消费时机
        messageListenerContainer.setAutoStartup(true);

        // 使用setAfterReceivePostProcessors方法可以增加消息后置处理器
        // messageListenerContainer.setAfterReceivePostProcessors();

        messageListenerContainer.setMessageListener(new MessageListener() {
            @Override
            public void onMessage(Message message) {
                try {
                    System.out.println(new String(message.getBody(), "UTF-8"));
                    System.out.println(message.getMessageProperties());
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });
        return messageListenerContainer;
    }
}

```
4.创建消费者启动类
```java
@ComponentScan(basePackages = "roberto.growth.process.rabbitmq.spring.amqp.auto.declare.consumer")
public class ConsumerApplication {
    public static void main(String[] args) {
        new AnnotationConfigApplicationContext(ConsumerApplication.class);
    }
}
```

该代码与上诉代码实现的效果一致，只是将交换机，队列，绑定进行了自动声明，不是上面代码中的手动声明

以上两个Demo在消费消息处理逻辑时往MessageListenerContainer中传递了MessageListener，但是我们有时候已经写好了消费逻辑对应的类，我们不希望它去扩展MessageListener/ChannelAwareMessageListener，因为这么做的话意味着我们需要改变现有代码。Spring AMQP提供了消息处理器适配器的功能，它可以把一个纯POJO类适配成一个可以处理消息的处理器，默认处理消息的方法为handleMessage，可以通过setDefaultListenerMethod方法进行修改

创建消费者消息处理器类，它可是是纯POJO类

```java
public class MessageHandle {
    public void add(byte[] message){
        try {
            System.out.println(new String(message,"UTF-8"));
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
    }
}

```
创建消费者配置类 配置自定义消息处理器(将roberto.order.add队列使用自定义消息处理类的add方法进行处理)

```java
// 新建消息处理器适配器
MessageListenerAdapter messageListenerAdapter = new MessageListenerAdapter(new MessageHandle());
// 设置默认处理消息方法
messageListenerAdapter.setDefaultListenerMethod("handleMessage");
Map<String, String> queueOrTagToMethodName = new HashMap<>();
// 将roberto.order.add队列的消息 使用add方法进行处理
queueOrTagToMethodName.put("roberto.order.add","add");
messageListenerAdapter.setQueueOrTagToMethodName(queueOrTagToMethodName);
messageListenerContainer.setMessageListener(messageListenerAdapter);
```

在上诉例子中我们定义的add(byte[] message)方法的参数是一个字节数组，但是有时候我们往RabbitMQ中发送的是一个JSON对象，我们希望在处理消息的时候它已经自动帮我们转为JAVA对象；又或者我们往RabbitMQ中发送的是一张图片或其他格式的文件，我们希望在处理消息的时候它已经自动帮我们转成文件格式，我们可以手动设置MessageConverter来实现如上需求，如果未设置MessageConverter则使用Spring AMQP默认提供的SimpleMessageConverter

1、当发送普通对象的JSON数据时，需要在消息的header中增加一个__TypeId__的属性告知消费者是哪个对象
2、当发送List集合对象的JSON数据时，需要在消息的header中将__TypeId__指定为java.util.List，并且需要额外指定属性__ContentTypeId__用户告知消费者List集合中的对象类型
3、当发送Map集合对象的JSON数据时，需要在消息的header中将__TypeId__指定为java.util.Map，并且需要额外指定属性__KeyTypeId__用于告知客户端Map中key的类型，__ContentTypeId__用于告知客户端Map中Value的类型

XXX

还有一种比较简便的消费方式，即使用RabbitListener注解进行消息消费，

1.在启动类上添加@EnableRabbit注解
2.在Spring容器中托管一个RabbitListenerContainerFactory，默认实现类SimpleRabbitListenerContainerFactory
3.编写一个消息处理器类托管到Spring容器中，并使用@RabbitListener注解标注该类为RabbitMQ的消息处理类
4.使用@RabbitHandler注解标注在方法上，表示当有收到消息的时候，就交给带有@RabbitHandler的方法处理，具体找哪个方法需要根据MessageConverter转换后的对象类型决定

创建消费者配置类 将RabbitListenerContainerFactory交由Spring托管

```java
@Configuration
public class SpringAMQPConsumerConfig {
    @Bean
    public ConnectionFactory connectionFactory() {
        com.rabbitmq.client.ConnectionFactory connectionFactory = new com.rabbitmq.client.ConnectionFactory();

        connectionFactory.setHost("192.168.56.128");
        connectionFactory.setPort(5672);
        connectionFactory.setVirtualHost("/");
        connectionFactory.setUsername("roberto");
        connectionFactory.setPassword("roberto");

        connectionFactory.setAutomaticRecoveryEnabled(true);
        connectionFactory.setNetworkRecoveryInterval(10000);

        Map<String, Object> connectionFactoryPropertiesMap = new HashMap();
        connectionFactoryPropertiesMap.put("principal", "RobertoHuang");
        connectionFactoryPropertiesMap.put("description", "RGP订单系统V2.0");
        connectionFactoryPropertiesMap.put("emailAddress", "RobertoHuang@foxmail.com");
        connectionFactory.setClientProperties(connectionFactoryPropertiesMap);

        CachingConnectionFactory cachingConnectionFactory = new CachingConnectionFactory(connectionFactory);
        return cachingConnectionFactory;
    }

    @Bean
    public RabbitAdmin rabbitAdmin(ConnectionFactory connectionFactory) {
        return new RabbitAdmin(connectionFactory);
    }

    @Bean
    public RabbitListenerContainerFactory<?> rabbitListenerContainerFactory(ConnectionFactory connectionFactory) {
        SimpleRabbitListenerContainerFactory simpleRabbitListenerContainerFactory = new SimpleRabbitListenerContainerFactory();
        simpleRabbitListenerContainerFactory.setConnectionFactory(connectionFactory);

        // 设置消费者线程数
        simpleRabbitListenerContainerFactory.setConcurrentConsumers(5);
        // 设置最大消费者线程数
        simpleRabbitListenerContainerFactory.setMaxConcurrentConsumers(10);

        // 设置消费者标签
        simpleRabbitListenerContainerFactory.setConsumerTagStrategy(new ConsumerTagStrategy() {
            @Override
            public String createConsumerTag(String s) {
                return "RGP订单系统ADD处理逻辑消费者";
            }
        });

        return simpleRabbitListenerContainerFactory;
    }
}

```

自定义消费者消息处理器类 在消息处理器类中使用@RabbitListener注解声明该类为RabbitMQ消息处理器类，并在bindings属性中声明了队列和交换机已经它们之间的绑定关系(监听roberto.order.add队列)，使用@RabbitHandler注解声明具体消息处理方法

```java
@Component
@RabbitListener(bindings = {@QueueBinding(value = @Queue(value = "roberto.order.add", durable = "true", autoDelete = "false", exclusive = "false"), exchange = @Exchange(name = "roberto.order"))})
public class SpringAMQPMessageHandle {
    @RabbitHandler
    public void add(byte[] body) {
        System.out.println("----------byte[]方法进行处理----------");
        try {
            System.out.println(new String(body, "UTF-8"));
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
    }
}

```

消费者启动类 添加@EnableRabbit注解

#### 4.RabbitMQ异常处理

使用JAVA客户端整合RabbitMQ进行的许多操作都会抛出异常，我们可以自定义异常处理器进行处理，比如我们希望在RabbitMQ消费消息失败时记录一条日志，又或者在消息消费失败时发送一则通知等操作

```java
// 设置自定义异常处理器
        connectionFactory.setExceptionHandler(new DefaultExceptionHandler() {
            @Override
            public void handleConsumerException(Channel channel, Throwable exception, Consumer consumer, String consumerTag, String methodName) {
                System.out.println("----------消息消费异常处理----------");
                System.out.println("消息消费异常日志记录:" + exception.getMessage());
                super.handleConsumerException(channel, exception, consumer, consumerTag, methodName);
            }
        });

```

说明当消费消息出现异常时，会进入我们自定义异常处理的逻辑。需要注意的是默认RabbitMQ Java Client在发生异常时会将Channel/Connection关闭，进而使程序未能按照预期的方向执行，所以我们在软件设计的时候应当考虑周全

Spring AMQP在监听器抛出一个异常的时候它会将该异常包装成ListenerExecutionFailedException，通常这个消息会被拒绝并且重新放入队列中，如果设置DefaultRequeueRejected属性为false将把这个消息直接丢弃。需要注意的是如果抛出的异常是 ARADRE 或其他被RabbitMQ认为是致命错误的异常，即便DefaultRequeueRejected的值为true该消息也不会重新加入队列，而是会被直接丢弃。当抛出异常为以下几种异常时消息将不会重新入队列

1.org.springframework.amqp.support.converter.MessageConversionException
2.org.springframework.messaging.converter.MessageConversionException
3.org.springframework.messaging.handler.invocation.MethodArgumentResolutionException
4.java.lang.NoSuchMethodException
5.java.lang.ClassCastException

#### 5.RabbitMQ的消息可靠性

在项目中使用RabbitMQ时，我们可能会遇到这样的问题:如一个订单系统当用户付款成功时我们往消息中间件添加一条记录期望消息消费者修改订单状态，但是最终实际订单状态并没有被修改成功。遇到这种问题我们排查的思路如下:

1.消息是否已经成功发送到消息中间件
2.消息是否有丢失的情况 消息是否已经被消费成功

**生产者保证消息可靠投递**

为了保证消息被正确投递到消息中间件，RabbitMQ提供了如下两个配置来保证消息投递的可靠性:

1、在发送消息的时候我们可以设置Mandatory属性。如果设置了Mandatory属性则当消息不能被正确路由到队列中去时将会触发Return Method，这样我们可以在Return Method中进行相关业务处理，如果Mandatory没有设置则当消息不能正确路由到队列中去的时候，Broker将会丢弃该消息

2、RabbitMQ还提供了消息确认机制(Publisher Confirm)。生产者将Channel设置成Confirm模式，当设置Confirm模式后所有在该信道上面发布的消息都会被指派一个唯一的ID(从1开始，ID在同个Channel范围是唯一的)，一旦消息被投递到所有匹配的队列之后Broker就会发送一个确认给生产者(包含消息的唯一ID)，这就使得生产者知道消息已经正确到达目的队列了，就是一个ACK

如果消息和队列是可持久化的那么确认消息会将消息写入磁盘之后出，Broker回传给生产者的确认消息中DeliverTag域包含了确认消息的序列号，此外Broker也可以设置basic.ack的multiple域，表示到这个序列号之前的所有消息都已经得到了处理(multiple如果为true则表示小于等于deliveryTag的消息都被投递成功，如果为false则表示只有等于deliveryTag的消息已经被投递成功)

除了使用Publisher Confirm方式，RabbitMQ还提供了事务机制保证消息投递，但是使用事务会大大降低系统的吞吐量，就失去了消息中间件存在的意义，Publisher Confirm模式最大的好处在于他是异步的，一旦发布一条消息生产者应用程序就可以在等信道返回确认的同时继续发送下一条消息，当消息最终得到确认之后，生产者应用可以通过回调ACK方法来处理该确认消息，如果RabbitMQ因为自身内部错误导致消息丢失，生产者应用可以通过回调NACK方法来处理该确认消息

Publisher Confirm机制无法进行回滚，一旦服务器崩溃生产者无法得到Confirm信息，生产者其实本身也不知道该消息是否已经被持久化，只有继续重发来保证消息不丢失，但是如果原先已经持久化的消息并不会被回滚，这样队列中就会存在两条相同的消息(恢复后持久化回来的还有一条)，系统需要支持去重。

将channel.basicPublish()方法的Mandatory属性设置为true，并为Channel添加ReturnListener，同时使用channel.confirmSelect()开启消息确认模式并且为channel添加ConfirmListener，发送两条消息(修改其中一条消息的Routing Key让消息不能被正确路由)

```java
/ 当消息没有被正确路由时 回调ReturnListener
        channel.addReturnListener(new ReturnListener() {
            @Override
            public void handleReturn(int replyCode, String replyText, String exchange, String routingKey, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.out.println("replyCode:" + replyCode);
                System.out.println("replyText:" + replyText);
                System.out.println("exchange:" + exchange);
                System.out.println("routingKey:" + routingKey);
                System.out.println("properties:" + properties);
                System.out.println("body:" + new String(body, "UTF-8"));
            }
        });

        // 开启消息确认
        channel.confirmSelect();
        channel.addConfirmListener(new ConfirmListener() {
            @Override
            public void handleAck(long deliveryTag, boolean multiple) throws IOException {
                System.out.println("----------Ack----------");
                System.out.println(deliveryTag);
                System.out.println(multiple);
            }

            @Override
            public void handleNack(long deliveryTag, boolean multiple) throws IOException {
                System.out.println("----------Nack----------");
                System.out.println(deliveryTag);
                System.out.println(multiple);
            }
        });

        // 将mandatory属性设置成true
        channel.basicPublish("roberto.order", "add", true, basicProperties, "订单信息".getBytes());
        channel.basicPublish("roberto.order", "addXXX", true, basicProperties, "订单信息".getBytes());
```
得到的信息如下

```html
replyCode:312
replyText:NO_ROUTE
exchange:roberto.order
routingKey:addXXX
properties:#contentHeader<basic>(content-type=UTF-8, content-encoding=null, headers=null, delivery-mode=2, priority=null, correlation-id=null, reply-to=null, expiration=null, message-id=null, timestamp=null, type=null, user-id=null, app-id=null, cluster-id=null)
body:订单消息

----------Ack----------
2
false

----------Ack----------
1
false
```

以上输出说明当消息不能被确认路由时调用了ReturnListener的handleReturn方法，同时说明如果消息没有被正确路由仍然走的是ACK方法，Publisher Confirm只能保证消息到达消息中间件。如果要测试NACK方法可以通过在发送消息还没被确认时，停止RabbitMQ服务进行测试

Spring AMQP原理差不多，总结

Spring AMQP以上输出说明当消息不能被确认路由时调用了ReturnCallback的returnedMessage方法，同时说明如果消息没有被正确路由ACK的值仍为true，Publisher Confirm只能保证消息到达消息中间件。如果要使得ACK的值为false可以通过在发送消息还没被确认时，停止RabbitMQ服务进行测试

同时Spring AMQP对Publisher Confirm进行了封装，我们可以在发送消息时传递CorrelationData，当调用消息确认回调方法时我们可以获取到发送消息时传递的CorrelationData，该功能为我们业务处理提供了极大便利，我们不再需要花成本去维护Delivery Tag，可以直接使用CorrelationData的getId()方法获取业务主键

**消费者保证消息可靠消费**

以上章节我们使用消息返回和消息确认机制保证消息能够到达消息中间件并被正确路由到队列，但是在消费者消费消息时我们无法得到反馈信息，我们无法得知消息是否已经被消费成功。为了实现该功能RabbitMQ提供了Consumer Acknowledgements机制，使用Consumer Acknowledgements能在消费者消费消息后给Broker进行反馈，Broker根据反馈对消息进行处理

使用channel.basicConsume()方法将消息自动确认设置为false，在消费消息的时候手动使用channel.basicAck()进行消息确认，channel.basicNack()进行消息拒绝

```java
channel.basicConsume(declareOk.getQueue(), false, "RGP订单系统ADD处理逻辑消费者", new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                try {
                    System.out.println(consumerTag);
                    System.out.println(envelope.toString());
                    System.out.println(properties.toString());
                    System.out.println("消息内容:" + new String(body));
                    if ("订单信息2".equals(new String(body))) {
                        throw new RuntimeException();
                    } else {
                        channel.basicAck(envelope.getDeliveryTag(), false);
                    }
                } catch (Exception e) {
                    channel.basicNack(envelope.getDeliveryTag(), false, true);
                }
            }
        });

```

两条消息先是处于未确认状态，其中一条消息在调用channel.basicAck()后消息被成功消费，另外一条消息由于未确认重新发送回队列，所有又回到Ready状态，如此循环

Spring AMQP原理差不多，总结

AcknowledgeMode有三个可选值分别是NONE，MANUAL，AUTO。NONE为自动确认等效于autoAck=true，MANUAL手动确认等效于autoAck=false，AUTO是根据方法的执行情况来决定是确认还是拒绝，如果消息被成功消费了则自动确认，如果在消费消息时抛出AmqpRejectAndDontRequeueException消息会被拒绝并且不会重新入队列，如果抛出ImmediateAcknowledgeAmqpException则消息会被确认，如果抛出其他异常则消息会被拒绝，并且重新入队列。更多详细信息可查阅SimpleMessageListenerContainer的doReceiveAndExecute()方法进行获取

**RabbitMQ持久化**

以上篇幅我们介绍了如何保证生产者和消费者消息的可靠性，但是假设在运行过程中RabbitMQ服务端宕机了，若此前没有进行持久化操作则消息就会丢失。所以使用RabbitMQ通常建议开启持久化功能

1.交换机持久化 在声明时指定durable为true
2.队列持久化 在声明时指定durable为true
3.消息持久化 在声明时指定delivery_mode为2

只有进行如上几个操作，我们才能保证RabbitMQ消息的可靠性

#### 6.RabbitMQ常用属性详解

1、Alternate Exchange简称AE，当消息不能被正确路由时，如果交换机设置了AE则消息会被投递到AE中，如果存在AE链则会按此继续投递，直到消息被正确路由或AE链结束消息被丢弃。通常建议AE的交换机类型为Fanout防止出现路由失败，如果一个交换机指定了AE那么意为着该交换机和AE链都无法被正确路由时才会触发消息返回

将roberto.order交换机的AE指向roberto.order.failure交换机 并发送一条不能被正确路由的消息

```java
// 声明AE 类型为Fanout
       channel.exchangeDeclare("roberto.order.failure", BuiltinExchangeType.FANOUT, true, false, false, new HashMap<>());
       // 为roberto.order设置AE
       Map<String, Object> exchangeProperties = new HashMap<>();
       exchangeProperties.put("alternate-exchange", "roberto.order.failure");
       channel.exchangeDeclare("roberto.order", BuiltinExchangeType.DIRECT, true, false, false, exchangeProperties);

       // 发送一条不能正确路由的消息
       AMQP.BasicProperties basicProperties = new AMQP.BasicProperties().builder().deliveryMode(2).contentType("UTF-8").build();
       channel.basicPublish("roberto.order", "addXXX", false, basicProperties, "订单信息".getBytes());
```

将roberto.order交换机AE指向roberto.order.failure交换机，并将roberto.order.add队列绑定到roberto.order交换机上routing key为add，将roberto.order.add.failure队列绑定到roberto.order.failure交换机上，同时监听这两个队列

```java
AMQP.Queue.DeclareOk declareOk = channel.queueDeclare("roberto.order.add", true, false, false, new HashMap<>());
        // 声明AE 类型为Fanout
        channel.exchangeDeclare("roberto.order.failure", BuiltinExchangeType.FANOUT, true, false, false, new HashMap<>());
        // 为roberto.order设置AE
        Map<String, Object> exchangeProperties = new HashMap<>();
        exchangeProperties.put("alternate-exchange", "roberto.order.failure");
        channel.exchangeDeclare("roberto.order", BuiltinExchangeType.DIRECT, true, false, false, exchangeProperties);
        channel.queueBind(declareOk.getQueue(), "roberto.order", "add", new HashMap<>());

        // 将roberto.order.add.failure队列绑定到roberto.order.failure交换机上 无需指定routing key
        AMQP.Queue.DeclareOk declareOk2 = channel.queueDeclare("roberto.order.add.failure", true, false, false, new HashMap<>());
        channel.queueBind(declareOk2.getQueue(), "roberto.order.failure", "", new HashMap<>());

        // 消费roberto.order.add队列
        channel.basicConsume(declareOk.getQueue(), false, "RGP订单系统ADD处理逻辑消费者", new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                try {
                    System.out.println("----------roberto.order.add----------");
                    System.out.println(new String(body, "UTF-8"));
                    channel.basicAck(envelope.getDeliveryTag(), false);
                } catch (Exception e) {
                    channel.basicNack(envelope.getDeliveryTag(), false, true);
                }
            }
        });

        // 消费roberto.order.add.failure队列
        channel.basicConsume(declareOk2.getQueue(), false, "RGP订单系统ADD FAILURE处理逻辑消费者", new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                try {
                    System.out.println("----------roberto.order.add.failure----------");
                    System.out.println(new String(body, "UTF-8"));
                    channel.basicAck(envelope.getDeliveryTag(), false);
                } catch (Exception e) {
                    channel.basicNack(envelope.getDeliveryTag(), false, true);
                }
            }
        });

```

2、消息存活时间，TTL(Time To Live)指的是消息的存活时间，当消息超过过期时间后，消息即成为死信。RabbitMQ可以分别为队列和消息本身设置消息过期时间，注意当队列过期时间和消息过期时间都存在时，取两者中较短的时间

```java
// 设置消息将在三秒后过期
new AMQP.BasicProperties().builder().expiration("3000")

// 设置队列中的消息 将在5秒后过期
Map<String, Object> queueProperties = new HashMap<>();
queueProperties.put("x-message-ttl", 5000);
Queue addFailureQueue = new Queue("roberto.order.add.failure", true, false, false, queueProperties);

```

3、消息数量限制，我们可以为队列设置x-max-length来指定队列中可存放的最大消息数量，或者设置x-max-length-bytes来指定队列中存放消息内容的最大字节长度，当超过这个长度后部分消息将成为死信

```java
// 设置队列可存放最大消息数量为2条
Map<String, Object> queueProperties = new HashMap<>();
queueProperties.put("x-max-length", 2);
AMQP.Queue.DeclareOk declareOk = channel.queueDeclare("roberto.order.add", true, false, false, queueProperties);

// 设置队列可存放消息内容的最大字节长度为20
Map<String, Object> queueProperties = new HashMap<>();
queueProperties.put("x-max-length-bytes", 20);
AMQP.Queue.DeclareOk declareOk = channel.queueDeclare("roberto.order.add", true, false, false, queueProperties);

```

4、死信队列，如果队列中设置了Dead Letter Exchange属性，当消息变成死信后它能重写被投递到另一个交换机。消息变成死信一般有如下几种情况

1.消息过期而被删除
2.消息数量超过队列最大限制而被删除
3.消息总大小超过队列最大限制而被删除
4.消息被拒绝(channel.basicReject()/channel.basicNack())，且requeue为false

同时也可以指定一个可选的x-dead-letter-routing-key表示默认的routing-key，如果没有指定则使用消息的routing-key

```java
Map<String, Object> queueProperties = new HashMap<>();
queueProperties.put("x-dead-letter-exchange","roberto.order.failure");
AMQP.Queue.DeclareOk declareOk = channel.queueDeclare("roberto.order.add", true, false, false, queueProperties);

```

5、优先级队列，

1.创建优先级队列需增加x-max-priority参数(指定一个数字 数值越大则优先级越高)
2.发送消息的时候需要设置priority属性，最好不要超过上面指定的x-max-priority，如果超过了x-max-priority则为x-max-priority，需要注意如果生产者发送消息较慢，消费者消费速度较快则可能不会严格的按照优先级队列进行消费

```java
// 创建队列时设置队列最大优先级
Map<String, Object> queueProperties = new HashMap<>();
queueProperties.put("x-max-priority", 10);
AMQP.Queue.DeclareOk declareOk = channel.queueDeclare("roberto.order.add", true, false, false, queueProperties);

// 发送消息时使用priority()方法指定消息优先级
AMQP.BasicProperties basicProperties = new AMQP.BasicProperties().builder().deliveryMode(2).contentType("UTF-8").priority(i).build();

```

#### 7.RabbitMQ实现异步RPC

服务端步骤:
1.服务端监听一个队列，监听客户端发送过来的消息
2.收到消息之后调用RPC服务得到调用结果
3.从消息属性中获取reply_to，correlation_id属性，把调用结果发送给reply_to指定的队列，发送的消息属性要带上correlation_id

客户端步骤:

1.监听reply_to队列
2.发送消息，消息属性需要带上reply_to,correlation_id
3.服务端处理完之后reply_to对应的队列就会收到异步处理结果消息
4.收到消息之后进行处理，根据消息的correlation_id找到对应的请求

XXX

#### 8.SpringBoot整合RabbitMQ

只需要添加依赖

```java
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.0.RELEASE</version>
</parent>

<dependencies>
   <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
    </dependency>
</dependencies>

```

创建生产者配置类

```java
@Configuration
public class RabbitMQProducerConfig {
    @Bean
    public Exchange exchange() {
        return new DirectExchange("roberto.order", true, false, new HashMap<>());
    }
}

```

创建消费者配置类

```java
@Configuration
public class RabbitMQConsumerConfig {
    @Bean
    public Exchange exchange() {
        return new DirectExchange("roberto.order", true, false, new HashMap<>());
    }

    @Bean
    public Queue queue() {
        return new Queue("roberto.order.add", true, false, false, new HashMap<>());
    }

    @Bean
    public Binding binding() {
        return new Binding("roberto.order.add", Binding.DestinationType.QUEUE, "roberto.order", "add", new HashMap<>());
    }

    @Bean
    public MessageListenerContainer messageListenerContainer(ConnectionFactory connectionFactory) {
        SimpleMessageListenerContainer messageListenerContainer = new SimpleMessageListenerContainer();
        messageListenerContainer.setConnectionFactory(connectionFactory);
        messageListenerContainer.setQueueNames("roberto.order.add");

        messageListenerContainer.setConcurrentConsumers(5);
        messageListenerContainer.setMaxConcurrentConsumers(10);

        Map<String, Object> argumentMap = new HashMap();
        messageListenerContainer.setConsumerArguments(argumentMap);
        messageListenerContainer.setConsumerTagStrategy(new ConsumerTagStrategy() {
            @Override
            public String createConsumerTag(String s) {
                return "RGP订单系统ADD处理逻辑消费者";
            }
        });

        messageListenerContainer.setMessageListener(new MessageListener() {
            @Override
            public void onMessage(Message message) {
                try {
                    System.out.println(new String(message.getBody(), "UTF-8"));
                    System.out.println(message.getMessageProperties());
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });
        return messageListenerContainer;
    }
}

```
#### 9.SpringCloud的Stream与RabbitMQ

既然已经实现了springboot与rabbitmq的集成了，为什么还会出现SpringCloudStream这个组件呢，相比较于传统的Spring项目、SpringBoot项目使用消息中间件的很多配置不同，SpringCloud Stream抽象了中间件产品的不同，在SpringCloud中你仅仅需要修改几行配置文件就可以灵活的切换中间件产品而不需要修改任何代码

这段话的意思可以这么理解，第一，springboot与rabbitmq的整合过程，如果做到精确的配置和高可用，稳定使用，其实还是需要一些运维成本的，尤其在面临复杂的业务场景中，在处理消息队列的问题是比较耗费精力的，从而开发人员不能将精力放在业务的开发上；第二点，随着分布式的应用场景越来越多也越来越复杂，通过feign或者http的形式调用在某些场景也不能很好的达到预期的效果，因此我们需要借助消息中间件进行解耦

springcloud由于其本身是基于springboot之上的开发和封装，因此对于消息中间件的接入是必须的，比如像springcloud生态中的zipkin服务链路追踪，对rabbitmq就很好的支持，SpringCloudStream目前支持rabbitmq和kafka两种消息中间件，我们可以这么理解，SpringCloudStream对于消息中间件的引其实就是简化配置和开发，应用程序只需要做非常简单的配置，程序中使用简单的API即可完成之前的功能，而且性能还很好，下面就用简单的代码来演示一下整合使用的流程

首先看下pom依赖

```java
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
            <version>2.0.1.RELEASE</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-starter-stream-rabbit -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
            <version>2.0.1.RELEASE</version>
        </dependency>

```

除了`starter-amqp`，多了下面的starter-stream，再看下配置，很清晰

```java
spring.cloud.stream.binders.rabbit.type=rabbit
spring.cloud.stream.binders.rabbit.environment.spring.rabbitmq.host=
spring.cloud.stream.binders.rabbit.environment.spring.rabbitmq.port=
spring.cloud.stream.binders.rabbit.environment.spring.rabbitmq.username=
spring.cloud.stream.binders.rabbit.environment.spring.rabbitmq.password=
spring.cloud.stream.binders.rabbit.environment.spring.rabbitmq.virtual-host=
```

然后结构也很简单

```java
interface HelloBinding {

    @Output("greetingChannel")
    MessageChannel greeting();
}
```
作为生产者，要将消息推送到这个channel上

```java
@RestController
public class ProducerController {

    private MessageChannel greet;

    public ProducerController(HelloBinding binding) {
        greet = binding.greeting();
    }

    @GetMapping("/greet/{name}")
    public void publish(@PathVariable String name) {
        String greeting = "Hello, " + name + "!";
        Message<String> msg = MessageBuilder.withPayload(greeting)
            .build();
        this.greet.send(msg);
    }
}
```
我们创建了一个ProducerController类，它有一个MessageChannel类型的属性 greet。这是通过我们前面声明的方法在构造函数中初始化的，我们将在的主类中添加@EnableBinding注解，传入HelloBinding告诉Spring加载。

```java
@EnableBinding(HelloBinding.class)
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```
注意这个@EnableBinding，然后就完成了消息的发送了，是不是很简单

消费者也类似，我们需要监听之前创建的通过到，

```java
public interface HelloBinding {

    String GREETING = "greetingChannel";

    @Input(GREETING)
    SubscribableChannel greeting();
}
```
与生产者不一样的就是这个SubscribableChannel 和 @Input，创建一个处理数据的方法

```java
@EnableBinding(HelloBinding.class)
public class HelloListener {

    @StreamListener(target = HelloBinding.GREETING)
    public void processHelloChannelGreeting(String msg) {
        System.out.println(msg);
    }
}
```
注意这个@StreamListener，这个方法需要一个字符串作为参数，我们刚刚在控制台打印了这个参数。我们还在类添加@EnableBinding启用了HelloBinding，这个@StreamListener是不是像上面的@RabbitListener，到这里其实流程就结束了

条件分派和消息分组

SpringCloudStream另外有个很有意思的用法就是条件分派，简单来说就是，根据特定的规则将消息路由到不同的队列或通道，实现消息的分流，这个用法在高并发的场景下处理瞬间激增的订单类消息是很有用的

```java
@Autowired
private MyProcessor processor;

@StreamListener(
  target = MyProcessor.INPUT,
  condition = "payload < 10")
public void routeValuesToAnOutput(Integer val) {
    processor.anOutput().send(message(val));
}

@StreamListener(
  target = MyProcessor.INPUT,
  condition = "payload >= 10")
public void routeValuesToAnotherOutput(Integer val) {
    processor.anotherOutput().send(message(val));
}
```

而消息分组的概念在mq中，都有类似的用法，这个在rocketMq中使用的比较多，其实际用法是，多个消费者消费同一批消息，任何一个消费者实例抢占了消息，另一个或多个消费者实例就不能消费同样的消息了，这样一定程度上可以分担消费的压力，即将多个消费端的实例归于同一组即可

#### 10.RabbitMQ的延迟队列的设计

RabbitMQ本身是没有延时队列的功能的，延时队列首先说下，是指当消息被发送以后，并不想让消费者立刻拿到消息，而是等待特定时间后，消费者才能拿到这个消息进行消费。

延迟队列的使用场景有很多，比如: 用户希望通过手机远程遥控家里的智能设备在指定的时间进行工作。这时候就可以将用户指令发送到延迟队列，当指令设定的时间到了再将指令推送到智能设备。

RabbitMQ中有一些官方插件可以用用，但是也可以用自身的死信队列+TTL时间进行模拟，消息过期上面提到，有消息的过期时间和队列的过期时间，我们这里模拟出5s，10s，30s，1分钟四个等级的延迟队列，生产者在发送消息的时候通过设置不同的路由键，将消息发送到与交换机绑定的不同延时队列中，这几个延迟队列分别设置5s，10s，30s和1分钟，同时也要配置DLX（x-dead-letter-routing-key）和DLX相对应的死信队列。当相应的消息过期后，就会转到相应的死信队列中（也可以叫它延时队列吧），那消费者根据自身情况，分别选择不同的死信队列

看下面的代码，怎么设计队列和binding的

```java
/**
 * @Description: 消息队列配置类
 * @Author: zw
 */
@Configuration
public class RabbitConfig {


    /**
     * 死信队列跟交换机类型没有关系  不影响该类型交换机的特性.
     */
    @Bean
    DirectExchange deadLetterExchange() {
        return new DirectExchange("Dead_Letter_Exchange");
    }

    /**
     * 声明一个死信队列.
     * 为队列设置过期时间
     */
    @Bean
    Queue queue_5s() {
        Map<String, Object> map = new HashMap<String, Object>(3);
        map.put("x-dead-letter-exchange", "Dead_Letter_Exchange");
        map.put("x-dead-letter-routing-key", "dlx1");
        map.put("x-message-ttl", 5000);
        return new Queue("queue_5s", true, false, false, map);
    }

    /**
     * 死信路由通过 5s 绑定键绑定到死信队列上.
     */
    @Bean
    public Binding queue_5sBinding() {
        return new Binding("queue_5s", Binding.DestinationType.QUEUE, "Dead_Letter_Exchange", "5s", null);
    }

    /**
     * 定义死信队列转发队列.
     */
    @Bean
    public Queue queue_delay_5s() {
        return new Queue("queue_delay_5s");
    }

    @Bean
    public Binding queue_delay_5sBinding() {
        return new Binding("queue_delay_5s", Binding.DestinationType.QUEUE, "Dead_Letter_Exchange", "dlx1", null);
    }


    /**
     * 声明一个死信队列.
     * 为队列设置过期时间
     */
    @Bean
    Queue queue_10s() {
        Map<String, Object> map = new HashMap<String, Object>(3);
        map.put("x-dead-letter-exchange", "Dead_Letter_Exchange");
        map.put("x-dead-letter-routing-key", "dlx2");
        map.put("x-message-ttl", 10000);
        return new Queue("queue_10s", true, false, false, map);
    }

    /**
     * 死信路由通过 10s 绑定键绑定到死信队列上.
     */
    @Bean
    public Binding queue_10sBinding() {
        return new Binding("queue_10s", Binding.DestinationType.QUEUE, "Dead_Letter_Exchange", "10s", null);
    }

    /**
     * 定义死信队列转发队列.
     */
    @Bean
    public Queue queue_delay_10s() {
        return new Queue("queue_delay_10s");
    }

    @Bean
    public Binding queue_delay_10sBinding() {
        return new Binding("queue_delay_10s", Binding.DestinationType.QUEUE, "Dead_Letter_Exchange", "dlx2", null);
    }

    /**
     * 声明一个死信队列.
     * 为队列设置过期时间
     */
    @Bean
    Queue queue_30s() {
        Map<String, Object> map = new HashMap<String, Object>(3);
        map.put("x-dead-letter-exchange", "Dead_Letter_Exchange");
        map.put("x-dead-letter-routing-key", "dlx3");
        map.put("x-message-ttl", 30000);
        return new Queue("queue_30s", true, false, false, map);
    }

    /**
     * 死信路由通过 30s 绑定键绑定到死信队列上.
     */
    @Bean
    public Binding queue_30sBinding() {
        return new Binding("queue_30s", Binding.DestinationType.QUEUE, "Dead_Letter_Exchange", "30s", null);
    }

    /**
     * 定义死信队列转发队列.
     */
    @Bean
    public Queue queue_delay_30s() {
        return new Queue("queue_delay_30s");
    }

    @Bean
    public Binding queue_delay_30sBinding() {
        return new Binding("queue_delay_30s", Binding.DestinationType.QUEUE, "Dead_Letter_Exchange", "dlx3", null);
    }


    /**
     * 声明一个死信队列.
     * 为队列设置过期时间
     */
    @Bean
    Queue queue_60s() {
        Map<String, Object> map = new HashMap<String, Object>(3);
        map.put("x-dead-letter-exchange", "Dead_Letter_Exchange");
        map.put("x-dead-letter-routing-key", "dlx4");
        map.put("x-message-ttl", 60000);
        return new Queue("queue_60s", true, false, false, map);
    }

    /**
     * 死信路由通过 30s 绑定键绑定到死信队列上.
     */
    @Bean
    public Binding queue_60sBinding() {
        return new Binding("queue_60s", Binding.DestinationType.QUEUE, "Dead_Letter_Exchange", "60s", null);
    }

    /**
     * 定义死信队列转发队列.
     */
    @Bean
    public Queue queue_delay_60s() {
        return new Queue("queue_delay_60s");
    }

    @Bean
    public Binding queue_delay_60sBinding() {
        return new Binding("queue_delay_60s", Binding.DestinationType.QUEUE, "Dead_Letter_Exchange", "dlx4", null);
    }

}

```

消费者：

```java
/**
 * author:zw
 */
@Component
public class RabbitMQConsumer {
    @RabbitListener(queues = "queue_delay_5s")
    public void delay_5s_lisener(Message msg, Channel channel) throws IOException {
        System.out.println("进入queue_delay_5s时间为" + new Date().toLocaleString());
        if (null != msg && null != msg.getBody() && 0 != msg.getBody().length) {
            channel.basicAck(msg.getMessageProperties().getDeliveryTag(), false);//消息确认
            System.out.println("消息确认");
        }
    }

    @RabbitListener(queues = "queue_delay_10s")
    public void delay_10s_lisener(Message msg, Channel channel) throws IOException {
        System.out.println("进入queue_delay_10s时间为" + new Date().toLocaleString());
        if (null != msg && null != msg.getBody() && 0 != msg.getBody().length) {
            channel.basicAck(msg.getMessageProperties().getDeliveryTag(), false);//消息确认
            System.out.println("消息确认");
        }
    }

    @RabbitListener(queues = "queue_delay_30s")
    public void delay_30s_lisener(Message msg, Channel channel) throws IOException {
        System.out.println("进入queue_delay_30s时间为" + new Date().toLocaleString());
        if (null != msg && null != msg.getBody() && 0 != msg.getBody().length) {
            channel.basicAck(msg.getMessageProperties().getDeliveryTag(), false);//消息确认
            System.out.println("消息确认");
        }
    }

    @RabbitListener(queues = "queue_delay_60s")
    public void delay_60s_lisener(Message msg, Channel channel) throws IOException {
        System.out.println("进入queue_delay_60s时间为" + new Date().toLocaleString());
        if (null != msg && null != msg.getBody() && 0 != msg.getBody().length) {
            channel.basicAck(msg.getMessageProperties().getDeliveryTag(), false);//消息确认
            System.out.println("消息确认");
        }
    }
}

```
发送端代码：

```java
/**
 * author:zw
 */
@Component
public class RabbitMQSender implements RabbitTemplate.ConfirmCallback, RabbitTemplate.ReturnCallback {

    private static final Logger logger = LoggerFactory.getLogger(RabbitMQSender.class);
    @Autowired
    RabbitTemplate rabbitTemplate;

    @PostConstruct
    public void init() {
        rabbitTemplate.setConfirmCallback(this);
        rabbitTemplate.setReturnCallback(this);
    }

    public void sendDirectMsg(Object msg) {
        CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());
        System.out.println("消息id" + correlationData.getId());
        Map<String, Object> map = (HashMap<String, Object>) msg;
        if (Integer.parseInt(map.get("age").toString()) == 5) {
            System.out.println("发送5s延时队列" + new Date().toLocaleString());
            rabbitTemplate.convertAndSend("Dead_Letter_Exchange", "5s", msg, correlationData);
        } else if (Integer.parseInt(map.get("age").toString()) == 10) {
            System.out.println("发送10s延时队列" + new Date().toLocaleString());
            rabbitTemplate.convertAndSend("Dead_Letter_Exchange", "10s", msg, correlationData);
        } else if (Integer.parseInt(map.get("age").toString()) == 30) {
            System.out.println("发送30s延时队列" + new Date().toLocaleString());
            rabbitTemplate.convertAndSend("Dead_Letter_Exchange", "30s", msg, correlationData);
        } else if (Integer.parseInt(map.get("age").toString()) == 60) {
            System.out.println("发送60s延时队列" + new Date().toLocaleString());
            rabbitTemplate.convertAndSend("Dead_Letter_Exchange", "60s", msg, correlationData);
        }
    }
}
```

#### RabbitMQ的常见问题

**1、解决消息的重复执行**

一个是消息的任务类型最好是支持幂等性的，这样的好处是 任务执行多少次都没关系，顶多消耗一些性能。但如果不支持幂等性，那么需要构建一个map来记录任务的执行情况！ 不仅仅是成功和失败，还要有心跳！！！  这个map在消费端实现就可以了

还有解决重复投递和重复消费的，在消息生产时，MQ内部针对每条生产者发送的消息生成一个inner-msg-id，作为去重的依据（消息投递失败并重传），避免重复的消息进入队列；在消息消费时，要求消息体中必须要有一个bizId（对于同一业务全局唯一，如支付ID、订单ID、帖子ID等）作为去重的依据，避免同一条消息被重复消费。

这里应该就是程序要实现幂等性，幂等数学上其实就是f(x)=f(f(x))，调用接口发生异常并且重复尝试时，总是会造成系统无法承受的损失，所以必须阻止这样的事情发生，比如电商平台进行支付后，因为网络原因导致系统提示你支付失败，于是又重新支付了一下，然后你发现扣了两次款，这是怎样的体验？

这种问题其实在电商，银行，互联网金融中要求很高，常见的解决思路是：

1、MVCC

多版本并发控制，乐观锁的一种实现，在更新数据的时候需要去比较持有数据的版本号，版本不一致则操作不成功，更新成功版本加1，如果喜好加1

```java
public boolean addCount(Long id, Long version);

update like set count = count + 1, version = version + 1 where id = 321 and version = 123
```

2、利用数据库去设计

就是建立一个唯一性索引，保证某一类数据一旦执行成功，后续的请求无法再完成

例如转账的ID和账户ID两个字段组成唯一索引，这样这些记录在这张表中至多只有一条记录

3、Token机制

这种机制就比较重要了，适用范围较广，有多种不同的实现方式。其核心思想是为每一次操作生成一个唯一性的凭证，也就是token。一个token在操作的每一个阶段只有一次执行权，一旦执行成功则保存执行结果。对重复的请求，返回同一个结果。

以电商平台为例子，电商平台上的订单id就是最适合的token。当用户下单时，会经历多个环节，比如生成订单，减库存，减优惠券等等。

每一个环节执行时都先检测一下该订单id是否已经执行过这一步骤，对未执行的请求，执行操作并缓存结果，而对已经执行过的id，则直接返回之前的执行结果，不做任何操作。这样可以在最大程度上避免操作的重复执行问题，缓存起来的执行结果也能用于事务的控制等

**2、解决消息的顺序执行**

如果对于消息顺序敏感，那么我们这里给出的方法是 消息体通过hash分派到队列里，**每个队列对应一个消费者**，多分拆队列。同一组的任务会被分配到同一个队列里，每个队列只能有一个worker来消费，这样避免了同一个队列多个消费者消费时，乱序的可能！主动去分配队列，单个消费者。就是FIFO的思想呗

这边源自一些资料

AMQP 0-9-1核心规范的4.7节解释了保证排序的条件：**在一个通道中发布的消息，通过一个交换机和一个队列以及一个传出通道的消息将以与发送它们相同的顺序接收。**自2.7.0版以来，RabbitMQ提供了更强大的保证。可以使用具有requeue参数（basic.recover，basic.reject和basic.nack）或由于保留未确认消息而关闭通道的AMQP方法将消息返回到队列。这些情况中的任何一种都会导致对于2.7.0之前的RabbitMQ版本，消息在队列的后面重新排队。从RabbitMQ版本2.7.0开始，即使在存在重新排队或关闭通道的情况下，消息也始终按发布顺序保留在队列中。（强调）

**3、kafka中的分组分区**

kafka还是跟传统JMS不太一样的，它的一个Topic可以对应多个Consumer Group。如果需要实现广播，只要每个Consumer有一个独立的Group就可以了。要实现单播只要所有的Consumer在同一个Group里。用Consumer Group还可以将Consumer进行自由的分组而不需要多次发送消息到不同的Topic。举个例子

首先创建一个Topic (名为topic1，包含3个Partition)，然后创建一个属于group1的Consumer实例，并创建三个属于group2的Consumer实例，最后通过Producer向topic1发送key分别为1，2，3的消息。结果发现属于group1的Consumer收到了所有的这三条消息，同时group2中的3个Consumer分别收到了key为1，2，3的消息。
