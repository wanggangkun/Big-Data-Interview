##一、SparkContext的初始化

在开始介绍SparkContext，SparkContext的初始化步骤如下：

1. 创建Spark执行环境SparkEnv；
2. 创建RDD清理器metadataCleaner；
3. 创建并初始化SparkUI；
4. Hadoop相关配置及Executor环境变量的设置
5. 创建任务调度TaskScheduler；
6. 创建和启动DAGScheduler；
7. TaskScheduler的启动；
8. 初始化块管理器BlockManager（BlockManager是存储体系的主要组件之一，将在第4章介绍）；
9. 启动测量系统MetricsSystem；
10. 创建和启动Executor分配管理器ExecutorAllocationManager；
11. ContextCleaner的创建与启动；
12. Spark环境更新；
13. 创建DAGSchedulerSource和BlockManagerSource；
14. 将SparkContext标记为激活。

 

SparkContext的主构造器参数为SparkConf，其实现如下。

```scala
class SparkContext(config: SparkConf) extends Logging withExecutorAllocationClient {  
private val creationSite: CallSite = Utils.getCallSite()  
  private val allowMultipleContexts:Boolean =  
   config.getBoolean("spark.driver.allowMultipleContexts", false)  
 SparkContext.markPartiallyConstructed(this, allowMultipleContexts)  
```
上面代码中的CallSite存储了线程栈中最靠近栈顶的用户类及最靠近栈底的Scala或者Spark核心类信息。Utils.getCallSite的详细信息见附录A。SparkContext默认只有一个实例（由属性spark.driver.allowMultipleContexts来控制，用户需要多个SparkContext实例时，可以将其设置为true），方法markPartiallyConstructed用来确保实例的唯一性，并将当前SparkContext标记为正在构建中。

接下来会对SparkConf进行拷贝，然后对各种配置信息进行校验，代码如下。

```scala
private[spark] val conf =config.clone()  
conf.validateSettings()  
  
if (!conf.contains("spark.master")) {  
  throw newSparkException("A master URL must be set in your configuration")  
}  
if (!conf.contains("spark.app.name")) {  
  throw newSparkException("An application name must be set in yourconfiguration")  
}  
```
从上面校验的代码看到必须指定属性spark.master 和spark.app.name，否则会抛出异常，结束初始化过程。spark.master用于设置部署模式，spark.app.name指定应用程序名称。

##二、创建执行环境SparkEnv

SparkEnv是Spark的执行环境对象，其中包括众多与Executor执行相关的对象。由于在local模式下Driver会创建Executor，local-cluster部署模式或者Standalone部署模式下Worker另起的CoarseGrainedExecutorBackend进程中也会创建Executor，所以SparkEnv存在于Driver或者CoarseGrainedExecutorBackend进程中。创建SparkEnv 主要使用SparkEnv的createDriverEnv，createDriverEnv方法有三个参数，conf、isLocal和 listenerBus。

```scala
 val isLocal = (master == "local" ||master.startsWith("local["))  
 private[spark] vallistenerBus = newLiveListenerBus  
 conf.set("spark.executor.id","driver")  
  
 private[spark] valenv =SparkEnv.createDriverEnv(conf,isLocal, listenerBus)  
SparkEnv.set(env)  
```

上面代码中的conf是对SparkConf的拷贝，isLocal标识是否是单机模式，listenerBus采用监听器模式维护各类事件的处理。

SparkEnv的方法createDriverEnv最终调用create创建SparkEnv。SparkEnv的构造步骤如下：

1. 创建安全管理器SecurityManager；
2. 创建基于Akka的分布式消息系统ActorSystem；
3. 创建Map任务输出跟踪器mapOutputTracker；
4. 实例化ShuffleManager；
5. 创建ShuffleMemoryManager；
6. 创建块传输服务BlockTransferService；
7. 创建BlockManagerMaster；
8. 创建块管理器BlockManager；
9. 创建广播管理器BroadcastManager；
10. 创建缓存管理器CacheManager；
11. 创建HTTP文件服务器HttpFileServer；
12. 创建测量系统MetricsSystem；
13. 创建SparkEnv；

 

###2.1 安全管理器SecurityManager

SecurityManager主要对权限、账号进行设置，如果使用Hadoop YARN作为集群管理器，则需要使用证书生成 secret key登录，最后给当前系统设置默认的口令认证实例，此实例采用匿名内部类实现，参见代码清单3-2。

