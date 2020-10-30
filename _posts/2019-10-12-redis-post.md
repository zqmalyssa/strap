---
layout: post
title: "Redis相关内容"
author-id: "zqmalyssa"
tags: [code, operations]
---

Redis作为比较常用的中间件，可以有装机版，也可以上云，包括使用及运维，需要整理一下

### Redis的基本介绍

Redis除了能存储普通的字符串键之外，还可以存储其他的4种数据结构，而memcached只能存储普通的字符串键，redis可以当主数据库(primary database)，也可以当辅助数据库(auxiliary database)

Redis可以存储的数据结构有String，List，SET(集合)，HASH，ZSET(有序集合)

String：可以是字符串，整数和浮点数
List： 一个链表，链表上的每个节点都包含了一个字符串
SET：包含字符串的无序收集器，并且被包含的每个字符串都是
HASH：包含键值对的无序散列表
ZSET：字符串成员(member)与浮点数分值(score)之间的有序映射，元素的排列顺序由分值的大小决定

整个数据结构的贯通使用可以看Action中的文章投票的例子，重点看下redis和Web的整合

1、登录和cookie缓存

对于用来登录的cookie，有两种常见的方法可以将登录信息存储在cookie里面，一种是签名cookie，一种是令牌cookie，签名通常会存储用户名，可能还有Id，最后一次成功登录时间，以及网站觉得有用的其他任何信息，签名cookie还包含一个签名，服务器可以使用它来验证浏览器发送的信息是否未经改动(比如登录姓名换成另一个用户)，令牌cookie里面存储一串随机字节令牌，服务器可以根据令牌在数据库中查找令牌的拥有者。随着时间的推移，旧令牌会被新令牌取代

除了用户登录信息外，也可以将用户的访问时长和已浏览的商品的数量等信息存储到数据库里面，这样便于通过分析这些信息来学习如何更好的向用户推销商品，这些信息如果用关系型数据库来存，每台数据库服务器每秒也只能插入，删除或者更新200-2000个数据行，500万的用户，平均情况下每秒约1200次的写入，高峰时期每秒近6000次写入，所以可能需要10台关系型数据库才能应对高峰时期的负载量，用redis取代关系型数据库实现登录功能呢？

用一个hash来存储登录cookie令牌与已登录用户之间的映射

2、redis购物车

cookie可以实现购物车，这样就不需要写入数据库了，但是缺点则是程序需要重新解析与验证cookie，确保cookie的格式正确，因为浏览器每次请求都会连cookie发送，所以cookie体积大的时候，处理速度会下降，我们用redis

每个用户的购物车都是一个hash，这个散列存储了商品ID和商品订购数量之间的映射

3、网页缓存

在动态生成网页的时候，通常会使用模板语言(templating language)来简化网页的生成操作，实际上多数页面实际上并不会经常发生大的变化，对于可以被缓存的请求，首先从缓存中拿取页面，有就直接拿取，没有就生成页面，并放在redis中缓存5分钟

4、数据行缓存

促销活动的时候，页面就不能被缓存了，商品的数量得从数据库里面拿出来，为了应对促销活动带来的大量负载，需要对数据行进行缓存，具体做法是编写一个持续运行的守护进程，让这个函数将指定的数据行缓存到redis里面，并不定期对这些缓存进行更新，缓存函数会将数据行编码为(encode)为json字典并存储在Redis的字符串里面，其中数据列的名字会被映射为json字典的键，而数据行的值会被映射为json字典的值

然后，用两个有序集合来记录应该在何时对缓存进行更新，第一个有序集合为调度(Schedule)有序集合，它的成员为数据行的行ID，而分值是一个时间戳，这个时间戳记录了应该在何时将指定的数据行缓存到redis里面，第二个有序集合为延时(delay)有序结合，它的成员也是数据行的行ID，而分值记录了指定数据行的缓存需要每隔多少秒更新一次(Redis是没有嵌套结构的，就是Hash里面能包含列表或有序集合如果需要，模拟)，为了让缓存函数定期的缓存数据行，程序首先需要将行ID和给定的延迟值添加到延迟有序集合里，然后将行ID和当前时间的时间戳添加到有序集合里面，实际执行缓存操作的函数需要用到数据行的延时值，如果某个数据行的延时值不存在，那么程序将取消对这个数据行的调度。如果我们想要移除某个数据行已有的缓存，并且让缓存函数不在缓存那个数据行，那么只需要把那个数据行的延迟值设置为小于或等于0就可以了。

如何进行数据缓存呢，负责缓存数据行的函数会尝试读取调度有序结合的第一元素和该元素的分值，如果调度有序集合没有包含任何元素，或者分值存储的时间戳所指定的时间尚未来临，那么函数会先休眠50ms，然后再重新进行检查，当缓存函数发现一个需要立即更新的数据行时，缓存函数会检查这个数据行的延迟值，如果数据行的延迟值小于或等于0，那么缓存函数会从延迟有序集合和调度有序集合里面移除这个数据行的ID，并删除缓存数据，然后再重新进行检查，对于延迟大于0的数据行来说，缓存函数会从数据库里面取出这些行，将他们编码成JSON格式并存储到redis里面，然后更新这些行的调度时间。

5、网页分析

网页可以从用户的访问，交互，购买行为中收集到有价值的信息，比如我们只想关注那些浏览量最高的页面，那么我们可以尝试修改页面的格局，配色等信息，通过记录商品的浏览次数并定期对记录浏览次数的有序几个进行修剪和分值调整，我们可以建立一个持续更新的最常浏览商品排行榜

