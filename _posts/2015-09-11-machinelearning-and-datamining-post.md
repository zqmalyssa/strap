---
layout: post
title: 机器学习与数据挖掘
tags: [code, algorithm, machinelearning, sad]
author-id: zqmalyssa
---

机器学习与数据挖掘，啥也不说了

reference:

#### 大数据

大数据本身是个很宽泛的概念，Hadoop生态圈(或者泛生态圈)基本上都是为了处理超过单机尺度的数据处理而诞生的。

大数据，首先你要能存的下大数据！！！传统的文件系统是单机的，不能横跨不同的机器。HDFS(Hadoop Distributed FileSystem)的设计本质上是为了大量的数据能横跨成百上千台机器，但是你看到的是一个文件系统而不是很多文件系统。比如你说我要获取/hdfs/tmp/file1的数据，你引用的是一个文件路径，但是实际的数据存放在很多不同的机器上。你作为用户，不需要知道这些，就好比在单机上你不关心文件分散在什么磁道什么扇区一样。HDFS为你管理这些数据。

存的下数据之后，你就开始考虑怎么处理数据。虽然HDFS可以为你整体管理不同机器上的数据，但是这些数据太大了。一台机器读取成T上P的数据(很大的数据哦，比如整个东京热有史以来所有高清电影的大小甚至更大)，一台机器慢慢跑也许需要好几天甚至好几周。对于很多公司来说，单机处理是不可忍受的，比如微博要更新24小时热博，它必须在24小时之内跑完这些处理。那么我如果要用很多台机器处理，我就面临了如何分配工作，如果一台机器挂了如何重新启动相应的任务，机器之间如何互相通信交换数据以完成复杂的计算等等。这就是MapReduce / Tez / Spark的功能。

MapReduce是第一代计算引擎，Tez和Spark是第二代。MapReduce的设计，采用了很简化的计算模型，只有Map和Reduce两个计算过程(中间用Shuffle串联)，用这个模型，已经可以处理大数据领域很大一部分问题了。

就来说说MapReduce！！考虑如果你要统计一个巨大的文本文件存储在类似HDFS上，你想要知道这个文本里各个词的出现频率。你启动了一个MapReduce程序。Map阶段，几百台机器同时读取这个文件的各个部分，分别把各自读到的部分分别统计出词频，产生类似(hello, 12100次)，(world，15214次)等等这样的Pair(我这里把Map和Combine放在一起说以便简化);这几百台机器各自都产生了如上的集合，然后又有几百台机器启动Reduce处理。Reducer机器A将从Mapper机器收到所有以A开头的统计结果，机器B将收到B开头的词汇统计结果(当然实际上不会真的以字母开头做依据，而是用函数产生Hash值以避免数据串化。因为类似X开头的词肯定比其他要少得多，而你不希望数据处理各个机器的工作量相差悬殊)。然后这些Reducer将再次汇总，(hello，12100)+(hello，12311)+(hello，345881)= (hello，370292)。每个Reducer都如上处理，你就得到了整个文件的词频结果。

Map+Reduce的简单模型很黄很暴力，虽然好用，但是很笨重。第二代的Tez和Spark除了内存Cache之类的新feature，本质上来说，是让Map/Reduce模型更通用，让Map和Reduce之间的界限更模糊，数据交换更灵活，更少的磁盘读写，以便更方便地描述复杂算法，取得更高的吞吐量。

有了MapReduce，Tez和Spark之后，程序员发现，MapReduce的程序写起来真麻烦。他们希望简化这个过程。这就好比你有了汇编语言，虽然你几乎什么都能干了，但是你还是觉得繁琐。你希望有个更高层更抽象的语言层来描述算法和数据处理流程。于是就有了Pig和Hive。Pig是接近脚本方式去描述MapReduce，Hive则用的是SQL。它们把脚本和SQL语言翻译成MapReduce程序，丢给计算引擎去计算，而你就从繁琐的MapReduce程序中解脱出来，用更简单更直观的语言去写程序了。（这边就提到了Hive）

