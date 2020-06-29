[](https://www.jianshu.com/p/a21b3be88afd)


####3.3、Spark RDD 分区函数

1. HashPartition

HashPartitioner确定分区的方式：partition = key.hashCode () % numPartitions
弊端：弊端是数据不均匀，容易导致数据倾斜，极端情况下某几个分区会拥有rdd的所有数据。

2. RangePartitioner

RangePartitioner会对key值进行排序，然后将key值被划分成分区份数key值集合。

特点:RangePartitioner分区则尽量保证每个分区中数据量的均匀，而且分区与分区之间是有序的，也就是说一个分区中的元素肯定都是比另一个分区内的元素小或者大；但是分区内的元素是不能保证顺序的。简单的说就是将一定范围内的数映射到某一个分区内。其原理是水塘抽样 -----水塘抽样(Reservoir Sampling)问题。

水塘抽样：

在总数不知道的情况下如何等概率地从中抽取一行？即是说如果最后发现文字档共有N行，则每一行被抽取的概率均为1/N？

　　我们可以：定义取出的行号为choice，第一次直接以第一行作为取出行 choice ，而后第二次以二分之一概率决定是否用第二行替换 choice ，第三次以三分之一的概率决定是否以第三行替换 choice ……，以此类推。由上面的分析我们可以得出结论，在取第n个数据的时候，我们生成一个0到1的随机数p，如果p小于1/n，保留第n个数。大于1/n，继续保留前面的数。直到数据流结束，返回此数，算法结束。



3.CustomPartitioner

CustomPartitioner可以根据自己具体的应用需求，自定义分区。

```java
class CustomPartitioner(numParts: Int) extends Partitioner {
 override def numPartitions: Int = numParts
 override def getPartition(key: Any): Int =
 {
       if(key==1)){
    0
       } else if (key==2){
       1} else{ 
       2 }
  } 
}
```
Api解释：

1）spark默认实现了HashPartitioner和RangePartitioner两种分区策略，我们也可以自己扩展分区策略，自定义分区器的时候继承org.apache.spark.Partitioner类，实现类中的三个方法:(同一个key)

+ def numPartitions: Int：这个方法需要返回你想要创建分区的个数；
+ def getPartition(key: Any): Int：这个函数需要对输入的key做计算，然后返回该key的分区ID，范围一定是0到numPartitions-1；
+ equals()：这个是Java标准的判断相等的函数，之所以要求用户实现这个函数是因为Spark内部会比较两个RDD的分区是否一样。

2）使用，调用parttionBy方法中传入自定义分区对象

理解Spark从HDFS读入文件默认是怎样分区的

Spark从HDFS读入文件的分区数默认等于HDFS文件的块数(blocks)，HDFS中的block是分布式存储的最小单元。如果我们上传一个30GB的非压缩的文件到HDFS，HDFS默认的块容量大小128MB，因此该文件在HDFS上会被分为235块(30GB/128MB)；Spark读取SparkContext.textFile()读取该文件，默认分区数等于块数即235。



4. RangePartitioner分区

  从HashPartitioner分区的实现原理可以看出，其结果可能导致每个分区中数据量的不均匀。而RangePartitioner分区则尽量保证每个分区中数据量的均匀，而且分区与分区之间是有序的，但是分区内的元素是不能保证顺序的。sortByKey底层就是RangePartitioner分区器。

  首先了解蓄水池抽样(Reservoir Sampling)，它能够在O(n)时间内对n个数据进行等概率随机抽取。首先构建一个可放k个元素的蓄水池，将序列的前k个元素放入蓄水池中。然后从第k+1个元素开始，以k/n的概率来替换掉蓄水池中国的某个元素即可。当遍历完所有元素之后，就可以得到随机挑选出的k个元素，复杂度为O(n)。

  RangePartitioner分区器的主要作用就是将一定范围内的数映射到某一个分区内。该分区器的实现方式主要是通过两个步骤来实现的，第一步，先从整个RDD中抽取出样本数据，将样本数据排序，计算出每个分区的最大key值，形成一个Array[KEY]类型的数组变量rangeBounds；第二步，判断key在rangeBounds中所处的范围，给出该key的分区ID。

RangePartitioner的重点是在于构建rangeBounds数组对象，主要步骤是：

1. 计算总体的数据抽样大小sampleSize，计算规则是：(math.min(20.0 * partitions, 1e6))，至少每个分区抽取20个数据或者最多1M的数据量。 对父RDD的分区进行抽样。

2. 根据sampleSize和分区数量计算每个分区的数据抽样样本数量sampleSizePrePartition(math.ceil(3.0 * sampleSize / rdd.partitions.length).toInt)，即每个分区抽取的数据量一般会比之前计算的大一点)

3. 调用RangePartitioner的sketch函数进行数据抽样，计算出每个分区的样本

4. 计算样本的整体占比以及数据量过多的数据分区，防止数据倾斜

5. 对于数据量比较多的RDD分区调用RDD的sample函数API重新进行数据抽取

6. 将最终的样本数据通过RangePartitoner的determineBounds函数进行数据排序分配，计算出rangeBounds

  RangePartitioner的sketch函数的作用是对RDD中的数据按照需要的样本数据量进行数据抽取，主要调用SamplingUtils类的reservoirSampleAndCount方法对每个分区进行数据抽取，抽取后计算出整体所有分区的数据量大小；reservoirSampleAndCount方法的抽取方式是先从迭代器中获取样本数量个数据(顺序获取), 然后对剩余的数据进行判断，替换之前的样本数据，最终达到数据抽样的效果。RangePartitioner的determineBounds函数的作用是根据样本数据记忆权重大小确定数据边界。

