###SparkContext、SparkConf、SparkSession

###SparkConf

SparkContext实例化的时候需要传进一个SparkConf作为参数，SparkConf描述整个Spark应用程序的配置信息，如果和系统默认的配置冲突就会覆盖系统默认的设置。我们经常会在单元测试的时候使用new SparkConf(fasle)(如果不传入参数，默认是true)实例化SparkConf，这样就不会加载“conf/”下默认的配置，这样无论在什么样的集群环境中运行单元测试，其配置都是一样的，不会随着环境的变化而变化。另外在程序运行的时候不能修改SparkConf，数据结构通过ConcurrentHashMap进行维护，只能通过clone的方式读取Spark的配置信息。最后需要说明的是SparkConf可以进行链式的调用，即：

>new SparkConf().setMaster("local").setAppName("TestApp")

因为这些方法在设置完配置信息后最终都返回了自己，即SparkConf本身，SparkConf的部分源码如下：

```scala
// 用来存储key-value的配置信息
private val settings = new ConcurrentHashMap[String, String]()
// 默认会加载“spark.*”格式的配置信息
if (loadDefaults) {
  // Load any spark.* system properties
  for ((key, value) <- Utils.getSystemProperties if key.startsWith("spark.")) {
    set(key, value)
  }
}
/** Set a configuration variable. */
def set(key: String, value: String): SparkConf = {
  if (key == null) {
    throw new NullPointerException("null key")
  }
  if (value == null) {
    throw new NullPointerException("null value for " + key)
  }
  logDeprecationWarning(key)
  settings.put(key, value)
  // 这里我们可以清楚的看到，每次进行设置后都会返回SparkConf自身，所以可以进行链式的调用
  this
}
```

**SparkContext**

SparkContext是整个Spark功能的入口，代表了应用程序与整个集群的连接点，通过SparkContext可以创建RDD、Accumulators和Broadcast。

Spark应用程序是通过SparkContext发布到Spark集群的，并且Spark程序的运行都是在SparkContext为核心的调度指挥下进行的，SparkContext崩溃或者结束就代表Spark应用程序执行结束，可见SparkContext在Spark中是多么的重要，下面我们结合源码进行详细分析(只选取重要部分)：

```scala
// 方便开发人员查看调用的信息
// The call site where this SparkContext was constructed.
private val creationSite: CallSite = Utils.getCallSite()
// 是否允许存在多个SparkContext，默认是false
// If true, log warnings instead of throwing exceptions when multiple SparkContexts are active
private val allowMultipleContexts: Boolean =
  config.getBoolean("spark.driver.allowMultipleContexts", false)
// In order to prevent multiple SparkContexts from being active at the same time, mark this
// context as having started construction.
// NOTE: this must be placed at the beginning of the SparkContext constructor.
SparkContext.markPartiallyConstructed(this, allowMultipleContexts)
val startTime = System.currentTimeMillis()
// 判断上下文状态
private[spark] val stopped: AtomicBoolean = new AtomicBoolean(false)

...

// 消息通信相关的内容，我们会单独进行说明
// An asynchronous listener bus for Spark events
private[spark] val listenerBus = new LiveListenerBus

...

// 追踪所有执行持久化的RDD
// Keeps track of all persisted RDDs
private[spark] val persistentRdds = new TimeStampedWeakValueHashMap[Int, RDD[_]]

...

// 传递给executors的环境变量信息
// Environment variables to pass to our executors.
private[spark] val executorEnvs = HashMap[String, String]()
// 配置SPARK_USER
// Set SPARK_USER for user who is running SparkContext.
val sparkUser = Utils.getCurrentUserName()
```
同时还有几个重载的构造方法，我们不进行一一说明，下面我们来看SparkContext中最重要的一个部分，即try里面的内容(大部分初始化的工作都在这里面，因为内容较多，大家可以根据具体的功能点进行查看)：

