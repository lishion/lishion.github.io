---
title: Python type-hints
date: 2018-07-24 09:41:31
tags:
  - python
  - 编程
categories: python笔记
author: lishion
toc: true

---

## 基本使用

昨天刷知乎的时候偶然刷到了 Python3.5 引入的新特性 [typing](https://docs.python.org/3/library/typing.html) 可以增强 IDE 或则编辑器对类型的推断。由于 python 本身是一种动态类型语言，因此在某些不知道变量类型的时候编辑器无法给出提示，下面涉及到的编辑器都为vscode。例如:

```python
def trimString(string):
    return string.strip("\n")
```

由于不知道参数 string 的类型，在按下 string. 的时候 vscode 无法给出关于 string 的方法提示。但是如果采用 typing 的写法:

```python
def tringString(string:str)->str:
	return string.strip("\n")
```

这样就能在输入 string. 的时候提示出 str 类型的方法。

## type

对于 Python 自带的类型，python 都放在 typing 包里可以引入，例如:

```python
from typing import List,Dict
a : Dict = {"python-v":"3.5"}
b : List = ["python3","python2"] 
```

更多的类型请参照官网给出的例子。

## 泛型

当然有了类型之后就要有泛型，当然我不知道这个是否应该称之为泛型，因为实际上type hints 故名思意只是*为了 hint ，而不提供类型检查*。可以通过 TypeVar 实现类似泛型的概念:

```python
from typing import TypeVar,List
T = TypeVar("T")
def match(list:List[T],item:T) -> int:
    for i in enumerate(list):
        if i == item:
            return i
    return -1
```

## DataClass

在 scala 中提供了 case class 来满足我们只想用来保存数据，而不需要方法的类。而最近在 pyhon 3.7 中也增加了 `@DataClass` 的装饰器来支持。但是在 Python 3.7之前也可以通过 collections.namedtuple  来实现曲线救国。而还有一种方法就是使用 typing.NamedTuple。例如:

```python
from typing import NamedTuple
class Employee(NamedTuple):
    name : str
    age : int = 15 # 含有默认值的属性必须位于不含有特征至的属性的后面
    
employee = Employee("bob",12)
```

等价于:

```
Employee = collections.namedtuple('Employee', ['name', 'id'])
```

## 官方文档

这里只给出了一些常用的方式，更多的请参考官方文档:

> https://docs.python.org/3/library/typing.html

