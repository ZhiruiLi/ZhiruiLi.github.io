---
layout: post
title: 计算的表示
excerpt: "计算不是计算机，计算也不是 1 + 2，它是一个更为本质的东西。通过用 lambda 演算来描述计算可以从另一个角度对其进行理解。"
date: 2016-07-08
modified: 2016-07-27
categories: articles
author: zrl
tags:
  - Functional Programming
  - Theory
  - Lisp
comments: true
share: true
---

# 伪哲学家

当提到「计算」这个词的时候，我们会想到什么，是想到「计算机」，或是「图灵机」，又或是操控计算机的「汇编语言」，还是说「1 + 1」这样的算式？这些都是计算，但它们都是计算的一种表示而非计算本身，计算本身是一个更加本质的东西，可以认为是一种柏拉图型相，或是理念，刚刚说到的东西都是对它的摹仿。

比如我们说到「4」的时候，我们在用「4」这个符号去摹仿「4」这个理念，这个理念可以用「4」来摹仿，也可以用「四」，也可以用「four」，具体是什么不重要，重要的是你不会走在路上突然见到一个「4」，而是会见到一个类似「4」的东西。那既然可以用这样一个来自阿拉伯的符号来摹仿数字，那是否有其他的方式来摹仿呢？更一般地说，是否有其他的计算表示方式，并以此来实现我们在汇编语言，C，Java，等语言中表示的计算呢？下面将介绍一个图灵完备的计算模型，称为 λ 演算（lambda calculus）[^lambda-paper]，该计算的表示由 Alonzo Church 在 20 世纪 30 年代发明，它可被称为是最小的通用程序设计语言。

[^lambda-paper]: Raul Rojas - A Tutorial Introduction to the Lambda Calculus

# λ 演算

λ 演算非常简练，而且相对于图灵机的计算模型来说非常优雅，其核心在于表达式（expression）。一个名字（name）又被称为变量（variable），是一个标识符（identifier），可以是任意的字母，如：a, b, c 等。而表达式的定义如下：

$$
\begin{array}{rcl}
\text{<expression>} & := & \text{<name> | <function> | <application>} \\
\text{<function>} & := & \lambda~\text{<name>.<expression>} \\
\text{<application>} & := & \text{<expression><expression>} \\
\end{array}
$$

至于变换规则则总共有三条，更加具体的描述可参考维基百科 [^lambda-wiki]：

α - conversion: 改变绑定变量的名称不影响函数本身；<br/>
β - reduction: 将函数应用于其参数；<br/>
η - conversion: 两个函数对于所有的参数得到的结果都一致，当且仅当它们是同一个函数。

