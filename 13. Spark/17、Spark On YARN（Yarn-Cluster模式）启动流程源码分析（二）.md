###概览

本章将针对yarn-cluster（--master yarn –deploy-mode cluster）模式下全面进行代码补充解读：

+ 什么时候初始化SparkContext；

+ 如何实现ApplicationMaster如何启动executor；

+ 启动后如何通过rpc实现executor与driver端通信，并实现分配任务的功能。

###一、总体流程

在YARN-Cluster模式中，当用户向YARN中提交一个应用程序后，YARN将分两个阶段运行该应用程序：

1. 第一个阶段是把Spark的Driver作为一个ApplicationMaster在YARN集群中先启动；

2. 第二个阶段是由ApplicationMaster创建应用程序，然后为它向ResourceManager申请资源，并启动Executor来运行Task，同时监控它的整个运行过程，直到运行完成

![](16/1.png)

说明如下：

+ Spark Yarn Client向YARN中提交应用程序，包括ApplicationMaster程序、启动ApplicationMaster的命令、需要在Executor中运行的程序等；

+ ResourceManager收到请求后，在集群中选择一个NodeManager，为该应用程序分配第一个Container，要求它在这个Container中启动应用程序的ApplicationMaster，其中ApplicationMaster进行SparkContext等的初始化；

+ ApplicationMaster向ResourceManager注册，这样用户可以直接通过ResourceManage查看应用程序的运行状态，然后它将采用轮询的方式通过RPC协议为各个任务申请资源，并监控它们的运行状态直到运行结束；

+ 一旦ApplicationMaster申请到资源（也就是Container）后，便与对应的NodeManager通信，要求它在获得的Container中启动CoarseGrainedExecutorBackend，CoarseGrainedExecutorBackend启动后会向ApplicationMaster中的SparkContext注册并申请Task。这一点和Standalone模式一样，只不过SparkContext在Spark Application中初始化时，使用CoarseGrainedSchedulerBackend配合YarnClusterScheduler进行任务的调度，其中YarnClusterScheduler只是对TaskSchedulerImpl的一个简单包装，增加了对Executor的等待逻辑等；

+ ApplicationMaster中的SparkContext分配Task给CoarseGrainedExecutorBackend执行，CoarseGrainedExecutorBackend运行Task并向ApplicationMaster汇报运行的状态和进度，以让ApplicationMaster随时掌握各个任务的运行状态，从而可以在任务失败时重新启动任务；

+ 应用程序运行完成后，ApplicationMaster向ResourceManager申请注销并关闭自己；

###二、SparkSubmit类流程

使用spark-submit.sh提交任务：

```scala
#/bin/sh
#LANG=zh_CN.utf8
#export LANG
export SPARK_KAFKA_VERSION=0.10
export LANG=zh_CN.UTF-8
jarspath=''
for file in `ls /home/dx/works/myapp001/sparks/*.jar`
do
jarspath=${file},$jarspath
done
jarspath=${jarspath%?}
echo $jarspath

spark-submit \
--jars $jarspath \
--properties-file ./conf/spark-properties-myapp001.conf \
--verbose \
--master yarn \
--deploy-mode cluster \#或者client
--name Streaming-$1-$2-$3-$4-$5-Agg-Parser \
--num-executors 16 \
--executor-memory 6G \
--executor-cores 2 \
--driver-memory 2G \
--driver-java-options "-XX:+TraceClassPaths" \
--class com.dx.myapp001.Main \
/home/dx/works/myapp001/lib/application-jar.jar $1 $2 $3 $4 $5
```

运行spark-submit.sh，实际上执行的是org.apache.spark.deploy.SparkSubmit的main：


**1）--master yarn --deploy-mode:cluster**

调用YarnClusterApplication进行提交

YarnClusterApplication这是org.apache.spark.deploy.yarn.Client中的一个内部类，在YarnClusterApplication中new了一个Client对象，并调用了run方法

```scala
private[spark] class YarnClusterApplication extends SparkApplication {
  override def start(args: Array[String], conf: SparkConf): Unit = {
    // SparkSubmit would use yarn cache to distribute files & jars in yarn mode,
    // so remove them from sparkConf here for yarn mode.
    conf.remove("spark.jars")
    conf.remove("spark.files")

    new Client(new ClientArguments(args), conf).run()
  }
}
```

**2）--master yarn --deploy-mode:client[可忽略]**

调用application-jar.jar自身main函数，执行的是JavaMainApplication

```scala
/core/src/main/scala/org/apache/spark/deploy/SparkApplication.scala
/**
 * Implementation of SparkApplication that wraps a standard Java class with a "main" method.
 *
 * Configuration is propagated to the application via system properties, so running multiple
 * of these in the same JVM may lead to undefined behavior due to configuration leaks.
 */
private[deploy] class JavaMainApplication(klass: Class[_]) extends SparkApplication {

  override def start(args: Array[String], conf: SparkConf): Unit = {
    val mainMethod = klass.getMethod("main", new Array[String](0).getClass)
    if (!Modifier.isStatic(mainMethod.getModifiers)) {
      throw new IllegalStateException("The main method in the given main class must be static")
    }

    val sysProps = conf.getAll.toMap
    sysProps.foreach { case (k, v) =>
      sys.props(k) = v
    }

    mainMethod.invoke(null, args)
  }

}
/core/src/main/scala/org/apache/spark/deploy/SparkApplication.scala
```

从JavaMainApplication实现可以发现，JavaSparkApplication中调用start方法时，只是通过反射执行application-jar.jar的main函数。

###三、YarnClusterApplication运行流程

当yarn-custer模式中，YarnClusterApplication类中运行的是Client中run方法，Client#run()中实现了任务提交流程：

```scala
  yarn/src/main/scala/org/apache/spark/deploy/yarn/Client.scala
  /**
   * Submit an application to the ResourceManager.
   * If set spark.yarn.submit.waitAppCompletion to true, it will stay alive
   * reporting the application's status until the application has exited for any reason.
   * Otherwise, the client process will exit after submission.
   * If the application finishes with a failed, killed, or undefined status,
   * throw an appropriate SparkException.
   */
  def run(): Unit = {
    this.appId = submitApplication()
    if (!launcherBackend.isConnected() && fireAndForget) {
      val report = getApplicationReport(appId)
      val state = report.getYarnApplicationState
      logInfo(s"Application report for $appId (state: $state)")
      logInfo(formatReportDetails(report))
      if (state == YarnApplicationState.FAILED || state == YarnApplicationState.KILLED) {
        throw new SparkException(s"Application $appId finished with status: $state")
      }
    } else {
      val YarnAppReport(appState, finalState, diags) = monitorApplication(appId)
      if (appState == YarnApplicationState.FAILED || finalState == FinalApplicationStatus.FAILED) {
        diags.foreach { err =>
          logError(s"Application diagnostics message: $err")
        }
        throw new SparkException(s"Application $appId finished with failed status")
      }
      if (appState == YarnApplicationState.KILLED || finalState == FinalApplicationStatus.KILLED) {
        throw new SparkException(s"Application $appId is killed")
      }
      if (finalState == FinalApplicationStatus.UNDEFINED) {
        throw new SparkException(s"The final status of application $appId is undefined")
      }
    }
  }
```

其中run的方法流程：

1）  运行submitApplication()初始化yarn，使用yarn进行资源管理，并运行spark任务提交接下来的流程：分配driver container，然后在Driver Containe中启动ApplicaitonMaster，ApplicationMaster中初始化SparkContext。

2）  状态成功，上报执行进度等信息。

3）  状态失败，报告执行失败。

其中submitApplication（）的实现流程：

```scala
   src/main/scala/org/apache/spark/deploy/yarn/Client.scala
  /**
   * Submit an application running our ApplicationMaster to the ResourceManager.
   *
   * The stable Yarn API provides a convenience method (YarnClient#createApplication) for
   * creating applications and setting up the application submission context. This was not
   * available in the alpha API.
   */
  def submitApplication(): ApplicationId = {
    var appId: ApplicationId = null
    try {
      launcherBackend.connect()
      yarnClient.init(hadoopConf)
      yarnClient.start()

      logInfo("Requesting a new application from cluster with %d NodeManagers"
        .format(yarnClient.getYarnClusterMetrics.getNumNodeManagers))

      // Get a new application from our RM
      val newApp = yarnClient.createApplication()
      val newAppResponse = newApp.getNewApplicationResponse()
      appId = newAppResponse.getApplicationId()

      new CallerContext("CLIENT", sparkConf.get(APP_CALLER_CONTEXT),
        Option(appId.toString)).setCurrentContext()

      // Verify whether the cluster has enough resources for our AM
      verifyClusterResources(newAppResponse)

      // Set up the appropriate contexts to launch our AM
      val containerContext = createContainerLaunchContext(newAppResponse)
      val appContext = createApplicationSubmissionContext(newApp, containerContext)

      // Finally, submit and monitor the application
      logInfo(s"Submitting application $appId to ResourceManager")
      yarnClient.submitApplication(appContext)
      launcherBackend.setAppId(appId.toString)
      reportLauncherState(SparkAppHandle.State.SUBMITTED)
      appId
    } catch {
      case e: Throwable =>
        if (appId != null) {
          cleanupStagingDir(appId)
        }
        throw e
    }
  }