对于redis中的键，除了可以手动删除外，也可以设置过期时间(expiration)来让一个键在给定的时限之后自动被删除，但是只能跟键相关，无法控制键中的元素，键中的元素使用了存储时间戳的有序集合来实现针对单个元素的过期操作

| 命令 | 说明 |
| :----: | :----:  |
| PERSIST | 移除键的过期时间 |
| TTL | 查看键距离过期还有多少秒 |
| EXPIRE | 让给定键在指定秒数之后过期 |
| EXPIREAT | 将给定的键的过期时间设置为给定的UNIX时间戳 |
| PTTL | 查看给性键距离过期还有多少毫秒 |
| PEXPIRE | 让给定键在指定毫米数之后过期 |
| PEXPIREAT | 将一个毫米级精度的UNIX时间戳设置为给定键的过期时间 |

#### Redis的事务

Redis中是有事务的，redis没有在两个不同类型之间移动元素的命令，为了对相同和不同类型的多个键执行操作，redis有5个命令可以让用户在不被打断的情况下对多个键执行操作，它们分别是WATCH，MULTI，EXEC，UNWATCH和DISCARD

MULTI和EXEC配合用时，如果执行了MULTI，redis会将这个client其他的命令全部放到队列中，直到执行了EXEC，然后reids就会在不被打断的情况下，一个接一个执行存储在队列里面的命令。

**在多个事务同时处理同一个对象时通常需要用到二阶段提交(two-phase commit)**，所以如果事务不能以一致的形式读取数据，那么二阶段提交将无法实现，从而导致一些原本可以成功执行的事务沦落至执行失败的地步。

很多redis的客户端会等到事务包含的命令都出现了再一次性的将MULTI命令，事务中执行的一系列命令，已经EXEC命令全部发送给Redis，然后等待直到接收到所有命令的回复为止，"这种一次性发送多个命令，然后等待所有回复出现"的做法通常被称为流水线(pipelining)，它可以通过减少客户端与Redis服务器之间的网络通信次数来提升Redis在执行多个命令时的性能

需要用例子来说明redis的事务问题，假设一个游戏里有用户信息和用户包裹，用户信息存储在一个散列里面，用户包裹用一个集合来存

```html
hash-user-1:
name: Frank
funds: 42

hash-user-2:
name: Bill
funds: 125

set-inventory-1:
ItemL
ItemM
ItemN

set-inventory-2:
ItemQ
ItemP
ItemO
```

商品买卖市场的需求非常简单，一个用户(卖家)可以将自己的商品按照给定的价格放到市场上销售，当另一个用户(买家)来购买商品的时候，卖家就会收到钱，为了将被销售商品的全部信息都存储到市场里面，我们将商品ID和卖家ID拼接起来，并将拼接结果用作陈冠存储到市场有序集合中，而商品的售价则成为分值

```html
zset-market:
ItemA.4  35
ItemC.7  48
ItemE.2  60
ItemG.3  73
```

根据score已经可以将商品按照价格排序了，为了将商品放到市场上销售，除了使用MULTI和EXEC命令外，还需要配合使用WATCH命令，有时候甚至还需要UNWATCH或DISCARD命令。**在用户使用WATCH命令对键进行监视之后，直到用户执行EXEC命令的这段时间里面，如果有其他客户端抢先对任何被监视的键进行了替换，更新或者删除等操作，那么当用户尝试执行EXEC命令的时候，事务将失败并返回一个错误(之后用户可以选择重试事务或者放弃事务)**，通过事务的操作，程序可以在执行某些重要操作的时候，通过确保自己正在使用的数据没有发生变化来避免数据出错

UNWATCH命令可以在WATCH命令执行之后，MULTI命令执行之前对连接进行重置(reset)，同样的，DISCARD命令也可以在MULTI命令执行之后，EXEC命令执行之前对连接进行重置，也就是说，用户在使用WATCH监视一个或多个键，接着使用MULTI开始一个新的事务，并将多个命令入队到事务队列之后，**仍然可以通过发送DISCARD命令来取消WATCH命令并清空所有已入队命令**

这里，在放入销售市场的动作中，需要对用户包裹进行监视(WATCH)，以免被销售的物品都不在了，如果商品都不在了，停止对包裹的监控并返回一个空，否则就正常执行，添加商品并从卖家的包裹中移除该商品。

买家这边就要监控(WATCH)市场和买家的包裹了，分别是商品是否在和钱是否够

为什么redis没有实现典型的加锁的功能？在访问以写入为目的的数据的时候(SQL中的select for update)，关系数据库会对被访问的数据进行加锁，直到事务被提交(commit)或者被回滚(rollback)，就是select会hang在那边，如果有其他客户端试图对被加锁的数据行进行写入，那么客户端会被阻塞，直到第一个事务执行完毕为止，加锁实际中非常有效，但是如果持有锁的客户端运行太慢，会影响等待的客户端。而redis为了解决这个，在WATCH时并不会对数据进行加锁，相反的，redis只会在数据已经被其他客户端抢先修改了的情况下，通知执行了WATCH命令的客户端，这种做法就是乐观锁，而关系型数据库的锁当然就是悲观锁，乐观锁在实际中非常有效，因为客户端不必花时间去等待第一个取得锁的客户端，它们只需要在自己的事务执行失败时进行重试就行了



### Redis的数据安全和性能保障

1、数据持久化

两种，一种是RDB(快照)，一种是AOF(只追加文件)，这两种持久化方法即可以同时使用也可以单独使用，