[^lambda-wiki]: [λ 演算 - 维基百科](https://zh.wikipedia.org/wiki/%CE%9B%E6%BC%94%E7%AE%97)

本文后面的部分均使用 Scheme 语言来描述这些计算，在 Scheme 中，有非常类似 λ 演算中表达式的表示，例如一个函数 $$\lambda x.y$$ 将在 Scheme 中表示为 `(lambda (x) y)`，而将函数应用于参数 $$x~y$$ 将在 Scheme 中表示为 `(x y)`。最大的区别可能在于，在 λ 演算中，`(x)` 和 `x` 一样，而在 Scheme 中，前者会变成一个对函数 `x` 的调用，而后面则是 `x` 本身。

# 自然数的表示

在考虑如何表示数之前，先思考一下数是什么，前面已经说了，数是一种理念，我们在去摹仿这个理念的时候，一般是做两件事，一是定义一些基本运算，将数进行组合获取新的数，比如四则运算；二是通过和上下文结合，用来表达某个意思，比如三个苹果，走了一步等。这样看来，只要能抽象出这两点，具体用什么来表示似乎就不重要了，其中一个方式就是邱奇数 [^church-num]。

[^church-num]: [邱奇数 - 维基百科](https://zh.wikipedia.org/wiki/%E9%82%B1%E5%A5%87%E6%95%B0)

在这种表示法下，数字被表现为一个函数应用于一个值多少次。也就是说，所谓的「0」，就是一个函数应用一个值 0 次，即 `x`，「1」是应用 1 次，即 `(f x)`，「2」则是先应用 1 次，再对作用后的返回值再应用 1 次，即 `(f (f x))`，以此类推。首先，要先有一个初始，也就是所谓的「0」，用 Scheme 表示如下：

```scheme
(define ZERO
  (lambda (f)
    (lambda (x) x)))
```

注意，这里的 `(define ZERO ...)` 只是给了后面的表达式一个名字，而非像 C 语言一样声明一个变量，这个过程完全是引用透明的，也就是此后任何出现 `ZERO` 的地方都可以用这里定义的部分直接代换。下面是函数 `succ` 的定义，作用是求一个邱奇数的后继（successor），即所谓的给一个数加「1」。

```scheme
(define succ
  (lambda (n)
    (lambda (f)
      (lambda (x) (f ((n f) x))))))
```

此时，我们已经可以用这个方式表示所有的自然数了，举例而言：

```scheme
; (f x)
(define ONE (succ ZERO))

; (f (f x))
(define TWO (succ (succ ZERO)))

; (f (f (f x)))
(define THREE (succ TWO))
```

前面说了，数只有在给定的上下文中才会有其意义，否则只是一个概念而已，这里为了方便计算机打印，可以将其设置为对我们常用的阿拉伯数字的转换：

```scheme
(define trans
  (lambda (n)
    ((n (lambda (x) (+ x 1))) 0)))
```

其中传入给邱奇数 `n` 的函数是一个给阿拉伯数字加一的函数，之后再传入的被作用对象是阿拉伯数字 `0`，故结果就是该邱奇数的概念对应的阿拉伯数字。此时，在 REPL 中可以这样进行转换：

```
>>> (trans ONE)
1
>>> (trans (succ (succ ONE)))
3
```

除了这样依赖于上下文的意义，数字的作用还有一些预定义的运算。其中，加法的运算最为简单直接：

```scheme
(define plus
  (lambda (m)
    (lambda (n)
      (lambda (f)
        (lambda (x) ((m f) ((n f) x)))))))
```

也就是，对于两个邱奇数 `n`，`m` 而言，其求和之后的邱奇数是 `l`，当给 `l` 一个函数 `f` 和一个参数 `x` 之后，它会先将 `f` 和 `x` 传给 `n` 得出结果后再将 `f` 和结果传给 `m`，这样 `f` 就在 `x` 上作用了所谓「m + n」次了。

```
>>> (trans ((plus ONE) TWO))
3
```

类似地，乘法的定义也相当直观：

```scheme
(define mult
  (lambda (m)
    (lambda (n)
      (lambda (f) (m (n f))))))
```

也就是将 `f` 作用 `n` 次这件事重复作用 `m` 次，这也正是乘法的意义：

```
>>> (trans ((mult TWO) THREE))
6
```

如果想得到一个邱奇数的前驱（predecessor）则比之前的计算要困难多了，可以表示如下：

```scheme
(define pred
  (lambda (n)
    (lambda (f)
      (lambda (x)
        (((n (lambda (g)
               (lambda (h) (h (g f))))) (lambda (u) x)) (lambda (u) u))))))
```

则有：

```
>>> (trans (pred THREE))
2
```

当然，这个 `pred` 的表示非常令人困惑，但可以使用代换模型直接验证，考虑将将该函数应用于 `ONE`，再将结果应用于 `f` 和 `x`。那么，首先肯定是不能将 `ONE` 直接应用于 `f` 了，这里将 `ONE` 应用于另一个函数，这个函数是：

```scheme
(lambda (g)
  (lambda (h) (h (g f))))
```

其中的 `f` 是传入的 `f`，此时 `ONE` 会返回另一个函数：

```scheme
(lambda (y)
  ((lambda (g)
     (lambda (h) (h (g f)))) y))
```

即：

```scheme
(lambda (y)
  (lambda (h) (h (y f))))
```

这里为了避免和传入的 `x` 冲突，将 `x` 替换为 `y`。然后将这个结果应用于另一个函数：

```scheme
(lambda (u) x)
```

其中的 `x` 是传入的 `x`，结果为：

```scheme
(lambda (h) ((lambda (u) x) f))
```

即：

```scheme
(lambda (h) (h x))
```

这个结果又被应用到了另一个函数上：

```scheme
((lambda (h) (h x)) (lambda (u) u))
```

即：

```scheme
x
```

即 `(pred ONE)` 的定义如下：

```scheme
(lambda (f)
  (lambda (x) x))
```

也就是说 `(pred ONE)` 就是 `ZERO`。

一但有了这些对自然数的运算，负数、浮点数都是可以定义的，毕竟我们使用的计算机也是通过一些约定的记法来表示负数和浮点数的。

# 逻辑的表示

上一节说明了如何用函数来表示数的概念，这一节将用函数来表示逻辑与断言。同样地，首先要思考的是，我们一般使用的 `True` 和 `False` 到底是用来做什么的。事实上，逻辑不是为了判断对错，而是对条件分支的选择，它说明了有两种情况，在一种情况下选择一种分支，在另一种情况下选择另一分支，另外还需要在其上进行逻辑运算的算子（与或非）。注意到 C 语言其实是没有布尔值这个类型的，但是依然不影响 `if` 的使用，因为 C 里面可以用其他类型的值来进行逻辑运算，在 `if` 里也可以用这些结果来进行分支的选择。

所以，我们可以这样来定义布尔值，下面这个定义法被称为邱奇布尔值：

```scheme
(define TRUE
  (lambda (x) (lambda (y) x)))

(define FALSE
  (lambda (x) (lambda (y) y)))
```

也就是说，给定两个分支，`TURE` 会选择第一个，而 `FALSE` 会选择第二个。至于逻辑运算可以基于这个表示方式进行定义，比如与运算定义如下：

```scheme
(define and
  (lambda (p)
    (lambda (q) ((p q) (lambda (x) (lambda (y) x)))))
```

即：

```scheme
(define and
  (lambda (p)
    (lambda (q) ((p q) FALSE)))
```

也就是，如果 `p` 为真，将会由 `q` 来选择分支，如果 `p` 为假，则由 `FALSE` 来选择条件分支。至于或和非也类似定义：

```scheme
(define or
  (lambda (p)
    (lambda (q) ((p TRUE) q))))

(define not
  (lambda (p) ((p FALSE) TRUE)))
```

当然也可以转成一个可以被打印的形式：

```
>>> ((((and (not FALSE)) TRUE) 'true) 'false)
true
```

# 序对的表示

一个序对（pair）就是一个二元组（2-tuple），这是一个非常简单而且非常强大的结构构件，如果在 C 中，表示形式大概是这样的：

```c
struct Pair {
  void* first;
  void* second;
}
```

这个表示方法是一个很典型的方式，它可以很显然地看出数据是如何存放的，但事实上，为了获得一个序对，我们根本不用关心数据是如何存放的，我们关心的是如何构建一个序对，然后如何将组成序对的元素取出来，换言之，我们希望有 `(first ((pair x) y))` 就是 `x`，`(second ((pair x) y))` 就是 `y`。里面具体是怎样的，并不值得关心，也就是我们只是需要一个 constructor 和两个 selector 而已。邱奇序对编码（Church encoding for pairs）的表示如下：

```scheme
(define pair
  (lambda (x)
    (lambda (y)
      (lambda (f) ((f x) y)))))

(define first
  (lambda (p) (p (lambda (x) (lambda (y) x)))))

(define second
  (lambda (p) (p (lambda (x) (lambda (y) y)))))
```

使用之前定义的 `TRUE` 和 `FALSE`：

```scheme
(define first (lambda (p) (p TRUE)))

(define second (lambda (p) (P FALSE)))
```

这个表示方式非常令人惊讶，因为数据看起来像是存在于一片虚无之中，语言在执行的时候构建了 closure，但是看起来似乎就像把函数传来传去，数据就这样在其中隐匿、出现。`pair` 函数接收了两个元素之后就返回了另一个函数，这个函数接收一个选择函数，然后 `pair` 就将两个元素交由选择函数进行选择，而 `first` 和 `second` 则将对应的选择函数交给这个函数，获取到对应的数据。

有了这样一个东西，我们只要再有一个能够表示 list 末尾的东西就可以构建一个 list 了。将这个东西命名为 `NIL`，将判断一个元素是否为 `NIL` 的函数命名为 `nil?`。`NIL` 具体是个什么东西不重要，最重要的是，将 `nil?` 应用于一个序对将返回 `FALSE` 而将 `nil?` 应用于 `NIL` 将返回 `TRUE`，可以表示如下：

```scheme
(define NIL (lambda (x) TRUE))

(define nil?
  (lambda (p) (p (lambda (x)
                   (lambda (y) FALSE))))))
```

`nil?` 接受一个序对或是一个 `NIL`，并传递给他一个接受两个参数再返回 `FALSE` 的函数，如果是一个序对，那么结果必然是 `FALSE`，而如果是 `NIL`，那么这个函数则不会被调用，而是 `NIL` 直接返回了 `TRUE`。这样就表示了 `NIL`。那么构建有两个元素的 list 的方法可以表示为：

```scheme
(lambda (x) (lambda (y) ((pair x) ((pair y) NIL))))
```

这样一个 list 的构建方式就是链表，通过 pair，可以构建更为复杂的数据结构。

# 进一步思考

在这种计算模型下除法如何实现？递归如何实现？这些都是更为进阶的话题，在 Wikipedia 上有更详细的说明 [^wiki-div] [^wiki-rec]。

[^wiki-div]: [Church encoding - Wikipedia](https://en.wikipedia.org/wiki/Church_encoding#Division)
[^wiki-rec]: [Lambda calculus - Wikipedia](https://en.wikipedia.org/wiki/Lambda_calculus#Recursion_and_fixed_points)

计算是一个非常本质的东西，跳出固有的思维来看，能产生更多的思考。

# 参考
