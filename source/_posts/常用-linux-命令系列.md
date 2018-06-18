---
title: 常用 Linux 命令系列
categories: 笔记
author: lishion
toc: true
date: 2018-06-09 10:42:45
tags: 
  - Linux
  - 编程
---

记录一下用到的 Linux 命令

## sort

sort 可以用于对文本进行排序，常用的参数有:

> * -t : 指定分隔符，默认为空格
> * -n : 按照数字规则排序
> * -k : 按空格分隔后的排序列数

例如:

```bash
# sort_test
1:2:3
3:2:1
2:3:4
5:6:11
sort -nk -t : sort_test # 按数字排序
3:2:1
1:2:3
2:3:4
5:6:11
sort -k -t : sort_test # 按首字母进行排序
3:2:1
5:6:11
1:2:3
2:3:4

```

