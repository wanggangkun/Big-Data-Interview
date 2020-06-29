###一、spark-submit的入口函数

一般提交一个spark作业的方式采用spark-submit来提交

```scala
# Run on a Spark standalone cluster
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master spark://207.184.161.138:7077 \
  --executor-memory 20G \
  --total-executor-cores 100 \
  /path/to/examples.jar \
  1000
```
这个是提交到standalone集群的方式，其中spark-submit内容如下：

```scala
more opt/cloudera/parcels/SPARK2-2.4.0.cloudera1-1.cdh5.13.3.p0.1007356/lib/spark2/bin/spark-submit

#!/usr/bin/env bash

#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
if [ -z "${SPARK_HOME}" ]; then
  source "$(dirname "$0")"/find-spark-home
fi

# disable randomized hash for string in Python 3.3+
export PYTHONHASHSEED=0

exec "${SPARK_HOME}"/bin/spark-class org.apache.spark.deploy.SparkSubmit "$@"
```
从spark-submit内容上来看，可以发现spark-submit提交任务时，实际上最终是调用了SparkSubmit类。

从SparkSubmit的伴生类上可以看到入口main函数：

```scala
类名：core/src/main/scala/org/apache/spark/deploy/SparkSubmit.scala
object SparkSubmit extends CommandLineUtils with Logging {
  // Cluster managers
  private val YARN = 1
  private val STANDALONE = 2
  private val MESOS = 4
  private val LOCAL = 8
  private val KUBERNETES = 16
  private val ALL_CLUSTER_MGRS = YARN | STANDALONE | MESOS | LOCAL | KUBERNETES

  // Deploy modes
  private val CLIENT = 1
  private val CLUSTER = 2
  private val ALL_DEPLOY_MODES = CLIENT | CLUSTER

  // Special primary resource names that represent shells rather than application jars.
  private val SPARK_SHELL = "spark-shell"
  private val PYSPARK_SHELL = "pyspark-shell"
  private val SPARKR_SHELL = "sparkr-shell"
  private val SPARKR_PACKAGE_ARCHIVE = "sparkr.zip"
  private val R_PACKAGE_ARCHIVE = "rpkg.zip"

  private val CLASS_NOT_FOUND_EXIT_STATUS = 101

  // Following constants are visible for testing.
  private[deploy] val YARN_CLUSTER_SUBMIT_CLASS =
    "org.apache.spark.deploy.yarn.YarnClusterApplication"
  private[deploy] val REST_CLUSTER_SUBMIT_CLASS = classOf[RestSubmissionClientApp].getName()
  private[deploy] val STANDALONE_CLUSTER_SUBMIT_CLASS = classOf[ClientApp].getName()
  private[deploy] val KUBERNETES_CLUSTER_SUBMIT_CLASS =
    "org.apache.spark.deploy.k8s.submit.KubernetesClientApplication"

  override def main(args: Array[String]): Unit = {
    val submit = new SparkSubmit() {
      self =>
      override protected def parseArguments(args: Array[String]): SparkSubmitArguments = {
        new SparkSubmitArguments(args) {
          override protected def logInfo(msg: => String): Unit = self.logInfo(msg)
          override protected def logWarning(msg: => String): Unit = self.logWarning(msg)
        }
      }
      override protected def logInfo(msg: => String): Unit = printMessage(msg)

      override protected def logWarning(msg: => String): Unit = printMessage(s"Warning: $msg")

      override def doSubmit(args: Array[String]): Unit = {
        try {
          super.doSubmit(args)
        } catch {
          case e: SparkUserAppException =>
            exitFn(e.exitCode)
        }
      }
    }
    submit.doSubmit(args)
  }
}
```

在SparkSubmit类中doSubmit函数实现十分简单：

```scala
  类名：core/src/main/scala/org/apache/spark/deploy/SparkSubmit.scala
  def doSubmit(args: Array[String]): Unit = {
    // Initialize logging if it hasn't been done yet. Keep track of whether logging needs to
    // be reset before the application starts.
    val uninitLog = initializeLogIfNecessary(true, silent = true)

    val appArgs = parseArguments(args)
    if (appArgs.verbose) {
      logInfo(appArgs.toString)
    }
    appArgs.action match {
      case SparkSubmitAction.SUBMIT => submit(appArgs, uninitLog)
      case SparkSubmitAction.KILL => kill(appArgs)
      case SparkSubmitAction.REQUEST_STATUS => requestStatus(appArgs)
      case SparkSubmitAction.PRINT_VERSION => printVersion()
    }
  }

```
不难明白这是一个主控函数，根据接受的action类型，调用对应的处理：

+ case SparkSubmitAction.SUBMIT => submit(appArgs, uninitLog)---提交spark任务

+ case SparkSubmitAction.KILL => kill(appArgs)---杀掉spark任务

+ case SparkSubmitAction.REQUEST_STATUS => requestStatus(appArgs)---获取任务状态

+ case SparkSubmitAction.PRINT_VERSION => printVersion()---打印版本信息

我们想明白spark任务提交的具体实现类，需要进入submit函数查看具体的业务：