有了Hive之后，人们发现SQL对比Java有巨大的优势。一个是它太容易写了。刚才词频的东西，用SQL描述就只有一两行，MapReduce写起来大约要几十上百行。而更重要的是，非计算机背景的用户终于感受到了爱：我也会写SQL!于是数据分析人员终于从乞求工程师帮忙的窘境解脱出来，工程师也从写奇怪的一次性的处理程序中解脱出来。大家都开心了。Hive逐渐成长成了大数据仓库的核心组件。甚至很多公司的流水线作业集完全是用SQL描述，因为易写易改，一看就懂，容易维护。

自从数据分析人员开始用Hive分析数据之后，它们发现，Hive在MapReduce上跑，真鸡巴慢!流水线作业集也许没啥关系，比如24小时更新的推荐，反正24小时内跑完就算了。但是数据分析，人们总是希望能跑更快一些。比如我希望看过去一个小时内多少人在充气娃娃页面驻足，分别停留了多久，对于一个巨型网站海量数据下，这个处理过程也许要花几十分钟甚至很多小时。而这个分析也许只是你万里长征的第一步，你还要看多少人浏览了跳蛋多少人看了拉赫曼尼诺夫的CD，以便跟老板汇报，我们的用户是猥琐男闷骚女更多还是文艺青年/少女更多。你无法忍受等待的折磨，只能跟帅帅的工程师蝈蝈说，快，快，再快一点!

于是Impala，Presto，Drill诞生了(当然还有无数非著名的交互SQL引擎，就不一一列举了)。三个系统的核心理念是，MapReduce引擎太慢，因为它太通用，太强壮，太保守，我们SQL需要更轻量，更激进地获取资源，更专门地对SQL做优化，而且不需要那么多容错性保证(因为系统出错了大不了重新启动任务，如果整个处理时间更短的话，比如几分钟之内)。这些系统让用户更快速地处理SQL任务，牺牲了通用性稳定性等特性。如果说MapReduce是大砍刀，砍啥都不怕，那上面三个就是剔骨刀，灵巧锋利，但是不能搞太大太硬的东西。

这些系统，说实话，一直没有达到人们期望的流行度。因为这时候有两个异类被造出来了。他们是Hive on Tez / Spark和SparkSQL。它们的设计理念是，MapReduce慢，但是如果我用新一代通用计算引擎Tez或者Spark来跑SQL，那我就能跑的更快。而且用户不需要维护两套系统。这就好比如果你厨房小，人又懒，对吃的精细程度要求有限，那你可以买个电饭煲，能蒸能煲能烧，省了好多厨具。（这边就是Hive on spark和Spark SQL）

上面的介绍，基本就是一个数据仓库的构架了。底层HDFS，上面跑MapReduce/Tez/Spark，在上面跑Hive，Pig。或者HDFS上直接跑Impala，Drill，Presto。这解决了中低速数据处理的要求。

##### 大数据之hadoop/hive/hbase的区别

hadoop：它是一个分布式计算+分布式文件系统，前者其实就是 MapReduce，后者是 HDFS 。后者可以独立运行，前者可以选择性使用，也可以不使用

补充：HDFS是Hadoop的三大核心组件之一

hive：通俗的说是一个数据仓库，仓库中的数据是被hdfs管理的数据文件，它支持类似sql语句的功能，你可以通过该语句完成分布式环境下的计算功能，hive会把语句转换成MapReduce，然后交给hadoop执行。这里的计算，仅限于查找和分析，而不是更新、增加和删除。

它的优势是对历史数据进行处理，用时下流行的说法是离线计算，因为它的底层是MapReduce，MapReduce在实时计算上性能很差。它的做法是把数据文件加载进来作为一个hive表（或者外部表），让你觉得你的sql操作的是传统的表。

