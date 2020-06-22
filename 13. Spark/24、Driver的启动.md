##“Driver“服务启动流程解析


首先Driver服务的开启是在创建Driver的运行时环境的时候完成的，如下所示：

SparkContext中：

```scala
// Create the Spark execution environment (cache, map output tracker, etc)
_env = createSparkEnv(_conf, isLocal, listenerBus)
SparkEnv.set(_env)
```

可以看到执行的是SparkEnv的createDriverEnv：

```scala
private[spark] def createSparkEnv(
    conf: SparkConf,
    isLocal: Boolean,
    listenerBus: LiveListenerBus): SparkEnv = {
  // 创建Driver的运行时环境，注意这里的numDriverCores是local模式下用来执行计算的cores的个数，如果不是本地模式的话就是0
  SparkEnv.createDriverEnv(conf, isLocal, listenerBus, SparkContext.numDriverCores(master))
}
```

numDriverCores的计算：

```scala
/**
 * The number of driver cores to use for execution in local mode, 0 otherwise.
 */
private[spark] def numDriverCores(master: String): Int = {
  def convertToInt(threads: String): Int = {
    if (threads == "*") Runtime.getRuntime.availableProcessors() else threads.toInt
  }
  master match {
    case "local" => 1
    case SparkMasterRegex.LOCAL_N_REGEX(threads) => convertToInt(threads)
    case SparkMasterRegex.LOCAL_N_FAILURES_REGEX(threads, _) => convertToInt(threads)
    case _ => 0 // driver is not used for execution
  }
}
```

在SparkEnv中创建Driver运行时环境的代码：

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
    SparkContext.DRIVER_IDENTIFIER,  // "driver"
    hostname,
    port,
    isDriver = true,
    isLocal = isLocal,
    numUsableCores = numCores,
    listenerBus = listenerBus,
    mockOutputCommitCoordinator = mockOutputCommitCoordinator
  )
}
```
我们在前面的文章中大致的浏览过，现在聚焦Driver服务启动相关的部分：

```scala
// 这里我们是Driver，所以actorSystemName是"sparkDriver"
// 注意Spark2.x中已经移除了对Akka的依赖，所以在Spark2.x中这里是driverSystemName和executorSystemName
// Create the ActorSystem for Akka and get the port it binds to.
val actorSystemName = if (isDriver) driverActorSystemName else executorActorSystemName
// 创建Driver的运行时环境，注意这里的clientMode等于false
val rpcEnv = RpcEnv.create(actorSystemName, hostname, port, conf, securityManager,
  clientMode = !isDriver)
