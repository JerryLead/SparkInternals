# 架构
前三章从 job 的角度介绍了用户写的 program 如何一步步地被分解和执行。这一章主要从架构的角度来讨论 master，worker，driver，和 executor 间怎么协调来完成整个 job 的运行。

## 部署图
重新贴一下 Overview 中给出的部署图：

![deploy](figures/deploy.pdf)

假设要在 master node 上运行第三章给出的 ComplexJob

![ComplexJobStage](figures/ComplexJobStage.pdf)

## 生成 Job 的逻辑执行图和物理执行图

### 生成 Job 逻辑执行图
在 driver 中完成。用户 program 中的 transformation() 建立 computing chain（一系列的 RDD），每个 RDD 的 compute() 定义数据来了怎么计算得到该 RDD 中 partition 的结果，getDependencies() 定义 RDD 之间 partition 的数据依赖。

### 生成 Job 物理执行图
在 driver 中完成。每个 action() 触发生成一个 job（使用 SparkContext.runJob()），在 dagScheduler.runJob()  的时候进行 stage 划分（dagScheduler 被 driver 持有），在 submitStage() 的时候 生成具体的 ShuffleMapTask 或者 ResultTask，然后将 task 交给 driver.SparkDeploy 去分配执行。

### Driver
先看看 driver 中有什么，等等，怎么分析 driver？driver 不就是用户 program 么？用户没写 dagScheduler，SparkDeploy 等对象啊，driver 怎么就包含了它们？

用户 program 会使用 `val sc = new SparkContext(sparkConf)`，这个语句会帮助 program 启动诸多有关 driver 通信、job 执行的对象、线程、actor等，该语句确立了 program 的 dirver 地位。

分析一下 driver 在 SparkContext 中关键对象及其类型：
taskScheduler: TaskSchedulerImpl
dagScheduler: DAGScheduler
backend: SparkDeploySchedulerBackend （如果是 local，启动 LocalBackend）
executorMemory: conf.get("spark.executor.memory", 512MB)

nextShuffleId = new AtomicInteger(0)
newShuffleId(): Int = nextShuffleId.getAndIncrement()
nextRddId = new AtomicInteger(0)
newRddId(): Int = nextRddId.getAndIncrement()

主要 Actors，在 SparkEnv.create() 中初始化：

httpFileServer: HttpFileServer
mapOutputTracker: MapOutputTrackerMaster
shuffleFetcher: "spark.shuffle.fetcher", BlockStoreShuffleFetcher
broadcastManager: BroadcastManager
blockManager: BlockManager
blockManagerMaster: BlockManagerMaster (contains BlockManagerMasterActor)

metricsSystem
actorSystem: ActorSystem = AkkaUtils.createActorSystem(spark://hostname:port)

runJob() => dagScheduler.runJob(), rdd.doCheckpoint()
getExecutorMemoryStatus(): Map[String, (Long, Long)]