补充：Hive本身不存储和计算数据，它完全依赖于HDFS和MapReduce（也就是hadoop），Hive中的表纯逻辑。hive需要用到hdfs存储文件，需要用到MapReduce计算框架。hive可以认为是map-reduce的一个包装。hive的意义就是把好写的hive的sql转换为复杂难写的map-reduce程序。

hbase：通俗的说，hbase的作用类似于数据库，传统数据库管理的是集中的本地数据文件，而hbase基于hdfs实现对分布式数据文件的管理，比如增删改查。也就是说，hbase只是利用hadoop的hdfs帮助其管理数据的持久化文件（HFile），它跟MapReduce没任何关系。

hbase的优势在于实时计算，所有实时数据都直接存入hbase中，客户端通过API直接访问hbase，实现实时计算。由于它使用的是nosql，或者说是列式结构，从而提高了查找性能，使其能运用于大数据场景，这是它跟MapReduce的区别。

补充：hbase是物理表，不是逻辑表，提供一个超大的内存hash表，搜索引擎通过它来存储索引，方便查询操作，hbase可以认为是hdfs的一个包装。他的本质是数据存储，是个NoSql数据库；hbase部署于hdfs之上，并且克服了hdfs在随机读写方面的缺点。

所以，hadoop是hive和hbase的基础，hive依赖hadoop，而hbase仅依赖hadoop的hdfs模块。hive适用于离线数据的分析，操作的是通用格式的（如通用的日志文件）、被hadoop管理的数据文件，它支持类sql，比编写MapReduce的java代码来的更加方便，它的定位是数据仓库，存储和分析历史数据。hbase适用于实时计算，采用列式结构的nosql，操作的是自己生成的特殊格式的HFile、被hadoop管理的数据文件，它的定位是数据库，或者叫DBMS。

Hbase和Hive在大数据架构中处在不同位置，Hbase主要解决实时数据查询问题，Hive主要解决数据处理和计算问题，一般是配合使用。

在大数据架构中，Hive和HBase是协作关系，数据流一般如下图：

1、通过ETL工具将数据源抽取到HDFS存储；
2、通过Hive清洗、处理和计算原始数据；
3、HIve清洗处理后的结果，如果是面向海量数据随机查询场景的可存入Hbase，如果不是就是直接提供查询了
4、数据应用从HBase查询数据；

这边讲的还是比较清晰的，注意定位，作用和依赖的东西就行了

#### hadoop

Hadoop与Google一样，都是小孩命名的，是一个虚构的名字，没有特别的含义。从计算机专业的角度看，Hadoop是一个分布式系统基础架构，由Apache基金会开发。Hadoop的主要目标是对分布式环境下的“大数据”以一种可靠、高效、可伸缩的方式处理。

Hadoop框架透明地为应用提供可靠性和数据移动。它实现了名为MapReduce的编程范式：应用程序被分割成许多小部分，而每个部分都能在集群中的任意节点上执行或重新执行。

Hadoop的核心组件分为：HDFS(分布式文件系统)、MapRuduce(分布式运算编程框架)、YARN(运算资源调度系统)，还有一个Hadoop Common(为其他Hadoop模块提供基础设施)

（数据量达到500G就可以考虑使用大数据处理了）

Apache Hadoop起源：

Apache Lucene
开源的高性能全文检索工具包

Apache Nutch
开源的Web搜索引擎

Google三大论文
MapReduce/GFS/BigTable

Apache Hadoop
大规模数据处理

#### HDFS

整个Hadoop的体系结构主要是通过HDFS(Hadoop分布式文件系统)来实现对分布式存储的底层支持，并通过MR来实现对分布式并行任务处理的程序支持。

HDFS是Hadoop体系中数据存储管理的基础。它是一个高度容错的系统，能检测和应对硬件故障，用于在低成本的通用硬件上运行。HDFS简化了文件的一致性模型，通过流式数据访问，提供高吞吐量应用程序数据访问功能，适合带有大型数据集的应用程序。