```
这段代码主要实现向ResourceManager申请资源，启动Container并运行ApplicationMaster。

其中createContainerLaunchContext(newAppResponse)中对应的启动主类amClass分支逻辑如下：

```scala
val amClass =
      if (isClusterMode) {
        Utils.classForName("org.apache.spark.deploy.yarn.ApplicationMaster").getName
      } else {
        Utils.classForName("org.apache.spark.deploy.yarn.ExecutorLauncher").getName
      }
```
当yarn-cluster模式下，会先通过Client#run()方法中调用Client#submitApplication()向Yarn的Resource Manager申请一个container，来启动ApplicationMaster。

启动ApplicationMaster的执行脚本示例：

```scala
[dx@hadoop143 bin]$ps -ef|grep ApplicationMaster
# yarn账户在执行
/bin/bash -c /usr/java/jdk1.8.0_171-amd64/bin/java \
-server \
-Xmx2048m \
-Djava.io.tmpdir=/mnt/data3/yarn/nm/usercache/dx/appcache/application_1554704591622_0340/container_1554704591622_0340_01_000001/tmp \
-Dspark.yarn.app.container.log.dir=/mnt/data4/yarn/container-logs/application_1554704591622_0340/container_1554704591622_0340_01_000001 \
org.apache.spark.deploy.yarn.ApplicationMaster \
--class 'com.dx.myapp001.Main' \
--jar file:/home/dx/works/myapp001/lib/application-jar.jar \
--arg '-type' \
--arg '0' \
--properties-file /mnt/data3/yarn/nm/usercache/dx/appcache/application_1554704591622_0340/container_1554704591622_0340_01_000001/__spark_conf__/__spark_conf__.properties \
1> /mnt/data4/yarn/container-logs/application_1554704591622_0340/container_1554704591622_0340_01_000001/stdout \
2> /mnt/data4/yarn/container-logs/application_1554704591622_0340/container_1554704591622_0340_01_000001/stderr
```

###四、ApplicationMaster运行流程

ApplicaitonMaster启动过程会通过半生类ApplicationMaster的main作为入口，执行：

```scala
  private var master: ApplicationMaster = _

  def main(args: Array[String]): Unit = {
    SignalUtils.registerLogger(log)
    val amArgs = new ApplicationMasterArguments(args)
    master = new ApplicationMaster(amArgs)
    System.exit(master.run())
  }
https://github.com/apache/spark/blob/branch-2.4/resource-managers/yarn/src/main/scala/org/apache/spark/deploy/yarn/ApplicationMaster.scala
```

通过ApplicationMasterArguments类对args进行解析，然后将解析后的amArgs作为master初始化的参数，并执行master#run()方法启动ApplicationMaster。

**ApplicationMaster实例化**

在ApplicationMaster类实例化中，ApplicationMaster的属性包含以下：

```scala
  private val isClusterMode = args.userClass != null

  private val sparkConf = new SparkConf()
  if (args.propertiesFile != null) {
    Utils.getPropertiesFromFile(args.propertiesFile).foreach { case (k, v) =>
      sparkConf.set(k, v)
    }
  }

  private val securityMgr = new SecurityManager(sparkConf)

  private var metricsSystem: Option[MetricsSystem] = None

  // Set system properties for each config entry. This covers two use cases:
  // - The default configuration stored by the SparkHadoopUtil class
  // - The user application creating a new SparkConf in cluster mode
  //
  // Both cases create a new SparkConf object which reads these configs from system properties.
  sparkConf.getAll.foreach { case (k, v) =>
    sys.props(k) = v
  }

  private val yarnConf = new YarnConfiguration(SparkHadoopUtil.newConfiguration(sparkConf))

  private val userClassLoader = {
    val classpath = Client.getUserClasspath(sparkConf)
    val urls = classpath.map { entry =>
      new URL("file:" + new File(entry.getPath()).getAbsolutePath())
    }

    if (isClusterMode) {
      if (Client.isUserClassPathFirst(sparkConf, isDriver = true)) {
        new ChildFirstURLClassLoader(urls, Utils.getContextOrSparkClassLoader)
      } else {
        new MutableURLClassLoader(urls, Utils.getContextOrSparkClassLoader)
      }
    } else {
      new MutableURLClassLoader(urls, Utils.getContextOrSparkClassLoader)
    }
  }

  private val client = doAsUser { new YarnRMClient() }
  private var rpcEnv: RpcEnv = null

  // In cluster mode, used to tell the AM when the user's SparkContext has been initialized.
  private val sparkContextPromise = Promise[SparkContext]()
```

ApplicationMaster属性解释：

+ --isClusterMode是否cluster模式，userClass有值则为true(来自ApplicationMaster参数--class 'com.dx.myapp001.Main' )

+ --sparkConf spark运行配置，配置信息来自args.propertiesFile（来自ApplicationMaster的参数--properties-file /mnt/data3/yarn/nm/usercache/dx/appcache/application_1554704591622_0340/container_1554704591622_0340_01_000001/__spark_conf__/__spark_conf__.properties）

+ --securityMgr安全管理类，初始化时需要传入sparkConf对象

+ --metricsSystem是测量系统初始化，用来记录执行进度，资源等信息。

+ --client是YarnRMClient的对象，它主要用来使用YARN ResourceManager处理注册和卸载application。https://github.com/apache/spark/blob/branch-2.4/resource-managers/yarn/src/main/scala/org/apache/spark/deploy/yarn/YarnRMClient.scala

+ --userClassLoader用户类加载器，从sparkConf中获取application的jar路径(实际来此ApplicationMaster初始化接收参数)，为后边通过它的main运行做辅助。

+ --userClassThread用来执行application main的线程

+ --rpcEnvrpc通信环境对象

+ --sparkContextPromise在cluster模式，当用户（application）的SparkContext对象已经被初始化用来通知ApplicationMaster

**ApplicationMaster执行Run方法**

ApplicationMaster#run()->ApplicationMaster#runImpl，在ApplicationMaster#runImpl方法中包含以下比较重要分支逻辑：
```scala
      if (isClusterMode) {
        runDriver()
      } else {
        runExecutorLauncher()
      }
```
因为args.userClass不为null，因此isCusterMode为true，则执行runDriver()方法。

ApplicationMaster#runDriver如下：

```scala
  private def runDriver(): Unit = {
    addAmIpFilter(None)
    userClassThread = startUserApplication()

    // This a bit hacky, but we need to wait until the spark.driver.port property has
    // been set by the Thread executing the user class.
    logInfo("Waiting for spark context initialization...")
    val totalWaitTime = sparkConf.get(AM_MAX_WAIT_TIME)
    try {
      val sc = ThreadUtils.awaitResult(sparkContextPromise.future,
        Duration(totalWaitTime, TimeUnit.MILLISECONDS))
      if (sc != null) {
        rpcEnv = sc.env.rpcEnv

        val userConf = sc.getConf
        val host = userConf.get("spark.driver.host")
        val port = userConf.get("spark.driver.port").toInt
        registerAM(host, port, userConf, sc.ui.map(_.webUrl))

        val driverRef = rpcEnv.setupEndpointRef(
          RpcAddress(host, port),
          YarnSchedulerBackend.ENDPOINT_NAME)
        createAllocator(driverRef, userConf)
      } else {
        // Sanity check; should never happen in normal operation, since sc should only be null
        // if the user app did not create a SparkContext.
        throw new IllegalStateException("User did not initialize spark context!")
      }
      resumeDriver()
      userClassThread.join()
    } catch {
      case e: SparkException if e.getCause().isInstanceOf[TimeoutException] =>
        logError(
          s"SparkContext did not initialize after waiting for $totalWaitTime ms. " +
           "Please check earlier log output for errors. Failing the application.")
        finish(FinalApplicationStatus.FAILED,
          ApplicationMaster.EXIT_SC_NOT_INITED,
          "Timed out waiting for SparkContext.")
    } finally {
      resumeDriver()
    }
  }
