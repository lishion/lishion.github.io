---
title: Spark 源码阅读计划 - 第一部分 - 迭代计算
date: 2018-06-06 13:46:45
tags:
  - Spark
  - 编程
categories: Spark源码阅读计划
author: lishion
toc: true
---
首先立一个flag，这将是一个长期更新的版块。

## 写在最开始

在我使用`spark`进行日志分析的时候感受到了`spark`的便捷与强大。在学习`spark`初期，我阅读了许多与`spark`相关的文档，在这个过程中了解了`RDD`，`分区`，`shuffle`等概念，但是我并没有对这些概念有更多具体的认识。由于不了解`spark`的基本运作方式使得我在编写代码的时候十分困扰。由于这个原因，我准备阅读`spark`的源代码来解决我对基本概念的认识。

## 本部分主要内容

虽然该章节的名字叫做**迭代计算**，但是本章会讨论包括**RDD、分区、Job、迭代计算**等相关的内容，其中会重点讨论分区与迭代计算。因此在阅读本章之前你至少需要对这些提到的概念有一定的了解。

## 准备工作

你需要准备一份 Spark 的源代码以及一个便于全局搜索的编辑器|IDE。

## 一个简单的例子

假设现在我们有这样一个需求 : 寻找从 0 to 100 的偶数并输出其总数。利用 spark 编写代码如下:

```scala
sc.parallelize(0 to 100).filter(_%2 == 0).count
//res0: Long = 51
```

这是一个非常简单的例子，但是里面却蕴涵了许多基础的知识。这一章的内容也是从这个例子展开的。

## 迭代计算

### comput 方法与 itertor 

在RDD.scala 定义的类 RDD中，有两个实现迭代计算的核心方法，分别是`compute`以及`itertor`方法。其中`itertor`方法如下:

```scala
  final def iterator(split: Partition, context: TaskContext): Iterator[T] = {
    if (storageLevel != StorageLevel.NONE) {//如果有缓存
      getOrCompute(split, context)
    } else {
      computeOrReadCheckpoint(split, context)
    }
  }
```

我们只关心这个方法的第一个参数**分区**，以及返回的迭代器。忽略有缓存的情况，我们继续看`computeOrReadCheckpoint`这个方法:

```scala
  private[spark] def computeOrReadCheckpoint(split: Partition, context: TaskContext): Iterator[T] =
  {
    if (isCheckpointedAndMaterialized) {//如果有checkpoint
      firstParent[T].iterator(split, context)
    } else {
      compute(split, context)
    }
  }
```

可以看到，在没有缓存和 checkpoint 的情况下，itertor 方法直接调用了 compute 方法。而compute方法定义如下:

```scala
 /**
   * Implemented by subclasses to compute a given partition.
   */
  def compute(split: Partition, context: TaskContext): Iterator[T]
```

从注释可以看出，compute 方法应该由子类实现，用以计算给定分区并生成一个迭代器。如果不考虑缓存和 checkpoint 的情况下，简化一下代码:

```scala
 final def iterator(split: Partition, context: TaskContext): Iterator[T] = {
    compute(split,context)
  }
```

那么实际上 iterator 的功能是: **接受一个分区，对这个分区进行计算，并返回计算结果的迭代器**。而之所以 compute 需要子类来实现，是因为只需要在子类中实现不同的 compute 方法，就能产生不同类型的 RDD。既然不同的RDD 有不同的功能，我们想知道之前的例子是如何进行迭代计算的，就需要知道这个例子中涉及到了哪些 RDD。

首先查看`SparkContext`中与`parallelize`相关的部分代码:

```scala
  def parallelize[T: ClassTag](
      seq: Seq[T],
      numSlices: Int = defaultParallelism): RDD[T] = withScope {
    assertNotStopped()
    new ParallelCollectionRDD[T](this, seq, numSlices, Map[Int, Seq[String]]())
  }
```

可以看到，`parallelize`实际返回了一个`ParallelCollectionRDD`。在`ParallelCollectionRDD`中并没有对`filter`方法进行重写，因此我们查看`RDD`中的`filter`方法:

```scala
 def filter(f: T => Boolean): RDD[T] = withScope {
    val cleanF = sc.clean(f)
    new MapPartitionsRDD[T, T](
      this,
      (context, pid, iter) => iter.filter(cleanF),
      preservesPartitioning = true)
  }
```

