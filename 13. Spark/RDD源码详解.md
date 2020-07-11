1、什么是RDD？

上一章讲了Spark提交作业的过程，这一章我们要讲RDD。简单的讲，RDD就是Spark的input，知道input是啥吧，就是输入的数据。

RDD的全名是Resilient Distributed Dataset，意思是容错的分布式数据集，每一个RDD都会有5个特征：

1、有一个分片列表。就是能被切分，和hadoop一样的，能够切分的数据才能并行计算。

2、有一个函数计算每一个分片，这里指的是下面会提到的compute函数。

3、对其他的RDD的依赖列表，依赖还具体分为宽依赖和窄依赖，但并不是所有的RDD都有依赖。

4、可选：key-value型的RDD是根据哈希来分区的，类似于mapreduce当中的Paritioner接口，控制key分到哪个reduce。

5、可选：每一个分片的优先计算位置（preferred locations），比如HDFS的block的所在位置应该是优先计算的位置。

 对应着上面这几点，我们在RDD里面能找到这4个方法和1个属性，别着急，下面我们会慢慢展开说这5个东东。

```scala 
  protected def getPartitions: Array[Partition]  
  //对一个分片进行计算，得出一个可遍历的结果
  def compute(split: Partition, context: TaskContext): Iterator[T]
  //只计算一次，计算RDD对父RDD的依赖
  protected def getDependencies: Seq[Dependency[_]] = deps
  //可选的，分区的方法，针对第4点，类似于mapreduce当中的Paritioner接口，控制key分到哪个reduce
  @transient val partitioner: Option[Partitioner] = None
  //可选的，指定优先位置，输入参数是split分片，输出结果是一组优先的节点位置
  protected def getPreferredLocations(split: Partition): Seq[String] = Nil
```

2、多种RDD之间的转换

下面用一个实例讲解一下吧，就拿我们常用的一段代码来讲吧，然后会把我们常用的RDD都会讲到。
```scala
    val hdfsFile = sc.textFile(args(1))
    val flatMapRdd = hdfsFile.flatMap(s => s.split(" "))
    val filterRdd = flatMapRdd.filter(_.length == 2)
    val mapRdd = filterRdd.map(word => (word, 1))
    val reduce = mapRdd.reduceByKey(_ + _)
```
这里涉及到很多个RDD，textFile是一个HadoopRDD经过map后的MappredRDD，经过flatMap是一个FlatMappedRDD，经过filter方法之后生成了一个FilteredRDD，经过map函数之后，变成一个MappedRDD，通过隐式转换成 PairRDD，最后经过reduceByKey。

我们首先看textFile的这个方法，进入SparkContext这个方法，找到它。

```scala
def textFile(path: String, minPartitions: Int = defaultMinPartitions): RDD[String] = {
    hadoopFile(path, classOf[TextInputFormat], classOf[LongWritable], classOf[Text], minPartitions).map(pair => pair._2.toString)
}
```
看它的输入参数，path，TextInputFormat，LongWritable，Text，同志们联想到什么？写过mapreduce的童鞋都应该知道哈。

1、hdfs的地址

2、InputFormat的类型

3、Mapper的第一个类型

4、Mapper的第二类型

这就不难理解为什么立马就对hadoopFile后面加了一个map方法，取pair的第二个参数了，最后在shell里面我们看到它是一个MappredRDD了。

言归正传，默认的defaultMinPartitions的2太小了，我们用的时候还是设置大一点吧。

####源码分析：HadoopRDD类

下面专门来看一下HadoopRDD是干什么的。

 >An RDD that provides core functionality for reading data stored in Hadoop (e.g., files in HDFS, sources in HBase, or S3)
 

看注释可以知道，HadoopRDD是一个专为Hadoop（HDFS、Hbase、S3）设计的RDD。

HadoopRDD主要重写了三个方法，可以在源码中找到加override标识的方法：

+ override def getPartitions: Array[Partition]

+ override def compute(theSplit: Partition, context: TaskContext): InterruptibleIterator[(K, V)]

