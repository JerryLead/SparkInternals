# Physical Plan

In the overview chapter, we briefly introduced the DAG-like physical plan, which contains stages and tasks. In this chapter, we'll look at **how the physical plan (e.g., the stages and tasks) is generated from a logical plan of a Spark application.**

## A Complex Logical Plan

![ComplexJob](../PNGfigures/ComplexJob.png)
The code of this application is attached at the end of this chapter.

**How to properly determine the stages and tasks within such a complex data dependency graph?**

An intuitive idea is to associate one RDD and its preceding RDD to form a stage. In this way, each arrow in the above figure will become a task. For the case of 2 RDDs aggregate into one, we may create a stage with these 3 RDDs. This strategy could be a working solution, but it's not efficient. It has a subtle, but severe problem: **lots of intermediate data needs to be stored**. For a physical task, its result will be stored either on local disk, or in the memory, or both. If a task is generated for each arrow in the data dependency graph, the system needs to store data of all the RDDs. It will cost a lot.

If we examine the logical plan more closely, we may find out that in each RDD, the partitions are independent from each other. That is to say, inside each RDD, the data within a partition will not interfere others. With this observation, an aggressive idea is to consider the whole diagram as a single stage and create one physical task for each partition of the final RDD (`FlatMappedValuesRDD`). The following diagram illustrates this idea:

![ComplexTask](../PNGfigures/ComplexTask.png)

All thick arrows in above diagram belong to task1, whose result is the first partition of the final RDD of the job. Note that in order to compute the first partition of the `CoGroupedRDD`, we need to compute all partitions of its preceding RDDs (e.g., all the partitions in UnionRDD) since it's a `ShuffleDependency`. After that, we can compute the `CoGroupedRDD`'s second and third partition using task2 (thin arrows) and task3 (dashed arrows), which are much simpler.

However, there's 2 problems within this idea:
  - The first task is too large. We have to compute all the partitions of the preceding RDDs (i.e., UnionRDD), because of the `ShuffleDependency`.
  - Need to design clever algorithms to determine which partitions (computed in the first task) need to be cached for the following tasks.

Although there are problems within this idea, there is still a good point in this idea, that is to **pipeline the data: the data is computed only if they are actually needed in a flow fashion**. For example in the first task, we check backwards from the final RDD (`FlatMappedValuesRDD`) to see which RDDs and partitions are actually needed to be computed. Moreover, between the RDDs with `NarrowDependency` relation, no intermediate data needs to be stored.

It will be clearer to understand the pipelining if we look at the partition relationship from a record-level point of view. The following diagram illustrates different  patterns of record processing for RDDs with `NarrowDependency`.

![Dependency](../PNGfigures/pipeline.png)

The first pattern (pipeline pattern) is equivalent to:

```scala
for (record <- records) {
  g(f(record))
}
```

Considering `records` as a stream, we can see that no intermediate results need to be stored. For example, once the result of `g(f(record1))` is generated, the original `record1` and result of `f(record1)` can be garbarge collected. Next, we can compute `g(f(record2))`. However, for other patterns (e.g., the third pattern as follows), this is not the case:

```scala
// The third pattern
def f(records) {
  var result
  for (record <- records)
    result.aggregate(process(record)) // need to store the intermediate result here
  result.iterator // return the iterator of newly-generated [record1, record2, record3]
}

val fResult = f(records) // newly-generated [record1, record2, record3]

for (record <- fResult) {
  g(record)
}
```

It's clear that `f`'s results need to be stored somewhere (e.g., in-memory data structures).

Let's go back to our problem with stages and tasks. The main issue of the above aggressive idea is that we can't pipeline the data flow if there's a `ShuffleDependency`. Since `NarrowDependency` can be pipelined, we can cut off the data flow at each `ShuffleDependency`, leaving chains of RDDs connected by `NarrowDependency`. For example, we can just divide the logical plan into stages like this:

![ComplextStage](../PNGfigures/ComplexJobStage.png)

The strategy for creating stages is to: **check backwards from the final RDD, add each `NarrowDependency` into the current stage, and break out for a new stage when there's a `ShuffleDependency`. In each stage, the task number is determined by the partition number of the last RDD in the stage.**

In above diagram, all thick arrows represent tasks. Since the stages are determined backwards, the last stage's id is 0, stage 1 and stage 2 are both parents of stage 0. **If a stage generates the final result, the tasks in this stage are of type `ResultTask`, otherwise they are `ShuffleMapTask`.** `ShuffleMapTask` gets its name because its results need to be shuffled to the next stage, which is similar to the mappers in Hadoop MapReduce. `ResultTask` can be regarded as reducer (when it gets shuffled data from its parent stages), or mapper (when the current stage has no parents).