filter 方法返回了`MapPartitionsRDD`。为了方便起见，我将 ParallelCollectionRDD 类型的 RDD 称为 a，MapPartitionsRDD 类型的 RDD 成为B。那么先看一下 b 的 compute 方法:

```scala
private[spark] class MapPartitionsRDD[U: ClassTag, T: ClassTag](
    var prev: RDD[T],
    f: (TaskContext, Int, Iterator[T]) => Iterator[U],  // (TaskContext, partition index, iterator)
    preservesPartitioning: Boolean = false)
  extends RDD[U](prev) {
  override val partitioner = if (preservesPartitioning) firstParent[T].partitioner else None
  override def getPartitions: Array[Partition] = firstParent[T].partitions
  override def compute(split: Partition, context: TaskContext): Iterator[U] =
    f(context, split.index, firstParent[T].iterator(split, context))
  override def clearDependencies() {
    super.clearDependencies()
    prev = null
  }
}

```

这里可以看到 compute 方法实际上调用 f 这个函数。而 f 这个函数是在 filter 方法中传入的`(context, pid, iter) => iter.filter(cleanF)` 。那么实际上 compute 进行的计算为:

```scala
(context, split.index, firstParent[T].iterator(split, context)) => firstParent[T].iterator(split, context).filter(cleanF)
```

前两个参数并没有用到，也就是最终的方法可以简化为:

```scala
firstParent[T].iterator(split, context) => firstParent[T].iterator(split, context).filter(cleanF)
```

这里出现了一个`firstParent`，我们知道 RDD 之间存在依赖关系，如果rdd3 依赖 rdd2，rdd2 依赖 rdd1。rdd2 与 rdd3 都是rdd1 的一个父rdd, spark也将其称为血缘关系。而故名意思 rdd2 是 rdd3 的 firstParent。在这个例子中 a 是 b的 firstParent。那么在b 的 compute 中又调用了 a 的 iterator 方法。而 a 的 iterator 方法又会调用a 的 comput 方法。用箭头表示调用关系:

```
b.iterotr -> b.compute -> a.iterotr -> a.compute ->| 调用
b.iterotr <- b.compute <- a.iterotr <- a.compute <-| 返回
```

而这里我们也可以看出来，b 调用 iterotr 返回的数据，是使用 b 中的方法对 a 返回数据进行处理。也就是b 的计算结果依赖于a的计算结果。如果将一个rdd比作一个函数，大概就是 `fb(fa())`，这样的。此时，我们已经得到了关于 spark 迭代的最简单的模型，假如我们有 n 个 rdd。那么之间的调用依赖关系便是:

```
n.iterotr -> n.compute -> (n-1).iterotr -> (n-1).compute ->...-> 1.iterotr -> 1.compute->| 
n.iterotr <- n.compute <- (n-1).iterotr <- (n-1).compute <-...<- 1.iterotr <- 1.compute<-|
```

那么细心的同学应该注意到了，所有的数据都是源自于第一个 RDD。这里把它称为源 RDD。例如本例中产生的 RDD 以及 textFile 方法产生的HadoopRDD都属于源RDD。既然属于源 RDD ，那么这个 RDD 一定会与内存或磁盘中的源数据产生交互，否则就没有真实的数据来源。这里的源数据是指内存的数组、本地文件、或者是HDFS上的分布式文件等。

那么我们再继续看 a 的 compute 方法:

```scala
  override def compute(s: Partition, context: TaskContext): Iterator[T] = {
    new InterruptibleIterator(context,s.asInstanceOf[ParallelCollectionPartition[T]].iterator)
  }
```

可以看到 a 的 compute 根据 ParallelCollectionPartition 类的 iterator直接生成了一个迭代器。再看看ParallelCollectionPartition 的定义:

```scala
rivate[spark] class ParallelCollectionPartition[T: ClassTag](
    var rddId: Long,
    var slice: Int,
    var values: Seq[T]
  ) extends Partition with Serializable {
  def iterator: Iterator[T] = values.iterator
  ...
```

iterator 是来自于 values 的一个迭代器，显然，values是内存中的数据。这说明 a 的 compute 方法直接返回了一个与源数据相关的迭代器。而这里 ParallelCollectionPartition 的数据是如何传入的，又是在什么时候被传入的呢?在**分区**小节将会提到。