+ override def getPreferredLocations(split: Partition): Seq[String]

下面分别看一下这三个方法。

getPartitions方法最后是返回了一个array。它调用的是inputFormat自带的getSplits方法来计算分片，然后把分片信息放到array中。

这里，我们是不是就可以理解，Hadoop中的一个分片，就对应到Spark中的一个Partition。
```scala
override def getPartitions: Array[Partition] = {
  val jobConf = getJobConf()
  // add the credentials here as this can be called before SparkContext initialized
  SparkHadoopUtil.get.addCredentials(jobConf)
  val inputFormat = getInputFormat(jobConf)
  val inputSplits = inputFormat.getSplits(jobConf, minPartitions)
  val array = new Array[Partition](inputSplits.size)
  for (i <- 0 until inputSplits.size) {
    array(i) = new HadoopPartition(id, i, inputSplits(i))
  }
  array
}

private[spark] class HadoopPartition(rddId: Int, override val index: Int, s: InputSplit)
```
compute方法的作用主要就是根据输入的partition信息生成一个InterruptibleIterator。如下面代码段。

```scala
override def compute(theSplit: Partition, context: TaskContext): InterruptibleIterator[(K, V)] = {
    val iter = new NextIterator[(K, V)] {......}
    new InterruptibleIterator[(K, V)](context, iter)
  }
```
下面看一下iter中做了什么逻辑处理。

把Partition转成HadoopPartition，然后通过InputSplit创建一个RecordReader
重写Iterator的getNext方法，通过创建的reader调用next方法读取下一个值。
从这里我们可以看得出来compute方法是通过分片来获得Iterator接口，以遍历分片的数据。

override def compute(theSplit: Partition, context: TaskContext): InterruptibleIterator[(K, V)] = {

 val iter = new NextIterator[(K, V)] {

      //将compute的输入theSplit，转换为HadoopPartition
      val split = theSplit.asInstanceOf[HadoopPartition]
      ......
      //c重写getNext方法
      override def getNext(): (K, V) = {
        try {
          finished = !reader.next(key, value)
        } catch {
          case _: EOFException if ignoreCorruptFiles => finished = true
        }
        if (!finished) {
          inputMetrics.incRecordsRead(1)
        }
        (key, value)
      }
     }
}

2.1 HadoopRDD

我们继续追杀下去，看看hadoopFile方法，里面我们看到它做了3个操作。

1、把hadoop的配置文件保存到广播变量里。

2、设置路径的方法

3、new了一个HadoopRDD返回

好，我们接下去看看HadoopRDD这个类吧，我们重点看看它的getPartitions、compute、getPreferredLocations。

先看getPartitions，它的核心代码如下：

```scala
    val inputSplits = inputFormat.getSplits(jobConf, minPartitions)
    val array = new Array[Partition](inputSplits.size)
    for (i <- 0 until inputSplits.size) {
      array(i) = new HadoopPartition(id, i, inputSplits(i))
    }
```
它调用的是inputFormat自带的getSplits方法来计算分片，然后把分片HadoopPartition包装到到array里面返回。

这里顺便顺带提一下，因为1.0又出来一个NewHadoopRDD，它使用的是mapreduce新api的inputformat，getSplits就不要有minPartitions了，别的逻辑都是一样的，只是使用的类有点区别。

我们接下来看compute方法，它的输入值是一个Partition，返回是一个Iterator[(K, V)]类型的数据，这里面我们只需要关注2点即可。

1、把Partition转成HadoopPartition，然后通过InputSplit创建一个RecordReader

2、重写Iterator的getNext方法，通过创建的reader调用next方法读取下一个值。