```html
//看下配置

//快照
sava 60 1000
stop-writes-on-bgsave-error no
rdbcompression yes
dbfilename dump.rdb

//AOF
appendonly no
appendsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

//都有的
dir ./
```

快照：它可以将存在于某一时刻的所有数据都写入硬盘里面，在创建快照后用户可以对快照进行备份，快照被写入dbfilename选项指定的文件里面，并存储在dir指定的路径上

创建快照的几个方法：

1、客户端可以向redis发送bgsave命令来创建一个快照，redis会调用fork来创建一个子进程，然后子进程负责将快照写入硬盘，父进程继续处理命令请求
2、发送save命令，在快照创建结束前不会响应其他命令
3、配置了save，那么意思是60秒内有10000次写入这个条件被满足，redis就会自动触发bgsave
4、当redis通过shutdown命令接收到关闭服务器请求时，会执行一个save
5、当一个redis服务器连接另一个redis服务器，并向对方发送sync命令开始一次复制操作时，如果主服务器目前没有在执行bgsave，那么主服务器就会执行bgsave

快照持久化适合那些即使丢失一部分数据也不会造成问题的应用程序，自己的开发环境

AOF：它会在执行写命令时，将被执行的写命令复制到硬盘里面，说AOF好，那为啥不所有持久化都选它呢，因为Redis会不断的将被执行的写命令记录到AOF文件里面，所以随着Redis不断的运行，AOF文件的体积也会不断增长，甚至能用光所有磁盘，还有因为redis重启后需要执行AOF文件记录的所有写命令，那么还原时间将非常长

BGREWRITEAOF命令会通过移除AOF文件中的冗余命令来重写AOF文件，使其体积尽可能小，但是它也是开了一个子进程auto-aof-rewrite-percentage 100和auto-aof-rewrite-min-size 64m都是用于配置BGREWRITEAOF的

这边还要提下复制，也就是主从复制，其实是使用了上面说到的快照，从服务器的数据会被替换成主服务器的数据。还有，redis是不支持主主复制的，因为redis允许用户在服务器启动之后使用slaveof命令来设置从服务器选项，但不要以为就能主主复制了

Redis的主从服务器并没有什么特别的地方，从服务器也可以拥有自己的从服务器，形成主从链。

### Redis分布式锁

一般来说，对数据进行加锁时，程序首先需要通过获取acquire锁来得到对数据进行排他性访问的能力，然后才能对数据执行一系列操作，最后还要释放锁(release)给其他程序，对于能够被多个线程访问的共享内存数据结构(shared-memory data structure)，这种先获取锁，然后执行操作，最后释放锁的动作非常常见，Redis使用WATCH命令来代替对数据进行加锁，因为WATCH只会在数据被其他客户端抢先修改了的情况下通知执行了这个命令的客户端，而不会阻止其他客户端对数据进行修改，所以这个命令被称为乐观锁(optimistic locking)

分布式锁也有类似"首先获取锁，然后执行操作，最后释放锁"的动作，**但这种锁即不是给同一个进程中的多个线程使用，也不是给同一台机器上的多个进程使用。而是由不同机器上的不同Redis客户端进行获取和释放**。何时使用以及是否使用WATCH或者锁取决于给定的应用程序，有的应用不需要使用锁就可以正确运行，而又的应用只需要使用少量的锁，还有的应用需要在每个步骤都使用锁。

为了对Redis存储的数据进行排他性访问，客户端需要访问一个锁，这个锁必须定义在一个可以让所有客户端都看得见的范围内，而这个范围就是Redis本身，因此我们需要把锁构建在redis里面，另一方面，虽然redis提供setex命令确实具有基本的加锁功能，但它的功能并不完整，并且也不具备分布式锁常见的一些高级特性，所以我们还是需要自己动手来构建分布式锁

例子用上面事务的游戏包裹案例来说，在卖商品时，需要用watch监控包裹是不是还在

```python

pipe.watch(inv)
if not pipe.sismember(inv, itemid)
 pipe.unwatch()
 return None

pipe.multi()
pipe.zadd("market", item, price)
pipe.srem(inv, itemid)
pipe.execute()
return true

```

现在来回顾下购物流程，当玩家在市场上购买商品的时候，程序首先需要使用WATCH去监视市场以及买家的个人信息散列，在得知买家现有的钱数以及商品的售价之后，程序会验证买家是否有足够的钱来购买指定的商品，如果买家有足够的钱，那么程序会将买家支付的钱转移给卖家，接着将商品添加到买家的包裹里面，并从市场里面移除已被售出的商品，相反的，如果买家没有足够的钱来购买商品，那么程序就会取消事务。如果在执行购买过程中，有其他玩家对市场进行了改动，或者因为记录买家个人信息的散列出现了变化而引发了WATCH错误，那么程序将重新执行购买操作

```python

pipe.watch("market", buyer) //监控市场以及买家个人信息变化
//检查商品是否已经售出，商品的价格是否已经发生了变化，以及买家是否有足够多的钱来购买这件商品
price=pipe.zscore("market", item)
funds=int(pipe.hget(buyer, 'funds'))
if price != lprice or price > funds:
  pipe.unwatch()
  return None

pipe.multi()
pipe.hincrby(seller, 'funds', int(price))
pipe.hincrby(buyerid, 'funds', int(-price))
pipe.sadd(inventory, itemid)
pipe.zrem("market", item)
pipe.execute()
return true

```
随着负载增加，完成一次交易所需的重试次数和购买等待时间都逐渐增加。但是为了保证数据正确性，需要这样，使用WATCH命令的做法并不完美，为了解决这个问题，并以可扩展的方式来处理市场交易，我们使用锁来保证市场在任一时刻只能上架或者销售一件商品

