---
title: spark源码阅读计划 - 第二部分 - shuffle write
date: 2018-06-08 13:46:45
tags:
  - Spark
  - 编程
categories: Spark源码阅读计划
author: lishion
toc: true
---
## 写在开始的话

shuffle 作为 spark 最重要的一部分，也是很多初学者感到头痛的一部分。简单来说，shuffle 分为两个部分 : shuffle write 和 shuffle read。shuffle write 在 map 端进行，而shuffle read 在 reduce 端进行。本章就准备就 shuffle write 做一个详细的分析。

## 为什么需要shuffle

在上一章[Spark-源码阅读计划-第一部分-迭代计算](https://lishion.github.io/2018/06/06/Spark-%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E8%AE%A1%E5%88%92-%E7%AC%AC%E4%B8%80%E9%83%A8%E5%88%86-%E8%BF%AD%E4%BB%A3%E8%AE%A1%E7%AE%97/)提到过每一个分区的数据最后都会由一个task来处理，那么当task 执行的类似于 reduceByKey 这种算子的时候一定需要满足相同的 key 在同一个分区这个条件，否则就不能按照 key 进行reduce了。但是当我们从数据源获取数据并进行处理后，相同的 key 不一定在同一个分区。那么在一个 map 端将相同的key 使用某种方式聚集在同一个分区并写入临时文件的过程就称为 shuffle write；而在 reduce 从 map 拉去执行 task 所需要的数据就是 shuffle read；这两个步骤组成了shuffle。

## 一个小例子

本章同样以一个小例子作为本章讲解的中心，例如一个统计词频的代码:

```scala
sc.parallelize(List("hello","world","hello","spark")).map(x=>(x,1)).reduceByKey(_+_)
```

相信有一点基础的同学都知道 reduceByKey 这算子会产生 shuffle。但是大部分的同学都不知 shuffle 是什么时候产生的，shuffle 到底做了些什么，这也是这一章的重点。

## shuffle task

在上一章提到过当 job 提交后会生成 result task。而在需要 shuffle 则会生成 ShuffleMapTask。这里涉及到 job 提交以及 stage 生成的的知识，将会在下一章提到。这里我们直接开始看 ShuffleMapTask 中的 runtask :

```scala
 override def runTask(context: TaskContext): MapStatus = {
    // Deserialize the RDD using the broadcast variable.
 ...
    var writer: ShuffleWriter[Any, Any] = null
    try {
      val manager = SparkEnv.get.shuffleManager
      writer = manager.getWriter[Any, Any](dep.shuffleHandle, partitionId, context)
       // 这里依然是rdd调用iterator方法的地方
      writer.write(rdd.iterator(partition, context).asInstanceOf[Iterator[_ <: Product2[Any, Any]]])
      writer.stop(success = true).get
    } catch {
      case e: Exception =>
        try {
          if (writer != null) {
            writer.stop(success = false)
          }
        } catch {
          case e: Exception =>
            log.debug("Could not stop writer", e)
        }
        throw e
    }
  }
```

可以看到首先获得一个 shuffleWriter。spark 中有许多不同类型的 writer ，默认使用的是 SortShuffleWriter。SortShuffleWriter 中的 write 方法实现了 shuffle write。源代码如下:

```scala
  override def write(records: Iterator[Product2[K, V]]): Unit = {
      //如果需要 map 端聚
    sorter = if (dep.mapSideCombine) {
      new ExternalSorter[K, V, C](
        context, dep.aggregator, Some(dep.partitioner), dep.keyOrdering, dep.serializer)
    } else {
      new ExternalSorter[K, V, V](
        context, aggregator = None, Some(dep.partitioner), ordering = None, dep.serializer)
    }
    sorter.insertAll(records)
    val output = shuffleBlockResolver.getDataFile(dep.shuffleId, mapId)
    val tmp = Utils.tempFileWith(output)
    try { 
      val blockId = ShuffleBlockId(dep.shuffleId, mapId, IndexShuffleBlockResolver.NOOP_REDUCE_ID)
      val partitionLengths = sorter.writePartitionedFile(blockId, tmp)
      shuffleBlockResolver.writeIndexFileAndCommit(dep.shuffleId, mapId, partitionLengths, tmp)
      mapStatus = MapStatus(blockManager.shuffleServerId, partitionLengths)
    } finally {
      if (tmp.exists() && !tmp.delete()) {
        logError(s"Error while deleting temp file ${tmp.getAbsolutePath}")
      }
    }
  }
```

使用了 ExternalSorter 的 insertAll 方法插入了上一个 RDD 生成的数据的迭代器。

#### insertAll

```scala
def insertAll(records: Iterator[Product2[K, V]]): Unit = {
    // TODO: stop combining if we find that the reduction factor isn't high
    val shouldCombine = aggregator.isDefined
    // 直接看需要 map 端聚合的情况
    // 不需要聚合的情况类似
    if (shouldCombine) {
      // Combine values in-memory first using our AppendOnlyMap
       
      val mergeValue = aggregator.get.mergeValue 
      val createCombiner = aggregator.get.createCombiner
      var kv: Product2[K, V] = null
      // 如果 key 不存在则直接创建 Combiner ，否则使用 mergeValue 将 value 添加到创建的 Combiner中
      val update = (hadValue: Boolean, oldValue: C) => {
        if (hadValue) mergeValue(oldValue, kv._2) else createCombiner(kv._2)
      }
      while (records.hasNext) {
        addElementsRead()
        kv = records.next()
        // 更新值
        // 这里需要注意的是之前的数据是(k,v)类型，这里转换成了((part,k),v)
        // 其中 part 是 key 对应的分区
        map.changeValue((getPartition(kv._1), kv._1), update)
        maybeSpillCollection(usingMap = true)
      }
    } else {
      // Stick values into our buffer
      while (records.hasNext) {
        addElementsRead()
        val kv = records.next()
        buffer.insert(getPartition(kv._1), kv._1, kv._2.asInstanceOf[C])
        maybeSpillCollection(usingMap = false)
      }
    }
  }
```

在这里使用 getPartition() 进行了数据的**分区**，也就是将不同的 key 与其分区对应起来。默认使用的是 HashPartitioner ，有兴趣的同学可以去阅读源码。再继续看 changeValue 方法。在这里使用非常重要的数据结构来存放分区后的数据，也就是代码中的 map 类型为 PartitionedAppendOnlyMap。其中是使用hask表存放数据，并且存放的方式为表中的偶数索index引存放key, index + 1 存放对应的value。例如:

```
k1|v1|null|k2|v2|k3|v3|null|....
```

之所以里面由空值，是因为在插入的时候使用了二次探测法。来看一看 changeValue 方法:

PartitionedAppendOnlyMap 中的  changeValue 方法继承自 

```scala
  def changeValue(key: K, updateFunc: (Boolean, V) => V): V = {
    assert(!destroyed, destructionMessage)
    val k = key.asInstanceOf[AnyRef]
    if (k.eq(null)) {
      if (!haveNullValue) {
        incrementSize()
      }
      nullValue = updateFunc(haveNullValue, nullValue)
      haveNullValue = true
      return nullValue
    }
    var pos = rehash(k.hashCode) & mask
    var i = 1
    while (true) {
      // 偶数位置为 key
      val curKey = data(2 * pos)
       // 如果该key不存在，就直接插入
      if (curKey.eq(null)) {
        val newValue = updateFunc(false, null.asInstanceOf[V])
        data(2 * pos) = k
        data(2 * pos + 1) = newValue.asInstanceOf[AnyRef]
        incrementSize()
        return newValue
      } else if (k.eq(curKey) || k.equals(curKey)) {//如果 key 存在，则进行聚合
        
        val newValue = updateFunc(true, data(2 * pos + 1).asInstanceOf[V])
        data(2 * pos + 1) = newValue.asInstanceOf[AnyRef]
        return newValue
      } else { // 否则进行下一次探测
        val delta = i
        pos = (pos + delta) & mask
        i += 1
      }
    }
    null.asInstanceOf[V] // Never reached but needed to keep compiler happy
  }
```

再调用 insertAll 进行了数据的聚合|插入之后，调用了 maybeSpillCollection 方法。这个方法的作用是判断 map 占用了内存，如果内存达到一定的值，则将内存中的数据 spill 到临时文件中。

```scala
private def maybeSpillCollection(usingMap: Boolean): Unit = {
    var estimatedSize = 0L
    if (usingMap) {
      estimatedSize = map.estimateSize()
      if (maybeSpill(map, estimatedSize)) {
        map = new PartitionedAppendOnlyMap[K, C]
      }
    } else {
      estimatedSize = buffer.estimateSize()
      if (maybeSpill(buffer, estimatedSize)) {
        buffer = new PartitionedPairBuffer[K, C]
      }
    }

    if (estimatedSize > _peakMemoryUsedBytes) {
      _peakMemoryUsedBytes = estimatedSize
    }
  }
```

maybeSpillCollection 继续调用了 maybeSpill 方法，这个方法继承自 Spillable 中

```scala
protected def maybeSpill(collection: C, currentMemory: Long): Boolean = {
    var shouldSpill = false
    if (elementsRead % 32 == 0 && currentMemory >= myMemoryThreshold) {
      // Claim up to double our current memory from the shuffle memory pool
      val amountToRequest = 2 * currentMemory - myMemoryThreshold
      val granted = acquireMemory(amountToRequest)
      myMemoryThreshold += granted
      // If we were granted too little memory to grow further (either tryToAcquire returned 0,
      // or we already had more memory than myMemoryThreshold), spill the current collection
      shouldSpill = currentMemory >= myMemoryThreshold
    }
    shouldSpill = shouldSpill || _elementsRead > numElementsForceSpillThreshold
    // Actually spill
    if (shouldSpill) {
      _spillCount += 1
      logSpillage(currentMemory)
      spill(collection)
      _elementsRead = 0
      _memoryBytesSpilled += currentMemory
      releaseMemory()
    }
    shouldSpill
  }
```

该方法首先判断了是否需要 spill，如果需要则调用 spill() 方法将内存中的数据写入临时文件中。spill 方法如下:

```scala
 override protected[this] def spill(collection: WritablePartitionedPairCollection[K, C]): Unit = {
    val inMemoryIterator = collection.destructiveSortedWritablePartitionedIterator(comparator)
    val spillFile = spillMemoryIteratorToDisk(inMemoryIterator)
    spills += spillFile
  }
```

其中的  comparator 如下:

```scala
 private def comparator: Option[Comparator[K]] = {
    if (ordering.isDefined || aggregator.isDefined) {
      Some(keyComparator)
    } else {
      None
    }
  }
```

也就是:

```scala
  private val keyComparator: Comparator[K] = ordering.getOrElse(new Comparator[K] {
    override def compare(a: K, b: K): Int = {
      val h1 = if (a == null) 0 else a.hashCode()
      val h2 = if (b == null) 0 else b.hashCode()
      if (h1 < h2) -1 else if (h1 == h2) 0 else 1
    }
  })
```

destructiveSortedWritablePartitionedIterator 位于 WritablePartitionedPairCollection 类中，实现如下:

```scala
def destructiveSortedWritablePartitionedIterator(keyComparator: Option[Comparator[K]])
    : WritablePartitionedIterator = {
    //这里调用的其实是　PartitionedAppendOnlyMap　中重写的　partitionedDestructiveSortedIterator    
    val it = partitionedDestructiveSortedIterator(keyComparator)
    // 这里还实现了一个　writeNext　的方法，后面会用到
    new WritablePartitionedIterator {
      private[this] var cur = if (it.hasNext) it.next() else null
       // 在map 中的数据其实是((partition,k),v)
       // 这里只写入了(k,v)
      def writeNext(writer: DiskBlockObjectWriter): Unit = {
        writer.write(cur._1._2, cur._2)
        cur = if (it.hasNext) it.next() else null
      }
      def hasNext(): Boolean = cur != null
      def nextPartition(): Int = cur._1._1
    }
  }
```

接下来看看partitionedDestructiveSortedIterator:

```scala
def partitionedDestructiveSortedIterator(keyComparator: Option[Comparator[K]])
    : Iterator[((Int, K), V)] = {
    val comparator = keyComparator.map(partitionKeyComparator).getOrElse(partitionComparator)
    destructiveSortedIterator(comparator)
 }
```

keyComparator.map(partitionKeyComparator)　等于 partitionKeyComparator(keyComparator)，再看下面的partitionKeyComparator，说明比较器是先对分区进行比较，如果分区相同，再对key的hash值进行比较:

```scala
 def partitionKeyComparator[K](keyComparator: Comparator[K]): Comparator[(Int, K)] = {
    new Comparator[(Int, K)] {
      override def compare(a: (Int, K), b: (Int, K)): Int = {
        val partitionDiff = a._1 - b._1
        if (partitionDiff != 0) {
          partitionDiff
        } else {
          keyComparator.compare(a._2, b._2)
        }
      }
    }
  }
```

然后看destructiveSortedIterator(),这是在 PartitionedAppendOnlyMap 的父类 destructiveSortedIterator 中实现的:

```scala
 def destructiveSortedIterator(keyComparator: Comparator[K]): Iterator[(K, V)] = {
    destroyed = true
    // Pack KV pairs into the front of the underlying array
    // 这里是因为　PartitionedAppendOnlyMap　采用的二次探测法　,kv对之间存在　null值，所以先把非空的kv对全部移动到hash表的前端
    var keyIndex, newIndex = 0
    while (keyIndex < capacity) {
      if (data(2 * keyIndex) != null) {
        data(2 * newIndex) = data(2 * keyIndex)
        data(2 * newIndex + 1) = data(2 * keyIndex + 1)
        newIndex += 1
      }
      keyIndex += 1
    }
    assert(curSize == newIndex + (if (haveNullValue) 1 else 0))
    // 这里对给定数据范围为 0 to newIndex，利用　keyComparator　比较器进行排序
    // 也就是对 map 中的数据根据　(partition,key) 进行排序
    new Sorter(new KVArraySortDataFormat[K, AnyRef]).sort(data, 0, newIndex, keyComparator)

    // 定义迭代器
    new Iterator[(K, V)] {
      var i = 0
      var nullValueReady = haveNullValue
      def hasNext: Boolean = (i < newIndex || nullValueReady)
      def next(): (K, V) = {
        if (nullValueReady) {
          nullValueReady = false
          (null.asInstanceOf[K], nullValue)
        } else {
          val item = (data(2 * i).asInstanceOf[K], data(2 * i + 1).asInstanceOf[V])
          i += 1
          item
        }
      }
    }
  }
```

可以看出整个　`val inMemoryIterator = collection.destructiveSortedWritablePartitionedIterator(comparator)`　这一步其实就是在对　map　或 buffer 进行排序，并且实现了一个 writeNext() 方法供后续调调用。随后调用`val spillFile = spillMemoryIteratorToDisk(inMemoryIterator)`将排序后的缓存写入文件中:

```scala
private[this] def spillMemoryIteratorToDisk(inMemoryIterator: WritablePartitionedIterator)
      : SpilledFile = {
   
    val (blockId, file) = diskBlockManager.createTempShuffleBlock()
    // These variables are reset after each flush
    var objectsWritten: Long = 0
    val spillMetrics: ShuffleWriteMetrics = new ShuffleWriteMetrics
    val writer: DiskBlockObjectWriter =
      blockManager.getDiskWriter(blockId, file, serInstance, fileBufferSize, spillMetrics)
    // List of batch sizes (bytes) in the order they are written to disk
    val batchSizes = new ArrayBuffer[Long]
    // How many elements we have in each partition
    // 用于记录每一个分区有多少条数据
    // 由于刚才数据写入文件没有写入key对应的分区，因此需要记录每一个分区有多少条数据，然后根据数据的偏移量就可以判断该数据属于哪个分区　
    val elementsPerPartition = new Array[Long](numPartitions)
    // Flush the disk writer's contents to disk, and update relevant variables.
    // The writer is committed at the end of this process.
    def flush(): Unit = {
      val segment = writer.commitAndGet()
      batchSizes += segment.length
      _diskBytesSpilled += segment.length
      objectsWritten = 0
    }
    var success = false
    try {
      while (inMemoryIterator.hasNext) {
        val partitionId = inMemoryIterator.nextPartition()
        require(partitionId >= 0 && partitionId < numPartitions,
          s"partition Id: ${partitionId} should be in the range [0, ${numPartitions})")
        inMemoryIterator.writeNext(writer)
        elementsPerPartition(partitionId) += 1
        objectsWritten += 1
        if (objectsWritten == serializerBatchSize) {　//写入　serializerBatchSize　条数据便刷新一次缓存
          // batchSize 在类中定义的如下:
          // 可以看出如果不存在配置默认为　10000　条
          // private val serializerBatchSize = conf.getLong("spark.shuffle.spill.batchSize", 10000)
          flush()
        }
      }
      if (objectsWritten > 0) {
        flush()
      } else {
        writer.revertPartialWritesAndClose()
      }
      success = true
    } finally {
      if (success) {
        writer.close()
      } else {
        // This code path only happens if an exception was thrown above before we set success;
        // close our stuff and let the exception be thrown further
        writer.revertPartialWritesAndClose()
        if (file.exists()) {
          if (!file.delete()) {
            logWarning(s"Error deleting ${file}")
          }
        }
      }
    }
    //最后记录临时文件的信息
    SpilledFile(file, blockId, batchSizes.toArray, elementsPerPartition)

  }
```

这就是整个　insertAll()　方法:

 merge 达到溢出值后进行排序并写入临时文件，这里要注意的是并非所有的缓存文件都被写入到临时文件中，所以接下来需要将临时文件中的数据加上内存中的数据进行一次排序写入到一个大文件中。

```scala
//首先获取需要写入的文件:　
   val output = shuffleBlockResolver.getDataFile(dep.shuffleId, mapId)
　　val tmp = Utils.tempFileWith(output)
    try { 
      val blockId = ShuffleBlockId(dep.shuffleId, mapId, IndexShuffleBlockResolver.NOOP_REDUCE_ID)
      val partitionLengths = sorter.writePartitionedFile(blockId, tmp)//这一句是重点 下面会讲解
      shuffleBlockResolver.writeIndexFileAndCommit(dep.shuffleId, mapId, partitionLengths, tmp)
      mapStatus = MapStatus(blockManager.shuffleServerId, partitionLengths)
    } finally {
      if (tmp.exists() && !tmp.delete()) {
        logError(s"Error while deleting temp file ${tmp.getAbsolutePath}")
    }
```
####　writePartitionedFile

继续看 writePartitionedFile :

```scala
//  ExternalSorter 中的　writePartitionedFile　方法
   def writePartitionedFile(
      blockId: BlockId,
      outputFile: File): Array[Long] = {

    // Track location of each range in the output file
    val lengths = new Array[Long](numPartitions)
    val writer = blockManager.getDiskWriter(blockId, outputFile, serInstance, fileBufferSize,
      context.taskMetrics().shuffleWriteMetrics)

    // 这里的spills 就是　刚才产生的临时文件的数据　如果为空逻辑就比较简单，直接进行排序并写入文件
    if (spills.isEmpty) {
      // Case where we only have in-memory data
      val collection = if (aggregator.isDefined) map else buffer
      val it = collection.destructiveSortedWritablePartitionedIterator(comparator)　//这里同样使用　destructiveSortedWritablePartitionedIterator　进行排序
      while (it.hasNext) {
        val partitionId = it.nextPartition()
        while (it.hasNext && it.nextPartition() == partitionId) {
          it.writeNext(writer)
        }
        val segment = writer.commitAndGet()
        lengths(partitionId) = segment.length
      }
    } else {//重点看不为空的时候，这里调用了　partitionedIterator　方法
      // We must perform merge-sort; get an iterator by partition and write everything directly.
      for ((id, elements) <- this.partitionedIterator) {
        if (elements.hasNext) {
          for (elem <- elements) {
            writer.write(elem._1, elem._2)
          }
          val segment = writer.commitAndGet()
          lengths(id) = segment.length
        }
      }
    }

    writer.close()
    context.taskMetrics().incMemoryBytesSpilled(memoryBytesSpilled)
    context.taskMetrics().incDiskBytesSpilled(diskBytesSpilled)
    context.taskMetrics().incPeakExecutionMemory(peakMemoryUsedBytes)

    lengths
  }
```

然后是 partitionedIterator 方法:

```scala
def partitionedIterator: Iterator[(Int, Iterator[Product2[K, C]])] = {
    val usingMap = aggregator.isDefined
    val collection: WritablePartitionedPairCollection[K, C] = if (usingMap) map else buffer
    if (spills.isEmpty) {// 这里又一次判断了是否为空，直接看有临时文件的部分
      // Special case: if we have only in-memory data, we don't need to merge streams, and perhaps
      // we don't even need to sort by anything other than partition ID
      if (!ordering.isDefined) {
        // The user hasn't requested sorted keys, so only sort by partition ID, not key
   groupByPartition(destructiveIterator(collection.partitionedDestructiveSortedIterator(None)))
      } else {
        // We do need to sort by both partition ID and key
        groupByPartition(destructiveIterator(
          collection.partitionedDestructiveSortedIterator(Some(keyComparator))))
      }
    } else {
      // Merge spilled and in-memory data
      // 这里传入了临时文件　spills　和　排序后的缓存文件
      merge(spills, destructiveIterator(
        collection.partitionedDestructiveSortedIterator(comparator)))
    }
  }
```

merge 方法:

```scala
 // merge方法
   // inMemory　是根据(partion,hash(k)) 排序后的内存数据
   private def merge(spills: Seq[SpilledFile], inMemory: Iterator[((Int, K), C)])
      : Iterator[(Int, Iterator[Product2[K, C]])] = {
    //　将所有缓存文件转化为 SpillReader 
    val readers = spills.map(new SpillReader(_))
    // buffered方法只是比普通的　itertor 方法多提供一个　head　方法，例如:
    // val a = List(1,2,3,4,5)
    // val b = a.iterator.buffered
    // b.head : 1
    // b.next : 1
    // b.head : 2
    // b.next : 2
    val inMemBuffered = inMemory.buffered
    (0 until numPartitions).iterator.map { p =>
      // 获得分区对应数据的迭代器 
      // 这里的　IteratorForPartition　就是得到分区 p 对应的数据的迭代器，逻辑比较简单，就不单独拿出来分析
      val inMemIterator = new IteratorForPartition(p, inMemBuffered)
      
      //这里获取所有文件的第　p　个分区　并与内存中的第　p 个分区进行合并
      val iterators = readers.map(_.readNextPartition()) ++ Seq(inMemIterator)
　　　// 这里的　iterators　是由　多个 iterator 组成的，其中每一个都是关于 p 分区数据的 iterator
      if (aggregator.isDefined) {
        // Perform partial aggregation across partitions
        (p, mergeWithAggregation(
          iterators, aggregator.get.mergeCombiners, keyComparator, ordering.isDefined))
      } else if (ordering.isDefined) {
        // No aggregator given, but we have an ordering (e.g. used by reduce tasks in sortByKey);
        // sort the elements without trying to merge them
        (p, mergeSort(iterators, ordering.get))
      } else {
        (p, iterators.iterator.flatten)
      }
    }
```
mergeWithAggregation
```scala

      private def mergeWithAggregation(
      iterators: Seq[Iterator[Product2[K, C]]],
      mergeCombiners: (C, C) => C,
      comparator: Comparator[K],
      totalOrder: Boolean)
      : Iterator[Product2[K, C]] =
  {
    if (!totalOrder) {
      new Iterator[Iterator[Product2[K, C]]] {
        val sorted = mergeSort(iterators, comparator).buffered
        val keys = new ArrayBuffer[K]
        val combiners = new ArrayBuffer[C]
        override def hasNext: Boolean = sorted.hasNext
        override def next(): Iterator[Product2[K, C]] = {
          if (!hasNext) {
            throw new NoSuchElementException
          }
          keys.clear()
          combiners.clear()
          val firstPair = sorted.next()
          keys += firstPair._1
          combiners += firstPair._2
          val key = firstPair._1
          while (sorted.hasNext && comparator.compare(sorted.head._1, key) == 0) {
            val pair = sorted.next()
            var i = 0
            var foundKey = false
            while (i < keys.size && !foundKey) {
              if (keys(i) == pair._1) {
                combiners(i) = mergeCombiners(combiners(i), pair._2)
                foundKey = true
              }
              i += 1
            }
            if (!foundKey) {
              keys += pair._1
              combiners += pair._2
            }
          }
          keys.iterator.zip(combiners.iterator)
        }
      }.flatMap(i => i)
    } else {
      // We have a total ordering, so the objects with the same key are sequential.
      new Iterator[Product2[K, C]] {
        val sorted = mergeSort(iterators, comparator).buffered
        override def hasNext: Boolean = sorted.hasNext
        override def next(): Product2[K, C] = {
          if (!hasNext) {
            throw new NoSuchElementException
          }
          val elem = sorted.next()
          val k = elem._1
          var c = elem._2
          // 这里的意思是可能存在多个相同的key 分布在不同临时文件或内存中
          // 所以还需要将不同的 key 对应的值进行合并 
          while (sorted.hasNext && sorted.head._1 == k) {
            val pair = sorted.next()
            c = mergeCombiners(c, pair._2)
          }
          (k, c)//返回
        }
      }
    }
  }
```

继续看 mergeSort 方法，首先选除头元素最小对应那个分区，然后选出这个分区的第一个元素，那么这个元素一定是最小的，然后将剩余的元素放回去:

```scala

  private def mergeSort(iterators: Seq[Iterator[Product2[K, C]]], comparator: Comparator[K])
      : Iterator[Product2[K, C]] =
  {
    val bufferedIters = iterators.filter(_.hasNext).map(_.buffered)
    type Iter = BufferedIterator[Product2[K, C]]
    //选取头元素最小的分区
    val heap = new mutable.PriorityQueue[Iter]()(new Ordering[Iter] {
      // Use the reverse of comparator.compare because PriorityQueue dequeues the max
      override def compare(x: Iter, y: Iter): Int = -comparator.compare(x.head._1, y.head._1)
    })
    heap.enqueue(bufferedIters: _*) 
    new Iterator[Product2[K, C]] {
      override def hasNext: Boolean = !heap.isEmpty
      override def next(): Product2[K, C] = {
        if (!hasNext) {
          throw new NoSuchElementException
        }
        val firstBuf = heap.dequeue()
        val firstPair = firstBuf.next()
        if (firstBuf.hasNext) {
          heap.enqueue(firstBuf)
        }
        firstPair
      }
    }
  }
```

接下来就使用 writeIndexFileAndCommit 将每一个分区的数据条数写入索引文件，便于 reduce 端获取。

## 总结

shuflle write 相比起 shuffe read 要复杂很多，因此本文的篇幅也较长。其实在 shuffle write 有两个较为重要的方法，其中一个是 insertAll。这个方法的作用就是将数据缓存到一个可以添加的 hash 表中。如果表占用的内存达到的一定值，就对其进行排序。排序方式先按照分区进行排序，如果分区相同则按key的hash进行排序，随后将排序后的数据写入临时文件中，这个过程可能会生成多个临时文件。

然后 writePartitionedFile 的作用是最后将内存中的数据和临时文件中的数据进行全部排序，写入到一个大文件中。

最后将每一个分区对应的索引数据写入文件。整个shuffle write 阶段就完成了
