---
layout: post
title: kafka相关
author-id: "zqmalyssa"
tags: [java, kafka]
---
Kafka MQ相关介绍，使用和运维

#### 1.Kafka基本介绍

kafka使用scala语言编写

kafka 设计理念

高吞吐、低延迟：kafka将消息写入操作系统页缓存而不直接操作磁盘io， 每秒可以处理几十万条消息，最低延迟几毫秒。

高伸缩性： 每个主题(topic) 包含多个分区(partition)，主题中的分区可以分布在不同的主机(broker)中。

负载均衡：kafka 使用 leader选举算法实现了负载均衡，极大提高了运行效率；

故障转移：kafka使用zookeeper 作为会话机制，搭建集群后kafka的容错机制极大提高，允许某个节点宕机而不影响其它节点。


使用场景

行为跟踪：Kafka 可以用来跟踪用户行为，比如我们经常回去京东购物，当你浏览购物的时候，你的浏览信息，你的搜索指数，你的购物爱好都会作为一个个消息传递给 Kafka ，这样就可以生成报告，可以做智能推荐，购买喜好等。

传递消息：Kafka 一个基本用途是传递消息，可以当作消息总线或者消息代理；

日志收集：企业的应用非常多，每个应用都会产生日志，kafka可以当作日志收集汇总方案，方便日志管理!

流式处理：流式处理是kafka Stream，kafka可以将接收的数据流提供给其它框架进行计算处理等！

限流削峰：Kafka 多用于互联网领域某一时刻请求特别多的情况下，可以把请求写入Kafka 中，避免直接请求后端程序导致服务崩溃。


kafka基本术语

消息：Kafka 中的数据单元被称为消息，也被称为记录；消息中有三要素，key为消息键，用作分区使用；value，消息体用于保存消息；timestamp，时间戳，表示消息发送的时间；

主题：消息的种类称为 主题（Topic）,一个主题代表了一类消息，经常使用topic来区分不同的业务，每个业务一个topic；

分区（Partition）：主题可以被分为若干个分区（partition），kafka 是 一个 topic-partition-message 的三级结构；分区存在的意义就是提高kafka的吞吐量；topic物理上的分组，每个partition都是一个顺序的、不可变的消息队列，且可以持续添加。Partition中的每条消息都会被分配一个有序的序列号，称为偏移量（offset），因此每个分区中偏移量都是唯一的。

副本：副本的目的是实现消息的可靠性，将f分区的数据复制一份到replica, 即防止消息丢失！Kafka 定义了两类副本：领导者副本（Leader Replica） 和 追随者副本（Follower Replica），前者对外提供服务，后者只是被动跟随Leader,相当于候补队员；

broker: 一个独立的 Kafka 服务器就被称为 broker；

ISR： Kafka在Zookeeper上针对每个Topic都维护了一个ISR（in-sync replica---已同步的副本）的集合，集合的增减Kafka都会更新该记录。如果某分区的Leader不可用，Kafka就从ISR集合中选择一个副本作为新的Leader。这样就可以容忍的失败数比较高，假如某Topic有N+1个副本，则可以容忍N个服务器不可用。

Consumer Group：每个Consumer属于一个特定的Consumer Group，这是kafka用来实现一个Topic消息的广播【发送给所有的consumer的发布订阅式消息模型】和单播【发送给任意一个consumer队列消息模型】的手段。一个topic可以有多个consumer group。

如果要实现广播，只要每个consumer有独立的consumer group就可以，此时就是发布订阅模型。
如果要实现单播，只要所有的consumer在同一个consumer group中就可以，此时就是队列模型。


kafka的序列化和反序列化

byte[]: org.apache.kafka.common.serialization.ByteArraySerializer
Integer: org.apache.kafka.common.serialization.IntegerSerializer
String: org.apache.kafka.common.serialization.StringSerializer

kafka的生产者和消费者

producer的参数

ACKS_CONFIG：应答，0标识producer不会确认消息发送broker是否成功，1表示leader接受消息确认，将消息写入本地日志，-1是all，需要leader和follower共同确认，从0到-1，吞吐量越来越差，持久化越来越好

consumer的参数

AUTO_OFFSET_RESET_CONFIG：有三个取值，earliest从最早开始，latest从最新开始，none新得消费者加入以后，由于之前不存在offset，则会直接抛出异常


consumer被归类到consumer group底下，每个group底下可能有多个consumer，由此引申出2个模型

队列模型（一对一）

点对点模型通常是一个基于拉取或者轮询的消息传送模型，这种模型从队列中请求信
息，而不是将消息推送到客户端。这个模型的特点是发送到队列的消息被一个且只有一个接
收者接收处理，即使有多个消息监听者也是如此。


发布订阅模型

发布订阅模型则是一个基于推送的消息传送模型。发布订阅模型可以有多种不同的订阅
者，临时订阅者只在主动监听主题时才接收消息，而持久订阅者则监听主题的所有消息，即
使当前订阅者不可用，处于离线状态。