```
其执行流程如下：

1. 初始化userClassThread=startUserApplication()，运行用户定义的代码，通过反射运行application_jar.jar（sparksubmit命令中--class指定的类）的main函数；

2. 初始化SparkContext，通过sparkContextPromise来获取初始化SparkContext,并设定最大等待时间。

　　a)   这也充分证实了driver是运行在ApplicationMaster上(SparkContext相当于driver)；

　　b)   该SparkContext的真正初始化是在application_jar.jar的代码中执行，通过反射执行的。

3. resumeDriver()当初始化SparkContext完成后，恢复用户线程。

4. userClassThread.join()阻塞方式等待反射application_jar.jar的main执行完成。

**提问：SparkContext初始化后是如何被ApplicationMaster主线程获取到的？**

在spark-submit任务提交过程中，当采用spark-submit --master yarn --deploy-mode cluster时，SparkContext（driver）初始化是在ApplicationMaster中子线程中，SparkContext初始化是运行在该

```scala
@volatile private var userClassThread: Thread = _
// In cluster mode, used to tell the AM when the user's SparkContext has been initialized.
private val sparkContextPromise = Promise[SparkContext]()

线程下

val sc = ThreadUtils.awaitResult(sparkContextPromise.future,
       Duration(totalWaitTime, TimeUnit.MILLISECONDS))

```
sparkContextPromise是怎么拿到userClassThread（反射执行用户代码线程）中的SparkContext的实例呢？

回答：

这个是在SparkContext初始化TaskScheduler时，yarn-cluster模式对应的是YarnClusterScheduler，它里边有一个后启动钩子：

```scala
/**
 * This is a simple extension to ClusterScheduler - to ensure that appropriate initialization of
 * ApplicationMaster, etc is done
 */
private[spark] class YarnClusterScheduler(sc: SparkContext) extends YarnScheduler(sc) {

  logInfo("Created YarnClusterScheduler")

  override def postStartHook() {
    ApplicationMaster.sparkContextInitialized(sc)
    super.postStartHook()
    logInfo("YarnClusterScheduler.postStartHook done")
  }
}
https://github.com/apache/spark/blob/branch-2.4/resource-managers/yarn/src/main/scala/org/apache/spark/scheduler/cluster/YarnClusterScheduler.scala
```

调用的ApplicationMaster.sparkContextInitialized()方法把SparkContext实例赋给前面的Promise对象：

```scala
  private def sparkContextInitialized(sc: SparkContext) = {
　　 sparkContextPromise.synchronized {
       // Notify runDriver function that SparkContext is available
　　　　sparkContextPromise.success(sc)
       // Pause the user class thread in order to make proper initialization in runDriver function.
　　　　sparkContextPromise.wait()
    }
  }
```
然后userClassThread是调用startUserApplication()方法产生的，这之后就是列举的那一句：

>val sc = ThreadUtils.awaitResult(sparkContextPromise.future,Duration(totalWaitTime, TimeUnit.MILLISECONDS))

这句就是在超时时间内等待sparkContextPromise的Future对象返回SparkContext实例。

其实可以理解为下边这个模拟代码：

```scala
object App {
  def main(args: Array[String]): Unit = {
    val userClassThread = startUserApplication()
    val totalWaitTime = 15000
    try {
      val sc = ThreadUtils.awaitResult(sparkContextPromise.future,
        Duration(totalWaitTime, TimeUnit.MILLISECONDS))
      if (sc != null) {
        println("the sc has initialized")
        val rpcEnv = sc.env.rpcEnv

        val userConf = sc.getConf
        val host = userConf.get("spark.driver.host")
        val port = userConf.get("spark.driver.port").toInt
      } else {
        // Sanity check; should never happen in normal operation, since sc should only be null
        // if the user app did not create a SparkContext.
        throw new IllegalStateException("User did not initialize spark context!")
      }
      resumeDriver()
      userClassThread.join()
    } catch {
      case e: SparkException if e.getCause().isInstanceOf[TimeoutException] =>
        println(
          s"SparkContext did not initialize after waiting for $totalWaitTime ms. " +
            "Please check earlier log output for errors. Failing the application.")
    } finally {
      resumeDriver()
    }
  }

  /**
    * Start the user class, which contains the spark driver, in a separate Thread.
    * If the main routine exits cleanly or exits with System.exit(N) for any N
    * we assume it was successful, for all other cases we assume failure.
    *
    * Returns the user thread that was started.
    */
  private def startUserApplication(): Thread = {
    val userThread = new Thread {
      override def run() {
        try {
          val conf = new SparkConf().setMaster("local[*]").setAppName("appName")
          val sc = new SparkContext(conf)
          sparkContextInitialized(sc)
        } catch {
          case e: Exception =>
            sparkContextPromise.tryFailure(e.getCause())
        } finally {
          sparkContextPromise.trySuccess(null)
        }
      }
    }

    userThread.setName("Driver")
    userThread.start()
    userThread
  }

  // In cluster mode, used to tell the AM when the user's SparkContext has been initialized.
  private val sparkContextPromise = Promise[SparkContext]()

  private def resumeDriver(): Unit = {
    // When initialization in runDriver happened the user class thread has to be resumed.
    sparkContextPromise.synchronized {
      sparkContextPromise.notify()
    }
  }

  private def sparkContextInitialized(sc: SparkContext) = {
    sparkContextPromise.synchronized {
      // Notify runDriver function that SparkContext is available
      sparkContextPromise.success(sc)
      // Pause the user class thread in order to make proper initialization in runDriver function.
      sparkContextPromise.wait()
    }
  }
}
```

**SparkContext初始化后注册AM，申请Container启动Executor**

ApplicationMaster#runDriver中逻辑包含内容挺多，因此单独提到这个小节来讲解。

```scala
    val totalWaitTime = sparkConf.get(AM_MAX_WAIT_TIME)
    try {
      val sc = ThreadUtils.awaitResult(sparkContextPromise.future,
        Duration(totalWaitTime, TimeUnit.MILLISECONDS))
      if (sc != null) {
        rpcEnv = sc.env.rpcEnv

        val userConf = sc.getConf
        val host = userConf.get("spark.driver.host")
        val port = userConf.get("spark.driver.port").toInt
        registerAM(host, port, userConf, sc.ui.map(_.webUrl))

        val driverRef = rpcEnv.setupEndpointRef(
          RpcAddress(host, port),
          YarnSchedulerBackend.ENDPOINT_NAME)
        createAllocator(driverRef, userConf)
      } else {
        // Sanity check; should never happen in normal operation, since sc should only be null
        // if the user app did not create a SparkContext.
        throw new IllegalStateException("User did not initialize spark context!")
      }
      resumeDriver()
      userClassThread.join()
    } catch {
      case e: SparkException if e.getCause().isInstanceOf[TimeoutException] =>
        logError(
          s"SparkContext did not initialize after waiting for $totalWaitTime ms. " +
           "Please check earlier log output for errors. Failing the application.")
        finish(FinalApplicationStatus.FAILED,
          ApplicationMaster.EXIT_SC_NOT_INITED,
          "Timed out waiting for SparkContext.")
    } finally {
      resumeDriver()
    }
