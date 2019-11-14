---
typora-root-url: ../
layout:     post
title:      Python 中的函数式编程
subtitle:   Python 中一切好的特性都是从其他语言中借鉴来的
date:       2018-02-10 14:00 +0800
author:     gsfish
header-img: img/post-bg-python.jpg
tags:
 - Python
---


从设计上看，不管函数式语言的定义如何，Python 都不是一门函数式语言。Python 只是从函数式语言中借鉴了一些好的想法。

以下为 Python 中涉及函数式编程的主要工具：

* `map`
* `reduce`
* `filter`
* `lambda`

## `map(func, seq)`

`map(func, seq)` （映射）函数接收两个参数，一个是函数，一个是序列，`map` 将传入的函数依次作用到序列的每个元素，并把结果作为新的 `list` 返回。

在 Python 3 中 `map` 函数直接返回一个生成器，需显式地使用 `list` 的构造方法创建列表。

使用示例：

```python
>>> def f(x):
...     return x * x
...
>>> list(map(f, [1, 2, 3, 4, 5, 6, 7, 8, 9]))
[1, 4, 9, 16, 25, 36, 49, 64, 81]
```

## `reduce(func, seq)`

`reduce` （归约）把一个函数作用在一个序列 `[x1, x2, x3...]` 上，这个函数必须接收两个参数，`reduce` 把结果继续和序列的下一个元素做累积计算。

使用示例：

```python
>>> from functools import reduce
>>>
>>> def add(x, y):
...     return x + y
...
>>> reduce(add, [1, 3, 5, 7, 9])
25
```

## `filter(func, seq)`

`filter` 用于过滤序列。`filter` 也接收一个函数和一个序列，其将传入的函数依次作用于每个元素，然后根据返回值是 `True` 还是 `False` 决定保留还是丢弃该元素。

在 Python 3 中 `filter` 函数直接返回一个生成器，需显式地使用 `list` 的构造方法创建列表。

使用示例：

```python
>>> def is_odd(n):
...     return n % 2 == 1
... 
>>> list(filter(is_odd, [1, 2, 4, 5, 6, 9, 10, 15]))
[1, 5, 9, 15]
```

## `lambda`

`lambda` 即 Python 中的匿名函数，`:` 左边为参数，右边为返回值的表达式。

`lambda`、`map`、`filter`、`reduce` 首次出现在 Lisp 中，而 Lisp 没有限制在 `lambda` 中能做什么，因为 Lisp 中的一切都是表达式。然而，Python 使用的是面向语句的句法，表达式中不能包含语句，因此限制了 `lambda` 函数的定义体只能使用纯表达式，即 `lambda` 定义体中不能赋值，也不能使用 `while` 和 `try` 等语句。

使用示例：

```python
>>> list(map(lambda x: x * x, [1, 2, 3, 4, 5, 6, 7, 8, 9]))
[1, 4, 9, 16, 25, 36, 49, 64, 81]
```

## 尾递归

尾调用是指一个函数里的最后一个动作是一个函数调用的情形：即这个调用的返回值直接被当前函数返回的情形。若这个函数在尾位置调用本身（或是一个尾调用本身的其他函数等等），则称这种情况为尾递归，是递归的一种特殊情形。

由于一般的递归在每一次进入函数调用时都会创建一层栈帧，当递归的深度很大时，这些用来保存环境的栈帧会占据大量的空间。而采用尾递归的方式，通过优化可以复用同一栈帧，从而达到节省空间的目的。

## 尾递归消除

Guido 在[一篇博客](https://neopythonic.blogspot.com/2009/04/tail-recursion-elimination.html)中提到了为什么 Python 不使用尾递归消除的方式进行优化。主要原因有三点：

1. 尾递归消除（Tail Recursion Elimination, TRE）不便于堆栈跟踪。在递归的过程中原有的栈帧会被覆盖，增加了 Debug 的难度
2. TRE 不仅是一种优化方式，还会影响开发者所编写的代码。若解释器未实现该特性则代码可能无法运行
3. Guido 认为递归并非基本的编程工具，它只是一个很好的理论方法

博客中还对一些现有的比较 Hack 的 TRE 优化方式进行了分析与反驳，十分有趣。


# 参考资料

1. http://www.ruanyifeng.com/blog/2015/04/tail-call.html
2. https://python-history.blogspot.com/2009/04/origins-of-pythons-functional-features.html
3. https://neopythonic.blogspot.com/2009/04/tail-recursion-elimination.html
4. https://www.amazon.cn/dp/B072HMKKPG
