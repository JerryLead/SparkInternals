# Spark Internals

Spark Version: 1.0.2
Doc Version: 1.0.2.0

## Authors
| Weibo Id | Name |
|:-----------|:-------------|
|[@JerryLead](http://weibo.com/jerrylead) | Lijie Xu |

## Introduction

This series discusses the design and implementation of Apache Spark, with focuses on its design principles, execution mechanisms, code architecture and performance optimization. In addition, there's some comparisons with Hadoop MapReduce in terms of design and implementation. I'm reluctant to call this document a "code walkthrough" because the goal is not to analyze each piece of code in the project, but to understand the whole system in a systematic way, through the process of a Spark job, from its creation to completion.

There're many ways to discuss a computer system, here I've chosen a **problem-driven** approach. Firstly one concret problem is introduced, then it gets analyzed step by step. We'll start from a typical Spark example job to discuss all the system modules and supports it needs for its creation and execution, to give a big picture. Then we'll selectively go into the design and implementation details of some system modules. I believe that this approach is better than diving into each module right from the beginning.

The target audience of this series are geeks who want to have a deeper understanding of Apache Spark as well as other distributed computing frameworks.

I'll try my best to keep this documentation up to date with Spark since it's a fast evolving project with an active community. The documentation's main version is in sync with Spark's version. The additional number at the end represents the documentation's update version.

For more academic oriented discussion, please check Matei's PHD paper and other related papers. You can also have a look my blog (in Chinese) [blog](http://www.cnblogs.com/jerrylead/archive/2013/04/27/Spark.html).

I haven't been writing such complete documentation for a while. Last time it was about three years ago when I was studying Andrew Ng's ML course. I was really motivated at that time! This time I've spent 20+ days on this document, from the summer break till now. Most of the time is spent on debugging, drawing diagrams and thinking how to put my ideas in the right way. I hope you find this series helpful.

## Contents
We start from the creation of a Spark job, then discuss its execution, finally we dive into some related system modules and features.

1. [Overview](https://github.com/JerryLead/SparkInternals/blob/master/markdown/1-Overview.md) Overview of Apache Spark
2. [Job logical plan](https://github.com/JerryLead/SparkInternals/blob/master/markdown/2-JobLogicalPlan.md) Logical plan of a job (data dependency graph)
3. [Job physical plan](https://github.com/JerryLead/SparkInternals/blob/master/markdown/3-JobPhysicalPlan.md) Physical plan
4. [Shuffle details](https://github.com/JerryLead/SparkInternals/blob/master/markdown/4-shuffleDetails.md) Shuffle process
5. [Architecture](https://github.com/JerryLead/SparkInternals/blob/master/markdown/5-Architecture.md) Coordination of system modules in job execution
6. [Cache and Checkpoint](https://github.com/JerryLead/SparkInternals/blob/master/markdown/6-CacheAndCheckpoint.md)  Cache and Checkpoint
7. [Broadcast](https://github.com/JerryLead/SparkInternals/blob/master/markdown/7-Broadcast.md) Broadcast feature
8. Job Scheduling TODO
9. Fault-tolerance TODO


The documentation is written in markdown. The pdf version is also available [here](https://github.com/JerryLead/SparkInternals/tree/master/pdf).

If you're under Max OS X, I recommand [MacDown](http://macdown.uranusjr.com/) with a github theme for reading.

## Examples
I've created some examples to debug the system during the writing, they are avaible under [SparkLearning/src/internals](https://github.com/JerryLead/SparkLearning/tree/master/src/internals).

## Acknowledgement

I appreciate the help from the following in providing solutions and ideas for some detailed issues:

- [@Andrew-Xia](http://weibo.com/u/1410938285) Participated in the discussion of BlockManager's implemetation's impact on broadcast(rdd).

- [@CrazyJVM](http://weibo.com/476691290) Participated in the discussion of BlockManager's implementation.

- [@王联辉](http://weibo.com/u/1685831233) Participated in the discussion of BlockManager's implementation.

Thanks to the following for complementing the document:

| Weibo Id | Chapter | Content | Revision status |
|:-----------|:-------------|:-------------|:-------------|
| [@OopsOutOfMemory](http://weibo.com/oopsoom) | Overview | Relation between workers and executors and [Summary on Spark Executor Driver's Resouce Management](http://blog.csdn.net/oopsoom/article/details/38763985) (in Chinese) | There's not yet a conclusion on this subject since its implementation is still changing, a link to the blog is added |

Thanks to the following for finding errors:

| Weibo Id | Chapter | Error/Issue | Revision status |
|:-----------|:-------------|:-------------|:-------------|
| [@Joshuawangzj](http://weibo.com/u/1619689670) | Overview | When multiple applications are running, multiple Backend process will be created | Corrected, but need to be confirmed. No idea on how to control the number of Backend processes |
| [@\_cs\_cm](http://weibo.com/u/1551746393) | Overview | Latest groupByKey() has removed the mapValues() operation, there's no MapValuesRDD generated | Fixed groupByKey() related diagrams and text |
| [@染染生起](http://weibo.com/u/2859927402) | JobLogicalPlan | N:N relation in FullDepedency N:N is a NarrowDependency | Modified the description of NarrowDependency into 3 different cases with detaild explaination, clearer than the 2 cases explaination before |
| [@zzl0](https://github.com/zzl0) | Fisrt four chapters | Lots of typos，such as "groupByKey has generated the 3 following RDDs"，should be 2. Check [pull request](https://github.com/JerryLead/SparkInternals/pull/3/files)。 | All fixed |
| [@左手牵右手TEL](http://weibo.com/w397090770) | Cache and Broadcast chapter | Lots of typos | All fixed |
| [@cloud-fan](https://github.com/cloud-fan) | JobLogicalPlan | Some arrows in the Cogroup() diagram should be colored red | All fixed |
| [@CrazyJvm](http://weibo.com/476691290) | Shuffle details | Starting from Spark 1.1, the default value for spark.shuffle.file.buffer.kb is 32k, not 100k | All fixed |

Special thanks to [@明风Andy](http://weibo.com/mingfengandy) for his great support.

Special thanks to the rockers (including researchers, developers and users) who participate in the design, implementation and discussion of big data systems.