```
1. SparkContext初始化过程，通过startUserApplication()反射application_jar.jar（用来代码）中的main初始化SparkContext。

2. 如果SparkContext初始化成功，就进入：

          i. 给rpcEnv赋值为初始化的SparkContext对象sc的env对象的rpcEnv.

         ii. 从sc获取到userConf（SparkConf），driver host，driver port，sc.ui，并将他们作为registerAM（注册ApplicationMaster）的参数。

        iii. 根据driver host、driver port和driver rpc server名称YarnSchedulerBackend.ENDPOINT_NAME获取到driver的EndpointRef对象driverRef。

        iv. 调用createAllocator(driverRef, userConf)

        v. resumeDriver() ---SparkContext初始化线程释放信号量（或者归还主线程）

        vi. userClassThread.join()等待运行application_jar.jar的程序运行完成。

3. 如果SparkContext初始化失败，则抛出异常throw new IllegalStateException("User did not initialize spark context!")

**向RM(ResourceManager)注册AM**

初始化SparkContext成功后将返回sc（SparkContext实例对象），然后从sc中获取到userConf（SparkConf），driver host，driver port，sc.ui，并将它们作为registerAM()方法的参数。其中registerAM()方法就是注册AM（ApplicationMaster）。

```scala
  private val client = doAsUser { new YarnRMClient() }

  private def registerAM(
      host: String,
      port: Int,
      _sparkConf: SparkConf,
      uiAddress: Option[String]): Unit = {
    val appId = client.getAttemptId().getApplicationId().toString()
    val attemptId = client.getAttemptId().getAttemptId().toString()
    val historyAddress = ApplicationMaster
      .getHistoryServerAddress(_sparkConf, yarnConf, appId, attemptId)

    client.register(host, port, yarnConf, _sparkConf, uiAddress, historyAddress)
    registered = true
  }
```
1. client是private val client = doAsUser { new YarnRMClient() }

2. 注册ApplicationMaster需要调用client#register(..)方法，该方法需要传入driver host、driver port、historyAddress

3. 在client#register(…)内部是通过org.apache.hadoop.yarn.client.api.AMRMClient#registerApplicationMaster(driverHost, driverPort, trackingUrl)方法来实现向YARN ResourceManager注册ApplicationMaster的。

上述代码中client#register（…）client是YarmRMClient实例，register方式具体实现如下：

```scala
/**
 * Handles registering and unregistering the application with the YARN ResourceManager.
 */
private[spark] class YarnRMClient extends Logging {

  private var amClient: AMRMClient[ContainerRequest] = _
  private var uiHistoryAddress: String = _
  private var registered: Boolean = false

  /**
   * Registers the application master with the RM.
   *
   * @param driverHost Host name where driver is running.
   * @param driverPort Port where driver is listening.
   * @param conf The Yarn configuration.
   * @param sparkConf The Spark configuration.
   * @param uiAddress Address of the SparkUI.
   * @param uiHistoryAddress Address of the application on the History Server.
   */
  def register(
      driverHost: String,
      driverPort: Int,
      conf: YarnConfiguration,
      sparkConf: SparkConf,
      uiAddress: Option[String],
      uiHistoryAddress: String): Unit = {
    amClient = AMRMClient.createAMRMClient()
    amClient.init(conf)
    amClient.start()
    this.uiHistoryAddress = uiHistoryAddress

    val trackingUrl = uiAddress.getOrElse {
      if (sparkConf.get(ALLOW_HISTORY_SERVER_TRACKING_URL)) uiHistoryAddress else ""
    }

    logInfo("Registering the ApplicationMaster")
    synchronized {
      amClient.registerApplicationMaster(driverHost, driverPort, trackingUrl)
      registered = true
    }
  }
。。。
}
复制代码
https://github.com/apache/spark/blob/branch-2.4/resource-managers/yarn/src/main/scala/org/apache/spark/deploy/yarn/YarnRMClient.scala
```

运行过程：

1. 需要先初始化 AMRMClient[ContainerRequest] 对象amClient并调动amClient#start()启动；

2. 同步方式执行 amClient#registerApplicationMaster(driverHost, driverPort, trackingUrl) 方法，使用YarnRMClient对象向Yarn Resource Manager注册ApplicationMaster；

3. 注册时，会传递一个trackingUrl，记录的是通过UI方式查看应用程序的运行状态的地址。

备注：ApplicationMaster 向 ResourceManager 注册，这样用户可以直接通过ResourceManage查看应用程序的运行状态，然后它将采用轮询的方式通过RPC协议为各个任务申请资源，并监控它们的运行状态直到运行结束。

**向RM申请资源启动Container**

接着向下分析ApplicationMaster#runDriver中逻辑，上边我们看到SparkContext初始化成功后返回sc对象，并将AM注册到RM，接下来：

1）根据driver host、driver port和driver rpc server名称YarnSchedulerBackend.ENDPOINT_NAME获取到driver的EndpointRef对象driverRef，方便AM与Driver通信；同时Container中Executor启动时也传递了driverRef的host、port等信息，这样Executor就可以通过driver host,dirver port获取到driverEndpointRef实现：executor与driver之间的RPC通信。

2）调用createAllocator(driverRef, userConf)，使用YarnRMClient对象向RM申请Container资源，并启动Executor。

在ApplicationMaster中定义了createAllocator(driverRef, userConf)方法如下

```scala
  private val client = doAsUser { new YarnRMClient() }
  private def createAllocator(driverRef: RpcEndpointRef, _sparkConf: SparkConf): Unit = {
    val appId = client.getAttemptId().getApplicationId().toString()
    val driverUrl = RpcEndpointAddress(driverRef.address.host, driverRef.address.port,
      CoarseGrainedSchedulerBackend.ENDPOINT_NAME).toString

    // Before we initialize the allocator, let's log the information about how executors will
    // be run up front, to avoid printing this out for every single executor being launched.
    // Use placeholders for information that changes such as executor IDs.
    logInfo {
      val executorMemory = _sparkConf.get(EXECUTOR_MEMORY).toInt
      val executorCores = _sparkConf.get(EXECUTOR_CORES)
      val dummyRunner = new ExecutorRunnable(None, yarnConf, _sparkConf, driverUrl, "<executorId>",
        "<hostname>", executorMemory, executorCores, appId, securityMgr, localResources)
      dummyRunner.launchContextDebugInfo()
    }

    allocator = client.createAllocator(
      yarnConf,
      _sparkConf,
      driverUrl,
      driverRef,
      securityMgr,
      localResources)

    credentialRenewer.foreach(_.setDriverRef(driverRef))

    // Initialize the AM endpoint *after* the allocator has been initialized. This ensures
    // that when the driver sends an initial executor request (e.g. after an AM restart),
    // the allocator is ready to service requests.
    rpcEnv.setupEndpoint("YarnAM", new AMEndpoint(rpcEnv, driverRef))

    allocator.allocateResources()
    val ms = MetricsSystem.createMetricsSystem("applicationMaster", sparkConf, securityMgr)
    val prefix = _sparkConf.get(YARN_METRICS_NAMESPACE).getOrElse(appId)
    ms.registerSource(new ApplicationMasterSource(prefix, allocator))
    ms.start()
    metricsSystem = Some(ms)
    reporterThread = launchReporterThread()
  }
```
上边这段代码主要通过client（YarnRMClient对象）创建的YarnAllocator对象allocator来进行container申请，通过ExecutorRunnable来启动executor，下边我们看下具体执行步骤：

1）通过yarn#createAllocator(yarnConf,_sparkConf,driverUrl,driverRef,securityMgr,localResources)创建allocator，该allocator是YarnAllocator的对象

>https://github.com/apache/spark/blob/branch-2.4/resource-managers/yarn/src/main/scala/org/apache/spark/deploy/yarn/YarnAllocator.scala

2）  初始化AMEndpoint（它是ApplicationMaster下的一个内部类）对象，用来实现与driver之间rpc通信。

其中需要注意：AMEndpoint初始化时传入了dirverRef的，在AMEndpoint的onStart（）方法中调用driver.send(RegisterClusterManager(self))，这时driver端接收该信息类是：

>https://github.com/apache/spark/blob/branch-2.4/resource-managers/yarn/src/main/scala/org/apache/spark/scheduler/cluster/YarnSchedulerBackend.scala

3）调用allocator.allocateResources()其内部实现是循环申请container，并通过ExecutorRunnable启动executor。

>https://github.com/apache/spark/blob/branch-2.4/resource-managers/yarn/src/main/scala/org/apache/spark/deploy/yarn/ExecutorRunnable.scala

4）上报测量数据，allocationThreadImpl()收集错误信息并做出响应。

上边提到AMEndpoint类（它是ApplicationMaster的一个内部类），下边看下他的具体实现：

ApplicationMaster发送

```scala
  /**
   * An [[RpcEndpoint]] that communicates with the driver's scheduler backend.
   */
  private class AMEndpoint(override val rpcEnv: RpcEnv, driver: RpcEndpointRef)
    extends RpcEndpoint with Logging {

    override def onStart(): Unit = {
      driver.send(RegisterClusterManager(self))
    }

    override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
      case r: RequestExecutors =>
        Option(allocator) match {
          case Some(a) =>
            if (a.requestTotalExecutorsWithPreferredLocalities(r.requestedTotal,
              r.localityAwareTasks, r.hostToLocalTaskCount, r.nodeBlacklist)) {
              resetAllocatorInterval()
            }
            context.reply(true)

          case None =>
            logWarning("Container allocator is not ready to request executors yet.")
            context.reply(false)
        }

      case KillExecutors(executorIds) =>
        logInfo(s"Driver requested to kill executor(s) ${executorIds.mkString(", ")}.")
        Option(allocator) match {
          case Some(a) => executorIds.foreach(a.killExecutor)
          case None => logWarning("Container allocator is not ready to kill executors yet.")
        }
        context.reply(true)

      case GetExecutorLossReason(eid) =>
        Option(allocator) match {
          case Some(a) =>
            a.enqueueGetLossReasonRequest(eid, context)
            resetAllocatorInterval()
          case None =>
            logWarning("Container allocator is not ready to find executor loss reasons yet.")
        }
    }

    override def onDisconnected(remoteAddress: RpcAddress): Unit = {
      // In cluster mode, do not rely on the disassociated event to exit
      // This avoids potentially reporting incorrect exit codes if the driver fails
      if (!isClusterMode) {
        logInfo(s"Driver terminated or disconnected! Shutting down. $remoteAddress")
        finish(FinalApplicationStatus.SUCCEEDED, ApplicationMaster.EXIT_SUCCESS)
      }
    }
  }