HDFS采用主从(Master/Slave)结构模型，一个HDFS集群是由一个NameNode和若干个DataNode组成的。NameNode作为主服务器，管理文件系统命名空间和客户端对文件的访问操作。DataNode管理存储的数据。HDFS支持文件形式的数据。

从内部来看，文件被分成若干个数据块，这若干个数据块存放在一组DataNode上。NameNode执行文件系统的命名空间，如打开、关闭、重命名文件或目录等，也负责数据块到具体DataNode的映射。DataNode负责处理文件系统客户端的文件读写，并在NameNode的统一调度下进行数据库的创建、删除和复制工作。NameNode是所有HDFS元数据的管理者，用户数据永远不会经过NameNode。

分析：NameNode是管理者，DataNode是文件存储者、Client是需要获取分布式文件系统的应用程序。

NameNode
◆Namenode 是一个中心服务器，单一节点（简化系统的设计和实现），负责管理文件系统的名字空间(namespace)以及客户端对文件的访问。
◆文件操作，NameNode 负责文件元数据的操作，DataNode负责处理文件内容的读写请求，跟 文件内容相关的数据流不经过NameNode，只会询问它跟那个DataNode联系，否则 NameNode会成为系统的瓶颈。
◆副本存放在哪些DataNode上由 NameNode来控制，根据全局情况做出块放置决定，读取文件时NameNode尽量让用户先读取最近的副本，降低带块消耗和读取延时；
◆Namenode 全权管理数据块的复制，它周期性地从集群中的每个Datanode接收心跳信号和块状态报告(Blockreport)。接收到心跳信号意味着该Datanode节点工作正常。块状态报告包含了一个该Datanode上所有数据块的列表。

DataNode
◆一个数据块在DataNode以文件存储在磁盘上，包括两个文件，一个是数据本身，一个是元数据包括数据块的长度，块数据的校验和，以及时间戳 ；
◆DataNode启动后向NameNode注册，通过后，周期性（1小时）的向NameNode上报所有的块信息；
◆心跳是每3秒一次，心跳返回结果带有NameNode给该DataNode的命令如复制块数据到另一台机器，或删除某个数据块。如果超过10分钟没有收到某个DataNode 的心跳，则认为该节点不可用；
◆集群运行中可以安全加入和退出一些机器。

文件
◆文件切分成块（默认大小128M），以块为单位，每个块有多个副本存储在不同的机器上，副本数可在文件生成时指定（默认3，是在hdfs-site.xml中配置的）；
◆NameNode 是主节点，存储文件的元数据如文件名，文件目录结构，文件属性（生成时间,副本数,文件权限），以及每个文件的块列表以及块所在的DataNode等等；
◆DataNode 在本地文件系统存储文件块数据，以及块数据的校验和。
◆可以创建、删除、移动或重命名文件，当文件创建、写入和关闭之后不能修改文件内容。


#### MapReduce(MR)

Hadoop MapReduce是google MapReduce 克隆版。

MapReduce是一种计算模型，用以进行大数据量的计算。其中Map对数据集上的独立元素进行指定的操作，生成键-值对形式中间结果。Reduce则对中间结果中相同“键”的所有“值”进行规约，以得到最终结果。MapReduce这样的功能划分，非常适合在大量计算机组成的分布式并行环境里进行数据处理。

(1)JobTracker

JobTracker叫作业跟踪器，运行到主节点(Namenode)上的一个很重要的进程，是MapReduce体系的调度器。用于处理作业(用户提交的代码)的后台程序，决定有哪些文件参与作业的处理，然后把作业切割成为一个个的小task，并把它们分配到所需要的数据所在的子节点。

Hadoop的原则就是就近运行，数据和程序要在同一个物理节点里，数据在哪里，程序就跑去哪里运行。这个工作是JobTracker做的，监控task，还会重启失败的task(于不同的节点)，每个集群只有唯一一个JobTracker，类似单点的NameNode，位于Master节点

(2)TaskTracker