到这里，我们已经基本了解了 spark 的迭代计算的原理以及数据来源。但是仍然还有许多不了解的地方，例如 iterotr 会接受一个  Partition 作为参数。那么这个Partition参数到底起到了什么作用。其次，我们知道 rdd 的 iterotr 方法 是从后开始往前调用，那么又是谁调用的最后一个 rdd 的 itertor 方法呢。接下来我们就探索一下与分区相关的内容.

### 分区

提到分区，大部分都是类似于“RDD 中最小的数据单元”，“只是数据的抽象分区，而不是真正的持有数据”等等这样的描述。

```scala
/**
 * An identifier for a partition in an RDD.
 */
trait Partition extends Serializable {
  def index: Int
  override def hashCode(): Int = index
  override def equals(other: Any): Boolean = super.equals(other)
}
```

可以看到分区的类十分的简单，只有一个 index 属性。 当将分区这个参数传入 itertor 时表示我们需要 index 对应的分区的数据经过一系列计算的之后的迭代器。那么显然，在源 RDD 中一定会有更据分区的index获取对应数据的代码部分。而我们又知道，对于`ParallelCollectionPartition`这个分区类是直接持有数据的迭代器，因此我们只需要知道这个类如何被建立，便知道了是如何根据分区获取数据的。我们在整个工程中搜索 ParallelCollectionPartition，发现:

```scala
 override def getPartitions: Array[Partition] = {
    val slices = ParallelCollectionRDD.slice(data, numSlices).toArray
    slices.indices.map(i => new ParallelCollectionPartition(id, i, slices(i))).toArray
  }
```

中生成了一系列的分区对象。其中 ParallelCollectionRDD.slice() 方法根据总共有多少个分区来将 data  进行切分。而 data 我们在调用 parallelize() 方法时传入的。看到这里就应该明白了，**ParallelCollectionRDD 会将我们输入的数组进行切分，然后将利用分区对象持有数据的迭代器，当调用到 itertor 方法时就返回该分区的迭代器，供后续的RDD**进行计算。虽然这里我们只看了 ParallelCollectionRDD 的原代码，但是其他例如 HadoopRDD 的远离基本相同。 只不过一个是从内存中获取数据进行切分，另一个是从磁盘上获取数据进行切分。

但是直到这里，我们仍然不知道 a 中的 itertor 方法的分区对象是如何传入的，因为这个方法间接被 b 的 itertor 方法调用。 而 b 的  itertor 方法同样也需要在外部被调用 ，因此要解决这个问题只需要找到 b 的 itertor 被调用的地方。不过我们首先可以根据代码猜测一下:

```scala
  override def compute(s: Partition, context: TaskContext): Iterator[T] = {
    new InterruptibleIterator(context,s.asInstanceOf[ParallelCollectionPartition[T]].iterator)
  }
```

a 中的 compute 方法直接将传入的分区对象转为了 ParallelCollectionPartition 并获取了对应的迭代器。根据对象间强转的

规则，传入的分区对象只能是 ParallelCollectionPartition 类型 ，因为其父类 RDD 并没有 iterator 方法。 并且传入的ParallelCollectionPartition 一定是持有源数据迭代器的对象，否则在 a 的 compute 中就无法向后返回迭代器。而且我们知道，spark 的计算是**惰性计算**，在调用 action 算子之后才会真正的开始计算。那么可以猜测，b 的 iterator 也是在调用了 count 方法之后才被调用的。为了证明这一点，我们继续查看代码:

```scala
def count(): Long = sc.runJob(this, Utils.getIteratorSize _).sum
```

可以看到是调用了 SparkContext 中的runJob方法:

```scala
 def runJob[T, U: ClassTag](
     ...
    dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)
    ...
  }
```

而 SparkContext 中的 runJob 又继续调用了 DAGScheduler 中的 runJob 方法， 继而调用了 submitJob 方法。之后由经历了 DAGSchedulerEventProcessLoop.post DAGSchedulerEventProcessLoop.doOnReceive 等方法，最后在 submitMissingTasks 看到建立了 ResultTask 对象。由于一个分区的数据会被一个task 处理，因此我们猜测在 ＲesultTask 中会有关于对应 rdd 调用 iterator 的信息，果然，在ResultTask中找到了 runTask 方法:

```scala
override def runTask(context: TaskContext): U = {
    // Deserialize the RDD and the func using the broadcast variables.
    val threadMXBean = ManagementFactory.getThreadMXBean
    val deserializeStartTime = System.currentTimeMillis()
    val deserializeStartCpuTime = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
      threadMXBean.getCurrentThreadCpuTime
    } else 0L
    val ser = SparkEnv.get.closureSerializer.newInstance()
    val (rdd, func) = ser.deserialize[(RDD[T], (TaskContext, Iterator[T]) => U)](
      ByteBuffer.wrap(taskBinary.value), Thread.currentThread.getContextClassLoader)
    _executorDeserializeTime = System.currentTimeMillis() - deserializeStartTime
    _executorDeserializeCpuTime = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
      threadMXBean.getCurrentThreadCpuTime - deserializeStartCpuTime
    } else 0L

    func(context, rdd.iterator(partition, context))//这里就 b 的iterator被真正调用的地方
  }


```

那么我们可以看到最后一行有一个rdd的调用，而这个rdd 是反徐序列化得的，很明显，这个rdd就是 b。此时，b 调用 iterator 方法的地方找到了，但是分区对象partition依然没有找到从哪里传入。由于 partition 是 ResultTask 构造时传入的，我们回到 submitMissingTasks 中，创建ResultTask时分区参数对应的为变量 part 。 

```scala
      //从需要计算的分区map中获取分区，并生成task
       partitionsToCompute.map { id =>
            val p: Int = stage.partitions(id)
            val part = partitions(p)
            val locs = taskIdToLocations(id)
            new ResultTask(stage.id, stage.latestInfo.attemptNumber,
              taskBinary, part, locs, id, properties, serializedTaskMetrics,
              Option(jobId), Option(sc.applicationId), sc.applicationAttemptId)
          }
```

继续往上看，可以看到 part 是 从 partitions 中获取的。这里我们也可以看出 **一个任务对应一个分区数据**。继续往上看，发现`partitions = stage.rdd.partitions`。实际上来自于 Stage 中的 rdd 中的 partitions。那么看Stage 对 rdd 这个属性的描述:

```scala
 *
 * @param rdd RDD that this stage runs on: for a shuffle map stage, it's the RDD we run map tasks
 *   on, while for a result stage, it's the target RDD that we ran an action on
 *   这里的意思是 rdd 表示调用 action 算子的 rdd 也就是 b 
 */
private[scheduler] abstract class Stage(
    val id: Int,
    val rdd: RDD[_],
    val numTasks: Int,
    val parents: List[Stage],
    val firstJobId: Int,
    val callSite: CallSite)
  extends Logging {
```

我们发现一个惊人的事实 ，这个 rdd 居然也是 b。也就是说，我们在 runTask函数中实际调用的方法是这样的:

```scala
  func(context, b.iterator(b.partitions(index), context))//这里就 b 的iterator被真正调用的地方
```

index 表示该task对应需要执行的分区标号。随即我们继续看 b 中的 partitions 的定义:

```scala
 override def getPartitions: Array[Partition] = firstParent[T].partitions
```

说明这个 partitions 来自于 a。 而 a 中的 partitions 实际上又来自于:

```scala
 override def getPartitions: Array[Partition] = {
    val slices = ParallelCollectionRDD.slice(data, numSlices).toArray
    slices.indices.map(i => new ParallelCollectionPartition(id, i, slices(i))).toArray
  }
```

这样最终 runTask 函数实际调用的方法为:

```scala
 func(context, b.iterator(a.partitions(index), context))//这里就 b 的iterator被真正调用的地方
```

这和我们猜测的相同，这个分区确实为 ParallelCollectionPartition 对象，并且也持有了源数据的迭代器。这里其实有一个很巧妙的地方，虽然 RDD  的迭代计算是从后往前调用，但是传入源 RDD 的分区对象依然来自于源 RDD 自身。

到了这里，我们也就明白分区对象是如何和数据关联起来的。ParallelCollectionPartition 对象中存在分区编号 index 以及 源数据的迭代器，通过分区编号就能获取到源数据的迭代器。

## 总结

经过如此大篇幅的讲解，终于对 spark 的迭代计算以及分区做了一个简单的分析。虽然还有很多地方不完善，例如还有其它类型的分区对象没有讲解，但是我相信你能完全看懂这篇文章所讲的内容，其他类型的分区对象对于你来说也很简单了。作者水平有限，并且这也是作者的第一篇关于 spark 源码阅读的博客，难免有遗漏和错误。如果想如给作者提建议和意见的话，请联系作者的微信或直接在 github 上提 issue 。

 