```
需要来说说这类：

Application Master 向driver发送

1） 在onStart()时，会向driver（SparkContext实例）发送一个RegisterClusterManager(self)请求，该用意用来告知driver，ClusterManger权限交给我，其中driver接收该AM参数代码在YarnSchedulerBackend（该对象是SparkContext的schedulerBackend属性）

备注：
```scala
YarnClusterSchedulerBackend、YarnSchedulerBackend、CoarseGrainedSchedulerBackend三者之间关系：

YarnClusterSchedulerBackend
extends
YarnSchedulerBackend

YarnSchedulerBackend
extends
CoarseGrainedSchedulerBackend

https://github.com/apache/spark/blob/branch-2.4/resource-managers/yarn/src/main/scala/org/apache/spark/scheduler/cluster/YarnClusterSchedulerBackend.scala

https://github.com/apache/spark/blob/branch-2.4/resource-managers/yarn/src/main/scala/org/apache/spark/scheduler/cluster/YarnSchedulerBackend.scala

https://github.com/apache/spark/blob/branch-2.4/core/src/main/scala/org/apache/spark/scheduler/cluster/CoarseGrainedSchedulerBackend.scala
```
2） 在receiveAndReply()方法包含了三种处理：

　　2.1）RequestExecutors请求分配executor；

    2.2）KillExecutors杀掉所有executor；

　　 2.3）GetExecutorLossReason获取executor丢失原因。

**通过ExecutorRunable启动Executor**

上边讲到调用ApplicationMaster.allocator.allocateResources()其内部实现是循环申请container，并通过ExecutorRunnable启动executor。ExecutorRunnable是用来启动CoarseGrainedExecutorBackend，CoarseGrainedExecutorBackend其实是一个进程，ExecutorRunnable包装了CoarseGrainedExecutorBackend进程启动脚本，并提供了通过nmClient(NameNode Client)启动Conatiner，启动container时附带CoarseGrainedExecutorBackend进程启动脚本。

```scala
private[yarn] class ExecutorRunnable(
    container: Option[Container],
    conf: YarnConfiguration,
    sparkConf: SparkConf,
    masterAddress: String,
    executorId: String,
    hostname: String,
    executorMemory: Int,
    executorCores: Int,
    appId: String,
    securityMgr: SecurityManager,
    localResources: Map[String, LocalResource]) extends Logging {

  var rpc: YarnRPC = YarnRPC.create(conf)
  var nmClient: NMClient = _

  def run(): Unit = {
    logDebug("Starting Executor Container")
    nmClient = NMClient.createNMClient()
    nmClient.init(conf)
    nmClient.start()
    startContainer()
  }

  def launchContextDebugInfo(): String = {
    val commands = prepareCommand()
    val env = prepareEnvironment()

    s"""
    |===============================================================================
    |YARN executor launch context:
    |  env:
    |${Utils.redact(sparkConf, env.toSeq).map { case (k, v) => s"    $k -> $v\n" }.mkString}
    |  command:
    |    ${commands.mkString(" \\ \n      ")}
    |
    |  resources:
    |${localResources.map { case (k, v) => s"    $k -> $v\n" }.mkString}
    |===============================================================================""".stripMargin
  }

  def startContainer(): java.util.Map[String, ByteBuffer] = {
    val ctx = Records.newRecord(classOf[ContainerLaunchContext])
      .asInstanceOf[ContainerLaunchContext]
    val env = prepareEnvironment().asJava

    ctx.setLocalResources(localResources.asJava)
    ctx.setEnvironment(env)

    val credentials = UserGroupInformation.getCurrentUser().getCredentials()
    val dob = new DataOutputBuffer()
    credentials.writeTokenStorageToStream(dob)
    ctx.setTokens(ByteBuffer.wrap(dob.getData()))

    val commands = prepareCommand()

    ctx.setCommands(commands.asJava)
    ctx.setApplicationACLs(
      YarnSparkHadoopUtil.getApplicationAclsForYarn(securityMgr).asJava)

    // If external shuffle service is enabled, register with the Yarn shuffle service already
    // started on the NodeManager and, if authentication is enabled, provide it with our secret
    // key for fetching shuffle files later
    if (sparkConf.get(SHUFFLE_SERVICE_ENABLED)) {
      val secretString = securityMgr.getSecretKey()
      val secretBytes =
        if (secretString != null) {
          // This conversion must match how the YarnShuffleService decodes our secret
          JavaUtils.stringToBytes(secretString)
        } else {
          // Authentication is not enabled, so just provide dummy metadata
          ByteBuffer.allocate(0)
        }
      ctx.setServiceData(Collections.singletonMap("spark_shuffle", secretBytes))
    }

    // Send the start request to the ContainerManager
    try {
      nmClient.startContainer(container.get, ctx)
    } catch {
      case ex: Exception =>
        throw new SparkException(s"Exception while starting container ${container.get.getId}" +
          s" on host $hostname", ex)
    }
  }

  private def prepareCommand(): List[String] = {
    // Extra options for the JVM
    val javaOpts = ListBuffer[String]()

    // Set the JVM memory
    val executorMemoryString = executorMemory + "m"
    javaOpts += "-Xmx" + executorMemoryString

    // Set extra Java options for the executor, if defined
    sparkConf.get(EXECUTOR_JAVA_OPTIONS).foreach { opts =>
      val subsOpt = Utils.substituteAppNExecIds(opts, appId, executorId)
      javaOpts ++= Utils.splitCommandString(subsOpt).map(YarnSparkHadoopUtil.escapeForShell)
    }

    // Set the library path through a command prefix to append to the existing value of the
    // env variable.
    val prefixEnv = sparkConf.get(EXECUTOR_LIBRARY_PATH).map { libPath =>
      Client.createLibraryPathPrefix(libPath, sparkConf)
    }

    javaOpts += "-Djava.io.tmpdir=" +
      new Path(Environment.PWD.$$(), YarnConfiguration.DEFAULT_CONTAINER_TEMP_DIR)

    // Certain configs need to be passed here because they are needed before the Executor
    // registers with the Scheduler and transfers the spark configs. Since the Executor backend
    // uses RPC to connect to the scheduler, the RPC settings are needed as well as the
    // authentication settings.
    sparkConf.getAll
      .filter { case (k, v) => SparkConf.isExecutorStartupConf(k) }
      .foreach { case (k, v) => javaOpts += YarnSparkHadoopUtil.escapeForShell(s"-D$k=$v") }

    // Commenting it out for now - so that people can refer to the properties if required. Remove
    // it once cpuset version is pushed out.
    // The context is, default gc for server class machines end up using all cores to do gc - hence
    // if there are multiple containers in same node, spark gc effects all other containers
    // performance (which can also be other spark containers)
    // Instead of using this, rely on cpusets by YARN to enforce spark behaves 'properly' in
    // multi-tenant environments. Not sure how default java gc behaves if it is limited to subset
    // of cores on a node.
    /*
        else {
          // If no java_opts specified, default to using -XX:+CMSIncrementalMode
          // It might be possible that other modes/config is being done in
          // spark.executor.extraJavaOptions, so we don't want to mess with it.
          // In our expts, using (default) throughput collector has severe perf ramifications in
          // multi-tenant machines
          // The options are based on
          // http://www.oracle.com/technetwork/java/gc-tuning-5-138395.html#0.0.0.%20When%20to%20Use
          // %20the%20Concurrent%20Low%20Pause%20Collector|outline
          javaOpts += "-XX:+UseConcMarkSweepGC"
          javaOpts += "-XX:+CMSIncrementalMode"
          javaOpts += "-XX:+CMSIncrementalPacing"
          javaOpts += "-XX:CMSIncrementalDutyCycleMin=0"
          javaOpts += "-XX:CMSIncrementalDutyCycle=10"
        }
    */

    // For log4j configuration to reference
    javaOpts += ("-Dspark.yarn.app.container.log.dir=" + ApplicationConstants.LOG_DIR_EXPANSION_VAR)

    val userClassPath = Client.getUserClasspath(sparkConf).flatMap { uri =>
      val absPath =
        if (new File(uri.getPath()).isAbsolute()) {
          Client.getClusterPath(sparkConf, uri.getPath())
        } else {
          Client.buildPath(Environment.PWD.$(), uri.getPath())
        }
      Seq("--user-class-path", "file:" + absPath)
    }.toSeq

    YarnSparkHadoopUtil.addOutOfMemoryErrorArgument(javaOpts)
    val commands = prefixEnv ++
      Seq(Environment.JAVA_HOME.$$() + "/bin/java", "-server") ++
      javaOpts ++
      Seq("org.apache.spark.executor.CoarseGrainedExecutorBackend",
        "--driver-url", masterAddress,
        "--executor-id", executorId,
        "--hostname", hostname,
        "--cores", executorCores.toString,
        "--app-id", appId) ++
      userClassPath ++
      Seq(
        s"1>${ApplicationConstants.LOG_DIR_EXPANSION_VAR}/stdout",
        s"2>${ApplicationConstants.LOG_DIR_EXPANSION_VAR}/stderr")

    // TODO: it would be nicer to just make sure there are no null commands here
    commands.map(s => if (s == null) "null" else s).toList
  }

  private def prepareEnvironment(): HashMap[String, String] = {
    val env = new HashMap[String, String]()
    Client.populateClasspath(null, conf, sparkConf, env, sparkConf.get(EXECUTOR_CLASS_PATH))

    // lookup appropriate http scheme for container log urls
    val yarnHttpPolicy = conf.get(
      YarnConfiguration.YARN_HTTP_POLICY_KEY,
      YarnConfiguration.YARN_HTTP_POLICY_DEFAULT
    )
    val httpScheme = if (yarnHttpPolicy == "HTTPS_ONLY") "https://" else "http://"

    System.getenv().asScala.filterKeys(_.startsWith("SPARK"))
      .foreach { case (k, v) => env(k) = v }

    sparkConf.getExecutorEnv.foreach { case (key, value) =>
      if (key == Environment.CLASSPATH.name()) {
        // If the key of env variable is CLASSPATH, we assume it is a path and append it.
        // This is kept for backward compatibility and consistency with hadoop
        YarnSparkHadoopUtil.addPathToEnvironment(env, key, value)
      } else {
        // For other env variables, simply overwrite the value.
        env(key) = value
      }
    }

    // Add log urls
    container.foreach { c =>
      sys.env.get("SPARK_USER").foreach { user =>
        val containerId = ConverterUtils.toString(c.getId)
        val address = c.getNodeHttpAddress
        val baseUrl = s"$httpScheme$address/node/containerlogs/$containerId/$user"

        env("SPARK_LOG_URL_STDERR") = s"$baseUrl/stderr?start=-4096"
        env("SPARK_LOG_URL_STDOUT") = s"$baseUrl/stdout?start=-4096"
      }
    }

    env
  }
}
```
ExecutorRunable该类包含以下方法：

1）prepareEnvironment():准备executor运行环境

2）prepareCommand():生成启动CoarseGrainedExecutorBackend进程启动脚本

3）startContainer(): 

+ 初始化executor运行环境；

+ 生成启动CoarseGrainedExecutorBackend进程启动脚本

+ 将生成启动CoarseGrainedExecutorBackend进程启动脚本附加到container中，并调用nmClient.startContainer(container.get, ctx)实现container启动，在container中运行启动CoarseGrainedExecutorBackend进程启动脚本来启动executor。

4）launchContextDebugInfo():打印测试日志

5）run(): 通过NMClient.createNMClient()初始化nmClient ，并启动nmClient ，并调用startContainer()。

启动CoarseGrainedExecutorBackend进程脚本示例：

```scala
launch_container.sh内容

