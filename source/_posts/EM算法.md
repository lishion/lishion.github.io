---
title: EM算法
date: 2018-08-02 14:25:48
tags:
  - 机器学习
categories: 机器学习
author: lishion
toc: true
---
EM算法是一种用来求解含有**隐变量**的概率模型的**最大似然估计**的一种通用方法。

##  基本推导

设所有的观测变量联合起来记作$X$，将所有的隐含变量记作$Z$，所有参数为$\theta$，似然函数为:
$$
p(X|\theta)=\sum_zp(X,z|\theta)
$$
由于含有求和，这里直接利用求导的方式来解$\theta$是非常困难的。这里先引入一个潜变量的分布函数$q(z)$，上式两边取对数，可以改写为:
$$
\tag1 \ ln( \ p(X|\theta) \ )=   ln( \ \sum_z q(z) \frac{p(X,z|\theta)}{ q(z)} \ )
$$
这时候需要引入**琴生不等式**，具体的推导这里不是重点，大概的意思是对于一个凸函数，函数的期望小于期望的函数，也就是说:
$$
E(f(x)) \le f(E(x))
$$
那么对于公式(1):
$$
ln( \ \sum_z q(z) \frac{p(X,z|\theta)}{ q(z)} \ ) \ge q(z) \sum_z ln(  \  \frac{p(X,z|\theta)}{ q(z)}  \ )
$$

$$
\tag2 \ ln( \ p(X|\theta) \ ) \ge   \sum_zq(z) ln(  \  \frac{p(X,z|\theta)}{ q(z)}  \ )
$$

得到公式(2)，可以将:
$$
\sum_zq(z) ln(  \  \frac{p(X,z|\theta)}{ q(z)}  \ )
$$
看作是我们需要最大花的对数似然函数的一个下界，通过最大化这个下界来最大化最终的对数似然函数。当然，这种情况下界与最终似然函数的越接近越好，也就是说**如果下界能够等于最终的似然函数**，那么优化下界就相当于优化似然函数。而下界等于似然函数的时候也就是琴生不等式取等号的时候。那么根据琴生不等式取等号的条件:
$$
\frac{p(X,z|\theta)}{ q(z)} = c,c为常数
$$
以及$q(z)$本身为概率密度函数，满足:
$$
\sum_zq(z)=1
$$
可以得到:
$$
c\sum_zq(z)=\sum_zp(X,z|\theta)=p(X|\theta)=c
$$
又:
$$
q(z)=\frac{p(X,z|\theta)}{c}=\frac{p(X,z|\theta)}{p(X|\theta)}=p(z|X,\theta)
$$
带入(2)，这里就求出了最优下界为:
$$
\sum_zp(z|X,\theta) ln(  \  \frac{p(X,z|\theta)}{ p(z|X,\theta)}  \ )
$$
对应着EM算法中的`E-Step`。这里虽然对 z 进行了求和，消去了隐变量，但是下界仍然是一个关于$\theta$的函数，只不过此时我们的对数似然函数中已经不包含求和项了，也就可以利用求导的方式来估计参数$\theta$:
$$
\hat\theta= arg \ max_\theta \sum_zp(z|X,\theta) ln(  \  \frac{p(X,z|\theta)}{ p(z|X,\theta)}  \ )
$$
这就对应着EM算法中的`M-Step`。不断的迭代这两步骤，直到算法收敛。

未完。。。。