**One problem remains:** `NarrowDependency` chain can be pipelined, but in our example application, we've showed only `OneToOneDependency` and `RangeDependency`, how about the `NarrowDependency (M:N)`?

Let's check back the cartesian operation in the previous chapter with complex `NarrowDependency` inside:

![cartesian](../PNGfigures/Cartesian.png)

This NarrowDependency only needs one stage as follows:

![cartesian](../PNGfigures/cartesianPipeline.png)

The thick arrows represent the first `ResultTask`. Since the stage directly outputs the final result, 6 `ResultTask` are generated. Different with `OneToOneDependency`, each `ResultTask` in this job needs to compute 3 RDDs (RDD a, b, and CartesianRDD) and read 2 data blocks, all executed in one single task. **We can see that regardless of the actual type of `NarrowDependency`, be it 1:1 or N:N, `NarrowDependency` chain can always be pipelined. The number of task  is the same as the partition number in the final RDD.**

## Execution of the Physical Plan
We have known how to generate stages and tasks, next problem is that: **how the tasks are executed?**

Let's go back to the physical plan of our example application. Recall that in Hadoop MapReduce, the tasks are executed in order, `map()` generates map outputs, which are partitioned and written to local disk. Then, `shuffle-sort-aggregate` process is applied to generate reduce inputs (i.e., \<k, list(v)\> records). Finally `reduce()` is performed to generate the final result. This process is illustrated in the following diagram:

![MapReduce](../PNGfigures/MapReduce.png)

This execution process cannot be used directly on Spark's physical plan since Hadoop MapReduce's physical plan is simple and fixed, and without pipelining.

The main idea of pipelining is that **the data is computed when they are actually needed in a flow fashion.** We start from the final result and check backwards the RDD chain to find what RDDs and partitions are needed for computing the final result. In most cases, we trace back to some partitions in the leftmost RDD and they are the first to be computed.

**For a stage without parent stages, its leftmost RDD can be evaluate directly (it has no dependency), and each record evaluated can be streamed into the subsequent computations (pipelining).** The computation chain is deduced backwards from the final step, but the actual execution streams the records forwards. One record goes through the whole computation chain before the computation of the next record starts.

For stages with parent stages, we need to execute its parent stages and then fetch the data through shuffle. Once it's done, it becomes the same case as a stage without parent stages.

> In the code, each RDD's `getDependency()` method declares its data dependency. `compute()` method is in charge of receiving upstream records (from parent RDD or data source) and applying computation logic. We often see code like this in RDDs: `firstParent[T].iterator(split, context).map(f)`. Here, `firstParent` is the first dependent RDD, `iterator()` shows that the records are consumed one by one, and `map(f)` applies the computation logic `f` on each record. The `compute()` method returns an iterator of the computed records in this RDD for next computation.

Summary so far: **The whole computation chain is created by checking backwards the data depency from the last RDD. Each `ShuffleDependency` separates stages. In each stage, each RDD's `compute()` method calls `parentRDD.iterator()` to receive the upstream record stream.**

Note that `compute()` method is declared only for computation logic. The actual dependent RDDs are declared in `getDependency()` method, and the dependent partitions are declared in `dependency.getParents()` method.

Let's check the `CartesianRDD` as an example:

```scala
 // RDD x is the cartesian product of RDD a and RDD b
 // RDD x = (RDD a).cartesian(RDD b)
 // Defines how many partitions RDD x should have, and the type of each partition
 override def getPartitions: Array[Partition] = {
    // Create the cross product split
    val array = new Array[Partition](rdd1.partitions.size * rdd2.partitions.size)
    for (s1 <- rdd1.partitions; s2 <- rdd2.partitions) {
      val idx = s1.index * numPartitionsInRdd2 + s2.index
      array(idx) = new CartesianPartition(idx, rdd1, rdd2, s1.index, s2.index)
    }
    array
  }

  // Defines the computation logic for each partition of RDD x (the result RDD)
  override def compute(split: Partition, context: TaskContext) = {
    val currSplit = split.asInstanceOf[CartesianPartition]
    // s1 shows that a partition in RDD x depends on one partition in RDD a
    // s2 shows that a partition in RDD x depends on one partition in RDD b
    for (x <- rdd1.iterator(currSplit.s1, context);
         y <- rdd2.iterator(currSplit.s2, context)) yield (x, y)
  }

  // Defines which are the dependent partitions and RDDs for partition i in RDD x
  //
  // RDD x depends on RDD a and RDD b, both are `NarrowDependency`
  // For the first dependency, partition i in RDD x depends on the partition with
  //     index "i / numPartitionsInRDD2" in RDD a
  // For the second dependency, partition i in RDD x depends on the partition with
  //     index "i % numPartitionsInRDD2" in RDD b
  override def getDependencies: Seq[Dependency[_]] = List(
    new NarrowDependency(rdd1) {
      def getParents(id: Int): Seq[Int] = List(id / numPartitionsInRdd2)
    },
    new NarrowDependency(rdd2) {
      def getParents(id: Int): Seq[Int] = List(id % numPartitionsInRdd2)
    }
  )
```