复制代码
#!/bin/bash
。。。
exec /bin/bash -c "$JAVA_HOME/bin/java 
-server -Xmx6144m 
-Djava.io.tmpdir=$PWD/tmp
'-Dspark.driver.port=50365' 
'-Dspark.network.timeout=10000000' 
'-Dspark.port.maxRetries=32' 
-Dspark.yarn.app.container.log.dir=/data4/yarn/container-logs/application_1559203334026_0010/container_1559203334026_0010_01_000003 
-XX:OnOutOfMemoryError='kill %p' 
org.apache.spark.executor.CoarseGrainedExecutorBackend
--driver-url spark://CoarseGrainedScheduler@CDH-143:50365 
--executor-id 2 
--hostname CDH-141 
--cores 2 
--app-id application_1559203334026_0010 
--user-class-path file:$PWD/__app__.jar 
--user-class-path file:$PWD/spark-sql-kafka-0-10_2.11-2.4.0.jar 
--user-class-path file:$PWD/spark-avro_2.11-3.2.0.jar  
--user-class-path file:$PWD/bijection-core_2.11-0.9.5.jar 
--user-class-path file:$PWD/bijection-avro_2.11-0.9.5.jar 
1>/data4/yarn/container-logs/application_1559203334026_0010/container_1559203334026_0010_01_000003/stdout 
2>/data4/yarn/container-logs/application_1559203334026_0010/container_1559203334026_0010_01_000003/stderr"
。。。。
```

**CoarseGrainedExecutorBackend启动**

**CoarseGrainedExecutorBackend入口函数**
和SparkSubmit半生对象、AppllicationMaster半生对象一样，CoarseGrainedExecutorBackend也包含一个半生对象，同样也包含了入口main函数。该main函数执行：

单例对象与类同名时，这个单例对象被称为这个类的伴生对象，而这个类被称为这个单例对象的伴生类。伴生类和伴生对象要在同一个源文件中定义，伴生对象和伴生类可以互相访问其私有成员。不与伴生类同名的单例对象称为孤立对象。类和它的伴生对象可以互相访问其私有成员

+ 单例对象不能new，所以也没有构造参数
+ 可以把单例对象当做java中可能会用到的静态方法工具类。
+ 作为程序入口的方法必须是静态的，所以main方法必须处在一个单例对象中，而不能写在一个类中。

```scala
  def main(args: Array[String]) {
    var driverUrl: String = null
    var executorId: String = null
    var hostname: String = null
    var cores: Int = 0
    var appId: String = null
    var workerUrl: Option[String] = None
    val userClassPath = new mutable.ListBuffer[URL]()

    var argv = args.toList
    while (!argv.isEmpty) {
      argv match {
        case ("--driver-url") :: value :: tail =>
          driverUrl = value
          argv = tail
        case ("--executor-id") :: value :: tail =>
          executorId = value
          argv = tail
        case ("--hostname") :: value :: tail =>
          hostname = value
          argv = tail
        case ("--cores") :: value :: tail =>
          cores = value.toInt
          argv = tail
        case ("--app-id") :: value :: tail =>
          appId = value
          argv = tail
        case ("--worker-url") :: value :: tail =>
          // Worker url is used in spark standalone mode to enforce fate-sharing with worker
          workerUrl = Some(value)
          argv = tail
        case ("--user-class-path") :: value :: tail =>
          userClassPath += new URL(value)
          argv = tail
        case Nil =>
        case tail =>
          // scalastyle:off println
          System.err.println(s"Unrecognized options: ${tail.mkString(" ")}")
          // scalastyle:on println
          printUsageAndExit()
      }
    }

    if (driverUrl == null || executorId == null || hostname == null || cores <= 0 ||
      appId == null) {
      printUsageAndExit()
    }

    run(driverUrl, executorId, hostname, cores, appId, workerUrl, userClassPath)
    System.exit(0)
  }