```scala
      // 转换成HadoopPartition
      val split = theSplit.asInstanceOf[HadoopPartition]
      logInfo("Input split: " + split.inputSplit)
      var reader: RecordReader[K, V] = null
      val jobConf = getJobConf()
      val inputFormat = getInputFormat(jobConf)
        context.stageId, theSplit.index, context.attemptId.toInt, jobConf)
      // 通过Inputform的getRecordReader来创建这个InputSpit的Reader
      reader = inputFormat.getRecordReader(split.inputSplit.value, jobConf, Reporter.NULL)

      // 调用Reader的next方法
      val key: K = reader.createKey()
      val value: V = reader.createValue()
      override def getNext() = {
        try {
          finished = !reader.next(key, value)
        } catch {
          case eof: EOFException =>
            finished = true
        }
        (key, value)
      }
```

从这里我们可以看得出来compute方法是通过分片来获得Iterator接口，以遍历分片的数据。

getPreferredLocations方法就更简单了，直接调用InputSplit的getLocations方法获得所在的位置。

2.2 依赖
下面我们看RDD里面的map方法

def map[U: ClassTag](f: T => U): RDD[U] = new MappedRDD(this, sc.clean(f))
直接new了一个MappedRDD，还把匿名函数f处理了再传进去，我们继续追杀到MappedRDD。

复制代码
private[spark]
class MappedRDD[U: ClassTag, T: ClassTag](prev: RDD[T], f: T => U)
  extends RDD[U](prev) {
  override def getPartitions: Array[Partition] = firstParent[T].partitions
  override def compute(split: Partition, context: TaskContext) =
    firstParent[T].iterator(split, context).map(f)
}
复制代码
MappedRDD把getPartitions和compute给重写了，而且都用到了firstParent[T]，这个firstParent是何须人也？我们可以先点击进入RDD[U](prev)这个构造函数里面去。

def this(@transient oneParent: RDD[_]) = this(oneParent.context , List(new OneToOneDependency(oneParent)))
就这样你会发现它把RDD复制给了deps，HadoopRDD成了MappedRDD的父依赖了，这个OneToOneDependency是一个窄依赖，子RDD直接依赖于父RDD，继续看firstParent。

protected[spark] def firstParent[U: ClassTag] = {
  dependencies.head.rdd.asInstanceOf[RDD[U]]
}
由此我们可以得出两个结论：

1、getPartitions直接沿用了父RDD的分片信息

2、compute函数是在父RDD遍历每一行数据时套一个匿名函数f进行处理

好吧，现在我们可以理解compute函数真正是在干嘛的了

它的两个显著作用：

1、在没有依赖的条件下，根据分片的信息生成遍历数据的Iterable接口

2、在有前置依赖的条件下，在父RDD的Iterable接口上给遍历每个元素的时候再套上一个方法

我们看看点击进入map(f)的方法进去看一下

  def map[B](f: A => B): Iterator[B] = new AbstractIterator[B] {
    def hasNext = self.hasNext
    def next() = f(self.next())
  }
看黄色的位置，看它的next函数，不得不说，写得真的很妙！

我们接着看RDD的flatMap方法，你会发现它和map函数几乎没什么区别，只是RDD变成了FlatMappedRDD，但是flatMap和map的效果还是差别挺大的。

比如((1,2),(3,4)), 如果是调用了flatMap函数，我们访问到的就是(1,2,3,4)4个元素；如果是map的话，我们访问到的就是(1,2),(3,4)两个元素。

有兴趣的可以去看看FlatMappedRDD和FilteredRDD这里就不讲了，和MappedRDD类似。

2.3 reduceByKey
前面的RDD转换都简单，可是到了reduceByKey可就不简单了哦，因为这里有一个同相同key的内容聚合的一个过程，所以它是最复杂的那一类。

那reduceByKey这个方法在哪里呢，它在PairRDDFunctions里面，这是个隐式转换，所以比较隐蔽哦，你在RDD里面是找不到的。

  def reduceByKey(partitioner: Partitioner, func: (V, V) => V): RDD[(K, V)] = {
    combineByKey[V]((v: V) => v, func, func, partitioner)
  }
它调用的是combineByKey方法，过程过程蛮复杂的，折叠起来，喜欢看的人看看吧。