接下来先介绍一个简易锁，一步步来，最后设计成一个好的锁

使用redis构建一个基本上正确的锁非常简单，获取锁，setnx命令天生就适合用来实现锁的获取功能，这个命令只会在键不存在的情况下为键设置值，而锁要做的就是将一个随机生成的128位UUID设置为键的值，并使用这个值来防止锁被其他进程取得。如果程序在尝试获取锁的时候失败，那么它将不断的进行重试，直到成功的取得锁或者超过给定的时限为止

```python
def acquire_lock(conn, lockname, acquire_timeout=10):
  identifier = str(uuid.uuid4())   //128位随机标识符
  end = time.time() + acquire_timeout
  while time.time() < end:
    if conn.setnx('lock:' + lockname, identifier):  //尝试取得锁
      return identifier
    time.sleep(.001)
  return false;  

```

上面的函数就是用setnx命令，尝试在代表锁的键不存在的情况下，为键设置一个值，以此来获得锁，在获取锁失败的时候，函数在给定的时限内进行重试，直到成功获取锁或者超过给定的时限为止，默认的重试时间为10秒，在实现了锁之后，就可以用锁来代替WATCH操作了，程序首先对市场进行加锁，接着检查商品的价格，并在确保买家有足够的钱来购买商品之后，对钱和商品进行相应的转移，当操作执行完毕后，程序就会释放锁

```python
locked = acquire_lock(conn, market)
  return false
pipe = conn.pipeline(True)
try:
  pipe.zscore("market:", item)
  pipe.hget(buyer, 'fund')
  price, funds = pipe.execute()
  if price is None or price > funds:
    return None
  pipe.multi()
  pipe.hincrby(seller, 'funds', int(price))
  pipe.hincrby(buyerid, 'funds', int(-price))
  pipe.sadd(inventory, itemid)
  pipe.zrem("market", item)
  pipe.execute()
  return True
finally:
  release_lock(conn, market, locked)
```

在执行整个操作中都必须持有锁，还有就是程序在持有锁期间，其他客户端可能会擅自对锁进行修改，所以锁的释放操作需要和加锁操作一样小心谨慎的进行。函数首先使用WATCH命令监视代表锁的键，接着检查键目前的值是否和加锁时设置的值相同，并在确认值没有不变化后删除该键

```python
def release_lock(conn, lockname, identifier)
  pipe = conn.pipeline(True)
  lockname = 'lock.' + lockname
  while True:
    try:
      pipe.watch(lockname)   //检查进程是否仍然有锁
      if pipe.get(lockname) == identifier:
        pipe.multi()
        pipe.delete(lockname)
        pipe.execute()
      pipe.unwatch()
      break
    except redis.exceptions.WatchError //有其他客户端修改了锁，重试
      pass
  return False  //进程已经失去了锁
```

release_lock()函数也做了很多措施来确保锁没有被修改，通过上锁和使用WATCH对比，重试的次数下降了，但是**不同上架和买入进程之间的竞争**限制了商品买卖操作性能的进一步提升，下面用细粒度锁来解决这个问题

之前是为了跟WATCH匹配的锁，锁住了整个市场，其实可以锁住被买卖的商品，可以减少锁竞争出现的几率并提升程序的性能，无论有多少个上架进程和买入进程在运行，程序可以在60秒内完成220000-230000次的上架和买入操作，并不引起任何重试

还有设置锁的超时时间，程序将在获得锁后，调用EXPIRE命令来为锁设置过期时间，使用Redis可以自动删除超时的锁。所以整体来说，redis并不会主动使用锁的，需要自己显示的去调用

### Redis的单机版安装

redis官网下载tar包，版本是4.0.14，解压

```html
# tar -zxvf redis-4.0.14.tar.gz
```

需要进行源码编译

```html
# cd redis-4.0.14
# make
# cd src
# make install PREFIX=/usr/local/redis
```

移动配置文件

```html
# cd ..
# mkdir /usr/local/redis/etc
# cp redis.conf /usr/local/redis/etc
```

配置redis为后台启动


```html
# vi /usr/local/redis/etc/redis.conf
```

将daemonize no配置改成yes，这样就是以守护进程的方式运行，默认不是


```html
# /usr/local/redis/bin/redis-server /usr/local/redis/etc/redis.conf 
```

### Redis的集群版安装

简单试验了下主从复制的redis集群，即master使用上面部署的redis，再有两个只读模式的节点，从节点其他配置都一样，可以修改成不同的端口号，但要加上如下的配置

```html
# slaveof 192.168.1.12 6379
# slave-read-only yes
```

配置是哪个redis服务（IP:PORT）的从节点，打开只读模式

### Redis在kubernetes集群中安装

因为Redis是有状态的，所以在kubernetes集群中要用statefulset的方式

1、首先找一台机器做NFS服务器，创建6个PV

```html
vi /etc/exports
/opt/kubernetes/redis/pv1  *(rw,no_root_squash)
/opt/kubernetes/redis/pv2  *(rw,no_root_squash)
/opt/kubernetes/redis/pv3  *(rw,no_root_squash)
/opt/kubernetes/redis/pv4  *(rw,no_root_squash)
/opt/kubernetes/redis/pv5  *(rw,no_root_squash)
/opt/kubernetes/redis/pv6  *(rw,no_root_squash)

mkdir -p /opt/kubernetes/redis/pv{1..6}
chmod -R 777 /opt/kubernetes/redis/pv{1..6}
```

