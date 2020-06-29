worker收到信息后，启动CoarseGrainedExecutorBackend，之后worker和master就不通信，通过CoarseGrainedExecutorBackend来代替worker与master
通信，负责管理 Executor





接下来我们具体的说说job的提交：

1. rdd.action()会调用DAGScheduler.runJob(rdd,processPartition,resultHandler)来生成job。
2. runJob()会首先通过rdd.getPartitions()来得到finalRDD中应该存在的partition的个数和类型：Array[Partition]。然后根据partition个数new出来将来要持有result的数组Array[Result](partitions.size)。
3. 最后调用DAGScheduler的runJob(rdd,cleanedFunc,partitions,allowLocal,resultHandler)来提交job。cleanedFunc是processParittion经过闭包清理后的结果，这样可以被序列化后传递给不同节点的task。
4. DAGScheduler的runJob继续调用submitJob(rdd,func,partitions,allowLocal,resultHandler)来提交job。
5. submitJob()首先得到一个jobId，然后再次包装func，向DAGSchedulerEventProcessActor发送JobSubmitted信息，该actor收到信息后进一步调用dagScheduler.handleJobSubmitted()来处理提交的job。之所以这么麻烦，是为
了符合事件驱动模型。
6. handleJobSubmmitted()首先调用finalStage=newStage()来划分stage，然后submitStage(finalStage)。由于finalStage可能有parentstages，实际先提交parentstages，等到他们执行完，finalStage需要再次提交执行。再次提交由handleJobSubmmitted()最后的submitWaitingStages()负责。

分析一下newStage()如何划分stage：
1. 该方法在newStage()的时候会调用finalRDD的getParentStages()。
2. getParentStages()从finalRDD出发，反向visit逻辑执行图，遇到NarrowDependency就将依赖的RDD加入到stage，遇到ShuffleDependency切开stage，并递归到ShuffleDepedency依赖的stage。
3. 一个ShuffleMapStage（不是最后形成result的stage）形成后，会将该stage最后一个RDD注册到MapOutputTrackerMaster.registerShuffle(shuffleDep.shuffleId,rdd.partitions.size)，这一步很重要，因为shuffle过程需要MapOutputTrackerMaster来指示ShuffleMapTask输出数据的位置。

分析一下submitStage(stage)如何提交stage和task：
1. 先确定该stage的missingParentStages，使用getMissingParentStages(stage)。如果parentStages都可能已经执行过了，那么就为空了。
2. 如果missingParentStages不为空，那么先递归提交missing的parentstages，并将自己加入到waitingStages里面，等到parentstages执行结束后，会触发提交waitingStages里面的stage。
3. 如果missingParentStages为空，说明该stage可以立即执行，那么就调用submitMissingTasks(stage,jobId)来生成和提交具体的task。如果stage是ShuffleMapStage，那么new出来与该stage最后一个RDD的partition数相
同的ShuffleMapTasks。如果stage是ResultStage，那么new出来与stage最后一个RDD的partition个数相同的ResultTasks。一个stage里面的task组成一个TaskSet，最后调用taskScheduler.submitTasks(taskSet)来提交一
整个taskSet。
4. 这个taskScheduler类型是TaskSchedulerImpl，在submitTasks()里面，每一个taskSet被包装成manager:
TaskSetMananger，然后交给schedulableBuilder.addTaskSetManager(manager)。schedulableBuilder可以是
FIFOSchedulableBuilder或者FairSchedulableBuilder调度器。submitTasks()最后一步是通
知backend.reviveOffers()去执行task，backend的类型是SchedulerBackend。如果在集群上运行，那么这个
backend类型是SparkDeploySchedulerBackend。

5. SparkDeploySchedulerBackend是CoarseGrainedSchedulerBackend的子类，backend.reviveOffers()其实是向
DriverActor发送ReviveOffers信息。SparkDeploySchedulerBackend在start()的时候，会启动DriverActor。
DriverActor收到ReviveOffers消息后，会调用launchTasks(scheduler.resourceOffers(Seq(new
WorkerOffer(executorId,executorHost(executorId),freeCores(executorId)))))来launchtasks。scheduler
就是TaskSchedulerImpl。scheduler.resourceOffers()从FIFO或者Fair调度器那里获得排序后的
TaskSetManager，并经过TaskSchedulerImpl.resourceOffer()，考虑locality等因素来确定task的全部信息
TaskDescription。调度细节这里暂不讨论。
6. DriverActor中的launchTasks()将每个task序列化，如果序列化大小不超过Akka的akkaFrameSize，那么直接将
task送到executor那里执行executorActor(task.executorId)!LaunchTask(new
SerializableBuffer(serializedTask))。



 4、DAGScheduler
