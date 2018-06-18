---
title: Spark源码阅读计划-第四部分-shuffle read
date: 2018-06-18 10:38:44
tags:
  - Spark
  - 编程
categories: Spark源码阅读计划
author: lishion
toc: true
---

拖了这么久终于把 shuffle read 部分的源码看了一遍了。虽然 shuffle read 再数据合并部分的逻辑要比 shuffle 简单，但是由于这个过程中 executor 要到 master 拉取 shuffle write 结果信息，就涉及到 spark 的 block manager 的一些东西，因此完整的 shuffle read 过程依然是很复杂的。但是由于我还是太菜了，对于 block manage 这一部分看的一知半解，因此关于 executor 与 master 进行元数据交互的部分也就不会写得很详细，当然还是会涉及到一些。有关 block manage 的这部分以后应该会写到，先立个 flag 在这里吧。

## MapStatus	

在 shuffle write 完成之后会返回一个 MapStatus，MapStatus记录了 BlockManagerId 以及最终每个分区的大小。其中 BlockManagerId 包含了 BlockManager 所在的 host 以及 port 等信息。返回的 MapStatus 最终会被 Driver 端获并存储以便于 mapper 端获取。 这里要稍微提一下 BlockManager 的知识，Spark 利用 BlockManager 对数据进行读写，而 Block 就是其中的基本单位。每一个 Block 拥有一个 id。而通过BlockManager 则可以操作这些 Block 。因此 mapper 端只需要知道 BlockManager 的位置以及所需要的文件在哪些 Block 就能获取到对应的数据。这里通过 MapOutputTracker 通过 shuffleid 对一次 shuffle write 中所有的 mapper 产生的 MapStatus 进行了记录。

在每一个 mapper 的 shuffle task 结束后，MapOutputTracker就会将其返回的 MapStatues 进行注册。在 DAGScheduler 中可以看到:

```scala
 case smt: ShuffleMapTask =>
              val shuffleStage = stage.asInstanceOf[ShuffleMapStage]
              val status = event.result.asInstanceOf[MapStatus]
              val execId = status.location.executorId
			  
              mapOutputTracker.registerMapOutput(
                shuffleStage.shuffleDep.shuffleId, smt.partitionId, status)
```

这里的 MapOutputTracker 实际的类型为 MapOutputTrackerMaster 。继续看关于注册的代码:

```scala
 def registerMapOutput(shuffleId: Int, mapId: Int, status: MapStatus) {
    shuffleStatuses(shuffleId).addMapOutput(mapId, status)
  }
```

shuffleStatuses 实际上是一个 hashmap:

```scala
val shuffleStatuses = new ConcurrentHashMap[Int, ShuffleStatus]().asScala
```

ShuffleStatus 是一个类，里面包含了一个 MapStatus 数组。也就是说，通过在 shuffle write 之前注册的 shuffle 所用的 shuffleid 作为索引存储了 shuffle write 过程产生的 MapStatus。而 MapOutputTrackerMaster 是用于 Driver 端的。用于 Executor 的则是 MapOutputTrackerWorker 。在 MapOutputTrackerWorker 中一开始是没有 shuffleStatuses 的。需要从 MapOutputTrackerMaster 中获取。

## shuffle read

shuffle read 代码开始于 Shuffled 中的 compute 方法:

```scala
override def compute(split: Partition, context: TaskContext): Iterator[(K, C)] = {
    val dep = dependencies.head.asInstanceOf[ShuffleDependency[K, V, C]]
    SparkEnv.get.shuffleManager.getReader(dep.shuffleHandle, split.index, split.index + 1, context)
      .read()
      .asInstanceOf[Iterator[(K, C)]]
  }
```

可以看到这里传入了需要计算的分区。代码中获取了一个 reader 并调用了其 read 方法。 getReader 代码如下:

```scala
  override def getReader[K, C](
      handle: ShuffleHandle,
      startPartition: Int,
      endPartition: Int,
      context: TaskContext): ShuffleReader[K, C] = {
    new BlockStoreShuffleReader(
      handle.asInstanceOf[BaseShuffleHandle[K, _, C]], startPartition, endPartition, context)
  }
```

实际上是返回了一个 BlockStoreShuffleReader 。read 方法较长，准备一段一段的讲解:

```scala
override def read(): Iterator[Product2[K, C]]
```

方法返回了一个 Iterator。方法的开始定义了一个:

```scala
 val wrappedStreams = new ShuffleBlockFetcherIterator(
      context,
      blockManager.shuffleClient,
      blockManager,
      // 获取 blockManagerId 与　blockId
      mapOutputTracker.getMapSizesByExecutorId(handle.shuffleId, startPartition, endPartition),
      serializerManager.wrapStream,
      // Note: we use getSizeAsMb when no suffix is provided for backwards compatibility
      SparkEnv.get.conf.getSizeAsMb("spark.reducer.maxSizeInFlight", "48m") * 1024 * 1024,
      SparkEnv.get.conf.getInt("spark.reducer.maxReqsInFlight", Int.MaxValue),
      SparkEnv.get.conf.get(config.REDUCER_MAX_BLOCKS_IN_FLIGHT_PER_ADDRESS),
      SparkEnv.get.conf.get(config.MAX_REMOTE_BLOCK_SIZE_FETCH_TO_MEM),
      SparkEnv.get.conf.getBoolean("spark.shuffle.detectCorrupt", true))
```

这个 wrappedStreams 实际上获取数据的输入流。看看:

```scala
 override def getMapSizesByExecutorId(shuffleId: Int, startPartition: Int, endPartition: Int)
      : Iterator[(BlockManagerId, Seq[(BlockId, Long)])] = {
    logDebug(s"Fetching outputs for shuffle $shuffleId, partitions $startPartition-$endPartition")
    // 通过本次的 shuffleId 先获取 mapper 端产生的 MapStatuses
    val statuses = getStatuses(shuffleId)
    try {
       // 
      MapOutputTracker.convertMapStatuses(shuffleId, startPartition, endPartition, statuses)
    } catch {
      case e: MetadataFetchFailedException =>
        // We experienced a fetch failure so our mapStatuses cache is outdated; clear it:
        mapStatuses.clear()
        throw e
    }
  }
```

随后调用了 convertMapStatuses:

```scala
def convertMapStatuses(
      shuffleId: Int,
      startPartition: Int,
      endPartition: Int,
      statuses: Array[MapStatus]): Iterator[(BlockManagerId, Seq[(BlockId, Long)])] = {
    assert (statuses != null)
    val splitsByAddress = new HashMap[BlockManagerId, ListBuffer[(BlockId, Long)]]
    for ((status, mapId) <- statuses.iterator.zipWithIndex) {
      if (status == null) {
        val errorMessage = s"Missing an output location for shuffle $shuffleId"
        logError(errorMessage)
        throw new MetadataFetchFailedException(shuffleId, startPartition, errorMessage)
      } else {
        for (part <- startPartition until endPartition) {
          val size = status.getSizeForBlock(part) // 这里只获取了需要处理的分区对应的数据
          if (size != 0) {
            // 这里获取到需要计算的分区数据所在的 block。其实就是由 shuffleId, mapId, part
            // 这三个组成的
            splitsByAddress.getOrElseUpdate(status.location, ListBuffer()) +=
                ((ShuffleBlockId(shuffleId, mapId, part), size))
          }
        }
      }
    }
    splitsByAddress.iterator
  }
```

这里涉及到 mapId 和 part 这两个变量。实际上，这两个都是分区的 Id 。只不过 mapId 是 mapper 端对应的分区Id，而 part 是经过 shuffle 之后 reducer 端对应的分区Id。通过`convertMapStatuses`就可以得到需要从哪些 Block 拉取数据。