```scala
/**
   * 运行包含两步：
   * 第一步，我们通过设置适当的类路径，系统属性和应用程序参数来准备启动环境，以便基于集群管理和部署模式运行子主类。
   * 第二步，我们使用这个启动环境来调用子主类的主方法。
   * Submit the application using the provided parameters.
   * 使用提供的参数信息来提交application
   * This runs in two steps. First, we prepare the launch environment by setting up
   * the appropriate classpath, system properties, and application arguments for
   * running the child main class based on the cluster manager and the deploy mode.
   * Second, we use this launch environment to invoke the main method of the child
   * main class.
   */
  @tailrec
  private def submit(args: SparkSubmitArguments, uninitLog: Boolean): Unit = {
    // 通过设置适当的类路径，系统属性和应用程序参数来准备启动环境，以便基于集群管理和部署模式运行子主类。
    val (childArgs, childClasspath, sparkConf, childMainClass) = prepareSubmitEnvironment(args)

    def doRunMain(): Unit = {
      if (args.proxyUser != null) {
        val proxyUser = UserGroupInformation.createProxyUser(args.proxyUser,
          UserGroupInformation.getCurrentUser())
        try {
          proxyUser.doAs(new PrivilegedExceptionAction[Unit]() {
            override def run(): Unit = {
              runMain(childArgs, childClasspath, sparkConf, childMainClass, args.verbose)
            }
          })
        } catch {
          case e: Exception =>
            // Hadoop's AuthorizationException suppresses the exception's stack trace, which
            // makes the message printed to the output by the JVM not very helpful. Instead,
            // detect exceptions with empty stack traces here, and treat them differently.
            if (e.getStackTrace().length == 0) {
              error(s"ERROR: ${e.getClass().getName()}: ${e.getMessage()}")
            } else {
              throw e
            }
        }
      } else {
        runMain(childArgs, childClasspath, sparkConf, childMainClass, args.verbose)
      }
    }

    // Let the main class re-initialize the logging system once it starts.
    if (uninitLog) {
      Logging.uninitialize()
}
    //在独立集群模式下，有两个提交网关：
    //（1）使用o.a.s.deploy.Client作为包装器的传统RPC网关
    //（2）Spark 1.3中引入了新的基于REST的网关
    //后者是Spark 1.3的默认行为，但如果主端点不是REST服务器，则Spark Submit将故障转移到使用旧网关。
    // In standalone cluster mode, there are two submission gateways:
    //   (1) The traditional RPC gateway using o.a.s.deploy.Client as a wrapper
    //   (2) The new REST-based gateway introduced in Spark 1.3
    // The latter is the default behavior as of Spark 1.3, but Spark submit will fail over
    // to use the legacy gateway if the master endpoint turns out to be not a REST server.
    if (args.isStandaloneCluster && args.useRest) {
      try {
        logInfo("Running Spark using the REST application submission protocol.")
        doRunMain()
      } catch {
        // Fail over to use the legacy submission gateway
        case e: SubmitRestConnectionException =>
          logWarning(s"Master endpoint ${args.master} was not a REST server. " +
            "Falling back to legacy submission gateway instead.")
          args.useRest = false
          submit(args, false)
      }
    // 其他模式，只需直接运行主类
    // In all other modes, just run the main class as prepared
    } else {
      doRunMain()
    }
  }
```

上边submit(…)函数最后一行会调用该函数内部自定义函数doRunMain()，该函数会根据应用程序参数（args.proxyUser）做一次判断处理：

1. 如果是代理用户，则使用proxyUser 对runMain()函数包装调用；

2. 如果非代理用户，则直接调用runMain()函数。

###二、任务运行环境准备

通过设置适当的类路径，系统属性和应用程序参数来准备启动环境，以便基于集群管理和部署模式运行子主类。

>val (childArgs, childClasspath, sparkConf, childMainClass) = prepareSubmitEnvironment(args)