Job=多个stage，Stage=多个同种task, Task分为ShuffleMapTask和ResultTask，Dependency分为ShuffleDependency和NarrowDependency

面向stage的切分，切分依据为宽依赖维护waiting jobs和active jobs，维护waiting stages、active stages和failed stages，以及与jobs的映射关系

主要职能：

1、接收提交Job的主入口，submitJob(rdd, ...)或runJob(rdd, ...)。在SparkContext里会调用这两个方法。 

生成一个Stage并提交，接着判断Stage是否有父Stage未完成，若有，提交并等待父Stage，以此类推。结果是：DAGScheduler里增加了一些waiting stage和一个running stage。
running stage提交后，分析stage里Task的类型，生成一个Task描述，即TaskSet。
调用TaskScheduler.submitTask(taskSet, ...)方法，把Task描述提交给TaskScheduler。TaskScheduler依据资源量和触发分配条件，会为这个TaskSet分配资源并触发执行。
DAGScheduler提交job后，异步返回JobWaiter对象，能够返回job运行状态，能够cancel job，执行成功后会处理并返回结果
2、处理TaskCompletionEvent 

如果task执行成功，对应的stage里减去这个task，做一些计数工作： 
如果task是ResultTask，计数器Accumulator加一，在job里为该task置true，job finish总数加一。加完后如果finish数目与partition数目相等，说明这个stage完成了，标记stage完成，从running stages里减去这个stage，做一些stage移除的清理工作
如果task是ShuffleMapTask，计数器Accumulator加一，在stage里加上一个output location，里面是一个MapStatus类。MapStatus是ShuffleMapTask执行完成的返回，包含location信息和block size(可以选择压缩或未压缩)。同时检查该stage完成，向MapOutputTracker注册本stage里的shuffleId和location信息。然后检查stage的output location里是否存在空，若存在空，说明一些task失败了，整个stage重新提交；否则，继续从waiting stages里提交下一个需要做的stage
如果task是重提交，对应的stage里增加这个task
如果task是fetch失败，马上标记对应的stage完成，从running stages里减去。如果不允许retry，abort整个stage；否则，重新提交整个stage。另外，把这个fetch相关的location和map任务信息，从stage里剔除，从MapOutputTracker注销掉。最后，如果这次fetch的blockManagerId对象不为空，做一次ExecutorLost处理，下次shuffle会换在另一个executor上去执行。
其他task状态会由TaskScheduler处理，如Exception, TaskResultLost, commitDenied等。
3、其他与job相关的操作还包括：cancel job， cancel stage, resubmit failed stage等

其他职能：

 cacheLocations 和 preferLocation

5、TaskScheduler
维护task和executor对应关系，executor和物理资源对应关系，在排队的task和正在跑的task。

内部维护一个任务队列，根据FIFO或Fair策略，调度任务。

TaskScheduler本身是个接口，spark里只实现了一个TaskSchedulerImpl，理论上任务调度可以定制。

主要功能：

1、submitTasks(taskSet)，接收DAGScheduler提交来的tasks 

为tasks创建一个TaskSetManager，添加到任务队列里。TaskSetManager跟踪每个task的执行状况，维护了task的许多具体信息。
触发一次资源的索要。 
首先，TaskScheduler对照手头的可用资源和Task队列，进行executor分配(考虑优先级、本地化等策略)，符合条件的executor会被分配给TaskSetManager。
然后，得到的Task描述交给SchedulerBackend，调用launchTask(tasks)，触发executor上task的执行。task描述被序列化后发给executor，executor提取task信息，调用task的run()方法执行计算。
2、cancelTasks(stageId)，取消一个stage的tasks 

调用SchedulerBackend的killTask(taskId, executorId, ...)方法。taskId和executorId在TaskScheduler里一直维护着。
3、resourceOffer(offers: Seq[Workers])，这是非常重要的一个方法，调用者是SchedulerBacnend，用途是底层资源SchedulerBackend把空余的workers资源交给TaskScheduler，让其根据调度策略为排队的任务分配合理的cpu和内存资源，然后把任务描述列表传回给SchedulerBackend 

从worker offers里，搜集executor和host的对应关系、active executors、机架信息等等
worker offers资源列表进行随机洗牌，任务队列里的任务列表依据调度策略进行一次排序
遍历每个taskSet，按照进程本地化、worker本地化、机器本地化、机架本地化的优先级顺序，为每个taskSet提供可用的cpu核数，看是否满足 
默认一个task需要一个cpu，设置参数为"spark.task.cpus=1"
为taskSet分配资源，校验是否满足的逻辑，最终在TaskSetManager的resourceOffer(execId, host, maxLocality)方法里
满足的话，会生成最终的任务描述，并且调用DAGScheduler的taskStarted(task, info)方法，通知DAGScheduler，这时候每次会触发DAGScheduler做一次submitMissingStage的尝试，即stage的tasks都分配到了资源的话，马上会被提交执行
4、statusUpdate(taskId, taskState, data),另一个非常重要的方法，调用者是SchedulerBacnend，用途是SchedulerBacnend会将task执行的状态汇报给TaskScheduler做一些决定 