```scala
try {
  // 读取SparkConf的信息，即Spark的配置信息，并检查是否有非法的配置信息
  _conf = config.clone()
  _conf.validateSettings()
  // 判断是否配置了Master，没有的话抛出异常
  if (!_conf.contains("spark.master")) {
    throw new SparkException("A master URL must be set in your configuration")
  }
  // 判断是否配置了AppName，没有的话抛出异常
  if (!_conf.contains("spark.app.name")) {
    throw new SparkException("An application name must be set in your configuration")
  }
  // Yarn模式下的判断
  // System property spark.yarn.app.id must be set if user code ran by AM on a YARN cluster
  // yarn-standalone is deprecated, but still supported
  if ((master == "yarn-cluster" || master == "yarn-standalone") &&
      !_conf.contains("spark.yarn.app.id")) {
    throw new SparkException("Detected yarn-cluster mode, but isn't running on a cluster. " +
      "Deployment to YARN is not supported directly by SparkContext. Please use spark-submit.")
  }
  if (_conf.getBoolean("spark.logConf", false)) {
    logInfo("Spark configuration:\n" + _conf.toDebugString)
  }
  
  // 设置Driver的host和port
  // Set Spark driver host and port system properties
  _conf.setIfMissing("spark.driver.host", Utils.localHostName())
  _conf.setIfMissing("spark.driver.port", "0")
  _conf.set("spark.executor.id", SparkContext.DRIVER_IDENTIFIER)
  _jars = _conf.getOption("spark.jars").map(_.split(",")).map(_.filter(_.size != 0)).toSeq.flatten
  _files = _conf.getOption("spark.files").map(_.split(",")).map(_.filter(_.size != 0))
    .toSeq.flatten
  _eventLogDir =
    if (isEventLogEnabled) {
      // 这里需要进行设置，否则默认路径是“/tmp/spark-events”，防止系统自动清除
      val unresolvedDir = conf.get("spark.eventLog.dir", EventLoggingListener.DEFAULT_LOG_DIR)
        .stripSuffix("/")
      Some(Utils.resolveURI(unresolvedDir))
    } else {
      None
    }
  _eventLogCodec = {
    val compress = _conf.getBoolean("spark.eventLog.compress", false)
    if (compress && isEventLogEnabled) {
      Some(CompressionCodec.getCodecName(_conf)).map(CompressionCodec.getShortName)
    } else {
      None
    }
  }
  _conf.set("spark.externalBlockStore.folderName", externalBlockStoreFolderName)
  if (master == "yarn-client") System.setProperty("SPARK_YARN_MODE", "true")
  // "_jobProgressListener" should be set up before creating SparkEnv because when creating
  // "SparkEnv", some messages will be posted to "listenerBus" and we should not miss them.
  _jobProgressListener = new JobProgressListener(_conf)
  // 关于ListenerBus我们会单独分析
  listenerBus.addListener(jobProgressListener)
  // Create the Spark execution environment (cache, map output tracker, etc)
  // 创建SparkEnv
  _env = createSparkEnv(_conf, isLocal, listenerBus)
  SparkEnv.set(_env)
  // 元数据清理器
  _metadataCleaner = new MetadataCleaner(MetadataCleanerType.SPARK_CONTEXT, this.cleanup, _conf)
  // 监控job和Stage的执行
  _statusTracker = new SparkStatusTracker(this)
  // 显示Stage的执行进度，从statusTracker定期拉取活动状态的Stages的进度，将在Stage至少运行500ms后显示，如果多个Stage在同一时间执行，则它们的状态将会合并到一起，在一行中显示，每200ms更新一次
  _progressBar =
    if (_conf.getBoolean("spark.ui.showConsoleProgress", true) && !log.isInfoEnabled) {
      Some(new ConsoleProgressBar(this))
    } else {
      None
    }
  // Spark UI部分
  _ui =
    if (conf.getBoolean("spark.ui.enabled", true)) {
      Some(SparkUI.createLiveUI(this, _conf, listenerBus, _jobProgressListener,
        _env.securityManager, appName, startTime = startTime))
    } else {
      // For tests, do not enable the UI
      None
    }
  // Bind the UI before starting the task scheduler to communicate
  // the bound port to the cluster manager properly
  _ui.foreach(_.bind())
  _hadoopConfiguration = SparkHadoopUtil.get.newConfiguration(_conf)
  // 添加Jar包的依赖
  // Add each JAR given through the constructor
  if (jars != null) {
    jars.foreach(addJar)
  }
  if (files != null) {
    files.foreach(addFile)
  }
  
  // 获取executor的内存配置信息，如果没有设置，默认就是1G
  _executorMemory = _conf.getOption("spark.executor.memory")
    .orElse(Option(System.getenv("SPARK_EXECUTOR_MEMORY")))
    .orElse(Option(System.getenv("SPARK_MEM"))
    .map(warnSparkMem))
    .map(Utils.memoryStringToMb)
    .getOrElse(1024)
  // Convert java options to env vars as a work around
  // since we can't set env vars directly in sbt.
  for { (envKey, propKey) <- Seq(("SPARK_TESTING", "spark.testing"))
    value <- Option(System.getenv(envKey)).orElse(Option(System.getProperty(propKey)))} {
    executorEnvs(envKey) = value
  }
  Option(System.getenv("SPARK_PREPEND_CLASSES")).foreach { v =>
    executorEnvs("SPARK_PREPEND_CLASSES") = v
  }
  // Mesos相关的配置
  // The Mesos scheduler backend relies on this environment variable to set executor memory.
  // TODO: Set this only in the Mesos scheduler.
  executorEnvs("SPARK_EXECUTOR_MEMORY") = executorMemory + "m"
  executorEnvs ++= _conf.getExecutorEnv
  executorEnvs("SPARK_USER") = sparkUser
  // We need to register "HeartbeatReceiver" before "createTaskScheduler" because Executor will
  // retrieve "HeartbeatReceiver" in the constructor. (SPARK-6640)
  _heartbeatReceiver = env.rpcEnv.setupEndpoint(
    HeartbeatReceiver.ENDPOINT_NAME, new HeartbeatReceiver(this))
  // 下面就是SparkContext中最重要的部分，即创建一系列调度器
  // Create and start the scheduler
  val (sched, ts) = SparkContext.createTaskScheduler(this, master)
  _schedulerBackend = sched
  _taskScheduler = ts
  _dagScheduler = new DAGScheduler(this)
  _heartbeatReceiver.ask[Boolean](TaskSchedulerIsSet)
  // start TaskScheduler after taskScheduler sets DAGScheduler reference in DAGScheduler's
  // constructor
  _taskScheduler.start()
  // 初始化应用程序Id信息
  _applicationId = _taskScheduler.applicationId()
  _applicationAttemptId = taskScheduler.applicationAttemptId()
  _conf.set("spark.app.id", _applicationId)
  _ui.foreach(_.setAppId(_applicationId))
  _env.blockManager.initialize(_applicationId)
  // 统计系统相关内容
  // The metrics system for Driver need to be set spark.app.id to app ID.
  // So it should start after we get app ID from the task scheduler and set spark.app.id.
  metricsSystem.start()
  // Attach the driver metrics servlet handler to the web ui after the metrics system is started.
  metricsSystem.getServletHandlers.foreach(handler => ui.foreach(_.attachHandler(handler)))
  _eventLogger =
    if (isEventLogEnabled) {
      val logger =
        new EventLoggingListener(_applicationId, _applicationAttemptId, _eventLogDir.get,
          _conf, _hadoopConfiguration)
      logger.start()
      listenerBus.addListener(logger)
      Some(logger)
    } else {
      None
    }
  // Optionally scale number of executors dynamically based on workload. Exposed for testing.
  val dynamicAllocationEnabled = Utils.isDynamicAllocationEnabled(_conf)
  if (!dynamicAllocationEnabled && _conf.getBoolean("spark.dynamicAllocation.enabled", false)) {
    logWarning("Dynamic Allocation and num executors both set, thus dynamic allocation disabled.")
  }
  _executorAllocationManager =
    if (dynamicAllocationEnabled) {
      Some(new ExecutorAllocationManager(this, listenerBus, _conf))
    } else {
      None
    }
  _executorAllocationManager.foreach(_.start())
  _cleaner =
    if (_conf.getBoolean("spark.cleaner.referenceTracking", true)) {
      Some(new ContextCleaner(this))
    } else {
      None
    }
  _cleaner.foreach(_.start())
  setupAndStartListenerBus()
  postEnvironmentUpdate()
  postApplicationStart()
  // Post init
  _taskScheduler.postStartHook()
  _env.metricsSystem.registerSource(_dagScheduler.metricsSource)
  _env.metricsSystem.registerSource(new BlockManagerSource(_env.blockManager))
  _executorAllocationManager.foreach { e =>
    _env.metricsSystem.registerSource(e.executorAllocationManagerSource)
  }
  // Make sure the context is stopped if the user forgets about it. This avoids leaving
  // unfinished event logs around after the JVM exits cleanly. It doesn't help if the JVM
  // is killed, though.
  _shutdownHookRef = ShutdownHookManager.addShutdownHook(
    ShutdownHookManager.SPARK_CONTEXT_SHUTDOWN_PRIORITY) { () =>
    logInfo("Invoking stop() from shutdown hook")
    stop()
  }
} catch {
  case NonFatal(e) =>
    logError("Error initializing SparkContext.", e)
    try {
      stop()
    } catch {
      case NonFatal(inner) =>
        logError("Error stopping SparkContext after init error.", inner)
    } finally {
      throw e
    }
}
```
其实SparkContext中最主要的三大核心对象就是DAGScheduler、TaskScheduler、SchedulerBackend，我们会在接下来的文章中详细进行分析。