```

1）  解析ExecutorRunnable传入的参数；

　　a)  var driverUrl: String = null  ---driver的Rpc通信Url

　　b)  var executorId: String = null ---executor的编号id(一般driver所在executor编号为0，其他一次加1，连续的)

　　c)  var hostname: String = null --- executor运行的集群节点的hostname

　　d)  var cores: Int = 0         ---executor可使用vcore个数

　　e)  var appId: String = null    ---当前应用程序的id

　　f)   var workerUrl: Option[String] = None ---worker UI地址

　　g)  val userClassPath = new mutable.ListBuffer[URL]() ---用户代码（当前应用程序）main所在的包和依赖包的路径列表。

2）  将解析后的参数传入run方法，执行CoarseGrainedExecutorBackend半生对象的run方法。

```scala
  private def run(
      driverUrl: String,
      executorId: String,
      hostname: String,
      cores: Int,
      appId: String,
      workerUrl: Option[String],
      userClassPath: Seq[URL]) {

    Utils.initDaemon(log)

    SparkHadoopUtil.get.runAsSparkUser { () =>
      // Debug code
      Utils.checkHost(hostname)

      // Bootstrap to fetch the driver's Spark properties.
      val executorConf = new SparkConf
      val fetcher = RpcEnv.create(
        "driverPropsFetcher",
        hostname,
        -1,
        executorConf,
        new SecurityManager(executorConf),
        clientMode = true)
      val driver = fetcher.setupEndpointRefByURI(driverUrl)
      val cfg = driver.askSync[SparkAppConfig](RetrieveSparkAppConfig)
      val props = cfg.sparkProperties ++ Seq[(String, String)](("spark.app.id", appId))
      fetcher.shutdown()

      // Create SparkEnv using properties we fetched from the driver.
      val driverConf = new SparkConf()
      for ((key, value) <- props) {
        // this is required for SSL in standalone mode
        if (SparkConf.isExecutorStartupConf(key)) {
          driverConf.setIfMissing(key, value)
        } else {
          driverConf.set(key, value)
        }
      }

      cfg.hadoopDelegationCreds.foreach { tokens =>
        SparkHadoopUtil.get.addDelegationTokens(tokens, driverConf)
      }

      val env = SparkEnv.createExecutorEnv(
        driverConf, executorId, hostname, cores, cfg.ioEncryptionKey, isLocal = false)

      env.rpcEnv.setupEndpoint("Executor", new CoarseGrainedExecutorBackend(
        env.rpcEnv, driverUrl, executorId, hostname, cores, userClassPath, env))
      workerUrl.foreach { url =>
        env.rpcEnv.setupEndpoint("WorkerWatcher", new WorkerWatcher(env.rpcEnv, url))
      }
      env.rpcEnv.awaitTermination()
    }
  }