复制代码
def combineByKey[C](createCombiner: V => C,
      mergeValue: (C, V) => C,
      mergeCombiners: (C, C) => C,
      partitioner: Partitioner,
      mapSideCombine: Boolean = true,
      serializer: Serializer = null): RDD[(K, C)] = {

    val aggregator = new Aggregator[K, V, C](createCombiner, mergeValue, mergeCombiners)
    if (self.partitioner == Some(partitioner)) {
      // 一般的RDD的partitioner是None，这个条件不成立，即使成立只需要对这个数据做一次按key合并value的操作即可
      self.mapPartitionsWithContext((context, iter) => {
        new InterruptibleIterator(context, aggregator.combineValuesByKey(iter, context))
      }, preservesPartitioning = true)
    } else if (mapSideCombine) {
      // 默认是走的这个方法，需要map端的combinber.
      val combined = self.mapPartitionsWithContext((context, iter) => {
        aggregator.combineValuesByKey(iter, context)
      }, preservesPartitioning = true)
      val partitioned = new ShuffledRDD[K, C, (K, C)](combined, partitioner)
        .setSerializer(serializer)
      partitioned.mapPartitionsWithContext((context, iter) => {
        new InterruptibleIterator(context, aggregator.combineCombinersByKey(iter, context))
      }, preservesPartitioning = true)
    } else {
      // 不需要map端的combine，直接就来shuffle
      val values = new ShuffledRDD[K, V, (K, V)](self, partitioner).setSerializer(serializer)
      values.mapPartitionsWithContext((context, iter) => {
        new InterruptibleIterator(context, aggregator.combineValuesByKey(iter, context))
      }, preservesPartitioning = true)
    }
  }
复制代码
按照一个比较标准的流程来看的话，应该是走的中间的这条路径，它干了三件事：

1、给每个分片的数据在外面套一个combineValuesByKey方法的MapPartitionsRDD。

2、用MapPartitionsRDD来new了一个ShuffledRDD出来。

3、对ShuffledRDD做一次combineCombinersByKey。

下面我们先看MapPartitionsRDD，我把和别的RDD有别的两行给拿出来了，很明显的区别，f方法是套在iterator的外边，这样才能对iterator的所有数据做一个合并。

  override val partitioner = if (preservesPartitioning) firstParent[T].partitioner else None
  override def compute(split: Partition, context: TaskContext) =
    f(context, split.index, firstParent[T].iterator(split, context))
}
 接下来我们看Aggregator的combineValuesByKey的方法吧。


复制代码
def combineValuesByKey(iter: Iterator[_ <: Product2[K, V]],
                         context: TaskContext): Iterator[(K, C)] = {
    // 是否使用外部排序，是由参数spark.shuffle.spill，默认是true
    if (!externalSorting) {
      val combiners = new AppendOnlyMap[K,C]
      var kv: Product2[K, V] = null
      val update = (hadValue: Boolean, oldValue: C) => {
        if (hadValue) mergeValue(oldValue, kv._2) else createCombiner(kv._2)
      }
      // 用map来去重，用update方法来更新值，如果没值的时候，返回值，如果有值的时候，通过mergeValue方法来合并
      // mergeValue方法就是我们在reduceByKey里面写的那个匿名函数，在这里就是（_ + _）
      while (iter.hasNext) {
        kv = iter.next()
        combiners.changeValue(kv._1, update)
      }
      combiners.iterator
    } else {  
      // 用了一个外部排序的map来去重，就不停的往里面插入值即可，基本原理和上面的差不多，区别在于需要外部排序   
      val combiners = new ExternalAppendOnlyMap[K, V, C](createCombiner, mergeValue, mergeCombiners)
      while (iter.hasNext) {
        val (k, v) = iter.next()
        combiners.insert(k, v)
      }
      combiners.iterator
}
复制代码
这个就是一个很典型的按照key来做合并的方法了，我们继续看ShuffledRDD吧。

ShuffledRDD和之前的RDD很明显的特征是

1、它的依赖传了一个Nil（空列表）进去，表示它没有依赖。

2、它的compute计算方式比较特别，这个在之后的文章说，过程比较复杂。

3、它的分片默认是采用HashPartitioner，数量和前面的RDD的分片数量一样，也可以不一样，我们可以在reduceByKey的时候多传一个分片数量即可。

在new完ShuffledRDD之后又来了一遍mapPartitionsWithContext，不过调用的匿名函数变成了combineCombinersByKey。

combineCombinersByKey和combineValuesByKey的逻辑基本相同，只是输入输出的类型有区别。combineCombinersByKey只是做单纯的合并，不会对输入输出的类型进行改变，combineValuesByKey会把iter[K, V]的V值变成iter[K, C]。

case class Aggregator[K, V, C] (
　　createCombiner: V => C,
　　mergeValue: (C, V) => C,
　　mergeCombiners: (C, C) => C)
　　......
}
 这个方法会根据我们传进去的匿名方法的参数的类型做一个自动转换。