## Job Creation

Now we've introduced the logical plan and physical plan, then **how and when a Spark job is created? What exactly is a job?**

The following table shows the typical [action()](http://spark.apache.org/docs/latest/programming-guide.html#actions). The second column is `processPartition()` method, it defines how to process the records in each partition and generate the result of finalRDD (we call it partial results). The third column is `resultHandler()` method, it defines how to process the partial results from each partition to form the final results.

| Action | finalRDD(records) => result | compute(results) |
|:---------| :-------|:-------|
| reduce(func) | (record1, record2) => result, (result, record i) => result | (result1, result 2) => result, (result, result i) => result
| collect() |Array[records] => result | Array[result] |
| count() | count(records) => result | sum(result) |
| foreach(f) | f(records) => result | Array[result] |
| take(n) | record (i<=n) => result | Array[result] |
| first() | record 1 => result | Array[result] |
| takeSample() | selected records => result | Array[result] |
| takeOrdered(n, [ordering]) | TopN(records) => result | TopN(results) |
| saveAsHadoopFile(path) | records => write(records) | null |
| countByKey() | (K, V) => Map(K, count(K)) | (Map, Map) => Map(K, count(K)) |

Each time there is an `action()` in user's driver program, a job will be created. For example, `foreach()` action will call `sc.runJob(this, (iter: Iterator[T]) => iter.foreach(f)))` to submit a job to the `DAGScheduler`. If there are other `action()`s in the driver program, there will be other jobs submitted. So, we will have as many jobs as the `action()` operations in a driver program. This is why a driver program is called as an application rather than a job.

The last stage of a job generates the job's results. For example in the `GroupByTest` in the first chapter, there're 2 jobs with 2 sets of results. When a job is submitted, the `DAGScheduler` applies the Application-LogicalPlan-PhysicalPlan strategy to figure out the stages, and submits firstly the **stages without parents** for execution. In this process, the number and type of tasks are also determined. A stage is executed after its parent stages' finish.

## Details in Job Submission

Let's briefly analyze the code for job creation and submission. We'll come back to this part in the Architecture chapter.

1. `rdd.action()` calls `DAGScheduler.runJob(rdd, processPartition, resultHandler)` to create a job.
2. `runJob()` gets the partition number and type of the final RDD by calling `rdd.getPartitions()`. Then it allocates `Array[Result](partitions.size)` for holding the results based on the partition number.
3. Finally, `runJob(rdd, cleanedFunc, partitions, allowLocal, resultHandler)` in `DAGScheduler` is called to submit the job. `cleanedFunc` is the closure-cleaned version of `processPartition`. In this way this function can be serialized and sent to the different worker nodes.
4. `DAGScheduler`'s `runJob()` calls `submitJob(rdd, func, partitions, allowLocal, resultHandler)` to submit a job.
5. `submitJob()` gets a `jobId`, then wrap the function once again and send a `JobSubmitted` message to `DAGSchedulerEventProcessActor`. Upon receiving this message, the actor calls `dagScheduler.handleJobSubmitted()` to handle the submitted job. This is an example of event-driven programming model.
6. `handleJobSubmitted()` firstly calls `finalStage = newStage()` to create stages, then it `submitStage(finalStage)`. If `finalStage` has parents, the parent stages will be submitted first. In this case, `finalStage` is actually submitted by `submitWaitingStages()`.

How `newStage()` divide an RDD chain in stages?
- This method calls `getParentStages()` of the final RDD when instantiating a new stage (`new Stage(...)`)
- `getParentStages()` starts from the final RDD, check backwards the logical plan. It adds the RDD into the current stage if it's a `NarrowDependency`. When it meets a `ShuffleDependency` between RDDs, it takes in the right-side RDD (the RDD after the shuffle) and then concludes the current stage. Then the same logic is applied on the left hand side RDD of the shuffle to form another stage.
- Once a `ShuffleMapStage` is created, its last RDD will be registered `MapOutputTrackerMaster.registerShuffle(shuffleDep.shuffleId, rdd.partitions.size)`. This is important since the shuffle process needs to know the data output location from `MapOuputTrackerMaster`.

Now let's see how `submitStage(stage)` submits stages and tasks:

1. `getMissingParentStages(stage)` is called to determine the `missingParentStages` of the current stage. If the parent stages are all executed, `missingParentStages` will be empty.
2. If `missingParentStages` is not empty, then recursively submit these missing stages, and the current stage is inserted into `waitingStages`. Once the parent stages are done, stages inside `waitingStages` will be run.
3. if `missingParentStages` is empty, then we know the stage can be executed right now. Then `submitMissingTasks(stage, jobId)` is called to generate and submit the actual tasks. If the stage is a `ShuffleMapStage`, then we'll allocate as many `ShuffleMapTask` as the partition number in the final RDD. In the case of `ResultStage`, `ResultTask` instances are allocated instead. The tasks in a stage form a `TaskSet`. Finally `taskScheduler.submitTasks(taskSet)` is called to submit the whole task set.
4. The type of `taskScheduler` is `TaskSchedulerImpl`. In `submitTasks()`, each `taskSet` gets wrapped in a `manager` variable of type `TaskSetManager`, then we pass it to `schedulableBuilder.addTaskSetManager(manager)`. `schedulableBuilder` could be `FIFOSchedulableBuilder` or `FairSchedulableBuilder`, depending on the configuration. The last step of `submitTasks()` is to inform `backend.reviveOffers()` to run the task. The type of backend is `SchedulerBackend`. If the application is run on a cluster, its type will be `SparkDeploySchedulerBackend`.
5. `SparkDeploySchedulerBackend` is a subclass of `CoarseGrainedSchedulerBackend`, `backend.reviveOffers()` actually sends `ReviveOffers` message to `DriverActor`. `SparkDeploySchedulerBackend` launches a `DriverActor` when it starts. Once `DriverActor` receives the `ReviveOffers` message, it will call `launchTasks(scheduler.resourceOffers(Seq(new WorkerOffer(executorId, executorHost(executorId), freeCores(executorId)))))` to launch the tasks. `scheduler.resourceOffers()` obtains the sorted `TaskSetManager` from the FIFO or Fair scheduler and gethers other information about the tasks from `TaskSchedulerImpl.resourceOffer()`. These information are stored in a `TaskDescription`. In this step the data locality information is also considered.
6. `launchTasks()` in the `DriverActor` serialize each task. If the serialized size does not exceed the `akkaFrameSize` limit of Akka, then the task is finally sent to the executor for execution: `executorActor(task.executorId) ! LaunchTask(new SerializableBuffer(serializedTask))`.

## Discussion
Up till now, we've discussed:
- How the driver program triggers jobs
- How to generate a physical plan from a logical plan
- What is pipelining in Spark and how it is used and implemented
- The code analysis of job creation and submission

However, there are subjects left for furhter discussion:
- The shuffle process
- Task execution procedure and execution location

In the next chapter, we'll discuss the shuffle process in Spark.

In my personal opinion, the transformation from logical plan to physical plan is really a masterpiece. The abstractions, such as dependencies, stages, and tasks are all well defined and the logic of the implementation algorithms are very clear.

## Source Code of the Example Job

```scala
package internals

import org.apache.spark.SparkContext
import org.apache.spark.SparkContext._
import org.apache.spark.HashPartitioner


object complexJob {
  def main(args: Array[String]) {

    val sc = new SparkContext("local", "ComplexJob test")

    val data1 = Array[(Int, Char)](
      (1, 'a'), (2, 'b'),
      (3, 'c'), (4, 'd'),
      (5, 'e'), (3, 'f'),
      (2, 'g'), (1, 'h'))
    val rangePairs1 = sc.parallelize(data1, 3)

    val hashPairs1 = rangePairs1.partitionBy(new HashPartitioner(3))


    val data2 = Array[(Int, String)]((1, "A"), (2, "B"),
      (3, "C"), (4, "D"))

    val pairs2 = sc.parallelize(data2, 2)
    val rangePairs2 = pairs2.map(x => (x._1, x._2.charAt(0)))


    val data3 = Array[(Int, Char)]((1, 'X'), (2, 'Y'))
    val rangePairs3 = sc.parallelize(data3, 2)


    val rangePairs = rangePairs2.union(rangePairs3)


    val result = hashPairs1.join(rangePairs)

    result.foreachWith(i => i)((x, i) => println("[result " + i + "] " + x))

    println(result.toDebugString)
  }
}
```