```
代码执行逻辑：

1）通过driverUrl与driver Endpoint建立通信，向driver需求Spark应用程序的配置信息，并来创建driverConf对象。RetrieveSparkAppConfig类型请求被driver的schedulerBackend属性接收，接收代码位置：https://github.com/apache/spark/blob/branch-2.4/core/src/main/scala/org/apache/spark/scheduler/cluster/CoarseGrainedSchedulerBackend.scala

2）通过SparkEnv.createExecutorEnv() 方法创建SparkEnv对象env ，SparkEnv#createExecutorEnv 内部会创建以下几类组件：RpcEnv，securityManager，broadcastManager，mapOutputTracker，shuffleManager，memoryManager，blockTransferService，blockManagerMaster，blockManager，metricsSystem，outputCommitCorrdinator，outputCommitCoordinatorRef等。

3）通过env.rpcEnv对象开放RPC通信接口“Executor”，对应RpcEndpoint类型是CoarseGrainedExecutorBackend类。

4）通过workerUrl开发RPC通信接口“WorkerWatcher”，用来监控worker运行。WorkerWatcher的功能：连接到工作进程并在连接断开时终止JVM的端点；提供工作进程及其关联子进程之间的命运共享。

5）调用env.rpcEnv.awaitTermination()来阻塞程序，直到程序退出。

**CoarseGrainedExecutorBackend类**

从该类的定义上可以看出它是一个RpcEndpoint，因此它是实现RPC通信数据处理功能类。

```scala
private[spark] class CoarseGrainedExecutorBackend(
    override val rpcEnv: RpcEnv,
    driverUrl: String,
    executorId: String,
    hostname: String,
    cores: Int,
    userClassPath: Seq[URL],
    env: SparkEnv)
  extends ThreadSafeRpcEndpoint with ExecutorBackend with Logging {

  private[this] val stopping = new AtomicBoolean(false)
  var executor: Executor = null
  @volatile var driver: Option[RpcEndpointRef] = None

  // If this CoarseGrainedExecutorBackend is changed to support multiple threads, then this may need
  // to be changed so that we don't share the serializer instance across threads
  private[this] val ser: SerializerInstance = env.closureSerializer.newInstance()

  override def onStart() {
    logInfo("Connecting to driver: " + driverUrl)
    rpcEnv.asyncSetupEndpointRefByURI(driverUrl).flatMap { ref =>
      // This is a very fast action so we can use "ThreadUtils.sameThread"
      driver = Some(ref)
      ref.ask[Boolean](RegisterExecutor(executorId, self, hostname, cores, extractLogUrls))
    }(ThreadUtils.sameThread).onComplete {
      // This is a very fast action so we can use "ThreadUtils.sameThread"
      case Success(msg) =>
        // Always receive `true`. Just ignore it
      case Failure(e) =>
        exitExecutor(1, s"Cannot register with driver: $driverUrl", e, notifyDriver = false)
    }(ThreadUtils.sameThread)
  }

  def extractLogUrls: Map[String, String] = {
    val prefix = "SPARK_LOG_URL_"
    sys.env.filterKeys(_.startsWith(prefix))
      .map(e => (e._1.substring(prefix.length).toLowerCase(Locale.ROOT), e._2))
  }

  override def receive: PartialFunction[Any, Unit] = {
    case RegisteredExecutor =>
      logInfo("Successfully registered with driver")
      try {
        executor = new Executor(executorId, hostname, env, userClassPath, isLocal = false)
      } catch {
        case NonFatal(e) =>
          exitExecutor(1, "Unable to create executor due to " + e.getMessage, e)
      }

    case RegisterExecutorFailed(message) =>
      exitExecutor(1, "Slave registration failed: " + message)

    case LaunchTask(data) =>
      if (executor == null) {
        exitExecutor(1, "Received LaunchTask command but executor was null")
      } else {
        val taskDesc = TaskDescription.decode(data.value)
        logInfo("Got assigned task " + taskDesc.taskId)
        executor.launchTask(this, taskDesc)
      }

    case KillTask(taskId, _, interruptThread, reason) =>
      if (executor == null) {
        exitExecutor(1, "Received KillTask command but executor was null")
      } else {
        executor.killTask(taskId, interruptThread, reason)
      }

    case StopExecutor =>
      stopping.set(true)
      logInfo("Driver commanded a shutdown")
      // Cannot shutdown here because an ack may need to be sent back to the caller. So send
      // a message to self to actually do the shutdown.
      self.send(Shutdown)

    case Shutdown =>
      stopping.set(true)
      new Thread("CoarseGrainedExecutorBackend-stop-executor") {
        override def run(): Unit = {
          // executor.stop() will call `SparkEnv.stop()` which waits until RpcEnv stops totally.
          // However, if `executor.stop()` runs in some thread of RpcEnv, RpcEnv won't be able to
          // stop until `executor.stop()` returns, which becomes a dead-lock (See SPARK-14180).
          // Therefore, we put this line in a new thread.
          executor.stop()
        }
      }.start()

    case UpdateDelegationTokens(tokenBytes) =>
      logInfo(s"Received tokens of ${tokenBytes.length} bytes")
      SparkHadoopUtil.get.addDelegationTokens(tokenBytes, env.conf)
  }

  override def onDisconnected(remoteAddress: RpcAddress): Unit = {
    if (stopping.get()) {
      logInfo(s"Driver from $remoteAddress disconnected during shutdown")
    } else if (driver.exists(_.address == remoteAddress)) {
      exitExecutor(1, s"Driver $remoteAddress disassociated! Shutting down.", null,
        notifyDriver = false)
    } else {
      logWarning(s"An unknown ($remoteAddress) driver disconnected.")
    }
  }

  override def statusUpdate(taskId: Long, state: TaskState, data: ByteBuffer) {
    val msg = StatusUpdate(executorId, taskId, state, data)
    driver match {
      case Some(driverRef) => driverRef.send(msg)
      case None => logWarning(s"Drop $msg because has not yet connected to driver")
    }
  }

  /**
   * This function can be overloaded by other child classes to handle
   * executor exits differently. For e.g. when an executor goes down,
   * back-end may not want to take the parent process down.
   */
  protected def exitExecutor(code: Int,
                             reason: String,
                             throwable: Throwable = null,
                             notifyDriver: Boolean = true) = {
    val message = "Executor self-exiting due to : " + reason
    if (throwable != null) {
      logError(message, throwable)
    } else {
      logError(message)
    }

    if (notifyDriver && driver.nonEmpty) {
      driver.get.send(RemoveExecutor(executorId, new ExecutorLossReason(reason)))
    }

    System.exit(code)
  }
}
```
包含的属性包含4个：

+ stopping ---标记executor运行状态

+ executor ---存储当前CoarseGrainedExecutorBackend进程中存储的Executor对象。

+ driver---存储与driver交互使用的RpcEndpointRef对象

+ ser---当前序列化使用的序列化工具

包含的方法解释：

+ onStart()：重写RpcEndpoint的onStart()方法，在该方法rpcEnv.asyncSetupEndpointRefByURI(driverUrl)根据driverUrl异步的方式获取driverEndpointRef并赋值给drvier属性，并发送RegisterExecutor(executorId, self, hostname, cores, extractLogUrls)到driver（schedulerBackend）。

在当前提交模式（yarn-cluster）下，实际driver处理该信息的类是CoarseGrainedSchedulerBackend，driver接收到该信息后会调用CoarseGrainedSchedulerBackend#driverEndpoint#receiveAndReply(context: RpcCallContext)做出响应，receiveAndReply方法内部拿到了executorRef，并使用它发送信息executorRef.send(RegisteredExecutor)给executor（CoarseGrainedExecutorBackend的receive方法将接收到并处理）

+ receive()：重写RpcEndpoint的onStart()方法，接收以下消息并处理：

1. RegisteredExecutor 接收到driver端已经注册了executor(注册时driver保留executorId，executorAddress等信息)，此时才在executor端调用executor = new Executor(executorId, hostname, env, userClassPath, isLocal = false)进行executor启动，executor主要负责执行task，上报task执行状态，进度，资源占用情况等。

2. RegisterExecutorFailed(message) 注册executor失败

3. LaunchTask(data) 加载任务，通过executor去执行 executor.launchTask(this, taskDesc)

4. KillTask(taskId, _, interruptThread, reason) 杀掉task任务

5. StopExecutor 停止executor

6. Shutdown  关闭executor

7. UpdateDelegationTokens(tokenBytes) 更新代理token

　　上边这些参数类型定义在CoarseGrainedClusterMessages中，这些接收到的消息发送者是driver端SparkContext下的schedulerBackend(CoarseGrainedSchedulerBackend)。

+ onDisconnected(remoteAddress: RpcAddress) ：重写RpcEndpoint的onDisconnected()方法

+ statusUpdate(taskId: Long, state: TaskState, data: ByteBuffer)：重写RpcEndpoint的statusUpdate()方法

+ exitExecutor(code: Int, reason: String, throwable: Throwable = null, notifyDriver: Boolean = true)

**driver端RpcEndpoint初始化过程**

CoarseGrainedExecutorBackend在它重写RpcEndpoint的onStart（）方法中，通过driverUrl获取到了driver的RpcEndpointRef，并给driver发送了请求：

>ref.ask[Boolean](RegisterExecutor(executorId, self, hostname, cores, extractLogUrls))

实际上这个接收对象是CoarseGrainedSchedulerBackend，对应的发送类型定义在CoarseGrainedClusterMessages中。

下面看下CoarseGrainedExecutorBackend引用的这个driver端schedulerBackend（CoarseGrainedSchedulerBackend）初始化过程具体过程。

**初始化schedulerBackend和taskScheduler**

在SparkContext初始化过程中，会初始化schedulerBackend和taskScheduler

```scala
  private var _schedulerBackend: SchedulerBackend = _
  private var _taskScheduler: TaskScheduler = _

  private[spark] def schedulerBackend: SchedulerBackend = _schedulerBackend

  private[spark] def taskScheduler: TaskScheduler = _taskScheduler
  private[spark] def taskScheduler_=(ts: TaskScheduler): Unit = {
    _taskScheduler = ts
  }

  // 构造函数中初始化赋值
  // Create and start the scheduler
  val (sched, ts) = SparkContext.createTaskScheduler(this, master, deployMode)
  _schedulerBackend = sched
  _taskScheduler = ts
https://github.com/apache/spark/blob/branch-2.4/core/src/main/scala/org/apache/spark/SparkContext.scala
```

初始化schedulerBackend和taskScheduler实现类YarnClusterManager

SparkContext中最终初始化schedulerBackend和taskScheduler的类是YarnClusterManager

```scala
/**
 * Cluster Manager for creation of Yarn scheduler and backend
 */
private[spark] class YarnClusterManager extends ExternalClusterManager {

  override def canCreate(masterURL: String): Boolean = {
    masterURL == "yarn"
  }

  override def createTaskScheduler(sc: SparkContext, masterURL: String): TaskScheduler = {
    sc.deployMode match {
      case "cluster" => new YarnClusterScheduler(sc)
      case "client" => new YarnScheduler(sc)
      case _ => throw new SparkException(s"Unknown deploy mode '${sc.deployMode}' for Yarn")
    }
  }

  override def createSchedulerBackend(sc: SparkContext,
      masterURL: String,
      scheduler: TaskScheduler): SchedulerBackend = {
    sc.deployMode match {
      case "cluster" =>
        new YarnClusterSchedulerBackend(scheduler.asInstanceOf[TaskSchedulerImpl], sc)
      case "client" =>
        new YarnClientSchedulerBackend(scheduler.asInstanceOf[TaskSchedulerImpl], sc)
      case  _ =>
        throw new SparkException(s"Unknown deploy mode '${sc.deployMode}' for Yarn")
    }
  }

  override def initialize(scheduler: TaskScheduler, backend: SchedulerBackend): Unit = {
    scheduler.asInstanceOf[TaskSchedulerImpl].initialize(backend)
  }
}
复制代码
https://github.com/apache/spark/blob/branch-2.4/resource-managers/yarn/src/main/scala/org/apache/spark/scheduler/cluster/YarnClusterManager.scala
```

YarnClusterManager#createTaskScheduler(...)：在该方法中会根据SparkContext对象的deployMode属性来进行分支判断：

1. client时，返回YarnScheduler(https://github.com/apache/spark/blob/branch-2.4/resource-managers/yarn/src/main/scala/org/apache/spark/scheduler/cluster/YarnScheduler.scala)实例对象；

2. cluster时，返回YarnClusterScheduler(https://github.com/apache/spark/blob/branch-2.4/resource-managers/yarn/src/main/scala/org/apache/spark/scheduler/cluster/YarnClusterScheduler.scala)实例对象。

YarnClusterManager#createSchedulerBackend(...)：在该方法中会根据SparkContext对象的deployMode属性来进行分支判断：

1. client时，返回YarnClientSchedulerBackend(https://github.com/apache/spark/blob/branch-2.4/resource-managers/yarn/src/main/scala/org/apache/spark/scheduler/cluster/YarnClientSchedulerBackend.scala)实例对象；

2. cluster时，返回YarnClusterSchedulerBackend(https://github.com/apache/spark/blob/branch-2.4/resource-managers/yarn/src/main/scala/org/apache/spark/scheduler/cluster/YarnClusterSchedulerBackend.scala)实例对象。

 