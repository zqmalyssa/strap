---
layout: post
title: 流式计算
tags: [stream-computing]
author-id: zqmalyssa
---

流式计算

#### 流式计算比较

Spark，Storm和Flink

涉及的流框架基于的实现方式分为两大类。第一类是Native Streaming，这类引擎中所有的data在到来的时候就会被立即处理，一条接着一条（HINT： 狭隘的来说是一条接着一条，但流引擎有时会为提高性能缓存一小部分data然后一次性处理），其中的代表就是storm和flink。第二种则是基于Micro-batch，数据流被切分为一个一个小的批次， 然后再逐个被引擎处理。这些batch一般是以时间为单位进行切分，单位一般是‘秒‘，其中的典型代表则是spark了，不论是老的spark DStream还是2.0以后推出的spark structured streaming都是这样的处理机制；另外一个基于Micro-batch实现的就是storm trident，它是对storm的更高层的抽象，因为以batch为单位，所以storm trident的一些处理变的简单且高效


#### Spark

Driver Program, Job和Stage是Spark中的几个基本概念。Spark官方文档中对于这几个概念的解释比较简单，对于初学者很难正确理解他们的涵义。

Driver Program: The process running the main() function of the application and creating the SparkContext.

Job: A parallel computation consisting of multiple tasks that gets spawned in response to a Spark action (e.g. save, collect); you’ll see this term used in the driver’s logs.

Stage: Each job gets divided into smaller sets of tasks called stages that depend on each other (similar to the map and reduce stages in MapReduce); you’ll see this term used in the driver’s logs.

```html

术语总是难以理解的，因为它取决于所处的上下文。在很多情况下，你可能习惯于“将Job提交给一个cluster”，但是对于spark而言却是提交了一个driver程序。

也就是说，对于Job，spark有它自己的定义，如下：
A parallel computation consisting of multiple tasks that gets spawned in response to a Spark action (e.g. save, collect); you’ll see this term used in the driver’s logs.

在这个例子中，假设你需要做如下一些事情：
1. 将一个包含人名和地址的文件加载到RDD1中
2. 将一个包含人名和电话的文件加载到RDD2中
3. 通过name来Join RDD1和RDD2，生成RDD3
4. 在RDD3上做Map，给每个人生成一个HTML展示卡作为RDD4
5. 将RDD4保存到文件
6. 在RDD1上做Map，从每个地址中提取邮编，结果生成RDD5
7. 在RDD5上做聚合，计算出每个邮编地区中生活的人数，结果生成RDD6
8. Collect RDD6，并且将这些统计结果输出到stdout


```

其中红色虚线表示输入和输出，蓝色实线是对RDD的操作，圆圈中的数字对应了以上的8个步骤。接下来解释driver program, job和stage这几个概念：

Driver program是全部的代码，运行所有的8个步骤。
第五步中的save和第八步中的collect都是Spark Job。Spark中每个action对应着一个Job，transformation不是Job。
其他的步骤（1、2、3、4、6、7）被Spark组织成stages，每个job则是一些stage序列的结果。对于一些简单的场景，一个job可以只有一个stage。但是对于数据重分区的需求（比如第三步中的join），或者任何破坏数据局域性的事件，通常会导致更多的stage。可以将stage看作是能够产生中间结果的计算。这种计算可以被持久化，比如可以把RDD1持久化来避免重复计算。
以上全部三个概念解释了某个算法被拆分的逻辑。相比之下，task是一个特定的数据片段，在给定的executor上，它可以跨越某个特定的stage。

到了这里，很多概念就清楚了。驱动程序就是执行了一个Spark Application的main函数和创建Spark Context的进程，它包含了这个application的全部代码。Spark Application中的每个action会被Spark作为Job进行调度。每个Job是一个计算序列的最终结果，而这个序列中能够产生中间结果的计算就是一个stage。

Action列表：

reduce
collect
count
first
take
takeSample
takeOrdered
saveAsTextFile
saveAsSequenceFile
saveAsObjectFile
countByKey
foreach

