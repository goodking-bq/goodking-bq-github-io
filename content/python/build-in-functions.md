---
title: "python 常用的内置函数"
date: 2020-05-14T15:43:56+08:00
draft: false
tags: ["python","reference"]
categories: ["python"]
---

# map 
> 对可迭代函数'iterable'中的每一个元素应用‘function’方法，将结果作为list返回。

例:
```
>>> map(lambda x:x*2,range(10))
[0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
>>> map(lambda x,y: (y + x) * 2, range(10),range(10))
[0, 4, 8, 12, 16, 20, 24, 28, 32, 36]
``` 
# filter
> **filter(function or None, sequence)** 对指定序列执行过滤操作。用sequence里的每个元素去调用function，最后结果保护调用结果为True的元素,如果function参数为None，返回结果和sequence参数相同。

例：
```
>>> filter(lambda x: x % 2 == 0, range(10))
[0, 2, 4, 6, 8]
```

# reduce
> **reduce(function, sequence[, initial])** 对参数序列中元素进行累积。function 必须要有两个参数。依次从sequence取一个元素，和上次调用的结果作为参数调用function。如果给了initial 参数，那么第一次调用就用initial 和sequence第一个元素作为参数。没有给就从sequence中去两个作为参数。

例：
```
>>> reduce(lambda x,y: x * y,range(1,10))
362880
```
# zip
> **zip([iterable,...])** 接受一系列可迭代的对象作为参数，将对象中对应的元素打包成一个个tuple（元组），然后返回由这些tuples组成的list（列表）。若传入参数的长度不等，则返回list的长度和参数中长度最短的对象相同。利用*号操作符，可以将list unzip（解压）。

例：
```
>>> a = ['a', 'b', 'c']
>>> b = [1, 2, 3, 4]
>>> c = ['~', '!', '@', '#', '$']
>>> zip(a, b, c)
[('a', 1, '~'), ('b', 2, '!'), ('c', 3, '@')]
>>> zip(*zip(a, b, c))
[('a', 'b', 'c'), (1, 2, 3), ('~', '!', '@')]
```