**SparkEnv**

下面我们再来分析一下SparkEnv，SparkEnv保存了一个正在运行的Spark实例(Master或者Worker)的运行时环境信息，包括序列化(serializer)、Akka的actor system(虽然1.6.x默认使用的是Netty但是有一些历史遗留代码，spark2.x开始已经不在依赖Akka)、BlockManager、MapOutPutTracker(Shuffle过程中非常重要)等，跟SparkContext一样，这些具体的功能点我们会用单独的文章分别进行说明，现在我们简单过滤一下SparkEnv的源码：

先来看createDriverEnv和createExecutorEnv，顾名思义，这两个方法就是分别创建Driver和Executor的运行时环境。

```scala
/**
 * Create a SparkEnv for the driver.
 */
private[spark] def createDriverEnv(
    conf: SparkConf,
    isLocal: Boolean,
    listenerBus: LiveListenerBus,
    numCores: Int,
    mockOutputCommitCoordinator: Option[OutputCommitCoordinator] = None): SparkEnv = {
  assert(conf.contains("spark.driver.host"), "spark.driver.host is not set on the driver!")
  assert(conf.contains("spark.driver.port"), "spark.driver.port is not set on the driver!")
  val hostname = conf.get("spark.driver.host")
  val port = conf.get("spark.driver.port").toInt
  create(
    conf,
    SparkContext.DRIVER_IDENTIFIER,
    hostname,
    port,
    isDriver = true,
    isLocal = isLocal,
    numUsableCores = numCores,
    listenerBus = listenerBus,
    mockOutputCommitCoordinator = mockOutputCommitCoordinator
  )
}
/**
 * Create a SparkEnv for an executor.
 * In coarse-grained mode, the executor provides an actor system that is already instantiated.
 */
private[spark] def createExecutorEnv(
    conf: SparkConf,
    executorId: String,
    hostname: String,
    port: Int,
    numCores: Int,
    isLocal: Boolean): SparkEnv = {
  val env = create(
    conf,
    executorId,
    hostname,
    port,
    isDriver = false,
    isLocal = isLocal,
    numUsableCores = numCores
  )
  SparkEnv.set(env)
  env
}
可以看到这两个方法最终都调用了create()方法：

private def create(
    conf: SparkConf,
    executorId: String,
    hostname: String,
    port: Int,
    isDriver: Boolean,
    isLocal: Boolean,
    numUsableCores: Int,
    listenerBus: LiveListenerBus = null,
    mockOutputCommitCoordinator: Option[OutputCommitCoordinator] = None): SparkEnv = {
  // Listener bus仅使用在Driver上，所以要进行判断
  // Listener bus is only used on the driver
  if (isDriver) {
    assert(listenerBus != null, "Attempted to create driver SparkEnv with null listener bus!")
  }
  // 实例化SecurityManager
  val securityManager = new SecurityManager(conf)
  // 这里是Spark1.6.x中遗留的一部分关于Akka的代码，spark2.x中已经完全移除
  // Create the ActorSystem for Akka and get the port it binds to.
  val actorSystemName = if (isDriver) driverActorSystemName else executorActorSystemName
  // 创建rpcEnv
  val rpcEnv = RpcEnv.create(actorSystemName, hostname, port, conf, securityManager,
    clientMode = !isDriver)
  val actorSystem: ActorSystem =
    if (rpcEnv.isInstanceOf[AkkaRpcEnv]) {
      rpcEnv.y[AkkaRpcEnv].actorSystem
    } else {
      val actorSystemPort =
        if (port == 0 || rpcEnv.address == null) {
          port
        } else {
          rpcEnv.address.port + 1
        }
      // Create a ActorSystem for legacy codes
      AkkaUtils.createActorSystem(
        actorSystemName + "ActorSystem",
        hostname,
        actorSystemPort,
        conf,
        securityManager
      )._1
    }
  // Figure out which port Akka actually bound to in case the original port is 0 or occupied.
  // In the non-driver case, the RPC env's address may be null since it may not be listening
  // for incoming connections.
  // 设置Driver或者Executor的端口号
  if (isDriver) {
    conf.set("spark.driver.port", rpcEnv.address.port.toString)
  } else if (rpcEnv.address != null) {
    conf.set("spark.executor.port", rpcEnv.address.port.toString)
  }
  
  // 根据给定的类的名字进行实例化操作
  // Create an instance of the class with the given name, possibly initializing it with our conf
  def instantiateClass[T](className: String): T = {
    val cls = Utils.classForName(className)
    // Look for a constructor taking a SparkConf and a boolean isDriver, then one taking just
    // SparkConf, then one taking no arguments
    try {
      cls.getConstructor(classOf[SparkConf], java.lang.Boolean.TYPE)
        .newInstance(conf, new java.lang.Boolean(isDriver))
        .asInstanceOf[T]
    } catch {
      case _: NoSuchMethodException =>
        try {
          cls.getConstructor(classOf[SparkConf]).newInstance(conf).asInstanceOf[T]
        } catch {
          case _: NoSuchMethodException =>
            cls.getConstructor().newInstance().asInstanceOf[T]
        }
    }
  }
  // Create an instance of the class named by the given SparkConf property, or defaultClassName
  // if the property is not set, possibly initializing it with our conf
  def instantiateClassFromConf[T](propertyName: String, defaultClassName: String): T = {
    instantiateClass[T](conf.get(propertyName, defaultClassName))
  }
  
  // 设置序列化器，可以看到默认使用的是Java的序列化器
  val serializer = instantiateClassFromConf[Serializer](
    "spark.serializer", "org.apache.spark.serializer.JavaSerializer")
  logDebug(s"Using serializer: ${serializer.getClass}")
  val closureSerializer = instantiateClassFromConf[Serializer](
    "spark.closure.serializer", "org.apache.spark.serializer.JavaSerializer")
  def registerOrLookupEndpoint(
      name: String, endpointCreator: => RpcEndpoint):
    RpcEndpointRef = {
    if (isDriver) {
      logInfo("Registering " + name)
      rpcEnv.setupEndpoint(name, endpointCreator)
    } else {
      RpcUtils.makeDriverRef(name, conf, rpcEnv)
    }
  }
  
  // 如果是Driver实例化MapOutputTrackerMaster，如果是Executor实例化MapOutputTrackerWorker，在Shuffle时会详细说明
  // 说明MapOutputTracker也是Master／Slaves的结构
  val mapOutputTracker = if (isDriver) {
    new MapOutputTrackerMaster(conf)
  } else {
    new MapOutputTrackerWorker(conf)
  }
  
  // 实例化向MapOutputTrackerMasterEndpoint并向MapOutputTracker注册
  // Have to assign trackerActor after initialization as MapOutputTrackerActor
  // requires the MapOutputTracker itself
  mapOutputTracker.trackerEndpoint = registerOrLookupEndpoint(MapOutputTracker.ENDPOINT_NAME,
    new MapOutputTrackerMasterEndpoint(
      rpcEnv, mapOutputTracker.asInstanceOf[MapOutputTrackerMaster], conf))
  // Shuffle的配置信息，默认使用的是SortShuffleManager
  // Let the user specify short names for shuffle managers
  val shortShuffleMgrNames = Map(
    "hash" -> "org.apache.spark.shuffle.hash.HashShuffleManager",
    "sort" -> "org.apache.spark.shuffle.sort.SortShuffleManager",
    "tungsten-sort" -> "org.apache.spark.shuffle.sort.SortShuffleManager")
  val shuffleMgrName = conf.get("spark.shuffle.manager", "sort")
  val shuffleMgrClass = shortShuffleMgrNames.getOrElse(shuffleMgrName.toLowerCase, shuffleMgrName)
  // 实例化ShuffleManager，默认是SortShuffleManager
  val shuffleManager = instantiateClass[ShuffleManager](shuffleMgrClass)
  // 是否使用原始的MemoryManager，即StaticMemoryManager
  val useLegacyMemoryManager = conf.getBoolean("spark.memory.useLegacyMode", false)
  // 默认使用的是UnifiedMemoryManager内存管理器
  val memoryManager: MemoryManager =
    if (useLegacyMemoryManager) {
      new StaticMemoryManager(conf, numUsableCores)
    } else {
      UnifiedMemoryManager(conf, numUsableCores)
    }
  // 实例化BlockTransferService，默认的实现方式是Netty，即NettyBlockTransferService
  val blockTransferService = new NettyBlockTransferService(conf, securityManager, numUsableCores)
  // 下面是BlockManager相关的初始化过程，我们会用单独的文章说明
  val blockManagerMaster = new BlockManagerMaster(registerOrLookupEndpoint(
    BlockManagerMaster.DRIVER_ENDPOINT_NAME,
    new BlockManagerMasterEndpoint(rpcEnv, isLocal, conf, listenerBus)),
    conf, isDriver)
  // NB: blockManager is not valid until initialize() is called later.
  val blockManager = new BlockManager(executorId, rpcEnv, blockManagerMaster,
    serializer, conf, memoryManager, mapOutputTracker, shuffleManager,
    blockTransferService, securityManager, numUsableCores)
  // 实例化BroadcastManager
  val broadcastManager = new BroadcastManager(isDriver, conf, securityManager)
  // 实例化cacheManager
  val cacheManager = new CacheManager(blockManager)
  // 统计系统
  val metricsSystem = if (isDriver) {
    // Don't start metrics system right now for Driver.
    // We need to wait for the task scheduler to give us an app ID.
    // Then we can start the metrics system.
    MetricsSystem.createMetricsSystem("driver", conf, securityManager)
  } else {
    // We need to set the executor ID before the MetricsSystem is created because sources and
    // sinks specified in the metrics configuration file will want to incorporate this executor's
    // ID into the metrics they report.
    conf.set("spark.executor.id", executorId)
    val ms = MetricsSystem.createMetricsSystem("executor", conf, securityManager)
    ms.start()
    ms
  }
  
  // 设置sparkFiles的存储目录，用来下载依赖。local模式下是一个临时的目录，分布式模式下是Executor的工作目录
  // Set the sparkFiles directory, used when downloading dependencies.  In local mode,
  // this is a temporary directory; in distributed mode, this is the executor's current working
  // directory.
  val sparkFilesDir: String = if (isDriver) {
    Utils.createTempDir(Utils.getLocalDir(conf), "userFiles").getAbsolutePath
  } else {
    "."
  }
  
  // outputCommitCoordinator相关的初始化及注册部分
  val outputCommitCoordinator = mockOutputCommitCoordinator.getOrElse {
    new OutputCommitCoordinator(conf, isDriver)
  }
  val outputCommitCoordinatorRef = registerOrLookupEndpoint("OutputCommitCoordinator",
    new OutputCommitCoordinatorEndpoint(rpcEnv, outputCommitCoordinator))
  outputCommitCoordinator.coordinatorRef = Some(outputCommitCoordinatorRef)
  val envInstance = new SparkEnv(
    executorId,
    rpcEnv,
    actorSystem,
    serializer,
    closureSerializer,
    cacheManager,
    mapOutputTracker,
    shuffleManager,
    broadcastManager,
    blockTransferService,
    blockManager,
    securityManager,
    sparkFilesDir,
    metricsSystem,
    memoryManager,
    outputCommitCoordinator,
    conf)
  // Add a reference to tmp dir created by driver, we will delete this tmp dir when stop() is
  // called, and we only need to do it for driver. Because driver may run as a service, and if we
  // don't delete this tmp dir when sc is stopped, then will create too many tmp dirs.
  if (isDriver) {
    envInstance.driverTmpDirToDelete = Some(sparkFilesDir)
  }
  envInstance
}
```

