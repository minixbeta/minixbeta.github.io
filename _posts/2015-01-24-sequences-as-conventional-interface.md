---
layout: post
title: 将列表作为通用的操作接口
comments: true
category: 技术
---

以前我一直不太理解，为什么在 Python 和 Haskell 中有一些函数专门用来处理列表，比如 filter, map 什么的。在学习了 SICP
这一节后，我明白了这里面的道理。

Richard Waters(1979) 开发了一个自动分析 Fortran 程序的工具，发现在 Fortran 的科学计算包中，90% 的程序都可以
使用列表的 map, filter, accumulation 的组合来表示。

这说明对列表的操作是一种极为常见的计算模式，许多时候，我们可以通过把任务转化为对列表的操作，来对程序进行模块化，同时
表达更加清晰。