然后启动nfs，有关这部分的详细步骤可以参考linux的post

```html
systemctl enable nfs
systemctl start nfs
```

2、kubernetes中创建pv

建6个，yaml文件如下，修改相应的name和path的路径

```html
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv1
spec:
  capacity:
    storage: 200M
  accessModes:
    - ReadWriteMany
  nfs:
    server: 10.144.91.121
    path: "/opt/kubernetes/redis/pv1"
```

3、创建configmap来存放redis的配置文件

```html
vi redis.conf

#enable AOF
appendonly yes

#cluster mod
cluster-enabled yes

#config-file
cluster-config-file /var/lib/redis/nodes.conf

#node timeout
cluster-node-timeout 5000

#AOF file location
dir /var/lib/redis

#port
port 6379
```

用下面的命令创建configmap

```html
kubectl create configmap redis-conf --from-file=redis.conf
```

4、创建一个Headless service

```html
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  labels:
    app: redis
spec:
  ports:
  - name: redis-port
    port: 6379
  clusterIP: None
  selector:
    app: redis
    appCluster: redis-cluster
```

可以看见clusterIp为None

5、创建Redis集群

```html
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: redis-app
spec:
  serviceName: "redis-service"
  replicas: 6
  template:
    metadata:
      labels:
        app: redis
        appCluster: redis-cluster
    spec:
      terminationGracePeriodSeconds: 20
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - redis
              topologyKey: kubernetes.io/hostname
      containers:
      - name: redis
        image: 10.144.91.121/library/redis:5.0
        command:
          - "redis-server"
        args:
          - "/etc/redis/redis.conf"
          - "--protected-mode"
          - "no"
        # command: redis-server /etc/redis/redis.conf --protected-mode no
        resources:
          requests:
            cpu: "100m"
            memory: "100Mi"
        ports:
            - name: redis
              containerPort: 6379
              protocol: "TCP"
            - name: cluster
              containerPort: 16379
              protocol: "TCP"
        volumeMounts:
          - name: "redis-conf"
            mountPath: "/etc/redis"
          - name: "redis-data"
            mountPath: "/var/lib/redis"
      volumes:
      - name: "redis-conf"
        configMap:
          name: "redis-conf"
          items:
            - key: "redis.conf"
              path: "redis.conf"
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes:
      - ReadWriteMany
      resources:
        requests:
          storage: 200M

```

6副本的创建，镜像push到harbor中`docker pull redis:5.0-rc5-alpine3.8`，检查一下

6、可以做相关验证

```html
kubectl get pod -owide
nslookup redis-app-1.redis-service.default.svc.cluster.local
kubectl get pv
```

7、初始化集群

Redis-trib进行初始化，这个东西`docker pull 314315960/redis-trib`，或者自己做一个镜像

```html
//先下载基础文件
wget http://download.redis.io/redis-stable/src/redis-trib.rb

//写dockerfile
FROM ruby:2.5-slim
MAINTAINER zqmalyssa<yzzqm@hotmail.com>
RUN gem install redis
RUN mkdir /redis
WORKDIR /redis
ADD ./redis-trib.rb /redis/redis-trib.rb

docker build -t redis-trib .
```
然后把这个redis-trib镜像做成脚本可以跑的，方便敲命令

```html
vi redis-trib
#!/bin/bash
/usr/bin/docker run -it --privileged --rm \
--net=host \
redis-trib:latest \
"$@"

chmod 777 redis-trib
mv redis-trib /usr/local/bin
//直接使用
redis-trib
```

可以初始化集群了

```html
# redis-trib create --replicas 1 \
    podIp1:6379 \
    podIp2:6379 \
    podIp3:6379 \
    podIp4:6379 \
    podIp5:6379 \
    podIp6:6379

//当然也可以不用自己去查redis的podIp，也不做脚本

kubectl run redis-trib --image=314315960/redis-trib -- create --replicas 1  \
$(kubectl get pod redis-app-0 -o jsonpath='{.status.podIP}'):6379 \
$(kubectl get pod redis-app-1 -o jsonpath='{.status.podIP}'):6379 \
$(kubectl get pod redis-app-2 -o jsonpath='{.status.podIP}'):6379 \
$(kubectl get pod redis-app-3 -o jsonpath='{.status.podIP}'):6379 \
$(kubectl get pod redis-app-4 -o jsonpath='{.status.podIP}'):6379 \
$(kubectl get pod redis-app-5 -o jsonpath='{.status.podIP}'):6379
```
replicas 1的意思，就是每个节点创建1个副本(即：slave)

8、初始化完成后验证

```html
kubectl exec -it redis-app-2 /bin/bash
//在容器中，-c的意思是Enable cluster mode (follow -ASK and -MOVED redirections)，否则set不是集群模式，会报(error) MOVED 866这种错
/usr/local/bin/redis-cli -c
cluster info
cluster nodes

//在宿主机上，看redis挂载的数据
ls /opt/kubernetes/redis/pv1
```

9、创建用于访问的Service

之前建的service没有cluster IP,因此不能用于外界访问.所以我们还需要创建一个service,专用于为Redis集群提供访问和负载均衡:

```html
apiVersion: v1
kind: Service
metadata:
  name: redis-access-service
  labels:
    app: redis
spec:
  ports:
  - name: redis-port
    protocol: "TCP"
    port: 6379
    targetPort: 6379
  selector:
    app: redis
    appCluster: redis-cluster

```
该Service名称为 redis-access-service，在K8S集群中暴露6379端口，并且会对labels name为app: redis或appCluster: redis-cluster的pod进行负载均衡。其实这部分吧，想想，反正可以根据clusterIp去check了

```html
redis-trib check clusterIp:6379
```

10、主从切换测试

```html
kubectl exec -it redis-app-2 /bin/bash
//容器中
redis-cli
role
//查看了角色及slave节点

//我们手动删除redis-app-2，还会再弹一个，IP改变
kubectl delete pods redis-app-2
//再进入redis-app-2
role
//发现自己是slave，之前的slave变成了它的master
```

11、动态扩容

现在这个集群中有6个节点三主三从,我现在添加两个pod节点,达到4主4从

```html
cat >> /etc/exports <<'EOF'
/opt/kubernetes/redis/pv7 *(rw,all_squash)
/opt/kubernetes/redis/pv8 *(rw,all_squash)
EOF
systemctl restart nfs

mkdir /opt/kubernetes/redis/pv{7..8}
chmod -R 777 /opt/kubernetes/redis/pv{7..8}
```
需要再建两个pv，这里不展示了，更改redis的yml文件里面的replicas:字段,把这个字段改为8,然后升级运行

```html
kubectl apply -f redis.yml
```
添加集群节点

```html
redis-trib add-node ip:6379
```
检查整个集群信息

### Redis的Win版安装

Win版一般用作测试用