```scala
private val secretKey =generateSecretKey()  
  
 // 使用HTTP连接设置口令认证  
 if (authOn) {  
  Authenticator.setDefault(  
     newAuthenticator() {  
       override defgetPasswordAuthentication(): PasswordAuthentication = {  
         var passAuth:PasswordAuthentication = null  
         val userInfo =getRequestingURL().getUserInfo()  
         if (userInfo !=null) {  
           val  parts = userInfo.split(":",2)  
           passAuth = newPasswordAuthentication(parts(0),parts(1).toCharArray())  
         }  
         return passAuth  
       }  
     }  
   )  
 }
```

###2.2 基于Akka的分布式消息系统ActorSystem

ActorSystem是Spark中最基础的设施，Spark既使用它发送分布式消息，又用它实现并发编程。怎么，消息系统可以实现并发？要解释清楚这个问题，首先应该简单的介绍下Scala语言的Actor并发编程模型：Scala认为Java线程通过共享数据以及通过锁来维护共享数据的一致性是糟糕的做法，容易引起锁的争用，而且线程的上下文切换会带来不少开销，降低并发程序的性能，甚至会引入死锁的问题。在Scala中只需要自定义类型继承Actor，并且提供act方法，就如同Java里实现Runnable接口，需要实现run方法一样。但是不能直接调用act方法，而是通过发送消息的方式(Scala发送消息是异步的)，传递数据。如：

Akka是Actor编程模型的高级类库，类似于JDK 1.5之后越来越丰富的并发工具包，简化了程序员并发编程的难度。ActorSystem便是Akka提供的用于创建分布式消息通信系统的基础类。Akka的具体信息见附录B。

正式因为Actor轻量级的并发编程、消息发送以及ActorSystem支持分布式消息发送等特点，Spark选择了ActorSystem。

ActorSystem的创建和启动

```scala
def createActorSystem(  
    name:String,  
    host:String,  
    port:Int,  
    conf:SparkConf,  
    securityManager: SecurityManager):(ActorSystem, Int) = {  
  val startService: Int=> (ActorSystem, Int) = { actualPort =>  
   doCreateActorSystem(name, host, actualPort, conf, securityManager)  
  }  
 Utils.startServiceOnPort(port, startService, conf, name)  
}  
```

###2.3 map任务输出跟踪器mapOutputTracker

mapOutputTracker用于跟踪map阶段任务的输出状态，此状态便于reduce阶段任务获取地址及中间输出结果。每个map任务或者reduce任务都会有其唯一标识，分别为mapId和reduceId。每个reduce任务的输入可能是多个map任务的输出，reduce会到各个map任务的所在节点上拉取Block，这一过程叫做shuffle。每批shuffle过程都有唯一的标识shuffleId。

这里先介绍下MapOutputTrackerMaster。MapOutputTrackerMaster内部使用mapStatuses：TimeStampedHashMap[Int,Array[MapStatus]]来维护跟踪各个map任务的输出状态。其中key对应shuffleId，Array存储各个map任务对应的状态信息MapStatus。由于MapStatus维护了map输出Block的地址BlockManagerId，所以reduce任务知道从何处获取map任务的中间输出。MapOutputTrackerMaster还使用cachedSerializedStatuses：TimeStampedHashMap[Int, Array[Byte]]维护序列化后的各个map任务的输出状态。其中key对应shuffleId，Array存储各个序列化MapStatus生成的字节数组。

Driver和Executor处理MapOutputTrackerMaster的方式有所不同：

+ 如果当前应用程序是Driver，则创建MapOutputTrackerMaster，然后创建MapOutputTrackerMasterActor，并且注册到ActorSystem中。
+ 如果当前应用程序是Executor，则创建MapOutputTrackerWorker，并从ActorSystem中找到MapOutputTrackerMasterActor。
+ 无论是Driver还是Executor，最后都由mapOutputTracker的属性trackerActor持有MapOutputTrackerMasterActor的引用

registerOrLookup方法用于查找或者注册Actor的实现

