---
title: Python笔记
date: 2018-07-23 16:32:56
tags:
  - python
  - 编程
categories: python笔记
author: lishion
toc: true
---

# slice-对切片进行命名

slice 可以用来对切片命名，例如:

```python
a = ['a','b','c','d']
indexes = slice(1,3)
print(a[slice])
#['b', 'c']
print(indexes.start) #1
print(indexes.end) #3
```

使用 slice 便可以将切片当成变量传递，某些时候很方便。

# itertools.compress - 构建非连续切片

使用itertools.compress()可以选择可迭代对象中的任意元素，例如:

```python
 from itertools import compress
 list(compress([1,2,3],[True,False,True]))
 #[1,3]
```

利用这个特性可以构建不连续下标切片:

```python
from itertools import compress
def getWithIndexes(_list,indexes):
    compressIndexes = [index in indexes for index in range(0,len(_list))]
    return list(compress(_list,compressIndexes))
a = ["a","b","c","d","e","f","g"]
indexes = [1,3,5]
print(getWithIndexes(a,indexes)) #['b', 'd', 'f']
```

# collections.namedtuple - 给元组命名 

在很多时候，我们会返回一个元祖来表示结构信息，例如我们要描述一个学生:

``` python
student = ("Bob",23,"man")
name = student[0]
age = student[1]
gneder = student[2]
```

然而这样的使用方式使得访问元素的值依赖与下标，并且使得代码可读性变差。这种情况可以使用 collections.namedtuple 来改善:

``` python
from collections import namedtuple
Student = namedtuple('Subscriber',['name','age','gender'])
student = Student('Bob',25,'man')
print(student.name,student.age,student.gender)
# Ｂob 23 man
```

此时访问元祖中的元素就无须通过下标访问，提高了代码易读性。

# ljust,rjust,center - 对齐字符串

在某些时候，我们希望我们的文本信息输出的更加美观，例如:

```python
print(*('alice',20,'woman'))
print(*('bob',20,'man'))
print(*('john',20,'man'))
print(*('lily',20,'woman'))
alice 20 woman
bob 20 man
john 20 man
lily 20 woman
```

其中由于名字性比长短不一导致输出不是很美观，这时候可以通过这几种方式调整输出:

```python
def format(strs):
    return [str(item).ljust(8) for item in strs]

print(*format(('alice',20,'woman')))
print(*format(('bob',20,'man')))
print(*format(('john',20,'man')))
print(*format(('lily',20,'woman')))

alice    20       woman
bob      20       man
john     20       man
lily     20       woman
```