Transformation列表：

map
filter
flatMap
mapPartitions
mapPartitionsWithIndex
sample
union
intersection
distinct
groupByKey
reduceByKey
aggregateByKey
sortByKey
join
cogroup
cartesian
pipe
coalesce
repartition
repartitionAndSortWithinPartitions

至于task，官方文档中是这么说的：Task is a unit of work that will be sent to one executor。再结合官方对Stage的解释，可以这样理解：
一个Job被拆分成若干个Stage，每个Stage执行一些计算，产生一些中间结果。它们的目的是最终生成这个Job的计算结果。而每个Stage是一个task set，包含若干个task。Task是Spark中最小的工作单元，在一个executor上完成一个特定的事情。


这边可以结合spark的UI去说明


上面就是Spark的UI主页，首先进来能看到的是Spark当前应用的job页面，在上面的导航栏：

1 代表job页面，在里面可以看到当前应用分析出来的所有任务，以及所有的excutors中action的执行时间。
2 代表stage页面，在里面可以看到应用的所有stage，stage是按照宽依赖来区分的，因此粒度上要比job更细一些
3 代表storage页面，我们所做的cache persist等操作，都会在这里看到，可以看出来应用目前使用了多少缓存
4 代表environment页面，里面展示了当前spark所依赖的环境，比如jdk,lib等等
5 代表executors页面，这里可以看到执行者申请使用的内存以及shuffle中input和output等数据
6 这是应用的名字，代码中如果使用setAppName，就会显示在这里
7 是job的主页面。

这里直接使用了分布式计算里面最经典的helloworld程序——WordCount,这个程序用于统计某一段文本中一个单词出现的次数。原始的文本如下:


```html

for the shadow of lost knowledge at leaset bala bala

```

```java

public static void main(String[] args) {

        SparkConf sparkConf = new SparkConf();
        sparkConf.setMaster("Local2");
        sparkConf.setAppName("test");
        JavaSparkContext sc = new JavaSparkContext(sparkConf);

        JavaPairRDD<String, Integer> counts = sc.textFile("D:\\Code\\WordCounts.txt")
            .flatMap(line -> Arrays.asList(line.split(" ")))
            .mapToPair(s -> new Tuple2<String, Integer>(s, 1))
            .reduceByKey((x, y) -> x + y);

        counts.cache();

        List<Tuple2<String, Integer>> result = counts.collect();

        for (Tuple2<String, Integer> t2 : result) {
            System.out.println(t2._1 + " : " + t2._2);
        }

        sc.stop();

    }

```

这个程序首先创建了SparkContext，然后读取文件，先使用` `进行切分，再把每个单词转换成二元组，再根据key进行累加，最后输出打印。为了测试storage的使用，我这对计算的结果添加了缓存。

Job主页可以分两个部份，一部分是event timeline，另一部分是进行中和完成的job任务。

第一部分event timeline展开后，可以看到executor创建的时间点，以及某个action触发的算子任务，执行的时间。通过这个时间图，可以快速的发现应用的执行瓶颈，触发了多少个action。

第二部分的图表，显示了触发action的job名字，它通常是某个count,collect等操作。有spark基础的人都应该知道，在spark中rdd的计算分为两类，一类是transform转换操作，一类是action操作，只有action操作才会触发真正的rdd计算。具体的有哪些action可以触发计算，可以参考api。collect at test2.java:27描述了action的名字和所在的行号，这里的行号是精准匹配到代码的，所以通过它可以直接定位到任务所属的代码，这在调试分析的时候是非常有帮助的。Duration显示了该action的耗时，通过它也可以对代码进行专门的优化。最后的进度条，显示了该任务失败和成功的次数，如果有失败的就需要引起注意，因为这种情况在生产环境可能会更普遍更严重。点击能进入该action具体的分析页面，可以看到DAG图等详细信息。

在Spark中job是根据action操作来区分的，另外任务还有一个级别是stage，它是根据宽窄依赖来区分的。

