---
layout: post
title: Http相关
tags: [http]
author-id: zqmalyssa
---

Jpa及Spring-data-jpa，及hibernate，mybatis，因为与数据库有比较大的联系

#### spring-data-jpa

spring-data-jpa的后面是hibernate，但是相比于hibernate的各种配置，做了不少简化

一些注意点

（1）可以通过自定义的 JPQL 完成 UPDATE 和 DELETE 操作。 注意： JPQL 不支持使用 INSERT；
（2）在 @Query 注解中编写 JPQL 语句， 但必须使用 @Modifying 进行修饰. 以通知 SpringData， 这是一个 UPDATE 或 DELETE 操作
（3）UPDATE 或 DELETE 操作需要使用事务，此时需要定义 Service 层，在 Service 层的方法上添加事务操作（必须的）
（4）默认情况下， SpringData 的每个方法上有事务， 但都是一个只读事务。 他们不能完成修改操作。

对第二点的一些细节性补充

```html

a.通过使用 @Query 来执行一个更新操作，为此，我们需要在使用 @Query 的同时，用 @Modifying 来将该操作标识为修改查询，这样框架最终会生成一个更新的操作，而非查询（这是关键）。

b.而Transactional可以是spring的也可以是java的，javax.transaction.Transactional或者org.springframework.transaction.annotation.Transactional

c.注意：在执行springDataJpa中使用jpql完成更新，删除操作时，需要手动添加事务的支持 必须的；因为默认会执行结束后，回滚事务。必须的意思是Service层 或者 Dao层在执行update或者delete的时候必须有个事务注解@Transactional
否则会报 Executing an update/delete query; nested exception is javax.persistence.TransactionRequiredException: Executing an update/delete query，b中的两种Transactional都可以

d.如果不加Modifying，update和delete用@Query的写法都会报 Not supported for DML operations，但是！！如果不用@Query，不用@Modifying，直接写个int deleteByKey(String key); 内置的语法，是可以的，当然要有Transactional注解，没有Transactional的报错不一样，是org.springframework.dao.InvalidDataAccessApiUsageException: No EntityManager with actual transaction available for current thread - cannot reliably process 'remove' call; nested exception is javax.persistence.TransactionRequiredException: No EntityManager with actual transaction available for current thread - cannot reliably process 'remove' call
，但这个报错在没有对应数据的时候是不会出现的。。

```


#### 异常处理

1、乐观锁异常处理

org.springframework.orm.ObjectOptimisticLockingFailureException

这种异常一般是由于，数据库里的某一条记录的某一个版本对应的信息，同时被两个事务读取到并试图进行写操作（包括更新和删除）；这种情况下第一个写成功的事务不会有影响，第二个事务再对同一版本的同一记录进行写操作时，抛出乐观锁异常（这是我觉得比较合适的说法了），注意哦，只有更新和删除的时候会有可能发生，创建不至于，创建会有什么？DataIntegrityViolationException异常，Key的冲突问题（但是如果用save即去创建又更新，两种错误就都要注意）


同悲观锁一样，乐观锁一样也是为了一定程度上解决并发问题。
乐观锁的实现是CAS机制（compare and swap），但需注意ABA问题。
在数据库上，乐观锁的实现一般是采用类似的版本号机制，相比悲观锁，它开销小，效率高，适用于冲突不多的多读场景；即保持最乐观的态度，赌你不会遇到多读少写下很小几率的并发问题。
如果运气不好遇到了并发问题，乐观锁的处理是只会抛出一个这样的异常，进一步处理的权力交由程序员。
乐观锁版本号机制示例如下：

```html

update process_bill set status = new_status, version = version + 1 where bill_no = "B12345" and version = 2;

```


```html
处理方法：

1、对此异常添加重试机制
2、@Query原生写操作，避免JPA的@Version乐观锁写操作的机制
3、写操作使用MySQL行锁的悲观锁（for update），在数据库层面最大程度避免了并发问题，但数据库锁时间长，开销大，更应该在应用层做处理
4、必要时在应用层对关键对象，使用zookeeper等分布式锁

```

这边两个系统，一个全量同步，一个消息，同步，会出现上面的乐观锁异常，另外可能还有unique异常，基本都是因为

系统用的JPA，底层还是Hibernate