```scala
/**  core/src/main/scala/org/apache/spark/deploy/SparkSubmit.scala
   * 未提交的应用程序准备环境
   * Prepare the environment for submitting an application.
   *
   * @param args the parsed SparkSubmitArguments used for environment preparation.
   * @param conf the Hadoop Configuration, this argument will only be set in unit test.
   * @return a 4-tuple:
   *        (1) the arguments for the child process,
   *        (2) a list of classpath entries for the child,
   *        (3) a map of system properties, and
   *        (4) the main class for the child
   *        返回一个4元组(childArgs, childClasspath, sparkConf, childMainClass)
   *        childArgs：子进程的参数
   *        childClasspath：子级的类路径条目列表
   *        sparkConf：系统参数map集合
   *        childMainClass：子级的主类
   *
   * Exposed for testing.
   */
  private[deploy] def prepareSubmitEnvironment(
      args: SparkSubmitArguments,
      conf: Option[HadoopConfiguration] = None)
      : (Seq[String], Seq[String], SparkConf, String) = {
    // Return values
    val childArgs = new ArrayBuffer[String]()
    val childClasspath = new ArrayBuffer[String]()
    val sparkConf = new SparkConf()
    var childMainClass = ""

    // 设置集群管理器，
    // 从这个列表中可以得到信息：spark目前支持的集群管理器包含：YARN,STANDLONE,MESOS,KUBERNETES,LOCAL，
    // 在spark-submit参数的--master中指定。
    // Set the cluster manager
    val clusterManager: Int = args.master match {
      case "yarn" => YARN
      case "yarn-client" | "yarn-cluster" => 
      // spark2.0之前可以使用yarn-cleint,yarn-cluster作为--master参数，从spark2.0起，不再支持，这里默认自动转化为yarn，并给出警告信息。
        logWarning(s"Master ${args.master} is deprecated since 2.0." +
          " Please use master \"yarn\" with specified deploy mode instead.")
        YARN
      case m if m.startsWith("spark") => STANDALONE
      case m if m.startsWith("mesos") => MESOS
      case m if m.startsWith("k8s") => KUBERNETES
      case m if m.startsWith("local") => LOCAL
      case _ =>
        error("Master must either be yarn or start with spark, mesos, k8s, or local")
        -1
    }

    // 设置部署模式--deploy-mode，默认为client模式。
    // Set the deploy mode; default is client mode
    var deployMode: Int = args.deployMode match {
      case "client" | null => CLIENT
      case "cluster" => CLUSTER
      case _ =>
        error("Deploy mode must be either client or cluster")
        -1
    }
    
    // 由于”yarn-cluster“和”yarn-client“方式已被弃用，因此封装了--master和--deploy-mode。
    // 如果只指定了一个--master和--deploy-mode，我们有一些逻辑来推断它们之间的关系；如果它们不一致，我们可以提前退出。
    // Because the deprecated way of specifying "yarn-cluster" and "yarn-client" encapsulate both
    // the master and deploy mode, we have some logic to infer the master and deploy mode
    // from each other if only one is specified, or exit early if they are at odds.
    if (clusterManager == YARN) {
      (args.master, args.deployMode) match {
        case ("yarn-cluster", null) =>
          deployMode = CLUSTER
          args.master = "yarn"
        case ("yarn-cluster", "client") =>
          error("Client deploy mode is not compatible with master \"yarn-cluster\"")
        case ("yarn-client", "cluster") =>
          error("Cluster deploy mode is not compatible with master \"yarn-client\"")
        case (_, mode) =>
          args.master = "yarn"
      }

      // 如果我们想去使用YARN的话，必须确保它包含在我们产品中。
      // Make sure YARN is included in our build if we're trying to use it
      if (!Utils.classIsLoadable(YARN_CLUSTER_SUBMIT_CLASS) && !Utils.isTesting) {
        error(
          "Could not load YARN classes. " +
          "This copy of Spark may not have been compiled with YARN support.")
      }
    }

    if (clusterManager == KUBERNETES) {
      args.master = Utils.checkAndGetK8sMasterUrl(args.master)
      // Make sure KUBERNETES is included in our build if we're trying to use it
      if (!Utils.classIsLoadable(KUBERNETES_CLUSTER_SUBMIT_CLASS) && !Utils.isTesting) {
        error(
          "Could not load KUBERNETES classes. " +
            "This copy of Spark may not have been compiled with KUBERNETES support.")
      }
    }

    // 下边的一些模式是不支持，尽早让它们失败。
    // Fail fast, the following modes are not supported or applicable
    (clusterManager, deployMode) match {
      case (STANDALONE, CLUSTER) if args.isPython =>
        error("Cluster deploy mode is currently not supported for python " +
          "applications on standalone clusters.")
      case (STANDALONE, CLUSTER) if args.isR =>
        error("Cluster deploy mode is currently not supported for R " +
          "applications on standalone clusters.")
      case (LOCAL, CLUSTER) =>
        error("Cluster deploy mode is not compatible with master \"local\"")
      case (_, CLUSTER) if isShell(args.primaryResource) =>
        error("Cluster deploy mode is not applicable to Spark shells.")
      case (_, CLUSTER) if isSqlShell(args.mainClass) =>
        error("Cluster deploy mode is not applicable to Spark SQL shell.")
      case (_, CLUSTER) if isThriftServer(args.mainClass) =>
        error("Cluster deploy mode is not applicable to Spark Thrift server.")
      case _ =>
    }
    
    // 如果args.deployMode为null的话，给它赋值更新。稍后它将作为Spark的属性向下传递
    // Update args.deployMode if it is null. It will be passed down as a Spark property later.
    (args.deployMode, deployMode) match {
      case (null, CLIENT) => args.deployMode = "client"
      case (null, CLUSTER) => args.deployMode = "cluster"
      case _ =>
    }
    // 根据资源管理器和部署模式，进行逻辑判断出几种特殊运行方式。
    val isYarnCluster = clusterManager == YARN && deployMode == CLUSTER
    val isMesosCluster = clusterManager == MESOS && deployMode == CLUSTER
    val isStandAloneCluster = clusterManager == STANDALONE && deployMode == CLUSTER
    val isKubernetesCluster = clusterManager == KUBERNETES && deployMode == CLUSTER
    val isMesosClient = clusterManager == MESOS && deployMode == CLIENT

    if (!isMesosCluster && !isStandAloneCluster) {
      // Resolve maven dependencies if there are any and add classpath to jars. Add them to py-files
      // too for packages that include Python code
      val resolvedMavenCoordinates = DependencyUtils.resolveMavenDependencies(
        args.packagesExclusions, args.packages, args.repositories, args.ivyRepoPath,
        args.ivySettingsPath)

      if (!StringUtils.isBlank(resolvedMavenCoordinates)) {
        args.jars = mergeFileLists(args.jars, resolvedMavenCoordinates)
        if (args.isPython || isInternal(args.primaryResource)) {
          args.pyFiles = mergeFileLists(args.pyFiles, resolvedMavenCoordinates)
        }
      }

      // install any R packages that may have been passed through --jars or --packages.
      // Spark Packages may contain R source code inside the jar.
      if (args.isR && !StringUtils.isBlank(args.jars)) {
        RPackageUtils.checkAndBuildRPackage(args.jars, printStream, args.verbose)
      }
    }

    args.sparkProperties.foreach { case (k, v) => sparkConf.set(k, v) }
    val hadoopConf = conf.getOrElse(SparkHadoopUtil.newConfiguration(sparkConf))
    val targetDir = Utils.createTempDir()

    // assure a keytab is available from any place in a JVM
    if (clusterManager == YARN || clusterManager == LOCAL || isMesosClient) {
      if (args.principal != null) {
        if (args.keytab != null) {
          require(new File(args.keytab).exists(), s"Keytab file: ${args.keytab} does not exist")
          // Add keytab and principal configurations in sysProps to make them available
          // for later use; e.g. in spark sql, the isolated class loader used to talk
          // to HiveMetastore will use these settings. They will be set as Java system
          // properties and then loaded by SparkConf
          sparkConf.set(KEYTAB, args.keytab)
          sparkConf.set(PRINCIPAL, args.principal)
          UserGroupInformation.loginUserFromKeytab(args.principal, args.keytab)
        }
      }
    }

    // Resolve glob path for different resources.
    args.jars = Option(args.jars).map(resolveGlobPaths(_, hadoopConf)).orNull
    args.files = Option(args.files).map(resolveGlobPaths(_, hadoopConf)).orNull
    args.pyFiles = Option(args.pyFiles).map(resolveGlobPaths(_, hadoopConf)).orNull
    args.archives = Option(args.archives).map(resolveGlobPaths(_, hadoopConf)).orNull

    lazy val secMgr = new SecurityManager(sparkConf)

    // In client mode, download remote files.
    var localPrimaryResource: String = null
    var localJars: String = null
    var localPyFiles: String = null
    if (deployMode == CLIENT) {
      localPrimaryResource = Option(args.primaryResource).map {
        downloadFile(_, targetDir, sparkConf, hadoopConf, secMgr)
      }.orNull
      localJars = Option(args.jars).map {
        downloadFileList(_, targetDir, sparkConf, hadoopConf, secMgr)
      }.orNull
      localPyFiles = Option(args.pyFiles).map {
        downloadFileList(_, targetDir, sparkConf, hadoopConf, secMgr)
      }.orNull
    }

    // When running in YARN, for some remote resources with scheme:
    //   1. Hadoop FileSystem doesn't support them.
    //   2. We explicitly bypass Hadoop FileSystem with "spark.yarn.dist.forceDownloadSchemes".
    // We will download them to local disk prior to add to YARN's distributed cache.
    // For yarn client mode, since we already download them with above code, so we only need to
    // figure out the local path and replace the remote one.
    if (clusterManager == YARN) {
      val forceDownloadSchemes = sparkConf.get(FORCE_DOWNLOAD_SCHEMES)

      def shouldDownload(scheme: String): Boolean = {
        forceDownloadSchemes.contains("*") || forceDownloadSchemes.contains(scheme) ||
          Try { FileSystem.getFileSystemClass(scheme, hadoopConf) }.isFailure
      }

      def downloadResource(resource: String): String = {
        val uri = Utils.resolveURI(resource)
        uri.getScheme match {
          case "local" | "file" => resource
          case e if shouldDownload(e) =>
            val file = new File(targetDir, new Path(uri).getName)
            if (file.exists()) {
              file.toURI.toString
            } else {
              downloadFile(resource, targetDir, sparkConf, hadoopConf, secMgr)
            }
          case _ => uri.toString
        }
      }

      args.primaryResource = Option(args.primaryResource).map { downloadResource }.orNull
      args.files = Option(args.files).map { files =>
        Utils.stringToSeq(files).map(downloadResource).mkString(",")
      }.orNull
      args.pyFiles = Option(args.pyFiles).map { pyFiles =>
        Utils.stringToSeq(pyFiles).map(downloadResource).mkString(",")
      }.orNull
      args.jars = Option(args.jars).map { jars =>
        Utils.stringToSeq(jars).map(downloadResource).mkString(",")
      }.orNull
      args.archives = Option(args.archives).map { archives =>
        Utils.stringToSeq(archives).map(downloadResource).mkString(",")
      }.orNull
    }

    // If we're running a python app, set the main class to our specific python runner
    。。。。
    // In YARN mode for an R app, add the SparkR package archive and the R package
    // archive containing all of the built R libraries to archives so that they can
    // be distributed with the job
    。。。。
    // TODO: Support distributing R packages with standalone cluster
    。。。。
    // TODO: Support distributing R packages with mesos cluster
    。。。。
    // If we're running an R app, set the main class to our specific R runner
    。。。。   

    // Special flag to avoid deprecation warnings at the client
    sys.props("SPARK_SUBMIT") = "true"

    // In client mode, launch the application main class directly
    // In addition, add the main application jar and any added jars (if any) to the classpath
    if (deployMode == CLIENT) {
      childMainClass = args.mainClass
      if (localPrimaryResource != null && isUserJar(localPrimaryResource)) {
        childClasspath += localPrimaryResource
      }
      if (localJars != null) { childClasspath ++= localJars.split(",") }
    }
    // Add the main application jar and any added jars to classpath in case YARN client
    // requires these jars.
    // This assumes both primaryResource and user jars are local jars, or already downloaded
    // to local by configuring "spark.yarn.dist.forceDownloadSchemes", otherwise it will not be
    // added to the classpath of YARN client.
    if (isYarnCluster) {
      if (isUserJar(args.primaryResource)) {
        childClasspath += args.primaryResource
      }
      if (args.jars != null) { childClasspath ++= args.jars.split(",") }
    }

    if (deployMode == CLIENT) {
      if (args.childArgs != null) { childArgs ++= args.childArgs }
    }

    // Map all arguments to command-line options or system properties for our chosen mode
    for (opt <- options) {
      if (opt.value != null &&
          (deployMode & opt.deployMode) != 0 &&
          (clusterManager & opt.clusterManager) != 0) {
        if (opt.clOption != null) { childArgs += (opt.clOption, opt.value) }
        if (opt.confKey != null) { sparkConf.set(opt.confKey, opt.value) }
      }
    }

    // In case of shells, spark.ui.showConsoleProgress can be true by default or by user.
    if (isShell(args.primaryResource) && !sparkConf.contains(UI_SHOW_CONSOLE_PROGRESS)) {
      sparkConf.set(UI_SHOW_CONSOLE_PROGRESS, true)
    }

    // Let YARN know it's a pyspark app, so it distributes needed libraries.
    if (clusterManager == YARN) {
      if (args.isPython) {
        sparkConf.set("spark.yarn.isPython", "true")
      }
    }

    // In yarn-cluster mode, use yarn.Client as a wrapper around the user class
    if (isYarnCluster) {
      childMainClass = YARN_CLUSTER_SUBMIT_CLASS
      if (args.isPython) {
        childArgs += ("--primary-py-file", args.primaryResource)
        childArgs += ("--class", "org.apache.spark.deploy.PythonRunner")
      } else if (args.isR) {
        val mainFile = new Path(args.primaryResource).getName
        childArgs += ("--primary-r-file", mainFile)
        childArgs += ("--class", "org.apache.spark.deploy.RRunner")
      } else {
        if (args.primaryResource != SparkLauncher.NO_RESOURCE) {
          childArgs += ("--jar", args.primaryResource)
        }
        childArgs += ("--class", args.mainClass)
      }
      if (args.childArgs != null) {
        args.childArgs.foreach { arg => childArgs += ("--arg", arg) }
      }
    }

    // Load any properties specified through --conf and the default properties file
    for ((k, v) <- args.sparkProperties) {
      sparkConf.setIfMissing(k, v)
    }

    // Ignore invalid spark.driver.host in cluster modes.
    if (deployMode == CLUSTER) {
      sparkConf.remove("spark.driver.host")
    }

    // Resolve paths in certain spark properties
    val pathConfigs = Seq(
      "spark.jars",
      "spark.files",
      "spark.yarn.dist.files",
      "spark.yarn.dist.archives",
      "spark.yarn.dist.jars")
    pathConfigs.foreach { config =>
      // Replace old URIs with resolved URIs, if they exist
      sparkConf.getOption(config).foreach { oldValue =>
        sparkConf.set(config, Utils.resolveURIs(oldValue))
      }
    }

    // Resolve and format python file paths properly before adding them to the PYTHONPATH.
    // The resolving part is redundant in the case of --py-files, but necessary if the user
    // explicitly sets `spark.submit.pyFiles` in his/her default properties file.
    sparkConf.getOption("spark.submit.pyFiles").foreach { pyFiles =>
      val resolvedPyFiles = Utils.resolveURIs(pyFiles)
      val formattedPyFiles = if (!isYarnCluster && !isMesosCluster) {
        PythonRunner.formatPaths(resolvedPyFiles).mkString(",")
      } else {
        // Ignoring formatting python path in yarn and mesos cluster mode, these two modes
        // support dealing with remote python files, they could distribute and add python files
        // locally.
        resolvedPyFiles
      }
      sparkConf.set("spark.submit.pyFiles", formattedPyFiles)
    }

    (childArgs, childClasspath, sparkConf, childMainClass)
  }
```