窄依赖是指前一个rdd计算能出一个唯一的rdd，比如map或者filter等；宽依赖则是指多个rdd生成一个或者多个rdd的操作，比如groupbykey reducebykey等，这种宽依赖通常会进行shuffle。

因此Spark会根据宽窄依赖区分stage，某个stage作为专门的计算，计算完成后，会等待其他的executor，然后再统一进行计算。

stage页面的使用基本上跟job类似，不过多了一个DAG图。这个DAG图也叫作血统图，标记了每个rdd从创建到应用的一个流程图，也是我们进行分析和调优很重要的内容。比如上面的wordcount程序，就会触发acton，然后生成一段DAG图：

从这个图可以看出，wordcount会生成两个dag，一个是从读数据到切分到生成二元组，第二个进行了reducebykey，产生shuffle。

点击进去还可以看到详细的DAG图，鼠标移到上面，可以看到一些简要的信息。

具体可以参考[这篇](https://blog.csdn.net/u013013024/article/details/73498508)

##### Spark RDD

RDD：Resilient Distributed Dataset 弹性分布式数据集，是Spark中的基本抽象。

RDD表示可以并行操作的元素的不变分区集合。

RDD提供了许多基本的函数（map、filter、reduce等）供我们进行数据处理。

通常来说，每个RDD有5个主要的属性组成：

1、分区列表

RDD是由多个分区组成的，分区是逻辑上的概念。RDD的计算是以分区为单位进行的。

2、用于计算每个分区的函数

作用于每个分区数据的计算函数。

3、对其他RDD的依赖关系列表

RDD中保存了对于父RDD的依赖，根据依赖关系组成了Spark的DAG（有向无环图），实现了spark巧妙、容错的编程模型

4、针对键值型RDD的分区器

分区器针对键值型RDD而言的，将key传入分区器获取唯一的分区id。在shuffle中，分区器有很重要的体现。

5、对每个分区进行计算的首选位置列表

根据数据本地性的特性，获取计算的首选位置列表，尽可能的把计算分配到靠近数据的位置，减少数据的网络传输。

```html

//创建此RDD的SparkContext
def sparkContext: SparkContext = sc
// 唯一的id
val id: Int = sc.newRddId()
// rdd友善的名字
@transient var name: String = _
// 分区器
val partitioner: Option[Partitioner] = None
// 获取依赖列表
// dependencies和partitions中都用到了checkpointRDD，如果进行了checkpoint，checkpointRDD表示进行checkpoint后的rdd
final def dependencies: Seq[Dependency[_]] = {
    // 一对一的窄依赖
    checkpointRDD.map(r => List(new OneToOneDependency(r))).getOrElse {
        if (dependencies_ == null) {
            dependencies_ = getDependencies
        }
        dependencies_
    }
}
// 获取分区列表
final def partitions: Array[Partition] = {
    checkpointRDD.map(_.partitions).getOrElse {
        if (partitions_ == null) {
            partitions_ = getPartitions
            partitions_.zipWithIndex.foreach { case (partition, index) =>
                require(partition.index == index,
                        s"partitions($index).partition == ${partition.index}, but it should equal $index")
            }
        }
        partitions_
    }
}
// 获取分区的首选位置
final def preferredLocations(split: Partition): Seq[String] = {
    checkpointRDD.map(_.getPreferredLocations(split)).getOrElse {
        getPreferredLocations(split)
    }
}
// 对应到每个分区的计算函数
def compute(split: Partition, context: TaskContext): Iterator[T]

```

主要就是围绕上面5个重要属性的一些操作

常用的函数/算子

```html

// 返回仅包含满足过滤条件的元素的新RDD。
def filter(f: T => Boolean): RDD[T] = withScope {
    val cleanF = sc.clean(f)
    new MapPartitionsRDD[T, T](
        this,
        (context, pid, iter) => iter.filter(cleanF),
        preservesPartitioning = true)
}
// 通过将函数应用于此RDD的所有元素来返回新的RDD。
def map[U: ClassTag](f: T => U): RDD[U] = withScope {
    val cleanF = sc.clean(f)
    new MapPartitionsRDD[U, T](this, (context, pid, iter) => iter.map(cleanF))
}
// 首先向该RDD的所有元素应用函数，然后将结果展平，以返回新的RDD。
def flatMap[U: ClassTag](f: T => TraversableOnce[U]): RDD[U] = withScope {
    val cleanF = sc.clean(f)
    new MapPartitionsRDD[U, T](this, (context, pid, iter) => iter.flatMap(cleanF))
}

```

我们可以发现几乎每个算子都会以当前RDD和对应的计算函数创建新的RDD，每个子RDD都持有父RDD的引用。

这就印证了RDD的不变性，也表明了RDD的计算是通过对RDD进行转换实现的。

比如

```html

val words = Seq("hello spark", "hello scala", "hello java")
val rdd = sc.makeRDD(words)
rdd
    .flatMap(_.split(" "))
    .map((_, 1))
    .reduceByKey(_ + _)
    .foreach(println(_))

```

上面是一个简单的RDD的操作，我们先调用makeRDD创建了一个RDD，之后对rdd进行一顿算子调用。

首先调用flatMap，flatMap内部会以当前rdd和我们传入的_.split(" ")构建新的MapPartitionsRDD；

之后map，map以上步生成的MapPartitionsRDD和我们传入的(_, 1)构造新的MapPartitionsRDD；

之后reduceByKey，reduceByKey构造新的RDD；

走到foreach，foreach是行动操作，触发计算，输出。

小总结
RDD内部的计算除action算子以外，其他算子都是懒执行，不会触发计算，只是进行RDD的转换。
RDD的计算是基于分区为单位计算的，我们传进去的函数，作用于分区进行计算

转换、行动算子

从上面知道RDD是懒执行的，只有遇到行动算子才执行计算。

转换操作：在内部对根据父RDD创建新的RDD，不执行计算

行动操作：内部会调用sc.runJob，提交作业、划分阶段、执行作业。

一些常见的行动操作
foreach、foreachPartition、collect、reduce、count（上面也有）

除行动操作外，都是转换操作

宽、窄依赖
宽窄依赖是shuffle和划分调度的重要依据。

先看看spark中与依赖有关的几个类(一层一层继承关系)：

```html

Dependency依赖的顶级父类
	NarrowDependency 窄依赖
		OneToOneDependency 表示父RDD和子RDD分区之间的一对一依赖关系的窄依赖
		RangeDependency 表示父RDD和子RDD中分区范围之间的一对一依赖关系的窄依赖
	ShuffleDependency 宽依赖

```

先说宽窄依赖的概念：

窄依赖：父RDD的每个分区只被一个子RDD分区使用

宽依赖：父RDD的每个分区都有可能被多个子RDD分区使用

其实就是父RDD的一个分区会被传到几个子RDD分区的区别。如果被传到一个子RDD分区，就可以不需要移动数据（移动计算）；如果被传到多个子RDD分区，就需要进行数据的传输。

接下来看看Dependency内部的一些属性及方法：

```html

// 依赖对应的rdd，其实就是当前rdd的父rdd。宽依赖和窄依赖都有这个属性
def rdd: RDD[T]
// 获取子分区对应的父分区（窄依赖的方法）
def getParents(partitionId: Int): Seq[Int]

// 以下是宽依赖的属性及方法
// 对应键值RDD的分区器
val partitioner: Partitioner
// 在数据传输时的序列化方法
val serializer: Serializer = SparkEnv.get.serializer
// 键的排序方式
val keyOrdering: Option[Ordering[K]] = None
// 一组用于聚合数据的功能
val aggregator: Option[Aggregator[K, V, C]] = None
// 是否需要map端预聚合
val mapSideCombine: Boolean = false
// 当前宽依赖的id
val shuffleId: Int = _rdd.context.newShuffleId()
// 向管理员注册一个shuffle，并获取一个句柄，以将其传递给任务
val shuffleHandle: ShuffleHandle =  _rdd.context.env.shuffleManager.registerShuffle(
shuffleId, _rdd.partitions.length, this)


```

一些常见的宽窄依赖
窄依赖：map、filter、union、mapPartitions、join(当分区器是HashPartitioner)

宽依赖：sortByKey、join(分区器不是HashPartitioner时)


最后说一下reduceByKey，顺便说一下为什么当分区器HashPartitioner时就是窄依赖。

reduceByKey是用来将key分组后，执行我们传入的函数。

它是窄依赖，它内部默认会使用HashPartitioner分区。

同一个key进去HashPartitioner得到的分区id是一样的，这样进行计算前后同一个key得到的分区都一样，父RDD的分区就只被子RDD的一个分区依赖，就不需要移动数据。

所以join、reduceByKey在分区器是HashPartitioner时是窄依赖。


`JavaRDD与JavaPairRDD的互相转换`

JavaRDD => JavaPairRDD: 通过mapToPair函数
JavaPairRDD => JavaRDD: 通过map函数转换

[RDD转换](https://blog.csdn.net/hellozhxy/article/details/82344515)

JavaPair的输出样式：

```html

(null,Hello World)
(null,Hello World)
(null,Hello World)
(null,Hello World)
(null,Hello World)
(null,Hello World)
(null,Hello World)
(null,Hello World)
(null,Hello World)
(null,Hello World)

```


`查看RDD中的内容`

```java

rdd.collect().forEach(System.out::println);
// 取前10个
rdd.take(10).forEach(System.out::println);


```


#### Spark Streaming

Spark streaming是Spark核心API的一个扩展，它对实时流式数据的处理具有可扩展性、高吞吐量、可容错性等特点。我们可以从kafka、flume、Twitter、 ZeroMQ、Kinesis等源获取数据，也可以通过由 高阶函数map、reduce、join、window等组成的复杂算法计算出数据。最后，处理后的数据可以推送到文件系统、数据库、实时仪表盘中

Spark Streaming支持一个高层的抽象，叫做离散流(discretized stream)或者DStream，它代表连续的数据流。DStream既可以利用从Kafka, Flume和Kinesis等源获取的输入数据流创建，也可以 在其他DStream的基础上通过高阶函数获得。在内部，DStream是由一系列RDDs组成。


#### Spark SQL

Spark SQL是Spark用来处理结构化数据的一个模块，它提供了两个编程抽象分别叫做DataFrame和DataSet，它们用于作为分布式SQL查询引擎。从下图可以查看RDD、DataFrames与DataSet的关系。

DataSets里面包含了 RDDs 和 DataFrames

我们已经学习了Hive，它是将Hive SQL转换成MapReduce然后提交到集群上执行，大大简化了编写MapReduce的程序的复杂性，由于MapReduce这种计算模型执行效率比较慢。所以Spark SQL的应运而生，它是将Spark SQL转换成RDD，然后提交到集群执行，执行效率非常快！所以我们类比的理解：Hive---SQL-->MapReduce，Spark SQL---SQL-->RDD。都是一种解析传统SQL到大数据运算模型的引擎，属于数据分析的范围。

首先，最简单的理解我们可以认为DataFrame就是Spark中的数据表（类比传统数据库），DataFrame的结构如下：

DataFrame（表）= Schema（表结构） + Data（表数据）

总结：DataFrame（表）是Spark SQL对结构化数据的抽象。可以将DataFrame看做RDD。

DataFrame是组织成命名列的数据集。它在概念上等同于关系数据库中的表，但在底层具有更丰富的优化。DataFrames可以从各种来源构建，

例如：

结构化数据文件(JSON)
外部数据库或现有RDDs

DataFrame API支持的语言有Scala，Java，Python和R。

从上图可以看出，DataFrame相比RDD多了数据的结构信息，即schema。RDD是分布式的 Java对象的集合。DataFrame是分布式的Row对象的集合。DataFrame除了提供了比RDD更丰富的算子以外，更重要的特点是提升执行效率、减少数据读取以及执行计划的优化。

DataSet

Dataset是数据的分布式集合。Dataset是在Spark 1.6中添加的一个新接口，是DataFrame之上更高一级的抽象。它提供了RDD的优点（强类型化）以及Spark SQL优化后的执行引擎的优点。一个Dataset 可以从JVM对象构造，然后使用函数转换（map， flatMap，filter等）去操作。 Dataset API 支持Scala和Java。 Python不支持Dataset API。

Apache Spark 2.0引入了SparkSession，其为用户提供了一个统一的切入点来使用Spark的各项功能，并且允许用户通过它调用DataFrame和Dataset相关API来编写Spark程序。最重要的是，它减少了用户需要了解的一些概念，使得我们可以很容易地与Spark交互。
在2.0版本之前，与Spark交互之前必须先创建SparkConf和SparkContext。然而在Spark 2.0中，我们可以通过SparkSession来实现同样的功能，而不需要显式地创建SparkConf, SparkContext 以及 SQLContext，因为这些对象已经封装在SparkSession中。

`这边再详细说下RDD->DataFrame->DataSet`

RDD的限制

1、没有内置的优化引擎
当对结构化的数据进行处理时，RDD没有使用Spark的高级优化器，比如catalyst优化器和Tungsten执行引擎。
2、处理结构化的数据
不像Dataframe或者Dataset，RDD不会主动推测出数据的schema，而是需要用户在代码里指示。

DataFrame

Spark从1.3版本开始引入Dataframe，它克服了RDD的最主要的挑战

主要描述：Dataframe是一个分布式的数据collection，而且将数据按照列名进行组织。在概念上它与关系型的数据库的表或者R/Python语言中的DataFrame类似。与之一起提供的还有，Spark引入了catalyst优化器，它可以优化查询。

DataFrame的限制

没有编译阶段的类型检查：
不能在编译时刻对安全性做出检查，而且限制了用户对于未知结构的数据进行操作。比如下面代码在编译时没有错误，但是在执行时会出现异常：

```java

case class Person(name: String, age :Int)
val dataframe = sqlContect.read.json("people.json")
dataframe.filter("salary > 10000").show

=> throws Exception : cannot resolve 'salary' given input age, name

```

不能保留类对象的结构：
一旦把一个类结构的对象转成了Dataframe，就不能转回去了。下面这个栗子就是指出了：

```java

case class Person(name: String, age :Int)
val personRDD = sc.makeRDD(Seq(Person("A",10), Person("B",20)))
val personDF = sqlContect.createDataframe(personRDD)
personDF.rdd // returns RDD[ROW], does not return RDD[Person]

```

DataSet

主要描述：Dataset API是对DataFrame的一个扩展，使得可以支持类型安全的检查，并且对类结构的对象支持程序接口。它是强类型的，不可变collection，并映射成一个相关的schema。
Dataset API的核心是一个被称为Encoder的概念。它是负责对JVM的对象以及表格化的表达（tabular representation）之间的相互转化。
表格化的表达在存储时使用了Spark内置的Tungsten二进制形式，允许对序列化数据操作并改进了内存使用。在Spark 1.6版本之后，支持自动化生成Encoder，可以对广泛的primitive类型（比如String，Integer，Long等）、Scala的case class以及Java Bean自动生成对应的Encoder。

DataSet的特性

1、支持RDD和Dataframe的优点：
包括RDD的类型安全检查，Dataframe的关系型模型，查询优化，Tungsten执行，排序和shuffling。
2、Encoder：
通过使用Encoder，用户可以轻松转换JVM对象到一个Dataset，允许用户在结构化和非结构化的数据操作。
3、编程语言：
Scala和Java
4、类型安全检查：
提供编译阶段的安全类型检查。比如下面这个栗子
5、相互转换：
Dataset可以让用户轻松从RDD和Dataframe转换到Dataset不需要额外太多代码。

DataSet的限制

需要把类型转成String：
Querying the data from datasets currently requires us to specify the fields in the class as a string. Once we have queried the data, we are forced to cast column to the required data type. On the other hand, if we use map operation on Datasets, it will not use Catalyst optimizer.