```scala
def registerOrLookup(name: String, newActor: => Actor): ActorRef ={  
      if (isDriver) {  
       logInfo("Registering" + name)  
        actorSystem.actorOf(Props(newActor),name = name)  
      } else {  
       AkkaUtils.makeDriverRef(name, conf, actorSystem)  
      }  
    }  
   
    val mapOutputTracker=  if (isDriver) {  
      newMapOutputTrackerMaster(conf)  
    } else {  
      newMapOutputTrackerWorker(conf)  
}  
   
    mapOutputTracker.trackerActor= registerOrLookup(  
     "MapOutputTracker",  
      newMapOutputTrackerMasterActor(mapOutputTracker.asInstanceOf[MapOutputTrackerMaster], conf))  
```
在后面章节大家会知道map任务的状态正是由Executor向持有的MapOutputTrackerMasterActor发送消息，将map任务状态同步到mapOutputTracker的mapStatuses和cachedSerializedStatuses的。Executor究竟是如何找到MapOutputTrackerMasterActor的？registerOrLookup方法通过调用AkkaUtils.makeDriverRef找到MapOutputTrackerMasterActor，实际正是利用ActorSystem提供的分布式消息机制实现的，具体细节参见附录B。这里第一次使用到了Akka提供的功能，以后大家会渐渐感觉到使用Akka的便捷。

###2.4 实例化ShuffleManager

ShuffleManager负责管理本地及远程的block数据的shuffle操作。ShuffleManager默认为通过反射方式生成的SortShuffleManager的实例，可以修改属性spark.shuffle.manager为hash来显式使用HashShuffleManager。SortShuffleManager通过持有的IndexShuffleBlockManager间接操作BlockManager中的DiskBlockManager将map结果写入本地，并根据shuffleId、mapId写入索引文件，也能通过MapOutputTrackerMaster中维护的mapStatuses从本地或者其他远程节点读取文件。有读者可能会问，为什么需要shuffle？Spark作为并行计算框架，同一个作业会被划分为多个任务在多个节点上并行执行，reduce的输入可能存在于多个节点上，因此需要通过“洗牌”将所有reduce的输入汇总起来，这个过程就是shuffle。

代码清单3-6  ShuffleManager的实例化及ShuffleMemoryManager的创建
```scala
val shortShuffleMgrNames =Map(  
  "hash"-> "org.apache.spark.shuffle.hash.HashShuffleManager",  
  "sort"-> "org.apache.spark.shuffle.sort.SortShuffleManager")  
val shuffleMgrName = conf.get("spark.shuffle.manager", "sort")  
val shuffleMgrClass = shortShuffleMgrNames.get  
se(shuffleMgrName.toLowerCase, shuffleMgrName)  
val shuffleManager = instantiateClass[ShuffleManager](shuffleMgrClass)  
  
val shuffleMemoryManager =new ShuffleMemoryManager(conf)  
```
###2.5 shuffle线程内存管理器ShuffleMemoryManager

ShuffleMemoryManager负责管理shuffle线程占有内存的分配与释放，并通过threadMemory：mutable.HashMap[Long, Long]缓存每个线程的内存字节数。
```scala
private[spark] class ShuffleMemoryManager(maxMemory: Long)extends Logging {  
  private val threadMemory = newmutable.HashMap[Long, Long]() // threadId -> memory bytes  
  def this(conf: SparkConf) = this(ShuffleMemoryManager.getMaxMemory(conf))  
 
getMaxMemory方法用于获取shuffle所有线程占用的最大内存，实现如下。

def getMaxMemory(conf: SparkConf): Long = {  
    val memoryFraction =conf.getDouble("spark.shuffle.memoryFraction", 0.2)  
    val safetyFraction =conf.getDouble("spark.shuffle.safetyFraction", 0.8)  
   (Runtime.getRuntime.maxMemory * memoryFraction *safetyFraction).toLong  
  }  
```
从上面代码可以看出，shuffle所有线程占用的最大内存的计算公式为：

Java运行时最大内存 * Spark的shuffle最大内存占比 * Spark的安全内存占比

可以配置属性spark.shuffle.memoryFraction修改Spark的shuffle最大内存占比，配置属性spark.shuffle.safetyFraction修改Spark的安全内存占比。

注意：ShuffleMemoryManager通常运行在Executor中， Driver中的ShuffleMemoryManager 只有在local模式下才起作用。

###2.6 块传输服务BlockTransferService

BlockTransferService默认为NettyBlockTransferService（可以配置属性spark.shuffle.blockTransferService使用NioBlockTransferService），它使用Netty提供的异步事件驱动的网络应用框架，提供web服务及客户端，获取远程节点上Block的集合。

```scala
val blockTransferService =  
     conf.get("spark.shuffle.blockTransferService","netty").toLowerCase match {  
        case "netty"=>  
          newNettyBlockTransferService(conf, securityManager, numUsableCores)  
        case "nio"=>  
          newNioBlockTransferService(conf, securityManager)  
      }  
```
NettyBlockTransferService的具体实现将在第4章详细介绍。这里大家可能觉得奇怪，这样的网络应用为何也要放在存储体系？大家不妨先带着疑问，直到你真正了解存储体系。

