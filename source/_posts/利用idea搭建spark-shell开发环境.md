---
title: 利用idea搭建spark-shell开发环境
date: 2018-06-22 14:54:33
tags:
  - 工具
  - 教程
  - 编程
  - Spark
categories: 工具
author: lishion
toc: true
---

之前一直使用 vscode 写的 spark-shell 脚本，然后上传到服务器跑。但是 vscode 对 scala 支持实在是太差了，连基本的自动补全和语法检查都不支持。后来实在是忍受不了了，决定使用 idea 搭建一个编写 spark-shell 脚本的环境。**注意搭建这个环境主要是为了自动补全和语法检测，并不是为了编写可以运行的代码**。当然你按照这个方法搭建只需要配置一下调试器应该就可以运行了。

## 用 maven 搭建 scala 运行环境

既然要写 spark-shell 脚本。事先肯定得搭建好 scala 的运行环境。至于像 JDK 这一类的依赖可以装也可以不装，可以使用 idea 自带的。scala项目可以使用 sbt 搭建也可以使用 maven 搭建。由于对 sbt 不是很熟悉，再加上 sbt 在国内慢如蜗牛，多疑还是选择 maven。

步骤如下:

1. 安装 scala 插件，建议直接下载离线版本，直接安装下载好的 zip 文件

2. 新建一个 maven 项目，稍后添加支持 scala

3. 新增 scala sdk：

   选择 `FIle -> setting -> Libraries -> 点击+ ->Scala SDK `

   应该会有多个选项，选择一个合适的版本就可以

到此如果可以新建 scala class 就说明安装成功了

##引入 spark 依赖

引入 spark 依赖其实很简单，只需在 pom.xml 中添加就可以 。一般来说只需要引入 spark-core 就行。如果还需要使用 spark sql 或则其他的依赖可以直接在[spark-maven](http://mvnrepository.com/artifact/org.apache.spark)中查找。

但是，例如像 sc spark 这种在 spark-shell 准备好的对象在我们搭建的环境中是不存在的。为了避免每次都手动生成这些对象，我们可以在根目录新建一个 Env 类。然后在类中准备好 sc spark 对象。然后写脚本的时候只需要引入这个类就可以。例如:

```scala
// Env.scala
import org.apache.spark.{SparkConf,SparkContext}
import org.apache.spark.sql.SparkSession
// 注意一定是 object
object Env {
    val sc = new SparkContext(new SparkConf().setMaster("123").setAppName("123"))
    val spark = new SparkSession()
}
```

然后需要写不同功能的脚本只需要新建一个包，然后引入:

```scala
import Env._
sc.xxx
spark.sql.xx
```

就可以获取到 sc 以及 spark 对象。