###三、准备Yarn(Cluster Manager)的执行类

使用spark-submit启动时，实际上执行的是exec "SPARKHOME"/bin/spark−classorg.apache.spark.deploy.SparkSubmit"@"

在SparkSubmit中

>private[deploy] def prepareSubmitEnvironment(args: SparkSubmitArguments,conf: Option[HadoopConfiguration] = None): (Seq[String], Seq[String], SparkConf, String)

方法中会为spark提交做准备，准备好运行环境相关。

其中这方法内部代码中，发现当cluster manager为yarn时：

####3.1、当--deploy-mode:cluster时

会调用YarnClusterApplication进行提交

YarnClusterApplication这是org.apache.spark.deploy.yarn.Client中的一个内部类，在YarnClusterApplication中new了一个Client对象，并调用了run方法

```scala
yarn/src/main/scala/org/apache/spark/deploy/yarn/Client.scala
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

####3.2、当--deploy-mode:client时

调用application-jar.jar自身main函数，执行的是JavaMainApplication

```scala
core/src/main/scala/org/apache/spark/deploy/SparkApplication.scala
/**
 * Entry point for a Spark application. Implementations must provide a no-argument constructor.
 */
private[spark] trait SparkApplication {
  def start(args: Array[String], conf: SparkConf): Unit
}

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
```
从JavaMainApplication实现可以发现，JavaSparkApplication中调用start方法时，只是通过反射执行application-jar.jar的main函数。

####4.3、提交到Yarn

**yarn-cluster运行流程：**

当yarn-custer模式中，YarnClusterApplication类中运行的是Client中run方法，Client#run()中实现了任务提交流程：

```scala
resource-managers/yarn/src/main/scala/org/apache/spark/deploy/yarn/Client.scala
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