TaskTracker叫任务跟踪器，MapReduce体系的最后一个后台进程，位于每个slave节点上，与datanode结合(代码与数据一起的原则)，管理各自节点上的task(由jobtracker分配)，每个节点只有一个tasktracker，但一个tasktracker可以启动多个JVM，运行Map Task和Reduce Task;并与JobTracker交互，汇报任务状态，Map Task：解析每条数据记录，传递给用户编写的map(),并执行，将输出结果写入本地磁盘(如果为map-only作业，直接写入HDFS)。

Reducer Task：从Map Task的执行结果中，远程读取输入数据，对数据进行排序，将数据按照分组传递给用户编写的reduce函数执行。


（基于磁盘IO进行迭代，开销较大）

#### Yarn

（主要是负责硬件资源的合理调用）

◆ YARN 总体上仍然是Master/Slave 结构，在整个资源管理框架中，ResourceManager 为Master，NodeManager 为Slave。
◆ ResourceManager 负责对各个NodeManager 上的资源进行统一管理和调度；
◆ 当用户提交一个应用程序时，需要提供一个用以跟踪和管理这个程序的ApplicationMaster（主管进程），它负责向ResourceManager 申请资源，并要求NodeManger 启动可以占用一定资源的任务。
◆ 由于不同的ApplicationMaster 被分布到不同的节点上，因此它们之间不会相互影响。

ResourceManager
◆ 全局的资源管理器，整个集群只有一个，负责集群资源的统一管理和调度分配。
◆ 功能
- 处理客户端请求
- 启动/监控ApplicationMaster
- 监控NodeManager
- 资源分配与调度

NodeManager
◆ 整个集群有多个，负责单节点资源管理和使用
◆ 功能：
- 单个节点上的资源管理和任务管理
- 处理来自ResourceManager的命令 （下文简称RM）
- 处理来自ApplicationMaster的命令
◆ NodeManager管理抽象容器，这些容器代表着可供一个特定应用程序使用的针对每个节点的资源。
◆ 定时地向RM汇报本节点上的资源使用情况和各个Container的运行状态

ApplicationManager
管理一个在YARN 内运行的应用程序的每个实例
◆ 功能

数据切分
为应用程序申请资源，并进一步分配给内部任务
任务监控与容错
◆ 负责协调来自ResourceManager的资源，幵通过NodeManager监视容器的执行和资源使用（CPU、内存等的资源分配）。

Container(容器)
◆ YARN中的资源抽象，封装某个节点上多维度资源，如内存、CPU、磁盘、网络等，当AM向RM申请资源时，RM向AM返回的资源便是用Container表示的。
◆ YARN 会为每个任务分配一个Container，且该任务只能使用该Container中描述的资源。
◆ 功能

对任务运行环境的抽象
描述一系列信息
任务启动命令
任务运行环境

◆ 资源调度和资源隔离是YARN作为一个资源管理系统，最重要和最基础的两个功能。资源调度由ResourceManager完成，而资源隔离由各个NM实现。
◆ ResourceManager将某个NodeManager上资源分配给任务（这就是所谓的“资源调度”）后，NodeManager需按照要求为任务提供相应的资源，甚至保证这些资源应具有独占性，为任务运行提供基础的保证，这就是所谓的资源隔离。
◆ 当谈及到资源时，我们通常指内存，CPU和IO三种资源。Hadoop YARN同时支持内存和CPU两种资源的调度。
◆ 内存资源的多少会会决定任务的生死，如果内存不够，任务可能会运行失败；相比之下，CPU资源则不同，它只会决定任务运行的快慢，不会对生死产生影响。

#### Hive

Hive是基于Hadoop的一个数据仓库工具，可以将结构化的数据文件映射为一张数据库表，并提供完整的sql查询功能，可以将sql语句转换为MapReduce任务进行运行。