```

接下来是RpcEnv的create方法：

```scala
def create(
    name: String,
    host: String,
    port: Int,
    conf: SparkConf,
    securityManager: SecurityManager,
    clientMode: Boolean = false): RpcEnv = {
  // Using Reflection to create the RpcEnv to avoid to depend on Akka directly
  // 封装成RpcEnvConfig，这里的name是"sparkDriver"，host是"driver"，clientMode是"false"
  val config = RpcEnvConfig(conf, name, host, port, securityManager, clientMode)
  // 这里实际上是通过反射的到的是NettyRpcEnvFactory，所以调用的是NettyRpcEnvFactory的create()方法
  getRpcEnvFactory(conf).create(config)
}
```
底层实现是NettyRpcEnvFactory的create方法：

```scala
def create(config: RpcEnvConfig): RpcEnv = {
  val sparkConf = config.conf
  // Use JavaSerializerInstance in multiple threads is safe. However, if we plan to support
  // KryoSerializer in future, we have to use ThreadLocal to store SerializerInstance
  val javaSerializerInstance =
    new JavaSerializer(sparkConf).newInstance().asInstanceOf[JavaSerializerInstance]
  // 实例化了NettyRpcEnv，名字为config.host，即driver
  val nettyEnv =
    new NettyRpcEnv(sparkConf, javaSerializerInstance, config.host, config.securityManager)
  // 传进来的clientMode为false，所以这里的判断为true
  if (!config.clientMode) {
    // 定义了一个函数startNettyRpcEnv
    val startNettyRpcEnv: Int => (NettyRpcEnv, Int) = { actualPort =>
      nettyEnv.startServer(actualPort)
      // 返回NettyRpcEnv及其端口号
      (nettyEnv, nettyEnv.address.port)
    }
    try {
      // 开启“sparkDriver”服务，注意此处传进了上面定义的函数，这里的config.name是"sparkDriver"，最后返回了NettyRpcEnv
      Utils.startServiceOnPort(config.port, startNettyRpcEnv, sparkConf, config.name)._1
    } catch {
      case NonFatal(e) =>
        nettyEnv.shutdown()
        throw e
    }
  }
  // 返回NettyRpcEnv
  nettyEnv
}
```
Utils中的startServiceOnPort方法：

```scala
def startServiceOnPort[T](
    startPort: Int,
    startService: Int => (T, Int),
    conf: SparkConf,
    serviceName: String = ""): (T, Int) = {
  // 我们传进来的startPort为0，所以会生成一个随机的端口号
  require(startPort == 0 || (1024 <= startPort && startPort < 65536),
    "startPort should be between 1024 and 65535 (inclusive), or 0 for a random free port.")
  // " 'sparkDriver'"
  val serviceString = if (serviceName.isEmpty) "" else s" '$serviceName'"
  // 通过"spark.port.maxRetries"设置，如果没有设置，而设置中包括"spark.testing"，
  // 最大重试次数就是100次，否则最大重试次数就是10次
  val maxRetries = portMaxRetries(conf)
  for (offset <- 0 to maxRetries) {
    // 设置端口号
    // Do not increment port if startPort is 0, which is treated as a special port
    val tryPort = if (startPort == 0) {
      startPort
    } else {
      // If the new port wraps around, do not try a privilege port
      ((startPort + offset - 1024) % (65536 - 1024)) + 1024
    }
    try {
      // 开启服务，并返回服务和端口号，注意这里的startService是上面传进来的那个函数startNettyRpcEnv
      // 所以我们实际上执行的是startNettyRpcEnv(tryPort)，而根据startNettyRpcEnv函数的定义，实际
      // 上是调用了nettyEnv.startServer(tryPort)方法
      val (service, port) = startService(tryPort)
      // 创建成功后打印日志，serviceString就是"sparkDriver"
      logInfo(s"Successfully started service$serviceString on port $port.")
      // 返回服务和端口号
      return (service, port)
    } catch {
      case e: Exception if isBindCollision(e) =>
        if (offset >= maxRetries) {
          val exceptionMessage = s"${e.getMessage}: Service$serviceString failed after " +
            s"$maxRetries retries! Consider explicitly setting the appropriate port for the " +
            s"service$serviceString (for example spark.ui.port for SparkUI) to an available " +
            "port or increasing spark.port.maxRetries."
          val exception = new BindException(exceptionMessage)
          // restore original stack trace
          exception.setStackTrace(e.getStackTrace)
          throw exception
        }
        logWarning(s"Service$serviceString could not bind on port $tryPort. " +
          s"Attempting port ${tryPort + 1}.")
    }
  }
  // Should never happen
  throw new SparkException(s"Failed to start service$serviceString on port $startPort")
}
```

下面我们就具体看一下NettyRpcEnv中的这个startServer方法，具体的启动方法我们不再追踪了，最后实际上创建了一个TransportServer。

```scala
def startServer(port: Int): Unit = {
  // 首先实例化bootstraps
  val bootstraps: java.util.List[TransportServerBootstrap] =
    if (securityManager.isAuthenticationEnabled()) {
      java.util.Arrays.asList(new SaslServerBootstrap(transportConf, securityManager))
    } else {
      java.util.Collections.emptyList()
    }
  // 实例化server
  server = transportContext.createServer(host, port, bootstraps)
  // 向dispatcher注册
  dispatcher.registerRpcEndpoint(
    RpcEndpointVerifier.NAME, new RpcEndpointVerifier(this, dispatcher))
}
```

再回到SparkEnv中，开启了"sparkDriver"服务后，又创建了Akka的ActorSystem，具体的创建过程就不分析了。

```scala
// 开启了sparkDriverActorSystem服务，spark2.x中已经移除了对Akka的依赖
val actorSystem: ActorSystem =
  if (rpcEnv.isInstanceOf[AkkaRpcEnv]) {
    rpcEnv.asInstanceOf[AkkaRpcEnv].actorSystem
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
  
// 最后使用开启的服务的端口替换掉原来的端口
if (isDriver) {
  conf.set("spark.driver.port", rpcEnv.address.port.toString)
} else if (rpcEnv.address != null) {
  conf.set("spark.executor.port", rpcEnv.address.port.toString)
}
```