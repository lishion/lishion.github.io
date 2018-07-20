---
title: scala集合类总结
date: 2018-07-20 11:11:04
tags:
  - scala
  - 编程
categories: scala笔记
author: lishion
toc: true
---

scala的集合类主要分为Seq、Set、Map这三种类型。其中 Set 与 Map与Java相似。而Seq的继承体系比较复杂，因此这里主要讲解Seq。

# Seq

Seq 是scala中三个集合类抽象借口之一。Seq 表示一类具有一定长度的可迭代访问的对象，其中每个元素均带有一个从0开始计数的固定索引位置，这表示Seq中的元素是**有序**的。

## 按照子类型分类

Seq 又分为两个子 Trait，分别为 LinearSeq 和 IndexedSeq:

```
Seq 
 |_LinearSeq       #线性序列  
 |   |_List
 |   |_ListBuffer
 |   |_Stream
 |   |_Stack
 |   |_Priority Queque
 |   |_Queue
 |   |_LinkedList
 |   |_Double LinkedList
 |
 |_IndexedSeq      #索引序列
     |_Array 
     |_Vector
     |_ArrayBuffer 
     |_Range
```

其中 Seq 又含有两个子 triat 分别是 LinearSeq和IndexSeq。

其中 LinearSeq提供了性能较高的head和tail操作。例如:

```scala
val a = List(1,2,3,4)
a.head
//res0: Int = 1
a.tail
//res1: List[Int] = List(2, 3, 4)
```

可以看到 List 的 tail 还是一个 List。

而 IndexSeq 提供了更高效的随机访问的能力。

默认 scala 的集合是不可变的，为了使得集合可变，可以使用 Buffer。List 版本的Buffer 为 ListBuffer，Array 版本的 Buffer 为 ArrayBuffer。

## 按可变与不可变分类:

Seq 按照不可变与可变可以进行以下分类:

```
Seq 
 |_mutable       
 |  |_LinearSeq
 |	|  |_Stack
 |	|  |_Queque
 |	|  |_Priority Queque
 |	|  |_LinkedList
 |	|  |_Double LinkedList
 | 	|  |_ListBuffer
 |	|
 |	|_IndexedSeq
 |     |_ArrayBuffer
 |
 |_immutable 
    |_LinearSeq
    |  |_List
    |  |_Stream
    |  |_Stack
    |  |_Quequ
    |
    |_IndexedSeq
       |_Vector
       |_Array
       |_Range
```

## List Array Vector

接下来 比较一下这三种比较常用的不可变类型的特点:

* List : 常数时间访问 head 与 tail，线性时间访问元素
* Vector : 常数时间访问，修改任意位置元素
* Array : 主要是与java中的数组对应，常数时间访问

## ArrayBuffer ListBuffer

ArrayBuffer 来源与数组，与数组特性相似，因此在随即访问方面有优势。

ListBuffer 来源于 list ，结构为链表，因此在添加删除方面有优势。

如果有大量的随即访问操作，应该使用ArrayBuffer。如果有大量添加删除操作，应该使用ListBuffer。