###2.7 BlockManagerMaster介绍

BlockManagerMaster负责对Block的管理和协调，具体操作依赖于BlockManagerMasterActor。Driver和Executor处理BlockManagerMaster的方式不同：

+ 如果当前应用程序是Driver，则创建BlockManagerMasterActor，并且注册到ActorSystem中。
+ 如果当前应用程序是Executor，则从ActorSystem中找到BlockManagerMasterActor。
+ 无论是Driver还是Executor，最后BlockManagerMaster的属性driverActor将持有对BlockManagerMasterActor的引用。BlockManagerMaster的创建代码如下。

```scala
val blockManagerMaster = new BlockManagerMaster(registerOrLookup(  
     "BlockManagerMaster",  
      newBlockManagerMasterActor(isLocal, conf, listenerBus)), conf, isDriver)  
```
registerOrLookup已在3.2.3节介绍过了，不再赘述。BlockManagerMaster及BlockManagerMasterActor的具体实现将在第4章详细介绍。

###2.8 创建块管理器BlockManager

BlockManager负责对Block的管理，只有在BlockManager的初始化方法initialize被调用后，它才是有效的。BlockManager作为存储系统的一部分，具体实现见第4章。BlockManager的创建代码如下。

```scala
val blockManager = new BlockManager(executorId, actorSystem, blockManagerMaster,  
     serializer, conf, mapOutputTracker, shuffleManager,blockTransferService, securityManager,   numUsableCores)  
```
3.2.9 创建广播管理器BroadcastManager

BroadcastManager用于将配置信息和序列化后的RDD、Job以及ShuffleDependency等信息在本地存储。如果为了容灾，也会复制到其他节点上。创建BroadcastManager的代码实现如下。

>val broadcastManager = new BroadcastManager(isDriver, conf,securityManager)  

BroadcastManager必须在其初始化方法initialize被调用后，才能生效。Initialize方法实际利用反射生成广播工厂实例broadcastFactory（可以配置属性spark.broadcast.factory指定，默认为org.apache.spark.broadcast.TorrentBroadcastFactory）。BroadcastManager的广播方法newBroadcast实际代理了工厂broadcastFactory的newBroadcast方法来生成广播或者非广播对象。BroadcastManager的Initialize及newBroadcast方法见代码清单3-8。

代码清单3-8  BroadcastManager的实现

```scala
  private def initialize(){  
   synchronized {  
      if(!initialized) {  
        val broadcastFactoryClass = conf.get("spark.broadcast.factory","org.apache.spark.broadcast.TorrentBroadcastFactory")  
       broadcastFactory =  
         Class.forName(broadcastFactoryClass).newInstance.asInstanceOf[BroadcastFactory]  
       broadcastFactory.initialize(isDriver, conf, securityManager)  
       initialized = true  
      }  
    }  
  }  
  private val nextBroadcastId = new AtomicLong(0)  
  defnewBroadcast[T: ClassTag](value_ : T, isLocal: Boolean) = {  
   broadcastFactory.newBroadcast[T](value_, isLocal, nextBroadcastId.getAndIncrement())  
  }  
  defunbroadcast(id: Long, removeFromDriver: Boolean, blocking: Boolean) {  
   broadcastFactory.unbroadcast(id, removeFromDriver, blocking)  
  }  
}
```

###2.10 创建缓存管理器CacheManager

CacheManager用于缓存RDD某个分区计算后中间结果，缓存计算结果发生在迭代计算的时候，将在6.1节讲到。而CacheManager将在4.14节详细描述。创建CacheManager的代码如下。

>val cacheManager = new CacheManager(blockManager)  

###2.11 HTTP文件服务器HttpFileServer

HttpFileServer主要提供对jar及其他文件的http访问，这些jar包包括用户上传的jar包。端口由属性spark.fileserver.port配置，默认为0，表示随机生成端口号。

代码清单3-9  HttpFileServer的创建

```scala
val httpFileServer =  
    if (isDriver) {  
      val fileServerPort = conf.getInt("spark.fileserver.port",0)  
      val server = newHttpFileServer(conf, securityManager, fileServerPort)  
     server.initialize()  
     conf.set("spark.fileserver.uri",  server.serverUri)  
     server  
    } else {  
      null  
    }  
```