#### 2.Kafka简单示例

项目中加个依赖

```java
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>2.8.0</version>
</dependency>
```

消费者
```java

public static void main(String[] args) {

    try {
      Properties properties = new Properties();
      properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getCanonicalName());
      properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getCanonicalName());
      properties.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "latest");
      Consumer<String, String> consumer =
          KafkaClientFactory.newConsumer("xxx.xxx.data.test", "xxx.xxx.test.notify", properties);

      while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000L));
        for (ConsumerRecord<String, String> record : records) {
          System.out.println(record);
        }
      }
    } catch (IOException e) {
      e.printStackTrace();
    }

  }

```
生产者

```java

public static void main(String[] args) throws Exception {

    Properties properties = new Properties();
    properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getCanonicalName());
    properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getCanonicalName());
    properties.put(ProducerConfig.ACKS_CONFIG, "0");
    properties.put(ProducerConfig.LINGER_MS_CONFIG, "200");
    properties.put(ProducerConfig.MAX_BLOCK_MS_CONFIG, "30000");
    // 设置超时响应时间
    properties.put(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG, "15000");
    // 压缩算法，GZIP,LZ4,Snappy
//    properties.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");
    // 重试次数
//    properties.put(ProducerConfig.RETRIES_CONFIG, "3");

    Producer<String, String> producer = KafkaClientFactory.newProducer("xxx.xxx.data.test", properties);
    for (int i = 0; i < 10; ++i) {

      producer.send(new ProducerRecord<>("xxx.xxx.data.test", null, "Hello World"));

    }

    // 这段就能接受消息了，异步的过程，主线程不能立即退出
    Thread.currentThread().join();

    // 跟close无关
    producer.close();

  }

```
注意:初次运行时一定要先运行消费者后运行生产者，因为初次运行时还没有声明队列和交换机的绑定关系(在消费者代码里)，如果先启动生产者会导致消息无法被正确路由而被丢弃

可以去rabbitmq的web界面上查看信息，比如连接信息，交换机信息，队列信息

#### 3.Spring整合Kafka



#### Kafka的常见问题

**1、kafka是如何确保消息可靠性**

消息的可靠性性一般需要从三个维度进行考量。分别是生产端、服务端、消费端。

生产端：生产者需要确保消费发送到了服务端机器上。

producer.send(msg)：俗称“发后即忘”，不管消息有没有成功写入到broker端，生产者都不会收到任何通知，那么到消息因为某些原因（如：网络异常，消息体过大等）并未被broker接受时，就产生了消息丢失。这种方式虽然吞吐量高，但无法保证消息的可靠性。

producer.send(msg).get()：俗称“同步发送”，以同步方式发送消息，每条消息发送的返回结果进行判断，是同步阻塞方式，只有返回了才能继续下一条消息发送。

producer.send(msg，callback)：带有回调函数，通过callback的回调来处理broker端的响应结果，如果未成功发送，那么就可以做响应的处理工作，如进行重试，记录日志等。在调用send方式发送消息时，指定一个回调函数，服务器在返回响应时会调用回调函数，通过回调能够对异常情况进行处理，只有回调函数执行完毕，生产者才会结束，否则一直阻塞。

在kafka中，对于某些异常，生产者捕获到异常，会进行异常重试，重试的次数是由retries参数来控制的，因此为了保证消费的可靠性，还需要将这个参数的值设置为大于0的值，一般可设置为3～5。

同时对于生产者而言，还有一个重要的参数需要设置，就是acks的值，acks可以设置为 1，0，-1三个值，每个值的含义如下

properties.put(“request.required.acks”, “1”);

acks=0，Kafka Producer只要把消息发送出去，不管那条数据有没有是否落到Partition Leader磁盘上，只要消息发出去就认为这个消息发送成功了。

acks=1，只要Partition Leader接收到消息而且写入本地磁盘了，就认为成功了，不管他其他的Follower有没有同步过去这条消息了。

acks=all/-1，意思就是说Partition Leader接收到消息之后，还必须要求ISR列表里跟Leader保持同步的那些Follower都要把消息同步过去，才能认为这条消息是写入成功了。

因此为了保证消息的可靠性，需要将acks参数设置为-1，这样可以避免leader结点宕机后，follower结点没有及时同步到消息，而产生的数据丢失。


如果业务要求消息必须是按顺序发送的，那么可以使用同步发送方式，并且只能在一个partation上，结合参数设置retries的值让发送失败时重试，设置max_in_flight_requests_per_connection=1，可以控制生产者在收到服务器晌应之前只能发送1个消息，从而控制消息顺序发送。

如果业务只关心消息的吞吐量，容许少量消息发送失败，也不关注消息的发送顺序，那么可以使用发送并忘记方式，并配合参数acks=0，这样生产者不需要等待服务器的响应，以网络能支持的最大速度发送消息。