在Client类的run()方法中会调用submitApplication()方法，该方法实现：

```scala
  /resource-managers/yarn/src/main/scala/org/apache/spark/deploy/yarn/Client.scala
  /**
   * Submit an application running our ApplicationMaster to the ResourceManager.
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


run()方法则是实现向yarn中的ResourceManager（后文全部简称RM）提交运行任务，并运行我们的ApplicationMaster(后文简称AM)。

稳定的Yarn API提供了一种方便的方法（YarnClient#createApplication），用于创建应用程序和设置应用程序提交上下文。

submitApplication()方法具体操作步骤：

1. 初始化并启动YarnClient，后边将使用yarnClient提供的各种API

2. 通过调用yarnClient#createApplication()方法，从RM获取一个newApp(application)，该newApp用于运行AM。通过newApp#getNewApplicationResponse()返回newApp需要资源情况(newAppResponse)。

3. 通过newAppResponse验证集群是否有足够的资源来运行AM。

4. 设置适当的上下文来以启动AM。

5. 用yarnClient#submitApplication(appContext)向yarn提交任务启动的请求，并监控application。

**yarn-client运行流程：**

对于部署方式是Client的情况，SparkSubmit的main函数中通过反射执行应用程序的main方法

在应用程序的main方法中，创建SparkContext实例

在创建SparkContext的实例过程中，通过如下语句创建Scheduler和Backend实例

```scala
  /core/src/main/scala/org/apache/spark/SparkContext.scala
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
```

###四、SparkContext初始化过程

在Yarn模式下，SparkContext初始化位置因--deploy-mode不同而不同：

yarn-cluster模式下：client会先申请向RM(Yarn Resource Manager)一个Container，来启动AM（ApplicationMaster）进程，而SparkContext运行在AM（ApplicationMaster）进程中；

yarn-client模式下  ：在提交节点上执行SparkContext初始化，由client类（JavaMainApplication）调用。

```scala
/**
   * Create a task scheduler based on a given master URL.
   * Return a 2-tuple of the scheduler backend and the task scheduler.
   */
  private def createTaskScheduler(。。。): (SchedulerBackend, TaskScheduler) = {
    。。。
    master match {
      case "local" =>
        。。。
      case LOCAL_N_REGEX(threads) =>
        。。。
      case LOCAL_N_FAILURES_REGEX(threads, maxFailures) =>
        。。。。
      case SPARK_REGEX(sparkUrl) =>
        。。。。
      case LOCAL_CLUSTER_REGEX(numSlaves, coresPerSlave, memoryPerSlave) =>
        。。。。
      case masterUrl =>
        val cm = getClusterManager(masterUrl) match {
          case Some(clusterMgr) => clusterMgr
          case None => throw new SparkException("Could not parse Master URL: '" + master + "'")
        }
        try {
          val scheduler = cm.createTaskScheduler(sc, masterUrl)
          val backend = cm.createSchedulerBackend(sc, masterUrl, scheduler)
          cm.initialize(scheduler, backend)
          (backend, scheduler)
        } catch {
          case se: SparkException => throw se
          case NonFatal(e) =>
            throw new SparkException("External scheduler cannot be instantiated", e)
        }
    }
  }

  private def getClusterManager(url: String): Option[ExternalClusterManager] = {
    val loader = Utils.getContextOrSparkClassLoader
    val serviceLoaders =
      ServiceLoader.load(classOf[ExternalClusterManager], loader).asScala.filter(_.canCreate(url))
    if (serviceLoaders.size > 1) {
      throw new SparkException(
        s"Multiple external cluster managers registered for the url $url: $serviceLoaders")
    }
    serviceLoaders.headOption
  }  
