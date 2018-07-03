---
title: 异常检测-Mass estimation与孤立森林
date: 2018-07-03 14:17:08
tags:
  - 数据挖掘
  - 机器学习
  - 异常检测
categories: 异常检测
author: lishion
toc: true

---

Mass Estimation 是一直种全新的评估数据分布的方法。与之前讲到过的孤立森林有很大的联系。

## Mass Estimation

在论文 *Mass Estimation and Its Applications* 一开始提到了现有的许多机器学习的方法都是基于**密度**估计的。这里的密度应该是指的**概率密度**，不知道与实际的数据密度有没有联系。看一下其中的一段话:

>Density estimation has been the base modelling mechanism used in many techniques designed for tasks such as classiflcation, clustering, anomaly detection and information retrieval. For example in classiflcation, density estimation is employed to estimate class-conditional density function (or likelihood function) p(xjj) or posterior probability  p(jjx)|the principal function underlying many classiflcation methods e.g., mixture models, Bayesian networks, and Naive Bayes. Examples of density estimation include 。
>
>kernel density estimation, k-nearest neighbours density estimation, maximum likelihood procedures or Bayesian methods

文中给出的几种方法都是基于概率密度的，而不是实际空间上的密度。有趣的是，Mass Estimation 虽然翻译过来应该是质量估计，但是实际它的性质与密度很类似。Mass Estimation 实际上是要先求得一个 mass distribution 。论文给出对质量分布描述如下:

>A mass distribution stipulates an ordering from core points to fringe points in a data cloud. In addition, this ordering accentuates the fringe points with a concave function|fringe points have markedly smaller mass than points close to the core points。

大致意思是质量分布给出了从数据中心到数据边缘的顺序。也就是对于处于中心的数据，质量分布函数的值越大，处于数据边缘的数据，质量分布函数值越小。这里和概率分布可以做一个对比：概率分布是出现次数多的数据，分布函数的值越大，反之越小。可见质量分布更注重数据分布的实际空间。这点对于异常检测非常重要，因为异常往往就是在实际空间中处于边缘部分的数据。

最开始的论文 *Mass Estimation and Its Applications* 只给出了在一维空间中 mass distribution 的定义。而后来 *Multi-Dimensional Mass Estimation and Mass-based Clustering* 以及 *Mass estimation* 给出了从一维空间推广到多维空间的方法，并且给出了利用**二分空间树(Half-Space Tree)** 构造  mass distribution 的方法。也就是利用二叉树将数据空间分为$2^h$个，这个和之前讲过的决策树和孤立树都差不多。其中从一维推广到多维的过程比较难理解，这里直接给出在多维空间中的定义:

设 $x$ 为 $R^d$ 空间中的一个点。$T^h(x)$表示$x$所在的空间。$T^h(.)$表示被划分的数据为$D$。$T^h(.|D_S)$表示被划分的数据为$D_s \in D$。$m$为落在$T^h(x)$或$T^h(.|D_S)$中训练样本的个数。因此质量函数(mass base function)可以定义为:
$$
m(T^h(x)) = m 如果 x 落在的空间有m个训练样本 否则 0
$$
而对数据进行抽样得到多个子集后的平均质量为:
$$
\overline{mass(x,h)} ≈\frac 1 c \sum_{k=1}^{c}m(T^h(x|D_{sk}))
$$
而最终，使用二分空间树求得的测试样本的 (augmented) mass 为:
$$
s(x) = m[l] * 2^l
$$
这里$m[l] $指的是样本落在 $l$ 层的叶子节点含有的训练样本数。而二分空间树在构造的时候叶子节点只有一个元素(不限制生长高度的情况)，因此最中的到:
$$
s(x) \approx 2^l \backsim l
$$
可以看出，最终求得的值正比于孤立森林的层数。论文中也提到了孤立森林的本质就是一种 mess estimation。因此这两种方法进行异常检测的性能相当。唯一不同，是二分空间树在构造的时候书中选择$\frac {q_{max}-q_{min}} {2}$来作为划分点 $q$ 表示用来划分的特征，而孤立森林随机选择数据作为划分点。

## 参考

mess estimation 目前还没有大规模应用的趋势，对其进行研究的论文也不多。因此我也是看得云里雾里，所以这篇文章也就是大概对比一下 mess estimation 和孤立森林的关系。如果想详细研究，可以参照:

>[1] Kai M T, Zhou G T, Liu F T, et al. Mass estimation and its applications[C]// ACM SIGKDD International Conference on Knowledge Discovery and Data Mining. ACM, 2010:989-998.
>
>[2] Kai M T, Wells J R. Multi-dimensional Mass Estimation and Mass-based Clustering[J]. 2010:511-520.
>
>[3] Kai M T, Zhou G T, Liu F T, et al. Mass estimation[J]. Machine Learning, 2013, 90(1):127-160.

其中 [1] 最开始提出了一维空间的 mass estimation。[2] 对其进行了推广到了多维空间。[3] 在 [1]的基础上增加了一些论文，并对比了其与孤立森林的关系。 