[这个地址](https://github.com/MicrosoftArchive/redis/tags)下载安装包，就用压缩包.zip好了

```html
//如下方式启动，配置文件可修改后台
./redis-server.exe redis.windows.conf
```
最好在cmd里面起client，Git_bash中起可能有问题，也可以将redis设置成windows服务

```html
./redis-server --service-install redis.windows.conf --loglevel verbose
```

这样在win的服务中就能发现这个服务了

```html
./redis-server --service-start
```

就是启动服务了，注意，redis的服务关闭后redis中的数据会丢失

### Redis基本使用

支持5种数据类型：String，hash，list，set和zset（有序的set）

```html
127.0.0.1:6379>set test "这是个test"
127.0.0.1:6379>del test

//hash结构的使用，下面的例子里test是key，field1和"test1"分别代表域和值，这是我们常规认知的key-value，然后这个可以用hmset做多次
127.0.0.1:6379>hmset test field1 "test1" field2 "test2"
127.0.0.1:6379>hget test field1
127.0.0.1:6379>del test

127.0.0.1:6379>lpush test redis
127.0.0.1:6379>lpush test mongodb
127.0.0.1:6379>lrange test 0 10
127.0.0.1:6379>del test
127.0.0.1:6379>sadd test redis
127.0.0.1:6379>sadd test rabbitmq
127.0.0.1:6379>sadd test rabbitmq (注意是不会有重复元素的)
127.0.0.1:6379>del test
127.0.0.1:6379>zadd test 0 redis
127.0.0.1:6379>zadd test 0 rabbitmq

//查看所有的key
keys *

```

### SpringBoot与Redis的整合

spring 可以通过两种方式连接redis，一种是jedis，一种是lettuce。通过debug我们发现spring boot默认注入的连接是LettuceConnectionFactory。为啥spring boot默认注入的是LettuceConnectionFactory而不是JedisConnectionFactory呢？
在idea工程的external libraries找到spring-boot-autoconfigure包，打开data -> redis。这里有两个configuration:JedisConnectionConfiguration和LettuceConnectionConfiguration。spring boot就是通过这两个类来自动配置redis。LettuceConnectionConfiguration的部分代码如下

```java
@Configuration
@ConditionalOnClass(RedisClient.class)
class LettuceConnectionConfiguration extends RedisConnectionConfiguration {
......

@Bean
	@ConditionalOnMissingBean(RedisConnectionFactory.class)
	public LettuceConnectionFactory redisConnectionFactory(
			ClientResources clientResources) throws UnknownHostException {
		LettuceClientConfiguration clientConfig = getLettuceClientConfiguration(
				clientResources, this.properties.getLettuce().getPool());
		return createLettuceConnectionFactory(clientConfig);
	}

```

@ConditionalOnClass注解是spring boot实现自动化配置的主要注解之一。其作用大概是当xxx.class在类加载路径下时@Configuration生效。所有当我们引入spring-boot-starter-data-redis后RedisClient.class也被引入了，此时spring就会初始化LettuceConnectionConfiguration 。

@ConditionalOnMissingBean(RedisConnectionFactory.class)原理类似，当spring容器没有RedisConnectionFactory bean的时候注入一个LettuceConnectionFactory。

接下来看看JedisConnectionFactory。
JedisConnectionFactory的部分代码如下：

```java
@Configuration
@ConditionalOnClass({ GenericObjectPool.class, JedisConnection.class, Jedis.class })
class JedisConnectionConfiguration extends RedisConnectionConfiguration {
......

@Bean
	@ConditionalOnMissingBean(RedisConnectionFactory.class)
	public JedisConnectionFactory redisConnectionFactory() throws UnknownHostException {
		return createJedisConnectionFactory();
	}

```

可以看到JedisConnectionFactory的注入条件更多，需要类路径下同时存在 GenericObjectPool.class, JedisConnection.class, Jedis.class时才会注入JedisConnectionFactory。

通过以上的分析，当我们只在pom.xml里面引入spring-boot-starter-data-redis时JedisConnectionConfiguration 需要的@ConditionalOnClass条件不满足，所以spring boot默认注入的是LettuceConnectionFactory。

```java
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>io.lettuce</groupId>
                    <artifactId>lettuce-core</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
            <version>2.6.0</version>
        </dependency>
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
        </dependency>
```

在这里我们排除了lettuce相关的包，并引入jedis。其中GenericObjectPool.class存在于commons-pool2包里面（jiedis自带commons-pool2）， Jedis.class 在jedis包里面。这样就满足了JedisConnectionConfiguration 的条件

我们看到connectionFactory被替换为了JedisConnectionFactory。lettuce和jedis对比

1、jedis：是Redis的Java实现客户端，提供了比较全面的Redis命令的支持。lettuce：高级Redis客户端，用于线程安全同步，异步和响应使用，支持集群，Sentinel，管道和编码器。

2、Jedis 在实现上是直接连接 Redis-Server，在多个线程间共享一个 Jedis 实例时是线程不安全的，如果想要在多线程场景下使用 Jedis，需要使用连接池（此点可以从JedisConnectionConfiguration 中GenericObjectPool.class是必要条件之一可以看出）来管理连接，而不是每次都生成一个新的连接。

3、Lettuce 则完全克服了其线程不安全的缺点：Lettuce 是一个可伸缩的线程安全的 Redis 客户端，支持同步、异步和响应式模式。多个线程可以共享一个连接实例，而不必担心多线程并发问题。它基于优秀 Netty NIO 框架构建，支持 Redis 的高级功能，如 Sentinel，集群，流水线，自动重新连接和 Redis 数据模型。

两者的连接池配置如下

```java
#redis lettuce连接池配置 使用的common-pool2的连接池
#shutdown的超时时间
#spring.redis.lettuce.shutdown-timeout=100ms
#连接池最大活跃连接数
#spring.redis.lettuce.pool.max-active=8
#连接池最大空闲连接数
#spring.redis.lettuce.pool.max-idle=8
#当连接池达到最大连接数时，等待可用的连接的等待时间
#spring.redis.lettuce.pool.max-wait=-1ms
#连接池最小空闲连接数
#spring.redis.lettuce.pool.min-idle=0

#redis jedis连接配置
#spring.redis.jedis.pool.max-active=8
#spring.redis.jedis.pool.max-idle=8
#spring.redis.jedis.pool.max-wait=-1ms
#spring.redis.jedis.pool.min-idle=0

```
lettuce既然是线程安全的那为什么也有连接池的配置呢？lettuce有如下几种情况你不能在线程之间复用连接：

1、请求批量下发，即禁止调用命令后立即flush
2、使用BLPOP这种阻塞命令
3、事务操作
4、有多个数据库的情况

所以LettuceConnectionFactory只有在使用阻塞命令或者事务操作的时候才会使用到连接池。

**Redis分布式锁与java**

setnx: 将key设置值为value，如果key不存在，这种情况下等同SET命令。 当key存在时，什么也不做。SETNX是”SET if Not eXists”的简写。

getset: 自动将key对应到value并且返回原来key对应的value。如果key存在但是对应的value不是字符串，就返回错误。

这两个命令在 java 中对应为 setIfAbsent 和 getAndSet

避免锁超时的redis实现简单分布式的伪代码，存在下面的三个问题

```java
//加锁
if（setnx（key，1） == 1）{
  //避免超时
    expire（key，30）
    try {
        do something ......
    } finally {
      //释放锁
        del（key）
    }
}
```
1、setnx和expire的非原子性，当setnx刚成功，还没来得及做expire，结点挂了，其他结点的进程不要想进来了，还好redis2.6版本以上提供了set(key, 1, 30, NX)这种指令替代了setnx

2、del导致误删，如果某些原因结点1的进程很慢很慢，30秒都没执行完，锁就自动释放了，另一个结点的某个进程中的线程B得到了锁，随后当前的线程A执行完后，del去释放锁，B还没执行完，实际上就是删的B加的锁，需要看下当前的锁是不是自己的锁，可以再删前判断下，根据自己的线程ID

```java
//加锁
String threadId = Thread.currentThread().getId()
set（key，threadId ，30，NX）

//解锁
if（threadId .equals(redisClient.get(key))）{
    del(key)
}
```
但是，这样做又隐含了一个新的问题，判断和释放锁是两个独立操作，不是原子性的。要想实现验证和删除过程的原子性，可以使用Lua脚本来实现。这样就能保证验证和删除过程的正确性了。

3、还是刚才第二点所描述的场景，虽然我们避免了线程A误删掉key的情况，但是同一时间有A，B两个线程在访问代码块，仍然是不完美的。怎么办呢？我们可以让获得锁的线程开启一个守护线程，用来给快要过期的锁“续航”。当过去了29秒，线程A还没执行完，这时候守护线程会执行expire指令，为这把锁“续命”20秒。守护线程从第29秒开始执行，每20秒执行一次。当线程A执行完任务，会显式关掉守护线程。如果节点1 忽然断电，由于线程A和守护线程在同一个进程，守护线程也会停下。这把锁到了超时的时候，没人给它续命，也就自动释放了。


分布式锁的实现还有哪些？

1、memcached分布式锁
利用Memcached的add命令。此命令是原子性操作，只有在key不存在的情况下，才能add成功，也就意味着线程得到了锁。
2、Redis分布式锁
和Memcached的方式类似，利用Redis的setnx命令。此命令同样是原子性操作，只有在key不存在的情况下，才能set成功。（setnx命令并不完善，后续会介绍替代方案）
3、Zookeeper分布式锁
利用Zookeeper的顺序临时节点，来实现分布式锁和等待队列。Zookeeper设计的初衷，就是为了实现分布式锁服务的。


**Redis实现session共享**

作为集中存储session的地方，这个功能还是经常被用到的，引入需要的pom

```java
<dependency>
  <groupId>org.springframework.session</groupId>
  <artifactId>spring-session-data-redis</artifactId>
</dependency>
```
启动类上面加注解

```java
@EnableRedisHttpSession
public class PomApplication {

  public static void main(String[] args) {
    SpringApplication.run(PomApplication.class, args);
  }

}
```
写一个简单的测试接口，调用下

```java
@RestController
public class SessionShareController {

  @RequestMapping(value = "/session/set", method = RequestMethod.POST)
  public Object setSession(HttpSession session, String name) {
    session.setAttribute("name", name);
    return "ok";
  }

}
```

可以看见redis中已经有session了

```html
127.0.0.1:6379> keys *
1) "spring:session:sessions:expires:2192c72d-4ea5-4bf2-b12e-50780102ae3d"
2) "aaa"
3) "ccc"
4) "bbb"
5) "dis"
6) "spring:session:sessions:2192c72d-4ea5-4bf2-b12e-50780102ae3d"
7) "spring:session:expirations:1581739860000"
```
然后看下key的详情

```html
127.0.0.1:6379> hgetall spring:session:sessions:2192c72d-4ea5-4bf2-b12e-50780102ae3d
1) "maxInactiveInterval"
2) "\xac\xed\x00\x05sr\x00\x11java.lang.Integer\x12\xe2\xa0\xa4\xf7\x81\x878\x02\x00\x01I\x00\x05valuexr\x00\x10java.lang.Number\x86\xac\x95\x1d\x0b\x94\xe0\x8b\x02\x00\x00xp\x00\x00\a\b"
3) "creationTime"
4) "\xac\xed\x00\x05sr\x00\x0ejava.lang.Long;\x8b\xe4\x90\xcc\x8f#\xdf\x02\x00\x01J\x00\x05valuexr\x00\x10java.lang.Number\x86\xac\x95\x1d\x0b\x94\xe0\x8b\x02\x00\x00xp\x00\x00\x01pF\xefK\x1e"
5) "lastAccessedTime"
6) "\xac\xed\x00\x05sr\x00\x0ejava.lang.Long;\x8b\xe4\x90\xcc\x8f#\xdf\x02\x00\x01J\x00\x05valuexr\x00\x10java.lang.Number\x86\xac\x95\x1d\x0b\x94\xe0\x8b\x02\x00\x00xp\x00\x00\x01pF\xefK\x1e"
7) "sessionAttr:name"
8) "\xac\xed\x00\x05t\x00\x0ctestsession2"
```
然后我们把应用起在两个端口上，用nginx将localhost:8082转发到这个两个端口上，通过配置server.port实现，同时idea要允许一个应用起两次，整个代码如下

```java
package com.qiming.pom.redis.sessionshare;

import javax.servlet.http.HttpSession;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class SessionShareController {

  @Value("${server.port}")
  private int port;

  @RequestMapping(value = "/session/set", method = RequestMethod.POST)
  public Object setSession(HttpSession session, String name) {
    session.setAttribute("name", name);
    return "ok";
  }

  @RequestMapping(value = "/session/get", method = RequestMethod.GET)
  public Object getSession(HttpSession session) {
    Object name = session.getAttribute("name");
    return "port: " + port + ", name: " + name;
  }

}

```

用postman调用`http://localhost/session/get`，能看见port值不停在变，但都能正常取到之前设置的值，说明session共享已经成功

### Redis运维

**1,默认是127启动，需要改成可以远程访问**

默认配置中bind 后面配的是127.0.0.1，如果要改成所有都能访问，配置成0.0.0.0，如果要改成启动在本地地址，改成本机IP，userdata中可以改成如下脚本

```html
#!/bin/bash

#disable_root: False
#password: qwer1234
#chpasswd:
#    list: |
#          root:qwer1234
#    expire: False
#ssh_pwauth: True

#instance_passwd=qwer1234

redis_init()
{
  ipaddr=$(ip a | grep "eth0" | grep "inet" | awk '{print $2}')
  ip=${ipaddr%/*}
  sed -i "s/127.0.0.1/$ip/g" /usr/local/redis/etc/redis.conf
  /usr/local/redis/bin/redis-server /usr/local/redis/etc/redis.conf
}

redis_init
```

**2,Redis用集合的sdiff不成功**

在命令行中操作，比较两个集合的差值报错

```html
(error) CROSSSLOT Keys in request don't hash to the same slot
```
原因是开启集群模式，虽然在同一个结点，但不在同一个哈希槽中，

```html
scan 0
//以下两个值不同
cluster keyslot set-key1
cluster keyslot set-key2
```
需要用到hash标签强制将两个set放到同一个槽中

```html
CLUSTER KEYSLOT {user}:set-key1
CLUSTER KEYSLOT {user}:set-key2
```
但是加了这个{user}发现比较没结果，而且{user}:set-key1和set-key1像是不同的集合，那又重新建了带标签的集合才好

```html
sadd {user}:set-key1 qi
sadd {user}:set-key1 mi
//下面这样才出效果
sdiff {user}:set-key1 {user}:set-key1
```