```

####4.1、SparkContext#createTaskScheduler

根据不同的资源管理方式cluster manager来创建不同的TaskScheduler，SchedulerBackend。

  1.1）SchedulerBackend与cluster manager资源管理器交互取得应用被分配的资源。

  1.2）TaskSheduler在不同的job之间调度，同时接收被分配的资源，之后由他来给每一个Task分配资源。

####4.2、SparkContext#createTaskScheduler

最后一个match case是对其他资源管理方式（除了local和standelone{spark://}外的mesos，yarn，kubernetes【外部资源管理器】的资源管理方式）的处理。

>SparkContext#createTaskScheduler(。。。)#master match#case masterUrl
下边调用了getClusterManager(masterUrl)方法，该方法返回对象是实现了ExternalClusterManager接口的YarnClusterManager类对象。

备注：实现了ExternalClusterManager接口的类还包含：

ExternalClusterManager接口定义：

```scala
core/src/main/scala/org/apache/spark/scheduler/ExternalClusterManager.scala
private[spark] trait ExternalClusterManager {
  def canCreate(masterURL: String): Boolean

  def createTaskScheduler(sc: SparkContext, masterURL: String): TaskScheduler
  
  def createSchedulerBackend(sc: SparkContext,
      masterURL: String,
      scheduler: TaskScheduler): SchedulerBackend
      
  def initialize(scheduler: TaskScheduler, backend: SchedulerBackend): Unit
}
```
ExternalClusterManager接口提供了4个方法：

+ canCreate(masterURL: String):Boolean  检查此群集管理器实例是否可以为某个masterURL创建scheduler组件。

+ createTaskScheduler(sc: SparkContext, masterURL: String):TaskScheduler  为给定的SparkContext创建TaskScheduler实例

+ createSchedulerBackend(sc: SparkContext,masterURL: String,scheduler: TaskScheduler): SchedulerBackend  为给定的SparkContext和调度程序创建SchedulerBackend 。这是在使用“ExternalClusterManager.createTaskScheduler（）”创建TaskScheduler后调用的。

+ initialize(scheduler: TaskScheduler, backend: SchedulerBackend): Unit  初始化TaskScheduler和SchedulerBackend，在创建调度程序组件之后调用。

YarnClusterManager类定义：

```scala
resource-managers/yarn/src/main/scala/org/apache/spark/scheduler/cluster/YarnClusterManager.scala
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
```


YarnClusterManager#createTaskScheduler(...)：

在该方法中会根据SparkContext对象的deployMode属性来进行分支判断：

+ client时，返回YarnScheduler(https://github.com/apache/spark/blob/branch-2.4/resource-managers/yarn/src/main/scala/org/apache/spark/scheduler/cluster/YarnScheduler.scala)实例对象；

+ cluster时，返回YarnClusterScheduler(https://github.com/apache/spark/blob/branch-2.4/resource-managers/yarn/src/main/scala/org/apache/spark/scheduler/cluster/YarnClusterScheduler.scala)实例对象。

YarnClusterManager#createSchedulerBackend(...)：

在该方法中会根据SparkContext对象的deployMode属性来进行分支判断：

+ client时，返回YarnClientSchedulerBackend(https://github.com/apache/spark/blob/branch-2.4/resource-managers/yarn/src/main/scala/org/apache/spark/scheduler/cluster/YarnClientSchedulerBackend.scala)实例对象；

+ cluster时，返回YarnClusterSchedulerBackend(https://github.com/apache/spark/blob/branch-2.4/resource-managers/yarn/src/main/scala/org/apache/spark/scheduler/cluster/YarnClusterSchedulerBackend.scala)实例对象。

###五、Yarn作业运行运行架构原理解析

####5.1、分析Spark on YARN的Cluster模式，从用户提交作业到作业运行结束整个运行期间的过程分析。

**客户端进行操作**

1. 根据yarnConf来初始化yarnClient，并启动yarnClient

2. 创建客户端Application，并获取Application的ID，进一步判断集群中的资源是否满足executor和ApplicationMaster申请的资源，如果不满足则抛出IllegalArgumentException；

3. 设置资源、环境变量：其中包括了设置Application的Staging目录、准备本地资源（jar文件、log4j.properties）、设置Application其中的环境变量、创建Container启动的Context等；

4. 设置Application提交的Context，包括设置应用的名字、队列、AM的申请的Container、标记该作业的类型为Spark；

5. 申请Memory，并最终通过yarnClient.submitApplication向ResourceManager提交该Application。

6. 当作业提交到YARN上之后，客户端就没事了，甚至在终端关掉那个进程也没事，因为整个作业运行在YARN集群上进行，运行的结果将会保存到HDFS或者日志中。

**提交到YARN集群，YARN操作**

1. 运行ApplicationMaster的run方法；

2. 设置好相关的环境变量。

3. 创建amClient，并启动；

4. 在Spark UI启动之前设置Spark UI的AmIpFilter；

5. 在startUserClass函数专门启动了一个线程（名称为Driver的线程）来启动用户提交的Application，也就是启动了Driver。在Driver中将会初始化SparkContext；

6. 等待SparkContext初始化完成，最多等待spark.yarn.applicationMaster.waitTries次数（默认为10），如果等待了的次数超过了配置的，程序将会退出；否则用SparkContext初始化yarnAllocator；

怎么知道SparkContext初始化完成？

其实在5步骤中启动Application的过程中会初始化SparkContext，在初始化SparkContext的时候将会创建YarnClusterScheduler，在SparkContext初始化完成的时候，会调用YarnClusterScheduler类中的postStartHook方法，而该方法会通知ApplicationMaster已经初始化好了SparkContext

7. 当SparkContext、Driver初始化完成的时候，通过amClient向ResourceManager注册ApplicationMaster

8. 分配并启动Executeors。在启动Executeors之前，先要通过yarnAllocator获取到numExecutors个Container，然后在Container中启动Executeors。如果在启动Executeors的过程中失败的次数达到了maxNumExecutorFailures的次数，maxNumExecutorFailures的计算规则如下：

```scala
// Default to numExecutors * 2, with minimum of 3
private val maxNumExecutorFailures =sparkConf.getInt("spark.yarn.max.executor.failures",
sparkConf.getInt("spark.yarn.max.worker.failures", math.max(args.numExecutors *2,3)))
　　那么这个Application将失败，将Application Status标明为FAILED，并将关闭SparkContext。其实，启动Executeors是通过ExecutorRunnable实现的，而ExecutorRunnable内部是启动CoarseGrainedExecutorBackend的。
```

9. 最后，Task将在CoarseGrainedExecutorBackend里面运行，然后运行状况会通过Akka通知CoarseGrainedScheduler，直到作业运行完成。

####5.2、Spark on YARN client 模式作业运行全过程分析

我们知道Spark on yarn有两种模式：yarn-cluster和yarn-client。这两种模式作业虽然都是在yarn上面运行，但是其中的运行方式很不一样，今天我就来谈谈Spark onYARN yarn-client模式作业从提交到运行的过程剖析。

和yarn-cluster模式一样，整个程序也是通过spark-submit脚本提交的。但是yarn-client作业程序的运行不需要通过Client类来封装启动，而是直接通过反射机制调用作业的main函数。下面就来分析：

1. 通过SparkSubmit类的launch的函数直接调用作业的main函数（通过反射机制实现），如果是集群模式就会调用Client的main函数。

2. 而应用程序的main函数一定都有个SparkContent，并对其进行初始化；

3. 在SparkContent初始化中将会依次做如下的事情：设置相关的配置、注册MapOutputTracker、BlockManagerMaster、BlockManager，创建taskScheduler和dagScheduler；其中比较重要的是创建taskScheduler和dagScheduler。在创建taskScheduler的时候会根据我们传进来的master来选择Scheduler和SchedulerBackend。由于我们选择的是yarn-client模式，程序会选择YarnClientClusterScheduler和YarnClientSchedulerBackend，并将YarnClientSchedulerBackend的实例初始化YarnClientClusterScheduler，上面两个实例的获取都是通过反射机制实现的，YarnClientSchedulerBackend类是CoarseGrainedSchedulerBackend类的子类，YarnClientClusterScheduler是TaskSchedulerImpl的子类，仅仅重写了TaskSchedulerImpl中的getRackForHost方法。

4. 初始化完taskScheduler后，将创建dagScheduler，然后通过taskScheduler.start()启动taskScheduler，而在taskScheduler启动的过程中也会调用SchedulerBackend的start方法。在SchedulerBackend启动的过程中将会初始化一些参数，封装在ClientArguments中，并将封装好的ClientArguments传进Client类中，并client.runApp()方法获取Application ID。

5. client.runApp里面的做是和前面客户端进行操作那节类似，不同的是在里面启动是ExecutorLauncher（yarn-cluster模式启动的是ApplicationMaster）。

6. 在ExecutorLauncher里面会初始化并启动amClient，然后向ApplicationMaster注册该Application。注册完之后将会等待driver的启动，当driver启动完之后，会创建一个MonitorActor对象用于和CoarseGrainedSchedulerBackend进行通信（只有事件AddWebUIFilter他们之间才通信，Task的运行状况不是通过它和CoarseGrainedSchedulerBackend通信的）。然后就是设置addAmIpFilter，当作业完成的时候，ExecutorLauncher将通过amClient设置Application的状态为FinalApplicationStatus.SUCCEEDED。

7. 分配Executors，这里面的分配逻辑和yarn-cluster里面类似，就不再说了。

8. 最后，Task将在CoarseGrainedExecutorBackend里面运行，然后运行状况会通过Akka通知CoarseGrainedScheduler，直到作业运行完成。

9. 在作业运行的时候，YarnClientSchedulerBackend会每隔1秒通过client获取到作业的运行状况，并打印出相应的运行信息，当Application的状态是FINISHED、FAILED和KILLED中的一种，那么程序将退出等待。

10. 最后有个线程会再次确认Application的状态，当Application的状态是FINISHED、FAILED和KILLED中的一种，程序就运行完成，并停止SparkContext。整个过程就结束了。
 

####5.3、YARN-Cluster运行架构原理

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

####5.4、YARN-Client运行架构原理

![](16/2.png)

说明如下：

+ Spark Yarn Client（提交代码的机器）向YARN的ResourceManager申请启动Application Master。同时在SparkContent初始化中将创建DAGScheduler和TASKScheduler等，由于我们选择的是Yarn-Client模式，程序会选择YarnClientClusterSchedulerYarnScheduler和YarnClientSchedulerBackend；

+ ResourceManager收到请求后，在集群中选择一个NodeManager，为该应用程序分配第一个Container，要求它在这个Container中启动应用程序的ApplicationMaster，与YARN-Cluster区别的是在该ApplicationMaster不运行SparkContext，只与SparkContext进行联系进行资源的分派；

+ Client中的SparkContext初始化完毕后，与ApplicationMaster建立通讯，向ResourceManager注册，根据任务信息向ResourceManager申请资源（Container）；

+ 一旦ApplicationMaster申请到资源（也就是Container）后，便与对应的NodeManager通信，要求它在获得的Container中启动CoarseGrainedExecutorBackend，CoarseGrainedExecutorBackend启动后会向Client中的SparkContext注册并申请Task；

+ client中的SparkContext分配Task给CoarseGrainedExecutorBackend执行，CoarseGrainedExecutorBackend运行Task并向Driver汇报运行的状态和进度，以让Client随时掌握各个任务的运行状态，从而可以在任务失败时重新启动任务；

+ 应用程序运行完成后，Client的SparkContext向ResourceManager申请注销并关闭自己。

####六、Client模式 vs Cluster模式

理解YARN-Client和YARN-Cluster深层次的区别之前先清楚一个概念：Application Master。在YARN中，每个Application实例都有一个ApplicationMaster进程，它是Application启动的第一个容器。它负责和ResourceManager打交道并请求资源，获取资源之后告诉NodeManager为其启动Container。

从深层次的含义讲YARN-Cluster和YARN-Client模式的区别其实就是ApplicationMaster进程的区别；

+ YARN-Cluster模式下，Driver运行在AM(Application Master)中，它负责向YARN申请资源，并监督作业的运行状况。当用户提交了作业之后，就可以关掉Client，作业会继续在YARN上运行，因而YARN-Cluster模式不适合运行交互类型的作业；

+ YARN-Client模式下，Application Master仅仅向YARN请求Executor，Client会和请求的Container通信来调度他们工作，也就是说Client不能离开；