Hive是建立在 Hadoop 上的数据仓库基础构架。它提供了一系列的工具，可以用来进行数据提取转化加载(ETL)，这是一种可以存储、查询和分析存储在 Hadoop 中的大规模数据的机制。

Hive 定义了简单的类 SQL 查询语言，称为 HQL，它允许熟悉 SQL 的用户查询数据。同时，这个语言也允许熟悉 MapReduce 开发者的开发自定义的 mapper 和 reducer 来处理内建的 mapper 和 reducer 无法完成的复杂的分析工作。

分析：Hive架构包括：CLI(Command Line Interface)、JDBC/ODBC、Thrift Server、WEB GUI、Metastore和Driver(Complier、Optimizer和Executor)，这些组件分为两大类：服务端组件和客户端组件

(1)客户端组件：

CLI：Command Line Interface，命令行接口。

Thrift客户端：上面的架构图里没有写上Thrift客户端，但是Hive架构的许多客户端接口是建立在Thrift客户端之上，包括JDBC和ODBC接口。

WEBGUI：Hive客户端提供了一种通过网页的方式访问Hive所提供的服务。这个接口对应Hive的HWI组件(Hive Web Interface)，使用前要启动HWI服务。

(2)服务端组件：

Driver组件：该组件包括Complier、Optimizer和Executor，它的作用是将HiveQL(类SQL)语句进行解析、编译优化，生成执行计划，然后调用底层的MapReduce计算框架。

Metastore组件：元数据服务组件，这个组件存储Hive的元数据，Hive的元数据存储在关系数据库里，Hive支持的关系数据库有Derby和Mysql。元数据对于Hive十分重要，因此Hive支持把Metastore服务独立出来，安装到远程的服务器集群里，从而解耦Hive服务和Metastore服务，保证Hive运行的健壮性;

Thrift服务：Thrift是Facebook开发的一个软件框架，它用来进行可扩展且跨语言的服务的开发，Hive集成了该服务，能让不同的编程语言调用Hive的接口。

`Hive与传统数据库的异同：`

(1)查询语言

由于 SQL 被广泛的应用在数据仓库中，因此专门针对Hive的特性设计了类SQL的查询语言HQL。熟悉SQL开发的开发者可以很方便的使用Hive进行开发。

(2)数据存储位置

Hive是建立在Hadoop之上的，所有Hive的数据都是存储在HDFS中的。而数据库则可以将数据保存在块设备或者本地文件系统中。

(3)数据格式

Hive中没有定义专门的数据格式，数据格式可以由用户指定，用户定义数据格式需要指定三个属性：列分隔符(通常为空格、”\t”、”\\x001″)、行分隔符(”\n”)以及读取文件数据的方法(Hive中默认有三个文件格式TextFile，SequenceFile以及RCFile)。

(4)数据更新

由于Hive是针对数据仓库应用设计的，而数据仓库的内容是读多写少的。因此，Hive中不支持对数据的改写和添加，所有的数据都是在加载的时候中确定好的。而数据库中的数据通常是需要经常进行修改的，因此可以使用INSERT INTO … VALUES添加数据，使用UPDATE … SET修改数据。（如何同步数据到Hive呢）

(5)索引

Hive在加载数据的过程中不会对数据进行任何处理，甚至不会对数据进行扫描，因此也没有对数据中的某些Key建立索引。Hive要访问数据中满足条件的特定值时，需要暴力扫描整个数据，因此访问延迟较高。由于MapReduce的引入， Hive可以并行访问数据，因此即使没有索引，对于大数据量的访问，Hive仍然可以体现出优势。数据库中，通常会针对一个或者几个列建立索引，因此对于少量的特定条件的数据的访问，数据库可以有很高的效率，较低的延迟。由于数据的访问延迟较高，决定了Hive不适合在线数据查询。（不适合在线数据，就是离线数据的查询和分析）

(6)执行

Hive中大多数查询的执行是通过Hadoop提供的MapReduce来实现的(类似select * from tbl的查询不需要MapReduce)。而数据库通常有自己的执行引擎。