**SparkSession**

SparkSession是Spark2.x后引入的概念。在2.x之前，对于不同的功能，需要使用不同的Context——

+ 创建和操作RDD时，使用SparkContext
+ 使用Streaming时，使用StreamingContext
+ 使用SQL时，使用sqlContext
+ 使用Hive时，使用HiveContext

在2.x中，为了统一上述的Context，引入SparkSession，实质上是SQLContext、HiveContext、SparkContext的组合。


SparkSession： SparkSession实质上是SQLContext和HiveContext的组合（未来可能还会加上StreamingContext），所以在SQLContext和HiveContext上可用的API在SparkSession上同样是可以使用的。

2.1 SparkSession的简单示例

SparkSession内部封装了SparkContext，所以计算实际上是由SparkContext完成的。

```scala
val sparkSession = SparkSession.builder
                    .master("master")
                    .appName("appName")
                    .getOrCreate()
//或者SparkSession.builder.config(conf=SparkConf())
```
上面代码类似于创建一个SparkContext，master设置为”master”，然后创建了一个SQLContext封装它。

2.2 创建支持Hive的SparkSession

如果你想创建hiveContext，可以使用下面的方法来创建SparkSession，以使得它支持Hive(HiveContext)：
```scala
val sparkSession = SparkSession.builder
                    .master("master")
                    .appName("appName")
                    .enableHiveSupport()
                    .getOrCreate()
//sparkSession 从csv读取数据：
val dq = sparkSession.read.option("header", "true").csv("src/main/resources/scala.csv")
```