到这里，作业都没有真正执行，只是将RDD各种嵌套，我们通过RDD的id和类型的变化观测到这一点，RDD[1]->RDD[2]->RDD[3]......



3、其它RDD
平常我们除了从hdfs上面取数据之后，我们还可能从数据库里面取数据，那怎么办呢？没关系，有个JdbcRDD！

复制代码
    val rdd = new JdbcRDD(
      sc,
      () => { DriverManager.getConnection("jdbc:derby:target/JdbcRDDSuiteDb") },
      "SELECT DATA FROM FOO WHERE ? <= ID AND ID <= ?",
      1, 100, 3,
      (r: ResultSet) => { r.getInt(1) } 
   ).cache()
复制代码
前几个参数大家都懂，我们重点说一下后面1, 100, 3是咋回事？

在这个JdbcRDD里面它默认我们是会按照一个long类型的字段对数据进行切分，（1,100）分别是最小值和最大值，3是分片的数量。

比如我们要一次查ID为1-1000,000的的用户，分成10个分片，我们就填（1, 1000,000， 10）即可，在sql语句里面还必须有"? <= ID AND ID <= ?"的句式，别尝试着自己造句哦！

最后是怎么处理ResultSet的方法，自己爱怎么处理怎么处理去吧。不过确实觉着用得不方便的可以自己重写一个RDD。

 

小结：

这一章重点介绍了各种RDD那5个特征，以及RDD之间的转换，希望大家可以对RDD有更深入的了解，下一章我们将要讲作业的运行过程，敬请关注！








1. 源码分析：SparkContext类
我们首先看textFile的这个方法，在SparkContext中。

看注释：

 Read a text file from HDFS, a local file system (available on all nodes), or any Hadoop-supported file system URI, and return it as an RDD of Strings.
 

其实textFile只是对hadoopFile方法做了一层封装。

注意： 此处有一个比较长的关系链，为了理解textfile中的逻辑，需要先看hadoopFile；hadoopFile最后返回的是一个HadoopRDD对象，然后HadoopRDD经过map变换后，转换成MapPartitionsRDD，由于HadoopRDD没有重写map函数，因此调用的是父类RDD的map；

def textFile(path: String, minPartitions: Int = defaultMinPartitions): RDD[String] = withScope {
  assertNotStopped()//暂时不用看
  hadoopFile(path, classOf[TextInputFormat], classOf[LongWritable], classOf[Text], minPartitions).map(pair => pair._2.toString).setName(path)
}
我们继续向下追，看一下hadoopFile方法，hadoopFile中做了这些事。

把hadoop的配置文件保存到广播变量里；
设置路径的方法；
new了一个HadoopRDD,并返回。
def hadoopFile[K, V](path: String,inputFormatClass: Class[_ <: InputFormat[K, V]],keyClass: Class[K],valueClass: Class[V],minPartitions: Int = defaultMinPartitions): RDD[(K, V)] = withScope {
  assertNotStopped()
  // A Hadoop configuration can be about 10 KB, which is pretty big, so broadcast it.
  val confBroadcast = broadcast(new SerializableConfiguration(hadoopConfiguration))
  val setInputPathsFunc = (jobConf: JobConf) => FileInputFormat.setInputPaths(jobConf, path)
  new HadoopRDD(this, confBroadcast,Some(setInputPathsFunc),inputFormatClass,keyClass,=valueClass,minPartitions).setName(path)
}
我们看一下它的输入参数。如果你写过MR程序，是不是特别熟悉？是不是和Mapper的API和接近？