如果业务需要知道消息发送是否成功，并且对消息的顺序不关心，那么可以异步发送+回调函数方式来发送消息，并将发送失败的消息记录到日志文件中。

broker端：服务端需要保证消息的持久化不丢失。

在kafka中，每条消费都会被存储到磁盘上进行持久化存储，即使broker因为异常进行重启，也不会消息丢失，并且在生产环境，kafka都是以集群的方式进行部署，同时因为kafka的分区和副本的特性，一般可以保证broker端的消息不丢失的情况，但是也有一些特殊情况下存在消息丢失的可能。

broker端的参数设置不合理：对于每个消费分区而言，副本数replication.factor >= 3，消息进行多余的冗余备份，可以防止因为broker端异常，导致的消息丢失。


消费端：消费端需要确认每条消息都被成功进行消费。

在kafka中，每一条消息就是自己offset偏移量，消费者每次消费完消息后，都会提交自己消费的位移，如下图所示，消费A消费到offset = 9的数据，消费者B消费到offset = 11的数据。consumer会提交自己的消费位移，用来知道自己消费的位置，如果消费位移提交不当，也会产生消息没有消费的情况。

对于consumer端来说，有两种提交消费位移的方式，分别是自动提交和手动提交。

手动提交，使用commitSync() 和 commitASync()API来进行手动提交，手动提交，可以让我们根据自己的实际消费情况来设置什么时间点进行提交位移，将位移提交交给用户自己，合理设置位移提交点可以保证消费的消费不丢失。


`消费不丢失参数配置`

1、在producer端使用，不要使用producer.send(msg)的API，要使用producer.seng(msg，callback)带有回调方法的API，来进行消息发送，可已经消息确认。

2、producer端设置acks=all，表示消息全部提交到ISR集合中的全部分区，才算消息提交成功。

3、producer端设置retries > 0，此参数，表示当生产者发送出现异常（如：broker出现网络抖动，导致超时）producer端进行重试的次数。

4、broker端，unclean.leader.election.enable = false，表示不允许，非ISR集合中的分区，进行leader选举，因为如果一个follower分区，消息落后于leader分区太远，当这个follower成为leader分区后，就会存在消息丢失。

5、broker端，replication.factor >= 3，表示副本的数量，消息进行多余的冗余备份，可以防止因为broker端异常，导致的消息丢失。

6、broker端，min.insync.replicas > 1，这个参数用来控制消息需要最小写入的副本数。Broker端参数，控制消息至少要被写入到多少个副本才算是“已提交”。设置成大于 1 可以提升消息持久性。在生产环境中不要使用默认值 1。确保replication.factor > min.insync.replicas。如果两者相等，那么只要有一个副本离线，整个分区就无法正常工作了。推荐设置成replication.factor = min.insync.replicas + 1。

7、consumer端，将自动提交改为手动提交，确认消息消费完成后，再进行提交。Consumer端有个参数enable.auto.commit，最好设置成false，并自己来处理offset的提交更新。


再另外补充一下：

任何消息组件不丢数据都是在特定场景下一定条件的，kafka要保证消息不丢，有两个核心条件。

第一，必须是已提交的消息，即committed message。kafka对于committed message的定义是，生产者提交消息到broker，并等到多个broker确认并返回给生产者已提交的确认信息。而这多个broker是由我们自己来定义的，可以选择只要有一个broker成功保存该消息就算是已提交，也可以是令所有broker都成功保存该消息才算是已提交。不论哪种情况，kafka只对已提交的消息做持久化保证。（结合上面理解）

第二，也就是最基本的条件，虽然kafka集群是分布式的，但也必须保证有足够broker正常工作，才能对消息做持久化做保证。也就是说 kafka不丢消息是有前提条件的，假如你的消息保存在 N 个kafka broker上，那么这个前提条件就是这 N 个broker中至少有 1 个存活。只要这个条件成立，kafka就能保证你的这条消息永远不会丢失。（broker要有一个存活）

在发送端：

处理发送失败的责任在Producer端而非Broker端。当然，如果此时broker宕机，那就另当别论，需要及时处理broker异常问题。

在消费端：

kafka通过先消费消息，后更新offset，来保证消息不丢失。但是这样可能会出现消息重复的情况，具体如何保证only-once，后续再单独分享。

当我们consumer端开启多线程异步去消费时，情况又会变得复杂一些。此时consumer自动地向前更新offset，假如其中某个线程运行失败了，它负责的消息没有被成功处理，但位移已经被更新了，因此这条消息对于consumer而言实际上是丢失了。这里的关键就在自动提交offset，如何真正地确认消息是否真的被消费，再进行更新offset。
这个问题的解决起来也简单：如果是多线程异步处理消费消息，consumer不要开启自动提交offset，consumer端程序自己来处理offset的提交更新。提醒你一下，单个consumer程序使用多线程来消费消息说起来容易，写成代码还是有点麻烦的，因为你很难正确地处理offset的更新，也就是说避免无消费消息丢失很简单，但极易出现消息被消费了多次的情况。
