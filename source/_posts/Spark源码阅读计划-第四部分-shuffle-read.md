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

## 从 shuffle write 完成开始说起