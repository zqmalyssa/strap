---
layout: post
title: "mysql的使用及运维"
author-id: "zqmalyssa"
tags: [code, operations]
---

本篇汇总了mysql的知识点，涉及到使用小技巧及运维部分

### mysql的基本介绍

查看mysql的版本

```html
mysql --version
```

#### mysql中的事务

4大特性，原子性(A)，一致性(C)，隔离性(I)，持久型(D)。而事务的ACID是通过InnoDB日志和锁来保证。事务的隔离性是通过数据库锁的机制实现的，持久性通过redo log（重做日志）来实现，原子性和一致性通过Undo log来实现。UndoLog的原理很简单，为了满足事务的原子性，在操作任何数据之前，首先将数据备份到一个地方（这个存储数据备份的地方称为UndoLog）。然后进行数据的修改。如果出现了错误或者用户执行了ROLLBACK语句，系统可以利用Undo Log中的备份将数据恢复到事务开始之前的状态。 和Undo Log相反，RedoLog记录的是新数据的备份。在事务提交前，只要将RedoLog持久化即可，不需要将数据持久化。当系统崩溃时，虽然数据没有持久化，但是RedoLog已经持久化。系统可以根据RedoLog的内容，将所有数据恢复到最新的状态。 对具体实现过程有兴趣的同学可以去自行搜索扩展

注意begin或者start可以开启一个事务，事务是有隐式提交的

1)新事物的开启会导致旧事物的隐式提交

2)InnoDB中所有的DDL或DCL操作都会开启一个新的事物，所以DDL或DCL语句会导致旧事物的隐式提交，注意不是DML语句
	- DDL（Data Definition Languages）语句：数据定义语言，这些语句定义了不同的数据段、数据库、表、列、索引等数据库对象的定义。常用的语句关键字主要包括 create、drop、alter等。
	- DML（Data Manipulation Language）语句：数据操纵语句，用于添加、删除、更新和查询数据库记录，并检查数据完整性，常用的语句关键字主要包括 insert、delete、udpate 和select 等。(增添改查）
	- DCL（Data Control Language）语句：数据控制语句，用于控制不同数据段直接的许可和访问级别的语句。这些语句定义了数据库、表、字段、用户的访问权限和安全级别。主要的语句关键字包括 grant、revoke 等。
3)过程的执行区结束End之前会有一次隐式提交

事务的并发的三个问题及四种隔离级别

1、脏读（Dirty read）：事务A读取了事务B更新的数据，然后B回滚操作，那么A读取到的数据是脏数据
2、不可重复读：事务 A 多次读取同一数据，事务 B 在事务A多次读取的过程中，对数据作了更新并提交，导致事务A多次读取同一数据时，结果不一致。重点是结果不一致了
3、幻读：系统管理员A将数据库中所有学生的成绩从具体分数改为ABCDE等级，但是系统管理员B就在这个时候插入了一条具体分数的记录，当系统管理员A改结束后发现还有一条记录没有改过来，就好像发生了幻觉一样，这就叫幻读。重点是数量上不一致了
小结：不可重复读的和幻读很容易混淆，不可重复读侧重于修改，幻读侧重于新增或删除。解决不可重复读的问题只需锁住满足条件的行，解决幻读需要锁表

| 事务隔离级别 | 脏读 | 不可重复读 | 幻读 |
| :----: | :----:  | :----: | :----: |
| 读未提交（read-uncommitted） | 会 | 会 | 会 |
| 读已提交/不可重复读（read-committed） | 不会 | 会 | 会 |
| 可重复读（repeatable-read） | 不会 | 不会 | 会 |
| 串行化（serializable） | 不会 | 不会 | 不会 |

大多数数据库的默认隔离级别是Read committed，但是mysql是Repeatable Read

查看当前的隔离级别

```html
select @@tx_isolation;
```
说明上面的情况

```html
create table tx (
	id int(6),
	num int(6)
);

insert into tx values (1, 1);
insert into tx values (2, 2);
insert into tx values (3, 3);
```
需要两个客户端

```html
set autocommit=0;
select @@autocommit;
```

1、读未提交的数据

```html
set session transaction isolation level read uncommitted;
show variables like '%tx%'; //检查下
```

A客户端
```html
start transaction;

//不停的做读操作
select * from tx;
```

B客户端
```html
start transaction;

update tx set num = 10 where id = 1; //这个一设置完，A应该就读到了10
select * from tx;

rollback;  //取消了设置，数据回滚，AB都读回原来的数，但是在没回滚这段时间，A客户端脏读了

select * from tx;

```

2、读已提交的数据

```html
set session transaction isolation level read committed;
show variables like '%tx%'; //检查下
```

换隔离级别最好是退出mysql再重新登一下

A客户端
```html
start transaction;

//不停的做读操作
select * from tx;
```
B客户端
```html
start transaction;

update tx set num = 10 where id = 1; //这个一设置完，A不应该读到10了，因为隔离级别提升了
select * from tx;

commit; //commit一完，A就读到不同的数据了，造成不可重复读

select * from tx;
```

3、可重复读

```html
set session transaction isolation level repeatable read; //默认的
show variables like '%tx%'; //检查下
```
A客户端
```html
start transaction;

//不停的做读操作
select * from tx;
```
B客户端
```html
start transaction;

update tx set num = 10 where id = 1; //这个一设置完，A不应该读到10了，因为隔离级别提升到了读已提交
select * from tx;

commit; //commit一完，A还是读原来的数据，不会读10，因为隔离级别又提高到了可重复度

select * from tx;

//插入数据
insert into tx values (4,4); //A中看不见
select * from tx;

commit; //A中还是看不见

```

只有等到A也commit的时候，才能看见改变和新增的数据。

