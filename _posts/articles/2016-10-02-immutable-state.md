---
layout: post
title: 不可变的状态
excerpt: "状态总与变化相关，如果我们不使用可变的变量，那要如何表示状态？又该如何使状态间的转换更加方便？"
date: 2016-10-02
modified: 2016-10-04
categories: articles
author: zrl
tags:
  - Functional Programming
  - Scala
  - Haskell
comments: true
share: true
---

# 可变与状态

在过程式的编程中，例如使用 C 语言，我们的工作是不断地以副作用的形式对状态进行修改，然后产生结果。例如我们可能会先令 `int x = 0`，然后进行一系列操作，将 `x` 修改以记录这些操作的过程和产生的效果，最后再产生结果。但是，如果一个语言建议一个值不可变（例如 Scala）或是强制要求一个值不可变（例如 Haskell）那又该怎么办？

例如说我们想要实现这样的一个函数，这个函数将遍历一棵二叉树，并给其每一个树叶打上标签 [^reason-effects]，二叉树的定义如下：

[^reason-effects]: Graham Hutton - Reasoning About Effects: Seeing the Wood Through the Trees 

```scala 
sealed trait Tree[A]
case class Leaf[A](value: A) extends Tree[A]
case class Branch[A](left: Tree[A], right: Tree[A]) extends Tree[A]
```

如果我们传递给 `labelTree` 函数一棵

```scala
val tree = Branch(Leaf('a'), Branch(Branch(Leaf('b'), Leaf('c')), Leaf('d')))
```

这样的树，我们想要得到这样的结果：

```scala
Branch(Leaf((0,'a')),Branch(Branch(Leaf((1,'b')),Leaf((2,'c'))),Leaf((3,'d'))))
```

这显然是一个涉及读写状态的问题，当函数决定给一个节点进行标记的时候，它必须需要知道当前应该打什么标签，并且用某种方法影响下一个节点所要被打的标签。一个简单的处理如下：

```scala 
var i = 0

def labelTree[A](tree: Tree[A]): Tree[(Int, A)] = tree match {
  case Leaf(value) =>
    val newLeaf = Leaf(i, value)
    i += 1
    newLeaf
  case Branch(left, right) =>
    Branch(labelTree(left), labelTree(right))
}
```

这个处理很简单直接，就是维护一个变量 `i`，当函数 `labelTree` 遍历一棵树的时候，如果看到了叶子节点，就打上标签 `i` 并将 `i` 加 1。如果看到一个树枝节点，就先递归标记左子树，然后再递归标记右子树，并用这两个结果构筑新树。使用方法如下：

```scala
val tree1 = Branch(Leaf('a'), Branch(Branch(Leaf('b'), Leaf('c')), Leaf('d')))
val tree2 = Branch(Leaf(1), Branch(Branch(Leaf(2), Leaf(3)), Leaf(4)))
// lbT1 = Branch(Leaf((0,'a')),Branch(Branch(Leaf((1,'b')),Leaf((2,'c'))),Leaf((3,'d'))))
val lbT1 = labelTree(tree1)
// lbT2 = Branch(Leaf((4,1)),Branch(Branch(Leaf((5,2)),Leaf((6,3))),Leaf((7,4))))
val lbT2 = labelTree(tree2)
```

# 使用变量的隐患

在这个实现中，函数 `labelTree` 是有状态的，同样是传递给它一棵树，它所给节点打上的标签是不同的。原因是它在不断读取一个可变的值 `i`，并根据 `i` 来决定其行为。在这个简单的例子中，这样的处理似乎没有什么问题，因为只有 `labelTree` 在修改 `i`，但是，如果放在一个更加复杂的场景中，这样做是有很大风险的。用户可能会先用某些方式修改这个 `i` 来控制 `labelTree` 所打的标签，而 `labelTree` 本身并不知道这个修改的过程，它并没有办法知道 `i` 是否已经是用户所期望的值，如果放在并发的场景下，这很可能造成灾难，因为并发的情形更加无法控制对一个变量的正确修改，这时可能就需要各种同步策略来保证程序的正确，在标记过程中，可能其它函数又将 `i` 修改了，造成结果的错误。同时，这个方式也对调试带来了困难，如果一个函数依赖了一个外部的可变状态，一旦需要测试这样的函数的正确性，就需要先构建状态，才能进行测试。