1

Mapper<Object, Text, Text, IntWritable>

2. 源码分析：HadoopRDD类
下面专门来看一下HadoopRDD是干什么的。

 An RDD that provides core functionality for reading data stored in Hadoop (e.g., files in HDFS, sources in HBase, or S3)
 

看注释可以知道，HadoopRDD是一个专为Hadoop（HDFS、Hbase、S3）设计的RDD。

HadoopRDD主要重写了三个方法，可以在源码中找到加override标识的方法：

override def getPartitions: Array[Partition]
override def compute(theSplit: Partition, context: TaskContext): InterruptibleIterator[(K, V)]
override def getPreferredLocations(split: Partition): Seq[String]
下面分别看一下这三个方法。

getPartitions方法最后是返回了一个array。它调用的是inputFormat自带的getSplits方法来计算分片，然后把分片信息放到array中。

这里，我们是不是就可以理解，Hadoop中的一个分片，就对应到Spark中的一个Partition。

override def getPartitions: Array[Partition] = {
  val jobConf = getJobConf()
  // add the credentials here as this can be called before SparkContext initialized
  SparkHadoopUtil.get.addCredentials(jobConf)
  val inputFormat = getInputFormat(jobConf)
  val inputSplits = inputFormat.getSplits(jobConf, minPartitions)
  val array = new Array[Partition](inputSplits.size)
  for (i <- 0 until inputSplits.size) {
    array(i) = new HadoopPartition(id, i, inputSplits(i))
  }
  array
}
compute方法的作用主要就是根据输入的partition信息生成一个InterruptibleIterator。如下面代码段。

override def compute(theSplit: Partition, context: TaskContext): InterruptibleIterator[(K, V)] = {
    val iter = new NextIterator[(K, V)] {......}
    new InterruptibleIterator[(K, V)](context, iter)
  }
下面看一下iter中做了什么逻辑处理。

把Partition转成HadoopPartition，然后通过InputSplit创建一个RecordReader
重写Iterator的getNext方法，通过创建的reader调用next方法读取下一个值。
从这里我们可以看得出来compute方法是通过分片来获得Iterator接口，以遍历分片的数据。

override def compute(theSplit: Partition, context: TaskContext): InterruptibleIterator[(K, V)] = {

 val iter = new NextIterator[(K, V)] {

      //将compute的输入theSplit，转换为HadoopPartition
      val split = theSplit.asInstanceOf[HadoopPartition]
      ......
      //c重写getNext方法
      override def getNext(): (K, V) = {
        try {
          finished = !reader.next(key, value)
        } catch {
          case _: EOFException if ignoreCorruptFiles => finished = true
        }
        if (!finished) {
          inputMetrics.incRecordsRead(1)
        }
        (key, value)
      }
     }
}
getPreferredLocations方法比较简单，直接调用SplitInfoReflections下的inputSplitWithLocationInfo方法获得所在的位置。

override def getPreferredLocations(split: Partition): Seq[String] = {
  val hsplit = split.asInstanceOf[HadoopPartition].inputSplit.value
  val locs: Option[Seq[String]] = HadoopRDD.SPLIT_INFO_REFLECTIONS match {
    case Some(c) =>
      try {
        val lsplit = c.inputSplitWithLocationInfo.cast(hsplit)
        val infos = c.getLocationInfo.invoke(lsplit).asInstanceOf[Array[AnyRef]]
        Some(HadoopRDD.convertSplitLocationInfo(infos))
      } catch {
        case e: Exception =>
          logDebug("Failed to use InputSplitWithLocations.", e)
          None
      }
    case None => None
  }
  locs.getOrElse(hsplit.getLocations.filter(_ != "localhost"))
}
3. 源码分析：MapPartitionsRDD类
先看一在RDD类中的map方法。

 Return a new RDD by applying a function to all elements of this RDD.
 