可重复读的隔离级别下使用了MVCC机制，select操作不会更新版本号，是快照读（历史版本）；insert、update和delete会更新版本号，是当前读（当前版本）。看下这篇[文章](https://www.cnblogs.com/wyaokai/p/10921323.html)

4、串行化

mysql中事务隔离级别为serializable时会锁表，因此不会出现幻读的情况，这种隔离级别并发性极低，开发中很少会用到。

补充说明：

1、事务隔离级别为读提交时，写数据只会锁住相应的行
2、事务隔离级别为可重复读时，如果检索条件有索引（包括主键索引）的时候，默认加锁方式是next-key 锁；如果检索条件没有索引，更新数据时会锁住整张表。一个间隙被事务加了锁，其他事务是不能在这个间隙插入记录的，这样可以防止幻读。
3、事务隔离级别为串行化时，读写数据都会锁住整张表
4、隔离级别越高，越能保证数据的完整性和一致性，但是对并发性能的影响也越大。

#### mysql的存储引擎

主要有InnoDB和MyISAM，其实最大的区别在于InnoDB是事务优先(行锁)，而MyISAM是性能优先(表锁)

```html
show engines;
```
查看所有引擎，这里面有个字段是`XA`，这个XA是用来支持分布式锁的吗？！

```html
show variables like "%storage_engine%";
```
查看正在使用的引擎和默认的

显示修改数据库对象的引擎，可以做到表级别，oracle没有

```html
create table tb(
	id int(4) auto_increment,
	name varchar(5) not null default '',
	loc varchar(5) default ''
) engine=MyISAM default charset=utf8;
```

B树和B+树

B树要了解是什么样的，就是key（排序的数据）要比指针少一个，指针用来索引，key其实就是分界线，可以很快的索引数据，还有data，data就是记录除key外的数据，B树有很多变种，比如B+树，有些不同

1. 每个节点的指针上限是2d，而不是B树的2d+1，这里d指数的度
2. B+树的内结点是不存储数据的，只存储key，叶结点只存储数据，不存储指针

三层B Tree的存放量可以到上百万条，B+树的所有数据都放在叶子结点中，所以查询任意的数据次数是B+树的高度，B+树比B树更适合实现外存储索引结构，这与计算机存取原理有关，而一般在数据库系统或文件系统中使用的B+树都在经典的B+树上进行优化，比如增加顺序访问指针，即在叶结点上增加一个指向相邻叶子结点的指针

红黑树等数据结构也可以用来实现索引，但是文件系统及数据库系统普遍采用B-/+Tree作为索引结构，一般来说，索引本身也很大，不可能全部存储在内存中，因此索引往往以索引文件的形式存储的磁盘上。这样的话，索引查找过程中就要产生磁盘I/O消耗，相对于内存存取，I/O存取的消耗要高几个数量级，所以评价一个数据结构作为索引的优劣最重要的指标就是在查找过程中磁盘I/O操作次数的渐进复杂度。换句话说，索引的结构组织要尽量减少查找过程中磁盘I/O的存取次数。

目前计算机使用的主存基本都是随机读写存储器（RAM），现代RAM的结构和存取原理比较复杂，这里只是一个抽象，主存是一系列的存储单元组成的矩阵，每个存储单元存储固定大小的数据。每个存储单元有唯一的地址，现代主存的编址规则比较复杂，这里将其简化成一个二维地址：通过一个行地址和一个列地址可以唯一定位到一个存储单元

当系统需要读取主存时，则将地址信号放到地址总线上传给主存，主存读到地址信号后，解析信号并定位到指定存储单元，然后将此存储单元数据放到数据总线上，供其它部件读取。写主存的过程类似，系统将要写入单元地址和数据分别放在地址总线和数据总线上，主存读取两个总线的内容，做相应的写操作。这里可以看出，主存存取的时间仅与存取次数呈线性关系，因为不存在机械操作，两次存取的数据的“距离”不会对时间有任何影响，例如，先取A0再取A1和先取A0再取D3的时间消耗是一样的。

索引一般以文件形式存储在磁盘上，索引检索需要磁盘I/O操作。与主存不同，磁盘I/O存在机械运动耗费，因此磁盘I/O的时间消耗是巨大的。一个磁盘由大小相同且同轴的圆形盘片组成，磁盘可以转动（各个磁盘必须同步转动）。在磁盘的一侧有磁头支架，磁头支架固定了一组磁头，每个磁头负责存取一个磁盘的内容。磁头不能转动，但是可以沿磁盘半径方向运动（实际是斜切向运动），每个磁头同一时刻也必须是同轴的，即从正上方向下看，所有磁头任何时候都是重叠的（不过目前已经有多磁头独立技术，可不受此限制），盘片被划分成一系列同心环，圆心是盘片中心，每个同心环叫做一个磁道，所有半径相同的磁道组成一个柱面。磁道被沿半径线划分成一个个小的段，每个段叫做一个扇区，每个扇区是磁盘的最小存储单元。为了简单起见，我们下面假设磁盘只有一个盘片和一个磁头。当需要从磁盘读取数据时，系统会将数据逻辑地址传给磁盘，磁盘的控制电路按照寻址逻辑将逻辑地址翻译成物理地址，即确定要读的数据在哪个磁道，哪个扇区。为了读取这个扇区的数据，需要将磁头放到这个扇区上方，为了实现这一点，磁头需要移动对准相应磁道，这个过程叫做寻道，所耗费时间叫做寻道时间，然后磁盘旋转将目标扇区旋转到磁头下，这个过程耗费的时间叫做旋转时间。

由于存储介质的特性，磁盘本身存取就比主存慢很多，再加上机械运动耗费，磁盘的存取速度往往是主存的几百分分之一，因此为了提高效率，要尽量减少磁盘I/O。为了达到这个目的，磁盘往往不是严格按需读取，而是每次都会预读，即使只需要一个字节，磁盘也会从这个位置开始，顺序向后读取一定长度的数据放入内存。这样做的理论依据是计算机科学中著名的局部性原理：

**当一个数据被用到时，其附近的数据也通常会马上被使用**

程序运行期间所需要的数据通常比较集中。由于磁盘顺序读取的效率很高（不需要寻道时间，只需很少的旋转时间），因此对于具有局部性的程序来说，预读可以提高I/O效率。预读的长度一般为页（page）的整倍数。页是计算机管理存储器的逻辑块，硬件及操作系统往往将主存和磁盘存储区分割为连续的大小相等的块，每个存储块称为一页（在许多操作系统中，页得大小通常为4k），主存和磁盘以页为单位交换数据。当程序要读取的数据不在主存中时，会触发一个缺页异常，此时系统会向磁盘发出读盘信号，磁盘会找到数据的起始位置并向后连续读取一页或几页载入内存中，然后异常返回，程序继续运行。

一般使用磁盘I/O次数评价索引结构的优劣。先从B-Tree分析，根据B-Tree的定义，可知检索一次最多需要访问h个节点（h是一个整数，是B树的高度）。数据库系统的设计者巧妙利用了磁盘预读原理，将一个节点的大小设为等于一个页，这样每个节点只需要一次I/O就可以完全载入。为了达到这个目的，在实际实现B-Tree还需要使用如下技巧：

1. 每次新建节点时，直接申请一个页的空间，这样就保证一个节点物理上也存储在一个页里，加之计算机存储分配都是按页对齐的，就实现了一个node只需一次I/O。
2. B-Tree中一次检索最多需要h-1次I/O（根节点常驻内存），一般实际应用中，出度d是非常大的数字，通常超过100，因此h非常小（通常不超过3）。

综上所述，用B-Tree作为索引结构效率是非常高的。

而**红黑树这种结构，h明显要深的多**。由于逻辑上很近的节点（父子）物理上可能很远，无法利用局部性，所以红黑树的I/O渐进复杂度也为O(h)，效率明显比B-Tree差很多。B+树更适合外存索引，原因和内节点出度d有关。从上面分析可以看到，d越大索引的性能越好（h越小），而**出度的上限取决于节点内key和data的大小，由于B+Tree内节点去掉了data域，因此可以拥有更大的出度，拥有更好的性能**

MyISAM引擎使用B+Tree作为索引结构，叶节点的data域存放的是**数据记录的地址**，MyISAM的索引文件仅仅保存数据记录的地址。在MyISAM中，主索引和辅助索引（Secondary key）在结构上没有任何区别，只是主索引要求key是唯一的，而辅助索引的key可以重复，MyISAM的索引方式也叫做“非聚集”的，之所以这么称呼是为了与InnoDB的聚集索引区分

虽然InnoDB也使用B+Tree作为索引结构，但具体实现方式却与MyISAM截然不同。第一个重大区别是InnoDB的数据文件本身就是索引文件。从上文知道，MyISAM索引文件和数据文件是分离的，索引文件仅保存数据记录的地址。而在InnoDB中，表数据文件本身就是按B+Tree组织的一个索引结构，这棵树的叶节点data域保存了完整的数据记录。这个索引的key是数据表的主键，因此InnoDB表数据文件本身就是主索引。叶节点包含了完整的数据记录。这种索引叫做聚集索引

因为InnoDB的数据文件本身要按主键聚集，所以InnoDB要求表必须有主键（MyISAM可以没有），如果没有显式指定，则MySQL系统会自动选择一个可以唯一标识数据记录的列作为主键，如果不存在这种列，则MySQL自动为InnoDB表生成一个隐含字段作为主键，这个字段长度为6个字节，类型为长整形

第二个与MyISAM索引的不同是InnoDB的辅助索引data域**存储相应记录主键的值**而不是地址（也就是存储primary key）。换句话说，InnoDB的所有辅助索引都引用主键作为data域。这里以英文字符的ASCII码作为比较准则。聚集索引这种实现方式使得按主键的搜索十分高效，但是辅助索引搜索需要检索两遍索引：首先检索辅助索引获得主键，然后用主键到主索引中检索获得记录

参考[这篇文章](https://blog.csdn.net/qq_27607965/article/details/79925288)

了解不同存储引擎的索引实现方式对于正确使用和优化索引都非常有帮助，可以看下面的注意事项

#### mysql优化

SQL优化，其实主要优化的就是`索引`，索引是高效获取数据的数据结构(B树，B+树，Hash树)，Mysql用的是B+树，Innodb和MyISAM，但有所不同，索引不是必须要的，因为索引本身是很大的，有些情况并不需要索引

1. 少量数据，别加了
2. 经常需要变得字段，不要加索引
3. 索引会提高查询，但是会降低增删改，因为需要改索引的东西

索引的分类：

1. 单值索引，一个表可以有多个单值索引
2. 唯一索引，数据不能重复，id （多字段的唯一索引，注意，如果其中某个字段是null值，是允许插入多条的，自然，某个允许null的字段，有唯一索引，对于null也是可以插入多条的）
3. 复合索引，多个列构成，相当于书的二级目录

创建索引的两种方式：

1. create index dept_index on tb(name);
2. alter table tb add index dept_name_index(dept, name);

删除：

drop index dept_index on tb;

查询：

show index from tb;

表删了，表上的索引就没了

```html
show status like 'last_query_cost';
```

上面的输出，mysql认为需要花费XXX个数据页的随机查找才能完成上面的查询，这个结果是跟汇聚一系列的统计信息计算出来的，这些统计信息包括：每张表或者索引的页面个数，索引的基数，索引和数据行的长度，索引的分布情况等等

B Screen，explain执行计划的使用

SQL的编写过程：

select dinstinct ... from ... join .. on .. where .. group by .. having .. order by ..

SQL的解析过程：

from .. on .. join .. where .. group by ... having ... select dinstinct .. order by ..

优化案例：

1. 单表优化
	- 根据SQL解析的顺序，调整索引的顺序，所以select的字段放在后面
	- 之前觉得不好的索引及时删除
	- where子句中有in的时候要注意了，一个是in语句可能导致复合索引的跨列情况，所以把in的表达式放到最后，一个是in会引起索引失效，导致using where，改成=号不出现using where，可以通过key_len  进行一些证明

2. 多表优化
	- 往哪里加索引，小表驱动大表，小表10条数据，大表300条数据，左边的是在外层循环，右边是内层循环，所以on子句后的连接语句有点说法
	- 索引建立在经常使用的字段上，就是上面说的左边的
	- 左外连接就给左表加索引，右外连接给右表加索引

避免索引失效的原则：

1. 复合索引的时候不要跨列或无序使用（最佳左前缀）
2. 复合索引尽量使用全索引匹配
3. 不要在索引上进行任何操作（计算，函数，类型转换），否则索引失效，**复合索引，左边失效了，索引全部失效**，但是独立的索引不受此影响。
4. 复合索引不能使用不等于(!= <>)或 is null，否则自身及右侧所有索引全部失效
5. SQL优化器会概念导致你想的优化失效
6. like尽量用常量开头，不要用%，否则索引失效，但是 索引覆盖会一定程度解决这个问题，查询的列全在索引中
7. 尽量不要使用or，否则索引失效，连左侧的索引的能给干失效了

补救的话尽量使用索引覆盖（using index）


优化方法：

1. 如果主查询的数据极大，使用in，如果子查询的数据极大，使用exist
	- exist，将主查询的结果放到子查询的结果中进行条件校验，看子查询是否有数据，如果有数据，则保留数据

注意事项：

1. 在InnoDB中，不要用过长的字段作为主键，因为辅助索引会引用主索引，过长的主索引会令辅助索引变得过大
2. 在InnoDB中，用非单调的字段作为主键不是个好主意，因为InnoDB数据文件本身是一颗B+Tree，非单调的主键会造成在插入新记录时数据文件为了维持B+Tree的特性而频繁的分裂调整，十分低效，而使用自增字段作为主键则是一个很好的选择（auto_increment）（就是UUID不好的原因，要保证有序）


慢查询日志：

记录Mysql中响应时间超过阈值的SQL，一般是10s还没出来，慢查询日志默认是关闭的，开发调优时再打开，部署时再关闭

```html
show variables like "%slow_query%";
```

临时开启，mysql服务重启就消失了

```html
set global slow_query_log = 1
```
10s在这个变量里看
```html
show variables like "%long_query_time%";
```
修改后重新登录才能看见效果

永久开启在配置文件中增加后重启mysql服务

```html
select sleep(4)
show global status like "%slow_queries%";  //一共有多少个慢查询
```
日志路径里面可以查执行，但是不方便，可以用自带的工具mysqldumpslow --help来用，可以通过过滤条件快速查找到慢SQL

1.获取返回记录最多的三个SQL

```html
mysqldumpslow -s r -t 3 /var/lib/mysql/epc01-slow.log
```

2.获取访问次数最多的三个SQL

```html
mysqldumpslow -s c -t 3 /var/lib/mysql/epc01-slow.log
```

3.按照时间排序，前10条包含left join的查询语句SQL

```html
mysqldumpslow -s t -t 10 -g "left join" /var/lib/mysql/epc01-slow.log
```

**分析海量数据**

```html
create database testdata;
use testdata;
create table dept(
	dno int(5) primary key default 0,
	dname varchar(20) not null default '',
	loc varchar(30) default ''
) engine=innodb default charset=utf8;

create table emp(
	eid int(5) primary key,
	ename varchar(20) not null default '',
	job varchar(20) not null default '',
	deptno int(5) not null default 0
) engine=innodb default charset=utf8;

//通过存储函数插入海量数据，存储过程和存储函数之间是有区别的，存储过程没有return，而存储函数是有返回值的

//分号不是结束符，下面可能需要一行一行执行
delimiter $

create function randstring(n int) returns varchar(255)
begin
	declare all_str varchar(100) default 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
	declare return_str varchar(255) default '';
	declare i int default 0;
	while i < n
	do
		set return_str = concat(return_str, substring(all_str,FLOOR(rand()*52+1),1));
		set i = i + 1;
	end while;
	return  return_str;
end $

create function randnum() returns int(5)
begin
	declare i int default 0;
	set i = FLOOR(rand()*100);
	return i;
end $

//在写个存储过程，关闭自动提交，不然每插一次数据提交一次，要提交100000次
create procedure insert_emp(in eid_start int(10), in data_times int(10))
begin
	declare i int default 0;
	set autocommit = 0;
	repeat
		insert into emp values(eid_start + i, randstring(5), "other", randnum());
		set i = i + 1;
		until i = data_times
	end repeat;
	commit;
end

create procedure insert_dept(in dno_start int(10), in data_times int(10))
begin
	declare i int default 0;
	set autocommit = 0;
	repeat
		insert into dept values(dno_start + i, randstring(6), randstring(8));
		set i = i + 1;
		until i = data_times
	end repeat;
	commit;
end

//调用，记得把delimiter改回;

delimiter ;

call insert_emp(1000, 800000);

call insert_dept(10, 30);

```

分析海量数据用profile

```html
show profiles;
```
发现没有，用下面语句检查一下，发现profiling是OFF的
```html
show variables like '%profiling%';
```

临时开启

```html
set profiling = ON;
```
然后再show profiles，就可以查看之后的查询语句了

`duration`记录语句花费的时间，但只能看到总共花费的时间，不能看到硬件花费的时间，精确分析的话，需要一个SQL诊断

```html
show profile all for query 2;
```
更精确的是看具体硬件

```html
show profile cpu,block io for query 2;
```

分析的话也可以看全局查询日志，Profile是很费性能的，调优的时候打开，在最终部署实施时关闭

```html
show variables like "%general_log%";
```
```html
set global general_log = 1;
//下面这句话也要写才能开启!!!!!将全部的记录记录在表中
set global log_output='table';
//也可以放在文件中
set global general_log_file='/opt/data/general.log'
//如果要在文件夹中显示还要再设置下，都要两步
set global log_output='file';
```
开启后就能在`mysql`数据库中的general_log中查询，这是上方的table模式，不是file模式

#### mysql中的锁

锁机制：解决因资源共享而造成的并发问题

操作类型分：
1. 读锁（共享锁）：对同一个数据的多个读操作可以同时进行，互不干扰
2. 写锁（互斥锁）：如果当前写操作没有完毕，则无法进行其他的读操作，写操作

操作范围分：
1. 表锁：一次性锁一张表，如MyISAM存储引擎使用表锁，开销小，加锁块，**无死锁**，但锁的范围大，容易发生锁冲突，并发度低
2. 行锁：一次性对一条数据加锁，使用行锁，开销大，开销大，加锁慢，容易**死锁**，不易发生锁冲突，并发度高（很小发生高并发概率：脏读，幻读，不可重复读，丢失更新等）
3. 页锁

增加锁：
```html
lock table tb1 read/write，tb2 read/write
```
释放锁：
```html
unlocak tables;
```

分析表锁定的严重程度：

```html
show status like 'table%';
```
出来两个值，`Table_locks_immediate`为可能获取到的锁数，`Table_locks_waited`为需要等待的表锁数（如果该值越大，说明存在越大的竞争），所以一般建议：Table_locks_immediate / Table_locks_waited > 5000，建议采用InnoDB引擎，否则用MyISAM

查看加锁的表：
```html
show open tables;
```

验证一些锁机制，注意下面的表是MyISAM存储引擎创建

```html
create table tablelock(
	id int(5) primary key auto_increment,
	name varchar(20) default ''
) engine=MyISAM;

insert into tablelock(name) values ('a1');
insert into tablelock(name) values ('a2');
insert into tablelock(name) values ('a3');
insert into tablelock(name) values ('a4');
insert into tablelock(name) values ('a5');
```
加读锁，开始试验

```html
lock table tablelock read;
```

Sesson1:

```html
select * from tablelock;
```

尝试写，删的时候发现报错了？？

```html
delete from tablelock where id =1;
```

尝试读别的表，之前优化创建的表，读的时候也报错了？？

```html
select * from emp;
```

肯定的，别的表，写也报错的，所以如果每个会话对A表加了read锁，则可以对A表进行读操作，不能进行写操作，该会话不能对其他表进行读或者写操作

Sesson2:

读tablelock，可以
删tablelock的数据，不行，一直等待，等待Session1将锁释放
读其他表，emp，可以的
删其他表，emp，可以的

加读锁，开始试验

```html
lock table tablelock write;
```
当前会话可以对加了写锁的表进行任何操作（增删改查），但是不能操作（增删改查）其他表
其他会话对Session1加写锁的表进行增删改查的前提是等待Session1释放写锁

MyISAM表级锁的模式：

1. MyISAM在执行查询语句前，会自动给涉及到的所有表加读锁，在执行更新操作前（DML），会自动给涉及到的表加写锁
2. 产生的情况就是对MyISAM的读操作，不会阻塞其他进程（会话）对同一表的读请求，但会阻塞对同一表的写请求，只有当读锁释放后，才会执行其他进程的写操作
3. 对MyISAM的写操作，会阻塞其他进程（会话）对同一表的读和写操作，只有当写锁释放后，才会执行其他进程的读写操作

注意下面的表示InnoDB创建，（Mysql默认自动提交，oracle默认不自动commit）

```html
create table linelock(
	id int(5) primary key auto_increment,
	name varchar(20) default ''
) engine=innodb;

insert into linelock(name) values ('a1');
insert into linelock(name) values ('a2');
insert into linelock(name) values ('a3');
insert into linelock(name) values ('a4');
insert into linelock(name) values ('a5');
```
研究行锁，要把自动commit关闭

```html
set autocommit = 0;
```
或者用start transaction 和 begin也可以

Session1:

```html
insert into linelock(id, name) values (6,'a6');
```

Session2:

```html
update linelock set name='a8' where id = 6;
```

Session1用`commit`后释放锁，Session2就可以了（其实也可以用rollback）

行锁通过事务来解锁，但是如果操作的是不同行的锁，是不影响的，行锁使用的注意事项：

1. 如果没有索引，则行锁会转为表锁
	- 如果索引列发生了类型转换，则索引失效，set name = "a6" 和 set name = a6，是有区别的，上升为表锁
2. 行锁的一种特殊情况，间隙锁，值在范围内，但却不存在
	- update linelock set name = 'x' where id > 1 and id < 9; 表里面2到8基本都有，但是没有id为7的数据，7就是间隙，Mysql会自动给间隙加锁

分析行锁用语句

```html
show status like '%innodb_row_lock%';
```
Innodb_row_lock_current_waits：当前正在等待锁的数量
Innodb_row_lock_time：从系统启动到现在，一共等待的时间
Innodb_row_lock_time_avg：平均等待的时间
Innodb_row_lock_time_max：最长一次等待的时间
Innodb_row_lock_waits：到现在等待的次数

行锁也可以对查询(select)进行加锁，就要用到`for update`

```html
select * from linelock where id = 2 for update;
```

锁这边还有要补充的就是如下：

innodb中的记录锁（也就叫行锁），间隙锁，next-key锁统统属于排他锁。

间隙锁是个什么鬼呢？编程的思想源于生活，生活中的例子能帮助我们更好的理解一些编程中的思想。生活中排队的场景，小明，小红，小花三个人依次站成一排，此时，如何让新来的小刚不能站在小红旁边，这时候只要将小红和她前面的小明之间的空隙封锁，将小红和她后面的小花之间的空隙封锁，那么小刚就不能站到小红的旁边。这里的小红，小明，小花，小刚就是数据库的一条条记录。他们之间的空隙也就是间隙，而封锁他们之间距离的锁，叫做间隙锁。

间隙锁的目的是为了防止幻读，其主要通过两个方面实现这个目的：

1、防止间隙内有新数据被插入
2、防止已存在的数据，更新成间隙内的数据（例如防止numer=3的记录通过update变成number=5）

innodb自动使用间隙锁的条件：
1、必须在RR级别下
2、检索条件必须有索引（没有索引的话，mysql会全表扫描，那样会锁定整张表所有的记录，包括不存在的记录，此时其他事务不能修改不能删除不能添加）

#### SQL注入

SQL注入单独说一下，SQL注入是比较常见的网络攻击方式之一，它不是利用操作系统的BUG来实现攻击，而是针对程序员编写时的疏忽，通过SQL语句，实现无账号登录，甚至篡改数据库。

SQL注入的总体思路：

1、寻找到SQL注入的位置
2、判断服务器类型和后台数据库类型
3、针对不同的服务器和数据库特点进行SQL注入攻击

实例：

```SQL
String sql = "select * from user_table where username=
' "+userName+" ' and password=' "+password+" '";

--当输入了上面的用户名和密码，上面的SQL语句变成：
SELECT * FROM user_table WHERE username=
'’or 1 = 1 -- and password='’

"""
--分析SQL语句：
--条件后面username=”or 1=1 用户名等于 ” 或1=1 那么这个条件一定会成功；

--然后后面加两个-，这意味着注释，它将后面的语句注释，让他们不起作用，这样语句永远都--能正确执行，用户轻易骗过系统，获取合法身份。
--这还是比较温柔的，如果是执行
SELECT * FROM user_table WHERE
username='' ;DROP DATABASE (DB Name) --' and password=''
--其后果可想而知…
"""
```

如何防御SQL注入

但凡有SQL注入漏洞的程序，都是因为程序要接受来自客户端用户输入的变量或URL传递的参数，并且这个变量或参数是组成SQL语句的一部分，对于用户输入的内容或传递的参数，我们应该要时刻保持警惕，这是安全领域里的「外部数据不可信任」的原则，纵观Web安全领域的各种攻击方式，大多数都是因为开发者违反了这个原则而导致的，所以自然能想到的，就是从变量的检测、过滤、验证下手，确保变量是开发者所预想的。

1、检查变量数据类型和格式

2、过滤特殊符号

3、绑定变量，使用预编译语句　　

通常我们的一条sql在db接收到最终执行完毕返回可以分为下面三个过程：

1、词法和语义解析
2、优化sql语句，制定执行计划
3、执行并返回结果

我们把这种普通语句称作Immediate Statements。　　

但是很多情况，我们的一条sql语句可能会反复执行，或者每次执行的时候只有个别的值不同（比如query的where子句值不同，update的set子句值不同,insert的values值不同）。
如果每次都需要经过上面的词法语义解析、语句优化、制定执行计划等，则效率就明显不行了。所谓预编译语句就是将这类语句中的值用占位符替代，可以视为将sql语句模板化或者说参数化，一般称这类语句叫Prepared Statements或者Parameterized Statements（想想JDBC吧），预编译语句的优势在于归纳为：一次编译、多次运行，省去了解析优化等过程；此外预编译语句能防止sql注入。

注意MySQL的老版本（4.1之前）是不支持服务端预编译的，但基于目前业界生产环境普遍情况，基本可以认为MySQL支持服务端预编译。

PREPARE的语法来预编译一条SQL语句
EXECUTE的语法来执行预编译语句

MySQL中的预编译语句作用域是session级，但我们可以通过max_prepared_stmt_count变量来控制全局最大的存储的预编译语句

如果我们想要释放一条预编译语句，则可以使用deallocate

还有就是要知道原理：

PrepareStatement可以防止sql注入

原理是采用了预编译的方法，先将SQL语句中可被客户端控制的参数集进行编译，生成对应的临时变量集，再使用对应的设置方法，为临时变量集里面的元素进行赋值，赋值函数setString()，会对传入的参数进行强制类型检查和安全检查，所以就避免了SQL注入的产生。下面具体分析

mybatis是如何防止SQL注入的　

就是#和$了，#将传入的数据都当成一个字符串，会对自动传入的数据加一个双引号。$将传入的数据直接显示生成在sql中，没有双引号

所以#方式能够很大程度防止sql注入，$方式无法防止Sql注入。

MyBatis框架作为一款半自动化的持久层框架，其SQL语句都要我们自己手动编写，这个时候当然需要防止SQL注入。其实，MyBatis的SQL是一个具有“输入+输出”的功能，类似于函数的结构，参考上面的两个例子。其中，parameterType表示了输入的参数类型，resultType表示了输出的参数类型。回应上文，如果我们想防止SQL注入，理所当然地要在输入参数上下功夫。上面代码中使用#的即输入参数在SQL中拼接的部分，传入参数后，打印出执行的SQL语句，会看到SQL是这样的：

```SQL
select id, username, password, role from user where username=? and password=?
```
不管输入什么参数，打印出的SQL都是这样的。这是因为MyBatis启用了预编译功能，在SQL执行前，会先将上面的SQL发送给数据库进行编译；执行时，直接使用编译好的SQL，替换占位符“?”就可以了。因为SQL注入只能对编译过程起作用，所以这样的方式就很好地避免了SQL注入的问题。

【底层实现原理】MyBatis是如何做到SQL预编译的呢？其实在框架底层，是JDBC中的PreparedStatement类在起作用，PreparedStatement是我们很熟悉的Statement的子类，它的对象包含了编译好的SQL语句。这种“准备好”的方式不仅能提高安全性，而且在多次执行同一个SQL时，能够提高效率。原因是SQL已编译好，再次执行时无需再编译


### mysql的运维

三台机器，某一次卡机后，恢复完机器，需要恢复mysql集群


```html
# systemctl restart mysql
```
在三台机器中尝试都不起作用，停掉集群


```html
# ansible -i ansible/hosts all -m shell -a 'systemctl stop mysql'
```

停到集群后将第一台节点配置改下,设置为可以启动

```html
# vi /var/lib/mysql/grastate.dat
```
然后以集群方式进行启动
```html
# mysqld --wsrep-new-cluster --user=root &
```
将另两台机器加进来，停掉第一个节点的集群启动方式，用正常方式启动

```html
# systemctl restart mysql
```

### mysql的使用技巧

1.多表查询转换成子查询

连表查询是可以转换成子查询的，子查询会先执行内存，那么explain的时候就会有多个ID，ID值越大越优先执行，如果有子查询也有连表查询，先看ID，在看从上往下

2.explain的使用

**select_type**

Primary: 包含子查询的主查询
Subquery：包含子查询的子查询
Simple：简单查询，不包含子查询和union
derived：衍生查询，在查询时用到了衍生表，a.在from子查询中只有一张表，b.在from子查询中，如果有table1 union table2，则table1是衍生表
union：上面例子的table2就是union
union result：有哪些是union的了

**type**

system > const > eq_ref > ref > range > index > all越往左效率越高，实际能达到的优化到ref及range就已经差不多了，没优化基本是all

system：只有一条数据的系统表，或只有一条数据的衍生表
const: 仅仅能查出一条数据的SQL，用于primary key和unique索引，**一般索引是不会生效的**，比如create index test_01 on table(id)
eq_ref: 唯一性索引，对于每个索引键的查询，返回匹配唯一行的数据（有且只有1个，不能多，**不能0**），**常见**于主键索引或唯一索引

ref和eq_ref的区别，这个还是明显的，上面说的这个0，如果不满足就退化到ref了，每条数据都有对应值则是eq_ref

ref: 非唯一性索引，返回匹配的所有行(0，多)，where后面跟的索引有多个值

```html
show index from tableName
show create table tableName
alter table tableName add constraint pk_id primary key(id);
alter table tableName add index index_name(tname);
```
range: 检索指定范围内的行，where后面是个范围查询(between，in(和数据有关，大于一半就会全表扫描)，>，<，>=)，前提也是有索引

index: 查询全部索引中数据，只将索引查了一遍

all: 查询全部表数据，全部数据，不只是索引列

**possible_keys**

可能用到的索引

**key**

实际用到的索引

**key_len**

用于判断复合索引是否被完全使用，(a,b,c)的复合索引，是只用了a，b，还是一起用了，通过这个字段可以判断。**utf-8是一个字符3个字节**，长度20的话，key_len就是60

**ref**

指明当前表所参照的字段，where name = "cw"，如果是这样写法，就是常量const，如果是c.name = t.name，那么就会展示成database.t.name这样

**rows**

被索引优化查询的数据个数，记得是看前面的table对应的表所匹配的行数

**extra**

using filesort: 性能消耗比较大，需要额外一次的排序（查询），常见于order by语句 select * from intances where name = "" order by id; 如果索引跨列，也会出现这种情况（a1，a2，a3），如果之间跳过了a2（就是索引的最佳左前缀）

select * from instances where a1="" order by a2; 这样是没有using filesort的
select * from instances where a1="" order by a3; select * from instances where a2="" order by a3; 这两种就会有，**这里一定要注意where跟order by是拼起来才算跨不跨列的**

where哪些字段就order by哪些字段

数据库的表要永久保存，必须在磁盘中啊，

filesort有双路排序（两次IO，扫描两次磁盘，第一次从磁盘中读取排序字段（比如主键ID），第二次扫描其他字段）和单路排序（一次IO，buffer中进行排序）

using temporary: 性能损耗也很大，用到了临时表，跟using filesort一样需要优化，一般出现在group by语句中，根据a1查，也要根据a1进行分组

select a1 from instances where a1 in () group by a1;
select a1 from instances where a1 in () group by a2; 这样就会出现using temporary

using index: 性能提升，索引覆盖，不读取原文件，只从索引文件中获取数据（不需要回表查询），只要使用到的列全在索引中，就是索引覆盖。

using index会对possible_keys和key有影响，前提是有索引覆盖，如果没有where，则索引只出现在key中，如果有where，则索引出现在key和possible_keys中

using where: 需要回表查询

impossible where: where子句永远为false，where a1 = "1" and a1 = "2"

using join buffer: 使用了缓存，mysql认为SQL写的太差，底层给你优化了

**SQL优化器**里面有办法就先优化你的SQL了，虽然编写的顺序和索引顺序不一致，但SQL优化器帮你进行优化了

3.union和union all的使用

先做好例子
```mysql
DROP TABLE IF EXISTS `ta`;
CREATE TABLE `ta` (
 `id` varchar(255) DEFAULT NULL,
 `num` int(11) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of ta
-- ----------------------------
INSERT INTO `ta` VALUES ('a', '5');
INSERT INTO `ta` VALUES ('b', '10');
INSERT INTO `ta` VALUES ('c', '15');
INSERT INTO `ta` VALUES ('d', '10');

-- ----------------------------
-- Table structure for `tb`
-- ----------------------------
DROP TABLE IF EXISTS `tb`;
CREATE TABLE `tb` (
 `id` varchar(255) DEFAULT NULL,
 `num` int(11) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of tb
-- ----------------------------
INSERT INTO `tb` VALUES ('b', '5');
INSERT INTO `tb` VALUES ('c', '15');
INSERT INTO `tb` VALUES ('d', '20');
INSERT INTO `tb` VALUES ('e', '99');
```
注意，这里面只有c为key的记录是所有字段都一样的

```mysql
SELECT id,SUM(num) FROM (
  SELECT * FROM ta
    UNION ALL
  SELECT * FROM tb) as tmp
  GROUP BY id
```

```mysql
SELECT id,SUM(num) FROM (
  SELECT * FROM ta
    UNION
  SELECT * FROM tb) as tmp
  GROUP BY id
```

使用Union，则所有返回的行都是唯一的，如同您已经对整个结果集合使用了DISTINCT,若果多表查询结果中有完全一致的数据，mysql将自动去重，使用Union all，则不会排重，返回所有的行，从效率上说，UNION ALL 要比UNION快很多，所以，如果可以确认合并的两个结果集中不包含重复的数据的话，那么就使用UNION ALL

4.mysql中join的使用

先造点数据

```mysql
create table tab1 (
	id int,
    size int
);

create table tab2 (
	size int,
    name varchar(20)
);

insert into tab1 values(1, 10);
insert into tab1 values(2, 20);
insert into tab1 values(3, 30);

insert into tab2 values(10, 'AAA');
insert into tab2 values(20, 'BBB');
insert into tab2 values(20, 'CCC');

```

内连接和外连接有区别，inner join又叫等值连接，只返回两个表中连接字段相等的行。外连接，left join，right join，full join

left join 和 left outer join 区别， 简单来说就是没有区别，只是写法不同

on 和 where 条件的区别 （只对外连接有区别）

数据库在通过连接两张或多张表来返回记录时，都会生成一张中间的临时表，然后再将这张临时表返回给用户。

1、on 条件是在生成临时表时使用的条件，它不管 on 中的条件是否为真，都会返回左边表中的记录。
2、where 条件是在临时表生成好后，再对临时表进行过滤的条件。这时已经没有 left join 的含义（必须返回左边表的记录）了，条件不为真的就全部过滤掉。

对于内连接没区别，因为内连接生成的临时表中只会保留符合on条件的数据，所以数据在 on 和 where 条件中过滤没区别。
对于外连接（左连接，右连接，全连接）有区别，外连接生成的临时表中会保留不符合on条件的数据，对于这些数据，在 on 和 where 条件中过滤就区别了。

```mysql
-- 4条记录，left join保证左边的记录都存在
select * from tab1 left join tab2 on tab1.size = tab2.size;

-- 3条记录
select * from tab1 join tab2 on tab1.size = tab2.size;

-- 1条记录
select * from tab1 left join tab2 on (tab1.size = tab2.size) where tab2.name='AAA';
-- 3条记录
select * from tab1 left join tab2 on (tab1.size = tab2.size and tab2.name='AAA');

-- 1条记录，内联 where和on都一样
-- 而外连接生成的临时表中会保留不符合on条件的数据，对于这些数据，在 on 和 where 条件中过滤就区别了
select * from tab1 join tab2 on (tab1.size = tab2.size) where tab2.name='AAA';
-- 1条记录
-- 因为内连接生成的临时表中只会保留符合on条件的数据，所以数据在 on 和 where 条件中过滤没区别
select * from tab1 join tab2 on (tab1.size = tab2.size and tab2.name='AAA');
```

5.mysql中如何把别的表的数据更新过来

```html
update table_2 m  set m.column = (select column from table_1 mp where mp.id= m.id);

update table_1 t1,table_2 t2 set t1.column = t2.column where t1.id = t2.pid;

```

另外，如何备份数据

```html

show create table xxxx // 创建备份表

insert into new_table select * from old_table where id = 2;

```

还有的方式是用export的按钮，然后选择 SQL INSERT statements

6.mysql中是否存储图片

mysql中一张表的数据是全部在一个数据文件中的。如果大字段的数据也存储在里面。程序展示列表，比如文章列表。这个时候根本不需要展示文章内容的。
但是仍然会影响速度，数据库查找数据其实就是扫描那个数据文件，文件容量越小，速度就会越快(为什么单表的容量在1g-2g的时候基本上要分表了)。拆分出去到一张单独的表，就是单独的文件了


三种东西永远不要放到数据库里,图片，文件，二进制数据。作者的理由是，

a、对数据库的读/写的速度永远都赶不上文件系统处理的速度
b、数据库备份变的巨大，越来越耗时间
c、对文件的访问需要穿越你的应用层和数据库层

select
table_schema as '数据库',
table_name as '表名',
table_rows as '记录数',
truncate(data_length/1024/1024, 2) as '数据容量(MB)',
truncate(index_length/1024/1024, 2) as '索引容量(MB)'
from information_schema.tables
order by data_length desc, index_length desc;

给自己行个方便吧，在数据库里只简单的存放一个磁盘上你的文件的相对路径，或者使用S3（备注：亚马逊云服务）或CDN之类的服务。

起初是想存个图片：

操作系统对单个目录的文件数量是有限制的。当文件数量很多的时候。从目录中获取文件的速度就会越来越慢。所以为了保持速度，才要按照固定规则去分散到多个目录中去。
图片分散到磁盘路径中去。数据库字段中保存的是类似于这样子的”images/2012/09/25/1343287394783.jpg”
并发访问量越大。就越精确就好了。

有个方面总结一下：为什么保存的磁盘路径，是”images/2012/09/25/1343287394783.jpg”，而不是” /images/2012/09/25/ 1343287394783.jpg”(最前面带有斜杠)

我的理解：

连那个斜杠都不要。这里也是做到方便以后系统扩展。

在页面中需要取出图片路径展示图片的时候，如果是相对路径，则可以使用”./”+”images/2012/09/25/1343287394783.jpg”进行组装。

如果需要单独的域名(比如做cdn加速的时候)域名，img1.xxx.com,img2.xxx.com这样的域名，

直接组装 “http://img1.xxx.com/”+”images/2012/09/25/1343287394783.jpg”

当然数据库是可以在前面加斜杠/保存起来,/images/2012/09/25/ 1343287394783.jpg

其实不方便统一。比如相对路径载入图片的时候，则是”.”+” /images/2012/09/25/ 1343287394783.jpg”

可能我还没体会到坏处，以后会遇到问题的。不过，遵循惯例不加斜杠” images/2012/09/25/ 1343287394783.jpg”就对了。

cdn，我理解其本质就是为了解决距离远产生的速度问题，使用就近的服务。

从中国请求美国一台服务器上的图片。一般比较慢，因为距离这么远，网络传输是存在损耗的，距离越远，传输的时间就越长。一般会看到浏览器左下角显示：“已响应,正在传输数据..”。这不是服务器本身问题了。实际上服务器早就响应请求，把数据发给客户端，但是网络问题，就一直在传输，没传完了。

一般的公司自己搭建cdn网络成本高，所以就有商业的cdn提供付费租用服务，这是一项很成熟的业务，很多这样的公司，大部分全国性的互联网公司都会使用到cdn。

总结：cdn服务。对于静态内容是非常适合的。所以像商品图片，随着访问量大了后，租用cdn服务，只需要把图片上传到他们的服务器上去。

曾经看过这个，这个是比较适合创业公司的。价格相对便宜

https://www.upyun.com/

7.Mysql中utf8和utf8mb4区别

一、介绍

MySQL在5.5.3之后增加了这个utf8mb4的编码，mb4就是most bytes 4的意思，专门用来兼容四字节的unicode。好在utf8mb4是utf8的超集，除了将编码改为utf8mb4外不需要做其他转换。当然，为了节省 空间，一般情况下使用utf8也就够了。

二、内容描述
那上面说了既然utf8能够存下大部分中文汉字,那为什么还要使用utf8mb4呢? 原来mysql支持的 utf8 编码最大字符长度为 3 字节，如果遇到 4 字节的宽字符就会插入异常了。三个字节的 UTF-8 最大能编码的 Unicode 字符是 0xffff，也就是 Unicode 中的基本多文种平面（BMP）。也就是说，任何不在基本多文本平面的 Unicode字符，都无法使用 Mysql 的 utf8 字符集存储。包括 Emoji 表情（Emoji 是一种特殊的 Unicode 编码，常见于 ios 和 android 手机上），和很多不常用的汉字，以及任何新增的 Unicode 字符等等。

三、问题根源

最初的 UTF-8 格式使用一至六个字节，最大能编码 31 位字符。最新的 UTF-8 规范只使用一到四个字节，最大能编码21位，正好能够表示所有的 17个 Unicode 平面。
utf8 是 Mysql 中的一种字符集，只支持最长三个字节的 UTF-8字符，也就是 Unicode 中的基本多文本平面。
Mysql 中的 utf8 为什么只支持持最长三个字节的 UTF-8字符呢？我想了一下，可能是因为 Mysql 刚开始开发那会，Unicode 还没有辅助平面这一说呢。那时候，Unicode 委员会还做着 “65535 个字符足够全世界用了”的美梦。Mysql 中的字符串长度算的是字符数而非字节数，对于 CHAR 数据类型来说，需要为字符串保留足够的长。当使用 utf8 字符集时，需要保留的长度就是 utf8 最长字符长度乘以字符串长度，所以这里理所当然的限制了 utf8 最大长度为 3，比如 CHAR(100) Mysql 会保留 300字节长度。至于后续的版本为什么不对 4 字节长度的 UTF-8 字符提供支持，我想一个是为了向后兼容性的考虑，还有就是基本多文种平面之外的字符确实很少用到。
要在 Mysql 中保存 4 字节长度的 UTF-8 字符，需要使用 utf8mb4 字符集，但只有 5.5.3 版本以后的才支持（查看版本： select version();）。我觉得，为了获取更好的兼容性，应该总是使用 utf8mb4 而非 utf8. 对于 CHAR 类型数据，utf8mb4 会多消耗一些空间，根据 Mysql 官方建议，使用 VARCHAR 替代 CHAR


8.varchar到底是多长

4.0版本以下，varchar(20)，指的是20字节，如果存放UTF8汉字时，只能存6个（每个汉字3字节）

5.0版本以上，varchar(20)，指的是20字符，无论存放的是数字、字母还是UTF8汉字（每个汉字3字节），都可以存放20个，最大大小是65532字节

注意是字符，但是大小又是按字节算的

如果是varchar(200)的大小，是200字符，那么就是字母如a，或者汉字"你"，都是能存储200个的，但是要注意，一个汉字是3个字节，所以当使用length和char_length的时候是有区别的

length(字母)  200

char_length(字母)  200

length(汉字)  600 这个是600哦，不要觉得设置的是varchar(200)，但length为什么出了600

char_length(汉字) 200

但是varchar的大小又是按照字节算的，如果全部存汉字，长度设置20000+差不多了，又是用行的，谨慎


9、一些小技巧

```html
// 改一部分的数据

insert into xxx_weight_xxx (
batch_id,
service_id,
data_code,
weight_year,
weight_season,
business_type,
business_weight,
sub_bu_weight,
father_business,
d_year,
d_quarter
) select 46, `service_id`, data_code, weight_year, weight_season, business_type, business_weight, sub_bu_weight, father_business, 2023, 'Q1' from xxx_weight_xxx where d_year = 2022 and d_quarter = "Q3";

```