计算机的「函数」和数学上的「函数」不同，数学的函数是一种映射 [^func-math]，例如 `double(x) = x * 2`，无论调用多少次，只要你给出了同样的输入，它就会给出同样的输出。你甚至不知道这个计算过程到底是真通过计算得出的还是查表得出的，因为没有区别，确定了输入，就确定了输出。计算机的「函数」则不一定，在大多数编程语言中，一个函数除了能接收参数并返回一个值之外，它还能有副作用，例如，它可以修改变量，可以在屏幕上打印字符串，可以读写文件等等，这些操作使得我们无法通过输入内容直接确定输出结果。如果我们在程序中定义的函数和数学函数一样，不依赖可变状态，也不产生副作用，那么我们就可以很好地解决之前提到的问题。这也是为什么一些语言在语法上就鼓励不可变。那么如果变量就是一个值，不可变，那我们还有办法实现我们要的功能么？

[^func-math]: [Function (mathematics) - Wikipedia](https://en.wikipedia.org/wiki/Function_(mathematics))

这显然可以，虽然一个变量不可变，但是我们可以创造新的变量，并用新的变量来确定下一步的操作：

```scala 
def labelTree[A](tree: Tree[A], label: Int): (Tree[(Int, A)], Int) = tree match {
  case Leaf(value) =>
    (Leaf((label, value)), label + 1)
  case Branch(left, right) =>
    val (newLeft, nextLabel1) = labelTree(left, label)
    val (newRight, nextLabel2) = labelTree(right, nextLabel1)
    (Branch(newLeft, newRight), nextLabel2)
}
```

这一次，`labelTree` 函数没有从外部的一个变量中读取状态，也没有去修改外部状态，它只接受一棵树和一个初始标签，然后递归执行，并返回结果，一旦输入确定，结果也就确定了。注意到 `labelTree` 的返回值类型并不是我们要的树 `Tree[(Int, A)]`，而是由树和标签构成的 Tuple `(Tree[(Int, A)], Int)`，这是因为 `labelTree` 的调用过程中，并不是无状态的，如果无状态，我们要如何改变对每一个叶子节点的标签呢？`labelTree` 只是要求状态被显式地传递，它接收一棵树和一个标签的状态，返回一个已被标记的树和一个新的状态，接下来的工作是根据这个新的状态进行的。此时，我们就可以这样调用获得最终结果（其中最后的 `_1` 方法可以获得一个 Tuple 的第一个值）：

```scala 
// lbT = Branch(Leaf((0,'a')),Branch(Branch(Leaf((1,'b')),Leaf((2,'c'))),Leaf((3,'d'))))
val lbT = labelTree(tree1, 0)._1
```

由于现在状态不是一个共享的变量了，所以如果想要标记两棵树，并且序号是连续的，就需要手工传递一下状态了，例如：

```scala
// lbT1 = Branch(Leaf((0,'a')),Branch(Branch(Leaf((1,'b')),Leaf((2,'c'))),Leaf((3,'d'))))
val (lbT1, lbState) = labelTree(tree1, 0)
// lbT2 = Branch(Leaf((4,1)),Branch(Branch(Leaf((5,2)),Leaf((6,3))),Leaf((7,4))))
val (lbT2, _) = labelTree(tree2, lbState)
```

我们可以对这个函数进行进一步的泛化，标签不一定是一个整型值，状态不一定和标签本身类型相同，状态的转换也不一定是对整型值进行 `+ 1` 的操作，我们可以将状态类型和状态转换的方式抽象出来：

```scala
def labelTree[A, L, S](transState: S => (L, S))
                      (state: S)
                      (tree: Tree[A]): (Tree[(L, A)], S) = tree match {
  case Leaf(value) =>
    val (label, nextState) = transState(state)
    (Leaf((label, value)), nextState)
  case Branch(left, right) =>
    val (newLeft, nextState1) = labelTree(transState)(state)(left)
    val (newRight, nextState2) = labelTree(transState)(nextState1)(right)
    (Branch(newLeft, newRight), nextState2)
}
```

类型参数 `S` 即为标签的状态，`L` 即为标签的值，函数 `transState` 将一个状态转换为下一个状态的同时，产生出一个标签，`labelTree` 就用这个标签来标记叶子节点。现在我们可以这样去使用 `labelTree`：

```scala
// lbT1 = Branch(Leaf((0,'a')),Branch(Branch(Leaf((1,'b')),Leaf((2,'c'))),Leaf((3,'d'))))
val lbT1 = labelTree((x: Int) => (x, x + 1))(0)(tree1)._1

// the label is a string, and the state is a list of string
val trans = (l: List[String]) => (l.head, l.tail)
val init = List("k", "u", "r", "i", "s", "u")
// lbT2 = Branch(Leaf(('k','a')),Branch(Branch(Leaf(('u','b')),Leaf(('r','c'))),Leaf(('i','d'))))
val lbT2 = labelTree(trans)(init)(tree1)._1
```

# 构建状态转变类型

在之前的实现中，我们显式地在 `labelTree` 的调用中传递了状态，并将这个过程泛化到可以处理任意标签的情况，我们此时可以发现，状态和状态的转变其实是一个非常一般的情况，对于这样的情况，我们可以构建一个新的类型专门用来表示它：

```scala
// transform state and produce something
class StateT[X, S](val trans: S => (X, S)) {
  // run state with a init state, and get the final result
  def run(init: S): X = trans(init)._1
}

object StateT {
  // Scala has syntax sugar for apply method,
  // we can create a new instance of StateT 
  // by writing: `StateT(func)` instead of: `new StateT(func)`
  def apply[X, S](trans: S => (X, S)): StateT[X, S] = new StateT(trans)
}

def labelTree[A, L, S](transState: S => (L, S))
                      (tree: Tree[A]): StateT[Tree[(L, A)], S] = {
  StateT((state: S) => tree match {
    case Leaf(value) =>
      val (label, nextState) = transState(state)
      (Leaf((label, value)), nextState)
    case Branch(left, right) =>
      val (newLeft, nextState1) = labelTree(transState)(left).trans(state)
      val (newRight, nextState2) = labelTree(transState)(right).trans(nextState1)
      (Branch(newLeft, newRight), nextState2)
  })
}
```

这里将这个类型命名为 `StateT`，不使用 `State` 作为名称的原因是 `StateT` 并不是状态本身，而是表示了一个状态的转换，其类型参数 `S` 才是状态的类型，而其中的 `X` 则是状态转换过程中的产物。现在 `labelTree` 的使用方法变为了：

```scala
// lbT = Branch(Leaf((0,'a')),Branch(Branch(Leaf((1,'b')),Leaf((2,'c'))),Leaf((3,'d'))))
val lbT = labelTree((x: Int) => (x, x + 1))(tree1).run(0)
```

我们用 `StateT` 描述了给树打标签的运行过程，并在最后传递一个初始的标签以获得最终结果。

到目前为止，`labelTree` 的不可变状态实现让我们陷入了手工传递状态的麻烦之中，整个过程充斥着转变状态，获取新状态，将函数应用于新状态之上这样的繁复代码之中，相比起最初的可变状态实现，这个维护过程并不令人愉快，这大概也是许多人非常喜欢全局变量的原因之一吧。

如何简化这一过程呢？直接空想相当困难，这里可以给出一个提示，那就是这里定义的 `StateT` 其实是一个 Monad，我们可以用 Scala 的 for-comprehension 来操作它。

什么是 Monad？在之前的文章中我们已经多次使用过它了，但是一直没有给出其定义和说明，只说了我们可以用 for-comprehension 来进行一些方便的操作。事实上，Monad 并不是太好理解，因为它来源于数学世界，如果你查阅 Wikipedia [^monad-ct-wiki]，你会看到这样的描述：

[^monad-ct-wiki]: [Monad (category theory) - Wikipedia](https://en.wikipedia.org/wiki/Monad_(category_theory))

> A monad (also triple, triad, standard construction and fundamental construction) is an endofunctor (a functor mapping a category to itself), together with two natural transformations.

看起来解释本身和被解释对象一样无法理解。在编程上，我们只需要知道如果一个参数化类型 M 上定义了如下两个操作：

```
unit : A => M[A]
bind : M[A] => (A => M[B]) => M[B]
```

并且这两个操作满足「Monad law」即可 [^monad-fp-wiki]。所谓「Monad law」是指 [^monad-laws]：

[^monad-fp-wiki]: [Monad (functional programming) - Wikipedia](https://en.wikipedia.org/wiki/Monad_(functional_programming))
[^monad-laws]: [Monad Laws - HaskellWiki](https://wiki.haskell.org/Monad_laws)

- Left identity: `bind(unit(x))(f) === f(x)`<br/>
即：`unit(x).flatMap(f) === f(x)`
- Right identity: `bind(m)(unit) === m`<br/>
即：`m.flatMap(unit) === m`
- Associativity: `bind(bind(m)(f))(g) === bind(m)(x => bind(f(x))(g))`<br/>
即：`m.flatMap(f).flatMap(g) === m.flatMap(x => f(x).flatMap(g))`

这个定义很精炼，但它和许多数学上的概念一样，知道了定义我们仍然不知道如何使用，所以更好的方法就是去多在实例中使用它，这里提一下 Monad 的定义的目的只是为了防止读者看到一个不明单词产生恐惧而已。从上面的定义可以大致看出 `unit` 是一个 Monad 的构造器，对于 `M` 类型的 Monad 而言，如果将 `unit` 应用于一个 `T` 类型的值，那么它将构造一个 `M[T]` 类型的值。而 `bind` 则处理了 Monad 之间的组合，可以将一个 Monad 转变为另一个 Monad。

刚刚提到了 `StateT` 是一个 Monad，这其实有点不准确，因为根据定义，Monad 只接收一个类型参数，所以 `StateT` 本身不是一个 Monad，但对于一个给定的状态类型 `S` 而言，`StateT` 就是一个 Monad 了，我们可以这样定义 `unit` 和 `bind`（即 `flatMap`）：

```scala
class StateT[X, S](val trans: S => (X, S)) {
  def run(state: S): X = trans(state)._1
  def flatMap[Y](func: X => StateT[Y, S]): StateT[Y, S] =
    StateT ((s: S) => {
      val (x, newS) = trans(s)
      func(x).trans(newS)
    })
}

object StateT {
  def apply[X, S](trans: S => (X, S)): StateT[X, S] = new StateT(trans)
  def unit[X, S](x: X): StateT[X, S] = StateT(s => (x, s))
  def bind[X, Y, S](sx: StateT[X, S])
                   (func: X => StateT[Y, S]): StateT[Y, S] = sx.flatMap(func)
}
```

可以看到，`flatMap` 和 `bind` 其实是一个意思，只是 `bind` 是在外部操作 `StateT` 对象的函数（所以需要先传入被操作的对象），而 `flatMap` 则是 `StateT` 对象的方法（所以不需要传入被操作的对象）。正如之前所提到的，一个类型是一个 Monad 不仅意味着在其上定义了 `unit` 和 `bind`，它们还需要满足 Monad law。如果你自己设计了一个 Monad，也必须使对应的两个函数满足 Monad law，否则用户在使用这个类型的时候就无法获得他期望的行为。这里的定义是符合 Monad law 的，可以手工推导验证一下。

如果看过之前的一些文章，可能会疑惑为什么之前的 Monad 没有定义 `unit`？原因是 Scala 比较注重工程实践，虽然 for-comprehension 可以用来方便地操作 Monad，但使用上它并没有去暴露 Monad 这个概念（将 `bind` 改名为 `flatMap` 可能也是因为这个原因）。所以 for-comprehension 仅要求用户定义 `flatMap` 和 `map`（部分 for-comprehension 的写法还要求定义 `filter` 和 `foreach`，此处暂不考虑）。而 `map` 方法本身是可以用 `flatMap` 和 `unit` 直接定义出来的：

```scala
class StateT[X, S](val trans: S => (X, S)) {
  // ...
  def map[Y](func: X => Y): StateT[Y, S] =
    flatMap(x => StateT.unit(func(x)))
}
```

此时我们就可以这样实现 `labelTree` 了：

```scala
def labelTree[A, L, S](genLabel: StateT[L, S])
                      (tree: Tree[A]): StateT[Tree[(L, A)], S] = tree match {
  case Leaf(value) =>
    for {
      label <- genLabel
    } yield Leaf(label, value)
  case Branch(left, right) =>
    for {
      newLeft <- labelTree(genLabel)(left)
      newRight <- labelTree(genLabel)(right)
    } yield Branch(newLeft, newRight)
}
```

我们可以这样使用这个版本的 `labelTree`：

```scala
val freshInt = StateT((i: Int) => (i, i + 1))
// lbT = Branch(Leaf((0,'a')),Branch(Branch(Leaf((1,'b')),Leaf((2,'c'))),Leaf((3,'d'))))
val lbT = labelTree(freshInt)(tree1).run(0)
```

非常漂亮，在这个版本的实现中，尽管我们显式地在类型上表示了状态、尽管状态依然是不可变的、尽管我们确实能获得正确的结果，但我们并没有去手工管理状态的更新，状态在 Monad 的包裹中传递。这看起来很像是全局共享可变状态，但它确实不是，我们可以试一下下面的代码：

```scala
val labelInt = labelTree(freshInt)
// lbT1 = Branch(Leaf((0,'a')),Branch(Branch(Leaf((1,'b')),Leaf((2,'c'))),Leaf((3,'d'))))
val lbT1 = labelInt(tree1).run(0)
// lbT2 = Branch(Leaf((0,1)),Branch(Branch(Leaf((1,2)),Leaf((2,3))),Leaf((3,4))))
val lbT2 = labelInt(tree2).run(0)
```

我们发现标签每次都是从 `0` 开始的。注意到，与共享可变状态的实现中使用 `i` 来记录状态不同，此处的状态并不是由 `labelInt` 来记录的（尽管看起来很像是），所以当我们调用两次 `labelInt` 给不同的树打上标签时，我们需要两次调用 `run` 方法传入初始状态，而这两次调用之间没有关系。所以，每一次的调用都是重新的计数。

但如果我们的需求就是要给两棵树打上连续的标签，那我们应该如何用这个版本的 `labelTree` 来实现呢？一个直接的方法就是像之前一样调用 `trans` 方法来传递状态：

```scala
// lbT1 = Branch(Leaf((0,'a')),Branch(Branch(Leaf((1,'b')),Leaf((2,'c'))),Leaf((3,'d'))))
// lbS = 4
val (lbT1, lbS) = labelInt(tree1).trans(0)
// lbT2 = Branch(Leaf((4,1)),Branch(Branch(Leaf((5,2)),Leaf((6,3))),Leaf((7,4))))
val lbT2 = labelInt(tree2).run(lbS)
```

这个方式很简单，但它使我们又重新陷入了手工管理状态的麻烦之中。回看一下我们之前使用 `flatMap` 来管理状态转换的方式，我们就可以发现其实并不需要调用 `trans`，我们可以用类似之前的方式来实现我们要的功能：

```scala
// lbST has type StateT[(Tree[(Int, Char)], Tree[(Int, Int)]), Int]
val lbST = for {
  t1 <- labelInt(tree1)
  t2 <- labelInt(tree2)
} yield (t1, t2)
// lbT1 = Branch(Leaf((0,'a')),Branch(Branch(Leaf((1,'b')),Leaf((2,'c'))),Leaf((3,'d'))))
// lbT2 = Branch(Leaf((4,1)),Branch(Branch(Leaf((5,2)),Leaf((6,3))),Leaf((7,4))))
val (lbT1, lbT2) = lbST.run(0)
```

这个版本的 `labelTree` 非常简洁，但这个过程似乎有点过于神奇了，以至于让人因难以理解它如何工作而感到不安，要想知道这个过程是怎么工作的，只需要展开 for-comprehension 的调用即可：

```scala
def labelTree[A, L, S](genLabel: StateT[L, S])
                      (tree: Tree[A]): StateT[Tree[(L, A)], S] = tree match {
  case Leaf(value) =>
    genLabel.map(label => Leaf(label, value))
  case Branch(left, right) =>
    labelTree(genLabel)(left).flatMap(newLeft =>
      labelTree(genLabel)(right).map(newRight =>
        Branch(newLeft, newRight)))
}
```

重新再看一下 `flatMap` 和 `map` 的实现，我们就会发现实际上是 `flatMap` 在帮助我们管理了状态的更新，在 `flatMap` 中，`trans` 被调用，记录了状态的转变，然后再通过传入的 `func` 将结果进行转换，通过这个调用，使得整个状态的转换管理的工作被抽象出来，不需要显式管理。这样我们就可以知道为什么 Monad 这个概念要被拿来在编程上使用，虽然它的定义本身有点不知所云，但它确实能构建一个强大的抽象，使得程序变得明晰。

但是，共享可变变量的实现中还有一个灵活之处，就是它可以很方便地获取和修改状态，例如，在给多棵树打连续的标签的过程中，我们可能需要在两棵树之间隔开一个标签，也就是说我们想在给一棵树打上标签后先令标签 `+ 1` 后再给第二棵树打标签。在最初的共享可变变量实现中，这非常容易实现：

```scala
// lbT1 = Branch(Leaf((0,'a')),Branch(Branch(Leaf((1,'b')),Leaf((2,'c'))),Leaf((3,'d'))))
val lbT1 = labelTree(tree1)
// current state value is i, set it to i + 1
i += 1
// lbT2 = Branch(Leaf((5,1)),Branch(Branch(Leaf((6,2)),Leaf((7,3))),Leaf((8,4))))
val lbT2 = labelTree(tree2)
```

换到 `StateT` 的实现中我们可以怎么做呢？用 `trans` 是可以做到的：

```scala
// lbT1 = Branch(Leaf((0,'a')),Branch(Branch(Leaf((1,'b')),Leaf((2,'c'))),Leaf((3,'d'))))
// lbS = 4
val (lbT1, lbS) = labelInt(tree1).trans(0)
// current state value is lbS, set it to lbS + 1, and pass it to `run` method
// lbT2 = Branch(Leaf((5,1)),Branch(Branch(Leaf((6,2)),Leaf((7,3))),Leaf((8,4))))
val lbT2 = labelInt(tree2).run(lbS + 1)
```

正如我们之前所说，我们不希望手工操作状态转换，我们要避免调用 `trans`。我们也许会想使用类似的方式在 for-comprehension 中设置状态，但我们目前只能通过 `run` 方法传入初始值的方式来控制初始状态，由于状态的转换过程交给了 `flatMap` 进行管理，我们没法在状态转换的过程中去获取和设置状态：

```scala
val lbST = for {
  t1 <- labelInt(tree1)
  // how to change the state here?
  // there is no way to get or set the current state
  t2 <- labelInt(tree2)
} yield (t1, t2)
```

也就是说，为了使得 `StateT` 更像我们一般使用的可变状态，我们应该设法提供一个能方便获取和设置状态的方式。幸运的是，这个需求并不难实现。

首先我们要考虑到，这两个和状态相关的函数一定是封装在 `StateT` 里的，假设状态的类型是 `S`，那么最终返回值的类型一定形如 `StateT[?, S]`。`?` 处应该是什么类型呢？对于状态获取函数 `getS` 而言，由于我们想获得状态，那显然这个类型就应该是 `S` 了，也就是说我们在状态转换的过程中并不产生其他类型的值，而是直接将当前状态本身作为转换过程的产物。那么下一个状态是什么呢？由于我们只想获得当前状态，所以下一个状态就应该和当前状态一样，不需要去改变它。因此，`getS` 的实现如下：

```scala 
val getS[S]: StateT[S, S] = StateT(s => (s, s))
```

至于设置状态的函数 `setS` 也是类似的，因为我们希望设置下一个状态，所以 `setS` 的类型应该形如 `S => StateT[?, S]`，所以在状态转换的过程中，我们要做的就是忽略当前状态，并将下一个状态设置为给定状态即可。由于设置状态这件事就像对变量的赋值操作一样，不需要返回什么有意义的值，但由于 `StateT` 要求在状态转换的过程中必须产生一个值，所以我们这里返回一个 dummy value，实现如下：

```scala
def setS[S](s: S): StateT[Unit, S] = StateT(_ => ((), s))
```

这样我们就可以很方便地实现刚刚提到的在两个树之间隔开一个标签的需求了：

```scala
// lbST has type StateT[(Tree[(Int, Char)], Tree[(Int, Int)]), Int]
val lbST = for {
  t1 <- labelInt(tree1)
  i <- getS             // get state value i
  _ <- setS(i + 1)      // set state to i + 1
  t2 <- labelInt(tree2)
} yield (t1, t2)
// lbT1 = Branch(Leaf((0,'a')),Branch(Branch(Leaf((1,'b')),Leaf((2,'c'))),Leaf((3,'d'))))
// lbT2 = Branch(Leaf((5,1)),Branch(Branch(Leaf((6,2)),Leaf((7,3))),Leaf((8,4))))
val (lbT1, lbT2) = lbST.run(0)
```

提供了这两个函数之后，我们就提供了完整的状态控制机制，使得在这样的实现下操作状态就如同使用一个变量一样轻松直观，同时又兼顾了不可变状态的优点。

这篇文章看起来到这里就已经可以结束了，但我们还可以更进一步。前面提到了，副作用并不止是修改变量一种，它还包括有读写文件、读入用户输入、在控制台打印输出等等，总之，一个函数如果除了接收参数和返回结果之外做了任何事情，它都产生了副作用。副作用这个名字不太好听，而且副作用的任意散布也确实是一件坏事，但是它本身非常重要，可以想象，如果一个程序完全不产生副作用，那么它除了能浪费电之外，什么都做不到。它不能读写文件，不能连接网络，甚至连最基础的「Hello World」都没法打印。既然副作用是必要的，而副作用又是必须得到控制的，所以我们希望能有某种方法能够对其进行更好的控制和封装。

# 封装所有副作用

读写变量这一副作用我们可以用前面构建的 `StateT` 实现，像输入输出这类操作我们有办法封装吗？有，而且实际上和 `StateT` 的构建方法没有太本质的区别。回忆一下，我们在封装可变状态这一副作用的时候是怎么做的？我们将状态的转变从隐式提升到显式在类型中展现，通过 Monad 的 `flatMap` 操作来使得状态的转换可以不需要手工管理。所以，我们可以类似地定义一个类型来代表所有能产生 IO 的操作，然后将这个类型实现为一个 Monad，并在其上进行操作，这里将其命名为 `IO`：

```scala
class IO[A](val run: () => A) {
  def flatMap[B](func: A => IO[B]): IO[B] = func(run())
  def map[B](func: A => B): IO[B] = flatMap(a => IO.unit(func(a)))
}

object IO {
  def apply[A](toA: => A): IO[A] = new IO(() => toA)
  def unit[A](a: A): IO[A] = IO(a)
}
```

在这个 `IO` 类型中，我们将通过一个不接受参数的函数 `run` 来表示正式运行这个 `IO`，这个 `run` 在数学上很不合理，既然不接受参数，那么 `() => A` 这个类型并没有什么意义，应该和 `A` 没有区别。但由于 `run` 将产生 `IO` 操作，所以必须实现为一个无参的函数以便延迟调用。下面我们简单地实现两个函数用于在命令行进行输入输出操作：

```scala
def putLine(str: String): IO[Unit] = IO(println(str))
def getLine: IO[String] = IO(StdIn.readLine())
```

经过了这一层的封装之后，所有的 `IO` 操作都可以像一般的值一样到处传递，并且方便组合，只有到最终运行的时候才会产生作用，就和 `StateT` 一样。例如，我们可以写一个读入人名并且打印问候信息的 `IO[Unit]` 值：

```scala
val sayHello: IO[Unit] = for {
  _ <- putLine("Input your name: ")
  name <- getLine
  _ <- putLine("Hello, " + name + "!")
} yield ()
```

注意到，正如 `StateT` 的例子中出现的代码一样，这个 `sayHello` 被定义后并没有产生副作用，它只是一个一般的值。要使它运行，我们需要这样做：

```scala
sayHello.run()
```

这就是封装所有副作用的方法，对比一下之前我们 `StateT` 的实现，我们可以发现，`IO` 和 `StateT` 一样，我们确实是在传递状态，并通过 `flatMap` 来对这个过程进行管理。只不过 `IO` 所管理的状态不是一个变量而是程序与整个世界之间交互的所有 IO 操作。在 Haskell 中，IO Monad 是一个基础的 Monad [^haskell-io]。Haskell 声称它是一个纯函数式的语言，也就是说你写的函数都是数学上的纯函数（除了少数后门之外），接收一个值，返回一个值，不能做其他操作。而在这样的环境下，Haskell 产生输入输出这样的副作用的方式就是使用 IO Monad。由于 Scala 允许在任何地方产生副作用，所以我们可以在任何地方调用 `run` 函数执行 `IO` 所封装的代码。但在 Haskell 中，并没有这样的方法，唯一能运行的方式是通过 `main` 运行，而 `main` 函数的类型就是 `IO ()`，这样就保证了 Haskell 的「纯」。

[^haskell-io]: [A Gentle Introduction to Haskell - Input/Output](https://www.haskell.org/tutorial/io.html)

# 将副作用提升到类型的缺点

既然将副作用提升到类型上有如此大的优点，为什么这样设计的语言占比如此之低呢？原因是太麻烦。有副作用的函数我们将其类型变为 `IO`，使得它可以像一般的值一样传递组合，这是优点，但我们也要注意到，一旦一个语言强制了这一实现，就会导致副作用标记如同病毒一样传播。例如我们一开始写了一个类型为 `Int => Int` 的函数 `f`，后来我们希望能够监控这个函数的执行，于是我们决定要在这个函数里添加一行打印日志的代码，这时候就出问题了，由于这个函数不能产生副作用，所以我们需要改变 `f` 的类型为 `Int => IO[Int]`，这样一改，结果是大部分调用这个函数的代码都需要进行更改，否则就会产生类型不匹配的错误。并且，由于 `Int` 被封装在 IO Monad 中，现在已经无法直接获取其值，调用 `f` 的代码的返回值也要用 IO Monad 封装起来，这又会造成新一轮的 IO Monad 的传播。因此，大多数语言并不会去强制用户不产生副作用，但一个设计精良的语言至少应该鼓励用户使用不可变的变量，例如在 Scala 中，声明一个不可变的变量的关键字是 `val`，声明一个可变的变量的关键字是 `var`，两者都很轻量化，而且，Scala 默认使用的容器也基本是不可变的容器。与之相对，在 Java 中，变量默认可变，如果你要将其标明为不可变，需要在其前面添加 `final` 关键字，这就使得这个过程比较啰嗦，同时，Java 默认的容器也是可变的。在工程实践中，除非必要，否则尽量使用不可变，这样可以使得程序更加可靠，也更利于测试与调试。

# 参考