它最后返回的是一个MapPartitionsRDD。并且对RDD中的每一个元素都调用了一个function。

/**
 *
 */
def map[U: ClassTag](f: T => U): RDD[U] = withScope {
  val cleanF = sc.clean(f)
  new MapPartitionsRDD[U, T](this, (context, pid, iter) => iter.map(cleanF))
}
那么MapPartitionsRDD是干什么的呢。

 An RDD that applies the provided function to every partition of the parent RDD.
 

可以看到，它重写了父类RDD的partitioner、getPartitions和compute。

private[spark] class MapPartitionsRDD[U: ClassTag, T: ClassTag](
    var prev: RDD[T],
    f: (TaskContext, Int, Iterator[T]) => Iterator[U],  // (TaskContext, partition index, iterator)
    preservesPartitioning: Boolean = false)
  extends RDD[U](prev) {
  override val partitioner = if (preservesPartitioning) firstParent[T].partitioner else None
  override def getPartitions: Array[Partition] = firstParent[T].partitions
  override def compute(split: Partition, context: TaskContext): Iterator[U] =
    f(context, split.index, firstParent[T].iterator(split, context))
  override def clearDependencies() {
    super.clearDependencies()
    prev = null
  }
}
可以看出在MapPartitionsRDD里面都用到一个firstParent函数。仔细看一下，可以发现，在MapPartitionsRDD其实没有重写partition和compute逻辑，只是从firstParent中取了出来。

那么firstParent是干什么的呢？其实是取到父依赖。

/** Returns the first parent RDD */
protected[spark] def firstParent[U: ClassTag]: RDD[U] = {
  dependencies.head.rdd.asInstanceOf[RDD[U]]
}
注意： 不要忽略细节。不然不太容易理解。

现在再看一下MapPartitionsRDD继承的RDD，它继承的是RDD[U](prev)。 这里的prev其实指的就是我们的HadoopRDD，也也就是说HadoopRDD变成了这个MapPartitionsRDD的OneToOneDependency依赖。OneToOneDependency是窄依赖。

def this(@transient oneParent: RDD[_]) =
    this(oneParent.context , List(new OneToOneDependency(oneParent)))
总结： 至此，我们阅读了第一行代码背后涉及的源码。val textFile = sc.textFile("hdfs://...")，我们继续进行。不要急，后面会快很多。

4. 源码分析：flatMap方法、filter方法
接下面看一下flatMap、filter和map操作，观察一下下面的代码，其实他们都是返回了MapPartitionsRDD对象，不同的仅仅是传入的function不同而已。

经过前面的分析我们也可以知道，这些都是窄依赖。

/**
 *  Return a new RDD by first applying a function to all elements of this
 *  RDD, and then flattening the results.
 */
def flatMap[U: ClassTag](f: T => TraversableOnce[U]): RDD[U] = withScope {
  val cleanF = sc.clean(f)
  new MapPartitionsRDD[U, T](this, (context, pid, iter) => iter.flatMap(cleanF))
}
/**
  * Return a new RDD containing only the elements that satisfy a predicate.
  */
 def filter(f: T => Boolean): RDD[T] = withScope {
   val cleanF = sc.clean(f)
   new MapPartitionsRDD[T, T](
     this,
     (context, pid, iter) => iter.filter(cleanF),
     preservesPartitioning = true)
 }
注意： 这里，我们可以明白了MapPartitionsRDD的compute方法的作用了：

在没有依赖的条件下，根据分片的信息生成遍历数据的Iterable接口
在有前置依赖的条件下，在父RDD的Iterable接口上给遍历每个元素的时候再套上一个方法
5. 源码分析：PairRDDFunctions 类
接下来，该reduceByKey操作了。它在PairRDDFunctions里面。

