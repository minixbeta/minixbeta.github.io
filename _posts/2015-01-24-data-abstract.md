---
layout: default
title: 数据抽象
comments: true
---

SICP 接下来讲述了数据抽象的思想。

## 数据抽象的目的
数据抽象的目的，是为了屏蔽下层的实现细节，让操作作用在抽象数据这一层，而不涉及抽象数据内部是怎么实现的。这样一来，可以在
操作抽象数据时，让表达更清晰，二来以后修改抽象数据的实现细节时，上层的应用不受影响。

## 数据抽象的本质 
这一部分一个比较有意思的思想是，抽象数据的本质是什么？

以前我们一直以为数据是数据，函数是函数，函数会操作数据。但是这一节提出一个观点，抽象数据的本质其实是：

* 多个函数
* 函数之前构成的限制

例如，pair，即 <a, b> 这种两个元素构成的数据类型。例如，我们说 x 是一个 pair 类型的数据，那么，我们实际上是在说什么呢？

我们其实是在说一种关系：

```
x = make-pair(a, b)
first(x) = a
second(x) = b
```

pair 这种数据类型，其实就是由 make-pair, first, second 这三个函数以及他们之间的关系限制来定义的。

我们看一下 pair 在 Scheme 中是如何通过过程来定义的就会对这一点有更深入的理解。

```
(define (make-pair x y)
  (define (dispatch m)
    (cond ((= m 0) x)
          ((= m 1) y)
          (else (error "Argument not 0 or 1: MAKE-PAIR" m))))
  dispatch)
  
(define (first z) (z 0))
(define (second z) (z 1))
```

我们可以看出，make-pair 实际上是一个高阶函数，返回了一个过程 dispatch。first, second 也是高阶函数，他们的参数就是个函数，
first, second 的函数体是对这个参数的调用。可以实际推断一下，就能明白这里面的原理。

这里的关键其实是 make-pair 返回的函数 dispatch 里已经记录下了 make-pair 的两个参数。之后的 first, second 是把这两个参数
提取出来。
