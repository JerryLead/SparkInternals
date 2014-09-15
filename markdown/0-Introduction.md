# Spark Internals

Spark Version: 1.0.2  
Doc Version: 1.0.2.0

## Authors
| Weibo Id | Name | 
|:-----------|:-------------|
|[@JerryLead](http://weibo.com/jerrylead) | Lijie Xu | 

## Introduction

本文主要讨论 Apache Spark 的设计与实现，重点关注其设计思想、运行原理、实现架构及性能调优，附带讨论与 Hadoop MapReduce 在设计与实现上的区别。不喜欢将该文档称之为“源码分析”，因为本文的主要目的不是去解读实现代码，而是尽量有逻辑地，从设计与实现原理的角度，来理解 job 从产生到执行完成的整个过程，进而去理解整个系统。

讨论系统的设计与实现有很多方法，本文选择 **问题驱动** 的方式，一开始引入问题，然后分问题逐步深入。从一个典型的 job 例子入手，逐渐讨论 job 生成及执行过程中所需要的系统功能支持，然后有选择地深入讨论一些功能模块的设计原理与实现方式。也许这样的方式比一开始就分模块讨论更有主线。

本文档面向的是希望对 Spark 设计与实现机制，以及大数据分布式处理框架深入了解的 Geeks。

因为 Spark 社区很活跃，更新速度很快，本文档也会尽量保持同步，文档号的命名与 Spark 版本一致，只是多了一位，最后一位表示文档的版本号。

由于技术水平、实验条件、经验等限制，当前只讨论 Spark core standalone 版本中的核心功能，而不是全部功能。诚邀各位小伙伴们加入进来，丰富和完善文档。

关于学术方面的一些讨论可以参阅相关的论文以及 Matei 的博士论文，也可以看看我之前写的这篇 [blog](http://www.cnblogs.com/jerrylead/archive/2013/04/27/Spark.html)。

好久没有写这么完整的文档了，上次写还是三年前在学 Ng 的 ML 课程的时候，当年好有激情啊。这次的撰写花了 20+ days，从暑假写到现在，大部分时间花在 debug、画图和琢磨怎么写上，希望文档能对大家和自己都有所帮助。


## Contents
本文档首先讨论 job 如何生成，然后讨论怎么执行，最后讨论系统相关的功能特性。具体内容如下：

1. [Overview](https://github.com/JerryLead/SparkInternals/blob/master/markdown/1-Overview.md) 总体介绍
2. [Job logical plan](https://github.com/JerryLead/SparkInternals/blob/master/markdown/2-JobLogicalPlan.md) 介绍 job 的逻辑执行图（数据依赖图）
3. [Job physical plan](https://github.com/JerryLead/SparkInternals/blob/master/markdown/3-JobPhysicalPlan.md) 介绍 job 的物理执行图
4. [Shuffle details](https://github.com/JerryLead/SparkInternals/blob/master/markdown/4-shuffleDetails.md) 介绍 shuffle 过程
5. [Architecture](https://github.com/JerryLead/SparkInternals/blob/master/markdown/5-Architecture.md) 介绍系统模块如何协调完成整个 job 的执行
6. [Cache and Checkpoint](https://github.com/JerryLead/SparkInternals/blob/master/markdown/6-CacheAndCheckpoint.md)  介绍 cache 和 checkpoint 功能
7. [Broadcast](https://github.com/JerryLead/SparkInternals/blob/master/markdown/7-Broadcast.md) 介绍 broadcast 功能
8. Job Scheduling 尚未撰写
9. Fault-tolerance 尚未撰写

可以直接点 md 文件查看。

喜欢看 pdf 版本的可以去 [这里](https://github.com/JerryLead/SparkInternals/tree/master/pdf) 下载。

如果使用 Mac OS X 的话，推荐下载 [MacDown](http://macdown.uranusjr.com/) 后使用 github 主题去阅读这些文档。

## Examples
写文档期间为了 debug 系统，自己设计了一些 examples，放在了 [SparkLearning/src/internals](https://github.com/JerryLead/SparkLearning/tree/master/src/internals) 下。

## Acknowledgement

文档写作过程中，遇到过一些细节问题，感谢下列同学给予的解答、讨论和帮助：

- [@Andrew-Xia](http://weibo.com/u/1410938285) 参与讨论了 BlockManager 的实现与 broadcast(rdd) 会出现的情况。

- [@CrazyJVM](http://weibo.com/476691290) 参与讨论了 BlockManager 的实现。

- [@王联辉](http://weibo.com/u/1685831233) 参与讨论了 BlockManager 的实现。

感谢下列同学对文档内容的补充：

| Weibo Id | 章节 | 补充内容 | 修改状态 | 
|:-----------|:-------------|:-------------|:-------------|
| [@OopsOutOfMemory](http://weibo.com/oopsoom) | Overview | workers 与 executors 的关系及 [Spark Executor Driver资源调度小结](http://blog.csdn.net/oopsoom/article/details/38763985) | 由于这部分内容的相关实现还在不断 update，本文暂不作结论性总结，已添加详情链接到该同学的 blog |

感谢下列同学指出文档中的不足或错误：

| Weibo Id | 章节 | 不足或错误 | 修改状态 | 
|:-----------|:-------------|:-------------|:-------------|
| [@Joshuawangzj](http://weibo.com/u/1619689670) | Overview | 多个 application 运行时 worker 应该会启动多个 Backend 进程 | 已修改，但需要进一步实验证实。怎么控制 Backend 的个数还不清楚 |
| [@\_cs\_cm](http://weibo.com/u/1551746393) | Overview | 最新的 groupByKey() 已经取消蕴含的 mapValues() 操作，没有MapValuesRDD 产生了 | 已修改 groupByKey() 相关的 figures 和描述 |
| [@染染生起](http://weibo.com/u/2859927402) | JobLogicalPlan | FullDepedency 中的 N:N 关系是否属于 NarrowDependency | 将原来的两种 NarrowDependency 描述改为更清楚的三种，已做更详细的说明 |
| [@zzl0](https://github.com/zzl0) | 前四章 | 很多 typos，比如 “groupByKey 产生了后面三个 RDD”，应该是两个。详见 [pull request](https://github.com/JerryLead/SparkInternals/pull/3/files)。 | 已经全部修改 | 
| [@左手牵右手TEL](http://weibo.com/w397090770) | Cache 和 Broadcast 两章 | 很多 typos | 已经全部修改 | 
| [@cloud-fan](https://github.com/cloud-fan) | JobLogicalPlan | Cogroup() 图中的某些剪头应该是红色的 | 已经全部修改 | 
| [@CrazyJvm](http://weibo.com/476691290) | Shuffle details |  从 Spark1.1开始spark.shuffle.file.buffer.kb的默认值为32k，而不是100k  | 已经全部修改 | 

特别感谢 [@明风Andy](http://weibo.com/mingfengandy) 同学给予的大力支持。

Special thanks to the rockers (including researchers, developers and users) who participate in the design, implementation and discussion of big data systems.