(7)执行延迟

Hive在查询数据的时候，由于没有索引，需要扫描整个表，因此延迟较高。另外一个导致Hive执行延迟高的因素是MapReduce框架。由于MapReduce本身具有较高的延迟，因此在利用MapReduce执行Hive查询时，也会有较高的延迟。相对的，数据库的执行延迟较低。当然，这个低是有条件的，即数据规模较小，当数据规模大到超过数据库的处理能力的时候，Hive的并行计算显然能体现出优势。

(8)可扩展性

由于Hive是建立在Hadoop之上的，因此Hive的可扩展性是和Hadoop的可扩展性是一致的(世界上比较大的Hadoop集群在Yahoo!，2009年的规模在4000台节点左右)。而数据库由于ACID语义的严格限制，扩展行非常有限。目前先进的并行数据库Oracle在理论上的扩展能力也只有100台左右。

(9)数据规模

由于Hive建立在集群上并可以利用MapReduce进行并行计算，因此可以支持很大规模的数据;对应的，数据库可以支持的数据规模较小。

#### Hbase

HBase – Hadoop Database，是一个高可靠性、高性能、面向列、可伸缩的分布式存储系统，利用HBase技术可在廉价PC Server上搭建起大规模结构化存储集群。

HBase是Google Bigtable的开源实现，类似Google Bigtable利用GFS作为其文件存储系统，HBase利用Hadoop HDFS作为其文件存储系统;

Google运行MapReduce来处理Bigtable中的海量数据，HBase同样利用Hadoop MapReduce来处理HBase中的海量数据;

Google Bigtable利用 Chubby作为协同服务，HBase利用Zookeeper作为协同服务。（Zookeeper作为协同服务）

分析：从上图可以看出：Hbase主要由Client、Zookeeper、HMaster和HRegionServer组成，由Hstore作存储系统。

Client

HBase Client使用HBase的RPC机制与HMaster和HRegionServer进行通信，对于管理类操作，Client与 HMaster进行RPC;对于数据读写类操作，Client与HRegionServer进行RPC

Zookeeper

Zookeeper Quorum 中除了存储了 -ROOT- 表的地址和 HMaster 的地址，HRegionServer 也会把自己以 Ephemeral 方式注册到 Zookeeper 中，使得 HMaster 可以随时感知到各个HRegionServer 的健康状态。

HMaster

HMaster 没有单点问题，HBase 中可以启动多个 HMaster ，通过 Zookeeper 的 Master Election 机制保证总有一个 Master 运行，HMaster 在功能上主要负责 Table和Region的管理工作：

管理用户对 Table 的增、删、改、查操作
管理 HRegionServer 的负载均衡，调整 Region 分布
在 Region Split 后，负责新 Region 的分配
在 HRegionServer 停机后，负责失效 HRegionServer 上的 Regions 迁移

HStore存储是HBase存储的核心了，其中由两部分组成，一部分是MemStore，一部分是StoreFiles。

MemStore是Sorted Memory Buffer，用户写入的数据首先会放入MemStore，当MemStore满了以后会Flush成一个StoreFile(底层实现是HFile)， 当StoreFile文件数量增长到一定阈值，会触发Compact合并操作，将多个 StoreFiles 合并成一个 StoreFile，合并过程中会进行版本合并和数据删除。

因此可以看出HBase其实只有增加数据，所有的更新和删除操作都是在后续的 compact 过程中进行的，这使得用户的写操作只要进入内存中就可以立即返回，保证了 HBase I/O 的高性能。（这边很重要，确保IO的高性能）

当StoreFiles Compact后，会逐步形成越来越大的StoreFile，当单个 StoreFile 大小超过一定阈值后，会触发Split操作，同时把当前 Region Split成2个Region，父 Region会下线，新Split出的2个孩子Region会被HMaster分配到相应的HRegionServer 上，使得原先1个Region的压力得以分流到2个Region上。

#### KNN