reduceByKey稍微复杂一点，因为这里有一个同相同key的内容聚合的一个过程，它调用的是combineByKey方法。

/**
   * Merge the values for each key using an associative reduce function. This will also perform
   * the merging locally on each mapper before sending results to a reducer, similarly to a
   * "combiner" in MapReduce.
   */
  def reduceByKey(partitioner: Partitioner, func: (V, V) => V): RDD[(K, V)] = self.withScope {
    combineByKeyWithClassTag[V]((v: V) => v, func, func, partitioner)
  }
下面详细看一下combineByKeyWithClassTag。英文的注释写的挺清晰的，不再做讲解了。只提一点，这个操作，最后会new一个ShuffledRDD，然后调用它的一些方法，下面专门分析一下这个步骤。

 Generic function to combine the elements for each key using a custom set of aggregation functions. Turns an RDD[(K, V)] into a result of type RDD[(K, C)], for a “combined type” C (Int, Int) into an RDD of type (Int, Seq[Int]). Users provide three functions:
 

createCombiner, which turns a V into a C (e.g., creates a one-element list)
mergeValue, to merge a V into a C (e.g., adds it to the end of a list)
mergeCombiners, to combine two C’s into a single one.
 In addition, users can control the partitioning of the output RDD, and whether to perform map-side aggregation (if a mapper can produce multiple items with the same key).
 

def combineByKeyWithClassTag[C]( createCombiner: V => C,mergeValue: (C, V) => C,mergeCombiners: (C, C) => C,partitioner: Partitioner,mapSideCombine: Boolean = true,serializer: Serializer = null)(implicit ct: ClassTag[C]): RDD[(K, C)] = self.withScope {
    require(mergeCombiners != null, "mergeCombiners must be defined") // required as of Spark 0.9.0
    // 判断keyclass是不是array类型，如果是array并且在两种情况下throw exception。
    if (keyClass.isArray) {
      if (mapSideCombine) {
        throw new SparkException("Cannot use map-side combining with array keys.")
      }
      if (partitioner.isInstanceOf[HashPartitioner]) {
        throw new SparkException("Default partitioner cannot partition array keys.")
      }
    }
    val aggregator = new Aggregator[K, V, C](
      self.context.clean(createCombiner),
      self.context.clean(mergeValue),
      self.context.clean(mergeCombiners))
    //虽然不太明白，但是此处基本上一直是false，感兴趣的看后面的参考文章
    if (self.partitioner == Some(partitioner)) {
      self.mapPartitions(iter => {
        val context = TaskContext.get()
        new InterruptibleIterator(context, aggregator.combineValuesByKey(iter, context))
      }, preservesPartitioning = true)
    } else {
      // 默认是走的这个方法
      new ShuffledRDD[K, V, C](self, partitioner)
        .setSerializer(serializer)
        .setAggregator(aggregator)
        .setMapSideCombine(mapSideCombine)
    }
  }
6. 源码分析：ShuffledRDD 类
看一下上一段代码最后做了什么？这里传入了partitioner，并分别set了三个值。

new ShuffledRDD[K, V, C](self, partitioner)
        .setSerializer(serializer)
        .setAggregator(aggregator)
        .setMapSideCombine(mapSideCombine)
shuffle的过程有点复杂，先不深入讲解，后面专门来分析。这里先看一下依赖的关系ShuffleDependency，它是一个宽依赖。

override def getDependencies: Seq[Dependency[_]] = {
  List(new ShuffleDependency(prev, part, serializer, keyOrdering, aggregator, mapSideCombine))
}
总结： 至此，我们大致理了一遍RDD的转换过程，其实到现在为止还都是RDD的变换，还没有真正的执行，真正的执行会在最后一句的地方出发。

0x03 总结
本来在这篇博客中是想把RDD的shuffle的原理也写清楚，但是错估了一些工作量，前面的东西从学习整理到写出来就画了5个多小时。有点累了，加上已经10点半了，准备休息。 下次会专门讲清楚。


 