``` scala
// 获取　block　对应的输入流
val recordIter = wrappedStreams.flatMap { case (blockId, wrappedStream) =>
      // Note: the asKeyValueIterator below wraps a key/value iterator inside of a
      // NextIterator. The NextIterator makes sure that close() is called on the
      // underlying InputStream when all records have been read.
      serializerInstance.deserializeStream(wrappedStream).asKeyValueIterator
    }

    // Update the context task metrics for each record read.
    val readMetrics = context.taskMetrics.createTempShuffleReadMetrics()
    val metricIter = CompletionIterator[(Any, Any), Iterator[(Any, Any)]](
      recordIter.map { record =>
        readMetrics.incRecordsRead(1)
        record
      },
      context.taskMetrics().mergeShuffleReadMetrics())

    // An interruptible iterator must be used here in order to support task cancellation
    //
    val interruptibleIter = new InterruptibleIterator[(Any, Any)](context, metricIter)
	
    val aggregatedIter: Iterator[Product2[K, C]] = if (dep.aggregator.isDefined) {
      　// 如果需要聚合
        if (dep.mapSideCombine) {
        // We are reading values that are already combined
        val combinedKeyValuesIterator = interruptibleIter.asInstanceOf[Iterator[(K, C)]]
        dep.aggregator.get.combineCombinersByKey(combinedKeyValuesIterator, context)
      } else {
        // We don't know the value type, but also don't care -- the dependency *should*
        // have made sure its compatible w/ this aggregator, which will convert the value
        // type to the combined type C
        val keyValuesIterator = interruptibleIter.asInstanceOf[Iterator[(K, Nothing)]]
        dep.aggregator.get.combineValuesByKey(keyValuesIterator, context)
      }
    } else {
      interruptibleIter.asInstanceOf[Iterator[Product2[K, C]]]
    }
```

这里进行聚合的代码实际上在 combineCombinersByKey 中:

```scala
  def combineCombinersByKey(
      iter: Iterator[_ <: Product2[K, C]],
      context: TaskContext): Iterator[(K, C)] = {
    val combiners = new ExternalAppendOnlyMap[K, C, C](identity, mergeCombiners, mergeCombiners)
    // 这里的 insertAll 和　shuffle write 中的 insertAll 效果一样，会排序生成临时文件
    combiners.insertAll(iter)
    updateMetrics(context, combiners)
    combiners.iterator
  }
```

最终返回了 iterator:

```scala
 override def iterator: Iterator[(K, C)] = {
    if (currentMap == null) {
      throw new IllegalStateException(
        "ExternalAppendOnlyMap.iterator is destructive and should only be called once.")
    }
     //　如果没有生成临时文件
    if (spilledMaps.isEmpty) {
      CompletionIterator[(K, C), Iterator[(K, C)]](
        destructiveIterator(currentMap.iterator), freeCurrentMap())
    } else {
      new ExternalIterator()
    }
```

如果有零时文件生成会返回一个 ExternalIterator :

```scala
private val mergeHeap = new mutable.PriorityQueue[StreamBuffer]
    private val sortedMap = CompletionIterator[(K, C), Iterator[(K, C)]](destructiveIterator(
      currentMap.destructiveSortedIterator(keyComparator)), freeCurrentMap())
    // 这里合并了临时文件与内存中的数据
    private val inputStreams = (Seq(sortedMap) ++ spilledMaps).map(it => it.buffered)
    inputStreams.foreach { it =>
      val kcPairs = new ArrayBuffer[(K, C)]
      readNextHashCode(it, kcPairs)
      if (kcPairs.length > 0) {
        mergeHeap.enqueue(new StreamBuffer(it, kcPairs))
      }
    }
```

这里涉及到了一个很重要的类 ArrayBuffer:

```scala
    private class StreamBuffer(
        val iterator: BufferedIterator[(K, C)],
        val pairs: ArrayBuffer[(K, C)])
      extends Comparable[StreamBuffer] {
      def isEmpty: Boolean = pairs.length == 0
      def minKeyHash: Int = {
        assert(pairs.length > 0)
        hashKey(pairs.head)
      }
      override def compareTo(other: StreamBuffer): Int = {
        if (other.minKeyHash < minKeyHash) -1 else if (other.minKeyHash == minKeyHash) 0 else 1
      }
```

StreamBuffer 实际上维持了一个 iterator 与一个数组。并且重写了 compareTo 方法。是通过数据中第一个元素的 key 也就是minKeyHash来比较大小。而代码:

```scala
 readNextHashCode(it, kcPairs)
      if (kcPairs.length > 0) {
        mergeHeap.enqueue(new StreamBuffer(it, kcPairs))
      }
```

再看 next 方法:

```scala
    override def next(): (K, C) = {
      if (mergeHeap.isEmpty) {
        throw new NoSuchElementException
      }
      // Select a key from the StreamBuffer that holds the lowest key hash
      //　这里需要注意的是每一个 StreamBuffer 中的数组的元素是按照 key 经过排序的，mergeHeap 中的 StreamBuffer 也是按照 minKeyHash 进行排序的。也就是从 mergeHeap 每取出一个StreamBuffer，其对应的数组中 key 的 hash 一定是目前 mergeHeap 中所有数组中 key 最小的，如果能理解这一点，那这里的 merge 就基本可以理解了。
      val minBuffer = mergeHeap.dequeue()
      val minPairs = minBuffer.pairs
      val minHash = minBuffer.minKeyHash
      val minPair = removeFromBuffer(minPairs, 0)
      val minKey = minPair._1
      var minCombiner = minPair._2
      assert(hashKey(minPair) == minHash)

      // For all other streams that may have this key (i.e. have the same minimum key hash),
      // merge in the corresponding value (if any) from that stream
      val mergedBuffers = ArrayBuffer[StreamBuffer](minBuffer)
      // 如果下一个 StreamBuffer 中的 minKeyHash 相同，则可能会含有相同的 key，则需要合并。这里由于是经过排序的，所以不用遍历所有的 StreamBuffer。只要下一个不同，则后面一定不会有与当前 minKeyHash 相同的的 StreamBuffer。
      while (mergeHeap.nonEmpty && mergeHeap.head.minKeyHash == minHash) {
        val newBuffer = mergeHeap.dequeue()
        // 这里如果有相同的 key 则进行合并
        // 注意，hash 相同 key 不一定相同
        minCombiner = mergeIfKeyExists(minKey, minCombiner, newBuffer)
        mergedBuffers += newBuffer
      }

      // Repopulate each visited stream buffer and add it back to the queue if it is non-empty
      
      mergedBuffers.foreach { buffer =>
        // 如果 key 被合并完了，就需要读取下一批hash相同的key的数据到ArrayBuffer的数组中
        if (buffer.isEmpty) {
          readNextHashCode(buffer.iterator, buffer.pairs)
        }
        if (!buffer.isEmpty) {
          // 刚才dequeue的 ArrayBuffer的数组中可能有没合并完的数据 
          // 或者有新读取的数据则需要继续放入 mergeHeap 中进行合并 
          mergeHeap.enqueue(buffer)
        }
      }
      // 返回合并完的数据
      // 注意这里的 key 只是按照 hash 进行排序的，在 ExternalSorter 才是按照用户定义的排序方式进行排序
      (minKey, minCombiner)
    }
```

如果不需要排序，shuffle read 就算完成了。但是如果需要排序则:

```scala
 val resultIter = dep.keyOrdering match {
      case Some(keyOrd: Ordering[K]) =>
        // Create an ExternalSorter to sort the data.
        val sorter =
          new ExternalSorter[K, C, C](context, ordering = Some(keyOrd), serializer = dep.serializer)
        sorter.insertAll(aggregatedIter)
        context.taskMetrics().incMemoryBytesSpilled(sorter.memoryBytesSpilled)
        context.taskMetrics().incDiskBytesSpilled(sorter.diskBytesSpilled)
        context.taskMetrics().incPeakExecutionMemory(sorter.peakMemoryUsedBytes)
        // Use completion callback to stop sorter if task was finished/cancelled.
        context.addTaskCompletionListener(_ => {
          sorter.stop()
        })
        CompletionIterator[Product2[K, C], Iterator[Product2[K, C]]](sorter.iterator, sorter.stop())
      case None =>
        aggregatedIter
    }

    resultIter match {
      case _: InterruptibleIterator[Product2[K, C]] => resultIter
      case _ =>
        // Use another interruptible iterator here to support task cancellation as aggregator
        // or(and) sorter may have consumed previous interruptible iterator.
        new InterruptibleIterator[Product2[K, C]](context, resultIter)
    }
```

这里依然使用了 ExternalSorter 进行排序，而最终使用了 ExternalSorter 的 iterator 方法。

```scala
  def iterator: Iterator[Product2[K, C]] = {
    isShuffleSort = false
    partitionedIterator.flatMap(pair => pair._2)
  }

```

其中 partitionedIterator 方法在 shuffle write 部分已经讲的很清楚了。有兴趣的可以去看看。这里 shuffle read 就基本结束了。其中从其他 Executor 拉取数据的部分由于涉及很多，这里就基本没有怎么讲解。以后可能会专门开一篇文章。