若TaskLost，找到该task对应的executor，从active executor里移除，避免这个executor被分配到其他task继续失败下去。
task finish包括四种状态：finished, killed, failed, lost。只有finished是成功执行完成了。其他三种是失败。
task成功执行完，调用TaskResultGetter.enqueueSuccessfulTask(taskSet, tid, data)，否则调用TaskResultGetter.enqueueFailedTask(taskSet, tid, state, data)。TaskResultGetter内部维护了一个线程池，负责异步fetch task执行结果并反序列化。默认开四个线程做这件事，可配参数"spark.resultGetter.threads"=4。
 TaskResultGetter取task result的逻辑

1、对于success task，如果taskResult里的数据是直接结果数据，直接把data反序列出来得到结果；如果不是，会调用blockManager.getRemoteBytes(blockId)从远程获取。如果远程取回的数据是空的，那么会调用TaskScheduler.handleFailedTask，告诉它这个任务是完成了的但是数据是丢失的。否则，取到数据之后会通知BlockManagerMaster移除这个block信息，调用TaskScheduler.handleSuccessfulTask，告诉它这个任务是执行成功的，并且把result data传回去。

2、对于failed task，从data里解析出fail的理由，调用TaskScheduler.handleFailedTask，告诉它这个任务失败了，理由是什么。





表面上看是数据在流动，实质上是算子在流动。
（1）数据不动代码动
（2）在一个Stage内部算子为何会流动（Pipeline）？首先是算子合并，也就是所谓的函数式编程的执行的时候最终进行函数的展开从而把一个Stage内部的多个算子合并成为一个大算子（其内部包含了当前Stage中所有算子对数据的计算逻辑）；其次，是由于Transformation操作的Lazy特性！在具体算子交给集群的Executor计算之前首先会通过Spark Framework(DAGScheduler)进行算子的优化（基于数据本地性的Pipeline）。

作者：SunnyMore
链接：https://www.jianshu.com/p/736a4e628f0f
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。





stage提交算法
在对于最后一个RDD划stage后,进行提交stage,主要的方法是:


 //TODO 递归提交stage，先将第一个stage提交
  private def submitStage(stage: Stage) {
    val jobId = activeJobForStage(stage)
    if (jobId.isDefined) {
      logDebug("submitStage(" + stage + ")")
      //TODO 判断stage是否有父Stage
      if (!waitingStages(stage) && !runningStages(stage) && !failedStages(stage)) {
        //TODO 获取没有提交的stage
        val missing = getMissingParentStages(stage).sortBy(_.id)
        logDebug("missing: " + missing)
        //TODO 这里是递归调用的终止条件，也就是第一个Stage开始提交
        if (missing == Nil) {
          logInfo("Submitting " + stage + " (" + stage.rdd + "), which has no missing parents")
          submitMissingTasks(stage, jobId.get)
        } else {
          //TODO 其实就是从前向后提交stage
          for (parent <- missing) {
            submitStage(parent)
          }
          waitingStages += stage
        }
      }
    } else {
      abortStage(stage, "No active job for stage " + stage.id)
    }
  }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
这里和划分stage的算法一样,拿到最后的stage然后找到第一个stage开始从第一个stage开始提交。

stage提交
下面的代码是submitMissingTasks(),主要是核心的代码:


//TODO 创建多少个Task，Task数量由分区数量决定
    val tasks: Seq[Task[_]] = if (stage.isShuffleMap) {
      partitionsToCompute.map { id =>
        val locs = getPreferredLocs(stage.rdd, id)
        val part = stage.rdd.partitions(id)
        //TODO 这里进行分区局部聚合,从上游拉去数据
        new ShuffleMapTask(stage.id, taskBinary, part, locs)
      }
    } else {
      val job = stage.resultOfJob.get
      partitionsToCompute.map { id =>
        val p: Int = job.partitions(id)
        val part = stage.rdd.partitions(p)
        val locs = getPreferredLocs(stage.rdd, p)
        //TODO 将结果写入持久化介质.比如HDFS等
        new ResultTask(stage.id, taskBinary, part, locs, id)
      }
      
       //TODO 调用taskScheduler来进行提交Task，这里使用TaskSet进行封装Task
      taskScheduler.submitTasks(
              new TaskSet(tasks.toArray, stage.id, stage.newAttemptId(), stage.jobId, properties))
