---
layout: post
title: Parser Combinator
excerpt: "直接手写词法语法分析往往非常痛苦，parser combinator 也许是一个不错的解决方案，其中还能反映一种有趣的编程思想。"
date: 2016-09-10
modified: 2016-09-14
categories: articles
author: zrl
tags:
  - Functional Programming
  - Compiler
  - Haskell
  - Scala
comments: true
share: true
---

# 词法分析和语法分析

词法分析（lexical analysis）[^wiki-lexical] 和语法分析（syntactic analysis，又称为 parsing）[^wiki-parsing]，同属于编译器的前端部分。词法分析器（lexer）将输入拆分为一个个的 token，然后语法分析器根据特定的语法规则将输入的 token 解析为一个结构化的表示，一般为抽象语法树（abstract syntax tree），供之后的语义分析器使用。

[^wiki-lexical]: [Lexical analysis - Wikipedia](https://en.wikipedia.org/wiki/Lexical_analysis)

[^wiki-parsing]: [Parsing - Wikipedia](https://en.wikipedia.org/wiki/Parsing)

在实际开发中，为了简化写词法分析和语法分析的过程，常常会使用生成器来代替人工操作，Lex 和 Yacc 就是生成器的经典实现 [^lex-yacc-home]。Lex 是 Lexical Analyzer 的简写，是一个词法分析器的生成器，Yacc 是 Yet Another Compiler Compiler 的简写，是一个语法分析器的生成器。这两个工具允许用户用类似 BNF 范式的写法声明一个词法描述和语法描述文件，然后自动生成能够解析对应词法语法的 C 语言程序。

[^lex-yacc-home]: [The Lex & Yacc Page](http://dinosaur.compilertools.net)

这个解决方案直观有效，实际应用也很普遍，不止是 C 语言，在其他平台也常有类似的实现，例如 Java 的 ANTLR。但是它也存在一些问题，首先是用机器生成的代码质量往往不如手写高。这个代码质量的问题在程序正常运作的时候倒也不算什么问题，毕竟可以将生成出来的代码当作一个黑盒来调用，不太需要理会内部的实现，但实际情况有时并不这么理想，如果描述的时候出现问题怎么办？甚至如果生成器本身就有 bug 又怎么办？由于生成出来的代码质量较低，所以这就带来了调试困难的问题。所以，在很多重要的应用中，parser 的部分往往是手写的而非用生成器生成 [^clang-zhihu]。不过手写解析器毕竟会有代码不直观的问题，而且这个过程往往比较枯燥乏味。

[^clang-zhihu]: [Clang parser 是完全手写的吗？ - 知乎](http://www.zhihu.com/question/31107048)

也就是说，我们希望有一个方法，能够使得我们在用某种规范形式描述出一个语言的语法后，就能构造出针对该语言的词法分析器和语法分析器，且这个特性必须要尽可能不损失可调试性，同时又足够简单易用。

# 使用 Parser Combinator 解析文本

Parser combinator [^wiki-parser-combinator] 也许是对上述问题的一个比较好的回答，虽然 parser combinator 也有不少缺点使得它解析复杂语法的时候往往力不从心，但在简单的情况下还是比较好用的，另外其中反映的编程思想也相当有趣。

[^wiki-parser-combinator]: [Parser combinator - Wikipedia](https://en.wikipedia.org/wiki/Parser_combinator)

举个例子，在 Java 中，bool 类型的字面值写法有 `true` 和 `false` 两种，用 BNF 范式表述大概是这样：

```
bool_literal ::= "true" | "false"
```

如果使用 Haskell 的 Megaparsec [^github-megaparsec] 来写，就可以写成这样：

[^github-megaparsec]: [Megaparsec - Github](https://github.com/mrkkrp/megaparsec)

```haskell 
boolLiteral = string "true" <|> string "false"
```

上面的 `<|>` 符号和 BNF 范式中 `|` 都是「或者」的意思，`<|>` 是一个函数，接收了两个 parser 构建一个新的 parser。注意这里使用了 Haskell 函数的中缀形式，当然你也可以写成这样：

```haskell 
boolLiteral = (<|>) (string "true") (string "false")
```

`string` 也是一个函数，它接收一个字符串构建了一个能解析该字符串的 parser，如果解析成功，将返回被解析的字符串。`boolLiteral` 将先尝试使用 `string "true"` 来解析输入的字符串，如果失败，就尝试使用 `string "false"` 去进行解析。如果它们都失败了，那么这个 `boolLiteral` 才会失败。

这个表述简单清晰，和语法描述非常接近，但它其实有点问题，因为这个描述只能将字符串解析，解析成功后返回的还是字符串，比如解析 `"true"` 的结果就是 `"true"`。但是现在需要的是一个结构化的内部表示，所以完整写法应该是这样：

```haskell 
data JBool = JBool Bool 

boolLiteral = (string "true" >> return (JBool True))
          <|> (string "false" >> return (JBool False))
```

这里的 `data JBool = JBool Bool` 声明了一个 Haskell 的数据类型 `JBool`，这个类型有一个构造器就是 `JBool`，它接收一个 Haskell 的 `Bool` 类型的值，返回一个 `JBool` 类型的值。至于下面的 `>>` 符号则接收两个 parser，先尝试运行前面的 parser，如果成功了，就丢弃返回值，并使用后一个 parser 来解析，如果后面的 parser 也成功了则返回后一个 parser 的结果。`return` 则接收一个参数，构建了一个必然成功的 parser，无论解析什么输入，都会直接返回该参数。这里的 `>>` 和 `return` 并不是 Megaparsec 特有的东西，Megaparsec 的 `Parser` 被实现为 Monad class [^hackage-monad] 的实例，所以会有这些操作符，但这里不需要考虑这些问题，只要将其当作是对 parser 本身的特定操作即可。

[^hackage-monad]: [Control.Monad - Hackage](https://hackage.haskell.org/package/base-4.9.0.0/docs/Control-Monad.html#t:Monad)

注意到，这里事实上是将词法分析和语法分析放在了一起，将输入的文本直接解析成了程序的内部表示，虽然这两个过程结合到了一起，但是整个过程并没有令人觉得有什么混乱或者不清晰的地方，依然非常直观，这个结合反而带来了方便。

前面提到了 parser combinator 有一些缺点，这里可以说一个，就是它不支持自动的回溯。Parser combinator 的本质是构建一个递归下降的解析器，但是，在 `<|>` 的前一个 parser 解析出错的时候，整个状态并不会自动返回到这个 parser 解析前的位置而是返回到最近出错的位置，例如，Scheme 语言的 boolean 字面值不是 `true` 和 `false` 而是 `#t` 和 `#f`，如果这样写：

```haskell 
data SBool = SBool Bool 

boolLiteral = (string "#t" >> return (SBool True))
          <|> (string "#f" >> return (SBool False))
```

那么在解析 `#t` 这个字符串的时候是正确的，而在解析 `#f` 这个字符串的时候就会出错，因为 `boolLiteral` 先尝试使用 `string "#t"` 这个 parser 来解析 `#f`，当它看到 `f` 这个字符时，发现无法和 `t` 匹配，就会返回错误，`boolLiteral` 将尝试第二个 parser，但此时 `string "#t"` 已经将 `#` 消耗掉了，使得当前的状态变为 `f`，当尝试第二个 parser `string "#f"` 时，会发现当前字符串开头不为 `#`，于是整个解析就失败了。为了应对这种情况，必须手动使其支持回溯，所幸 Megaparsec 提供了这个函数：

```haskell 
boolLiteral = try (string "#t" >> return (SBool True))
          <|> (string "#f" >> return (SBool False))
```

这样一来这个 parser 就能正确解析 Scheme 的 boolean 字面值了。这从一方面看确实有点不够智能，我们可能会希望有某个方法能够神奇地进行某些判断然后进行回溯，可惜这个特性并没有实现，但是自己加一个显式的回溯还是可以接受的，而且这个特性给了用户选择的自由，我们可以在不希望它自动回溯的时候取消自动回溯。

相对于上面这个可归为设计取向的问题，下面说的这个问题就大概算是其相对于解析器生成器的比较严重的缺陷了。这个缺陷是 parser combinator 无法很好地处理左递归文法，举个例子来说，在 JavaScript 中，由于函数是「一等公民」，所以一个函数的返回值可能是另一个函数，所以以下代码是合法的：

```javascript 
x = myFunc(a, b)(c)
```

这个语法可以描述为（省略 `expression` 的其他情况，用 `...` 占位）：

```
expression ::= ... | var_name | function_call | ...
function_call ::= expression "(" argument_list ")"
```

即一个表达式可能是一个一般的变量（例如：`myFunc`，`a`，`b` 和 `c`），也可能是一个函数调用（例如：`myFunc(a, b)` 和 `myFunc(a, b)(c)`），而一个函数调用是一个表达式后面接用括号包裹的参数列表。直接翻译过来 parser combinator 这边，就应该这么写：

```haskell 
data JSExpression = 
    ... 
  | JSVar String
  | JSFuncCall JSExpression [JSExpression] 
  | ...

expression = ... <|> try variable <|> try functionCall <|> ...

functionCall = do func <- expression
                  args <- between (symbol "(") (symbol ")") argumentList
                  return (JSFuncCall func args)
```

这里的代码也很好理解，Haskell 的 do-comprehension 类似于 Scala 的 for-comprehension，也是一个语法糖，不过即使不知道它将具体被翻译成什么样，这段代码所表述的内容也是非常清晰的。首先，`expression` 将依次尝试使用解析不同表达式的 parser 来解析输入文本，其中一种表达式是函数调用，用 `functionCall` 表示。`functionCall` 首先解析一个函数，这个函数是由一个表达式求出来的，所以是解析一个 `expression`，将其命名为 `func`，然后再解析一个由括号包裹的参数列表，将其命名为 `args`，最后将 `func` 和 `args` 传入 `JSFuncCall` 构造器，构造并返回一个 `JSExpression` 类型的值。其中使用到了 `between` 组合子，它用来构建一个被另外两个 parser 包裹的 parser。它依次使用第一个、第三个、第二个 parser 来解析文本，当三个 parser 都成功解析时，返回第三个 parser 的返回值，否则解析失败。我们希望的是它在解析上述的文本时能产生这样的结果：

```haskell 
JSFuncCall 
  (JSFuncCall (JSVar "myFunc") [JSVar "a", JSVar "b"]) 
  [JSVar "c"]
```

意思很清晰，也很直观，但问题在于这段代码会陷入无穷递归。`expression` 的解析需要调用 `functionCall` 来解析输入，而 `functionCall` 本身又先要调用 `expression`。同样地，以下 JavaScript 表达式也会陷入同样的问题：

```javascript 
arr[1][2][3]
obj.field1.method1()
```

所以实际上在上述情况下并不能非常方便地直接翻译语法构建 parser，需要自行进行一些转换。对于上述的情况来说，一个可行的写法是：

```haskell 
expression = do 
  varExpr <- variable
  tailExpression varExpr

expressionTail expr = checkAhead expr <|> return expr

checkAhead expr = do 
  c <- lookAhead (oneOf "[(.")
  case c of
    '[' -> brcketField
    '(' -> funcCall
    '.' -> dotField
  where
    brcket = between (symbol "[") (symbol "]") 
    parens = between (symbol "(") (symbol ")")
    argumentList = sepBy expression (symbol ",")
    brcketField = do inside <- braket expression
                     expressionTail (JSBracketField expr inside)
    funcCall    = do inside <- parens argumentList
                     expressionTail (JSFuncCall expr inside)
    dotField    = do after <- expression
                     expressionTail (JSDotField expr after)
```

相比起直接翻译出来的代码，这个手工转化的代码就显得有点长，首先，`expression` 会去解析一个变量表达式（注意在 JavaScript 中一个函数和一个一般的变量在表示上并没有什么区别），如果成功了，就将这个表达式传入 `expressionTail` 来解析表达式的尾部。`expressionTail` 接受一个表达式作为参数，它需要调用 `checkAhead` 来先检查表达式后面有没有跟 `'[', '(', '.'` 这三个字符中的任意一个，如果没有，`checkAhead` 将会失败，`expressionTail` 将会返回刚刚传入的表达式本身。如果检查到后面确实有跟 `[(.` 三个字符中的一个，那就要根据这个字符是什么来进行不同的，例如看到 `(` 就去解析函数调用的情况。这里将递归调用 `expressionTail`，因为可能会有连续的调用。拿刚刚的 `myFunc(a, b)(c)` 函数调用来说，运行的过程如下：

1. `expression` 调用 `variable` 解析出 `myFunc` 构建了 `JSVar "myFunc"`，并将其传入 `expressionTail` 中；
2. `expressionTail` 调用 `checkAhead` 并将表达式传入 `checkAhead`；
3. `checkAhead` 向前看一个字符，发现这个字符是 `(`，它调用 `funcCall` 来进行解析；
4. `funcCall` 解析了用括号包裹的参数列表，参数列表是一个由 `,` 分割的表达式列表；
5. `funcCall` 将解析出来的数据和传入的表达式一起，构造器构建了一个表达式 `JSFuncCall (JSVar "myFunc") [JSVar "a", JSVar "b"]`，并将其传入 `expressionTail`，进行递归调用；
6. 重复步骤 2 ~ 5，`funcCall` 构建了 `JSFuncCall (JSFuncCall (JSVar "myFunc") [JSVar "a", JSVar "b"]) [JSVar "c"]` 并再次递归调用 `expressionTail`；
7. `expressionTail` 调用 `checkAhead`，而 `checkAhead` 发现前面没有 `[(.` 中的任意一个，于是返回错误，于是 `expressionTail` 将尝试第二个分支，而第二个分支是直接将接收到的表达式返回，所以最终的结果就是 `JSFuncCall (JSFuncCall (JSVar "myFunc") [JSVar "a", JSVar "b"]) [JSVar "c"]` 了。

这里使用了一些新的组合子，`lookAhead` 接收一个 parser 然后尝试用这个 parser 进行解析，如果这个 parser 解析成功了，那么 `lookAhead` 就会成功并返回解析结果，但是并不会消耗输入；`oneOf` 接收一个字符串，并匹配其中任意一个字符；`sepBy` 即 separated by，接收两个 parser，对输入用第一个 parser 解析多次，并在每两次之间使用第二个 parser 解析一次，即第一个 parser 用于解析内容，第二个 parser 用于解析分隔符，最终返回的结果是一个由第一个 parser 解析出的内容组成的列表。

这里虽然依然存在递归调用，但是并不会陷入之前所说的无穷递归的情况，因为现在在递归调用 `expressionTail` 之前，会先查看下一个字符，只有当需要递归解析的时候才会递归解析，只要没有看到需要递归解析的情况，解析器就会将结果返回。

说到这里，parser combinator 看来也没什么特别的，不过是换了种方式进行描述而已，而且功能上还如此受限，在一些情况下需要手动增加不少代码。既然都是去调用接口，为何不直接用生成器来生成一个解析器供我们调用呢？下面就要说明，parser combinator 不仅比较易于使用，而且自己从头开始构建一套 parser combinator 也并非难事。虽然一个错误提示优秀，功能强大的 parser combinator 库并不容易实现，但是基础的功能还是比较容易做的。

# 构建一套简易的 Parser Combinator

下面用 Scala 说明如何构建一套简易的 parser combinator [^fpscala]。

[^fpscala]: Paul Chiusano, Rúnar Bjarnason - Functional Programming in Scala

前面直接使用了 Megaparsec 这个库，它通过提供基础的组合子封装了 parser 本身的实现，以至于前面一直在说某某函数会返回一个 parser，用某某 parser 会解析出什么，但却一直不知道这个 parser 是个什么东西，它的具体表示是什么。事实上，它的具体表示可以有很多种，甚至对于大多数组合子而言，这个表示都是不重要或者不可见的，Megaparsec 就可以灵活地选择解析 `String`、`ByteString`、`Text` 等，这里为简单起见，将 parser 表示为一个需要一个类型参数 `A` 的 `trait`，带有一个 `parse` 方法，能将输入的字符串解析为 `A` 类型的值，当然，由于解析可能失败，所以返回值的类型不是 `A`，而是 `Option[A]`。当然，`Option[A]` 仅仅有 `Some[A]` 和 `None` 两种情况，也就是说，对于解析错误的情况，错误信息被直接抛弃了，但对于整体实现思想而言，这个显得不那么重要。显然，要做到能够组合不同的 parser，光是有一个 `parse` 方法还不够，总还是需要有某个东西能记录下当前解析的状态，所以还要添加另一个方法，姑且命名为 `run` 吧，写出来的代码是这样的：

```scala 
trait Parser[+A] {
  def parse(input: String): Option[A]
  def run(state: State): Result[A]
}
```

那么 `State` 和 `Result` 应该怎么表示呢？首先，解析的状态就只和两件事有关，一个是整个输入字符串是什么，另一个是当前解析的位置，定义如下：

```scala 
case class State(input: String, cursor: Int) {
  def moveForward(n: Int): State = copy(cursor = cursor + n)
  def currentInput: String = input.substring(cursor)
}
```

这里还给 `State` 加了两个方法，`moveForward` 用于方便地移动游标的位置，`currentInput` 用于方便地获取输入字符串中尚未被解析的部分。

至于 `Result`，它应该有两种情况，一个用于表示成功情况，一个用于表示失败情况：

```scala 
sealed trait Result[+A] {
  // convert Result to Option
  def toOption: Option[A] = this match {
    case Success(res, _) => Some(res)
    case Failure(_) => None
  }
}
case class Success[A](res: A, consumed: Int) extends Result[A]
case class Failure(cursor: Int) extends Result[Nothing]
```

对于成功的情况，我们想要知道解析出了什么结果，同时，我们还想知道本次解析消耗掉了多少个字符，以便于确定后续的解析从何处开始，对于解析失败的情况，我们则简单地记录当前游标的位置即可。由于我们的最终结果要被定为 `Option[A]` 类型，这里给 `Result[A]` 添加一个方法方便地转为 `Option[A]`。有了 `State` 和 `Result` 的实现，我们已经可以知道 `parse` 方法怎么表示了，它就是从 0 开始解析输入的字符串：

```scala 
// defined inside Parser[A] trait

// run from 0, and convert the result to Option type
def parse(input: String): Option[A] = run(State(input, 0)).toOption
```

下面提供最基础的组合子。什么是基础的组合子？这是一个需要考虑的问题，在写之前，先定义一个 `Parser` trait 的伴生对象，用来存放这些组合子：

```scala
object Parser {
  // convert a function to a Parser
  def apply[A](func: State => Result[A]): Parser[A] = 
    new Parser[A] { 
      def run(state: State): Result[A] = func(state)
    }
}
```

这个伴生对象提供一个 `apply` 方法，用于方便地构建 parser。受益于 Scala 的语法糖，调用 `apply` 方法时方法名可以省略，所以我们可以写：

```scala 
val parser = Parser(someFunc)
```

现在继续思考基础的组合子有什么，这个可以先随便想，越简单越基础越好，之后可以再进行重构，比如容易想到的：无论何时都成功的组合子、无论何时都失败的组合子、解析出任意字符的组合子、解析出一个特定字符的组合子等等，我们先将其添加进 `Parser` 对象中：

```scala 
// defined inside Parser object

// always succeed without consuming any input
def success[A](a: A): Parser[A] = Parser(state => Success(a, 0))
// always fail without consuming any input
def failure: Parser[Nothing] = Parser(state => Failure(state.cursor))
/**
 * if currentInput is empty string, then the parsing will fail, otherwise,
 * it will succeed and consume one char
 */
def item: Parser[Char] = Parser(state => state.currentInput match {
  case "" => Failure(state.cursor)
  case s => Success(s.head, 1)
})
/**
 * if the head char of the input string equals to the given char, 
 * the parsing will succeed, otherwise, it will fail
 */
def char(c: Char): Parser[Char] = Parser(state =>
  state.currentInput.headOption match {
    case Some(h) => 
      if (h == c) Success(c, 1) else Failure(state.cursor)
    case None => Failure(state.cursor)
  }
)
```

现在考虑如何解析一个特定的字符串呢？只要将一个字符串看作许多字符，然后用 `char` 组合子依次解析即可。此时就涉及到了之前提到过的 for-comprehension 了：

```scala 
// defined inside Parser object

def string(s: String): Parser[String] = {
  
  def charList(lst: List[Char]): Parser[List[Char]] = lst match {
    case Nil => success(Nil)
    case notNil => for {
      // parse the first char
      h <- char(notNil.head)
      // parse tail chars
      t <- charList(notNil.tail)
    } yield h::t // append the char at the head of tail list
  }
  
  for {
    // parse chars
    lst <- charList(s.toList)
  } yield lst.mkString // convert the char list and build result string
}
```

这里先在 `string` 内部定义了一个 `charList`，这用于解析出一个特定的字符列表，解析的方式是先查看字符列表是否为空，对于空列表，当然是返回一个 `success(Nil)` 了，因为无论解析什么输入字符串都应该可以成功地解析出一个空列表。如果不为空，则使用 `char` 组合子解析列表头字符，如果成功解析出了列表的头字符，那么就继续调用 `charList` 依次解析列表尾的全部字符。`string` 将输入字符串转为了字符列表，然后输入给 `charList`，再将解析出的结果拼接为一个字符串。

正如在 [使用 Future 进行并发编程](/articles/handle-concurrency-using-future) 一文中提到的那样，想要在 Scala 中使用 for-comprehension，我们需要给 `Parser` trait 提供几个方法，其中最基础的是 `flatMap` 方法，`map` 和 `filter` 都可以基于 `flatMap` 来定义：

```scala 
// defined inside Parser[A] trait

// type of ??? should be Parser[B]
def flatMap[B](func: A => Parser[B]): Parser[B] = ???

def map[B](f: A => B): Parser[B] = flatMap(a => success(f(a)))
  
def filter(predicate: A => Boolean): Parser[A] =
  flatMap(x => if (predicate(x)) success(x) else failure)
```

`flatMap` 应该如何定义？这时候先看一下它的类型声明，它接收一个类型为 `A => Parser[B]` 的函数，返回一个类型为 `Parser[B]` 的对象。那么，能产生这个 `Parser[B]` 的结果的方式只有通过调用这个函数以及直接构建一个 `Parser[B]` 类型的对象两种，但是我们此时没有任何 `A` 类型的值，所以我们并没有办法去调用这个函数，所以我们选择直接构建这个对象：

```scala
// type of ??? should be State => Result[B]
def flatMap[B](func: A => Parser[B]): Parser[B] = Parser(???)
```

`Parser` 的 `apply` 方法需要一个类型为 `State => Result[B]` 的函数，所以我们写成这样：

```scala
// type of ??? should be Result[B]
def flatMap[B](func: A => Parser[B]): Parser[B] = Parser(state => ???)
```

现在我们需要构建一个 `Result[B]` 类型的对象，先看看我们有什么东西。我们现在有一个 `A => Parser[B]` 类型的函数 `func`，有一个 `State` 类型的对象 `state`，别忘了还有自身的一个 `run` 方法，它的类型是 `State => Result[A]`。拥有 `State` 和 `State => Result[A]` 意味着我们可以构建一个 `Result[A]`，拥有 `Result[A]` 意味着我们可以获得 `A`；拥有 `A` 和 `A => Parser[B]` 意味着我们可以获得 `Parser[B]`，拥有 `Parser[B]` 和 `State` 意味着我们可以获得 `Result[B]`。非常好！

```scala 
// defined inside Parser[A] trait

def flatMap[B](func: A => Parser[B]): Parser[B] = Parser(state => 
  run(state) match {
    case Success(a, c1) =>
      val newState = state.moveForward(c1)
      func(a).run(newState) match {
        case Success(b, c2) => Success(b, c1 + c2)
        case f@Failure(_) => f
      }
    case f@Failure(_) => f
  }
)
```

`flatMap` 接收一个类型为 `A => Parser[B]` 的函数，产生一个新的 parser。在解析输入字符串时，它先用原 parser 来进行解析，如果成功，就将结果传给这个函数，产生一个新的 parser，再用这个 parser 解析余下的输入，并以该结果为最终结果。

很有趣的一件事是，一般而言，我们在实现一个方法时，首先想的是这个方法是有什么功能，这个功能应该如何拆分，然后再考虑每个部分如何实现。而在 `flatMap` 的实现过程中，我们不是根据 `flatMap` 的功能来实现它的，我们甚至不需要管它究竟要做什么，仅仅依靠类型就大概导出了一个大致的实现思路，然后根据这个思路将其实现。我们之所以可以这样做是因为，我们整个的计算过程都没有涉及到赋值等副作用，所以一个函数能做的事情就是接收一个值然后再返回一个值，而一个值全部的特点都在于它的类型，于是我们就可以通过类型声明推演出大致实现。

现在有了 `string` 这个常用实现，我们可以试试使用：

```
>>> string("hello").parse("hello world")
Some(hello)
>>> string("hello").parse("world")
None
```

现在回头看一下之前使用的组合子，看看还缺少什么？显然我们没法解析类似 `boolLiteral = string "true" <|> string "false"` 这样的分支的情况，所以我们添加新的组合子 `or`，并给它起一个别名 `|` 方便使用：

```scala 
// defined inside Parser[A] trait

def or[B >: A](other: => Parser[B]): Parser[B] = Parser(state =>
  // run first parser on current state
  run(state) match {
    // if parsing succeeds, return the result
    case s1@Success(_, _) => s1
    // if parsing fails, fetch the latest position
    case Failure(pos1) => {
      // run the second parser on the latest position
      other.run(state.copy(cursor = pos1)) match {
        case s2@Success(_, _) => s2
        case f@Failure(_) => f
      }
    }
  }
)

def |[B >: A](other: => Parser[B]): Parser[B] = or(other)
```

如果对 Scala 不了解，这里可能有两个东西看不懂，一个是参数 `other` 的类型声明，`=> Parser[B]` 将 `other` 声明为传名调用，也就是 `other` 的值将在使用它的时候才进行计算，这么做是因为在原 parser 解析成功的情况下，我们并不需要用到 `other`，在使用的时候才计算其值有利于节约计算资源，提高性能。另外一点是，这里的类型参数。`[B >: A]` 的意思是 `B` 是 `A` 的父类，为什么要这样写呢？写成这样：

```scala 
def or(other: => Parser[A]): Parser[A]
```

不是更加清晰吗？[协变、逆变与不变](/articles/covariant-and-contravariant) 一文曾提到 `Parser[+A]` 这样的写法将 `Parser` 声明为在类型参数 `A` 上协变，但是在 `or` 方法中，`A` 类型出现在了函数参数中这个逆变的位置，所以这会导致一个类型错误。因为用户可以向解决方法就是声明另一个类型参数 `B`，这个 `B` 是 `A` 的父类，那么，当原 parser 解析成功时返回的是 `A` 类型，由于 `A` 类是 `B` 类的子类，所以可以将该结果作为 `B` 类型的值返回，如果原 parser 失败而另一个 parser 成功了，那么返回的类型将是 `B`，自然也是正确的。现在我们可以试一下分支的情况了：

```
>>> (string("true") | string("false")).parse("false")
Some(false)
>>> (string("#t") | string("#f")).parse("#f")
None
```

我们又遇到了之前提到过的问题，我们需要支持显式的回溯，由于 Scala 的 `try` 是关键字，我们把这个回溯过程命名为 `attempt`：

```scala 
// defined inside Parser[A] trait

def attempt: Parser[A] = Parser(state => 
  run(state) match {
    case s@Success(_, _) => s
    // if parsing fails, the position will be set to the position of input state
    case Failure(_) => Failure(state.cursor)
  }
)
```

`attempt` 方法很简单，如果原 parser 成功解析出了结果，那么它直接将结果返回，如果出错了，它就将错误结果记录的位置抛弃，使用解析前的位置。观察 `or` 方法的实现我们可以发现，其实在这个实现里，我们完全可以做到隐式的回溯，只要这样写就可以了：

```scala 
// defined inside Parser[A] trait

def or[B >: A](other: => Parser[B]): Parser[B] = Parser(state =>
  run(state) match {
    case s1@Success(_, _) => s1
    // if parsing fails, ignore the latest position
    case Failure(_) => {
      // run the second parser on the original position 
      other.run(state) match {
        case s2@Success(_, _) => s2
        case f@Failure(_) => f
      }
    }
  }
)
```

其原因在于，我们的实现里状态是不可变的，每次修改实际上是创建了一个新的 `State` 类型对象，而 `or` 方法在发现原 parser 解析出错后，直接在原状态上使用 `other` parser 来进行解析。这时候就要考虑我们到底是支持显式的回溯还是隐式回溯呢？这视乎个人的设计选择，由于隐式回溯无法做到显式回溯不加 `attempt` 的那种效果，为了给用户更高的自由度，这里选择支持显式回溯。不管怎么说，现在我们可以正确进行回溯了：

```
>>> (string("true") | string("false")).parse("false")
Some(false)
>>> (string("#t").attempt | string("#f")).parse("#f")
Some(#f)
```

写到这里，我们可以回头看一下基础的组合子有没有问题。可以发现我们可能常常会用到 `string`，但 `string` 的效率非常低，它将字符串转为列表来进行处理，然后再将解析出来的字符组合成字符串。一个显然的优化方案是不去保存中间的那个字符列表，因为没有意义，我们已经知道如果成功，最后要返回什么了：

```scala 
// defined inside Parser object

def string(s: String): Parser[String] = {
  
  def charList(lst: List[Char]): Parser[Unit] = lst match {
    case Nil => success(())
    case notNil => for {
      _ <- char(notNil.head)
      _ <- charList(notNil.tail)
    } yield ()
  }
  
  charList(s.toList).map(_ => s)
}
```

但这个实现还是很有问题，由于 for-comprehension 将被展开为一个嵌套的 `flatMap` 调用，所以对于一个长的字符串，展开的 for-comprehension 可能导致栈溢出。在 Haskell 中，如果要处理字符串，将用于解析一个特定字符的 parser 作为基础组合子并用其构建解析特定字符串的 parser 是合理的，因为 Haskell 将字符串表示为字符列表。但在 Scala 中，`String` 类型是一个独立的类型，拼接、分割字符串会产生一个新的字符串，再加上栈溢出的问题，所以，这里选择将解析一个特定字符串的 parser 做成基础的组合子：

```scala 
// defined inside Parser object

def string(s: String): Parser[String] = Parser(state =>
  if (state.currentInput.startsWith(s)) Success(s, s.length)
  else Failure(state.cursor)
)
```

如果此时再回头看一下，会发现我们不需要单独定义 `char` 了，因为它可以被组合出来：

```scala 
// defined inside Parser object

def char(c: Char): Parser[Char] = string(c.toString).map(_.head)
```

当然，具体使用什么表示方式是可以自由选择的，我们也可以选择其他组合方式，例如：

```scala 
// defined inside Parser object

def char(c: Char): Parser[Char] = item.filter(_ == c)
```

除此之外，我们还可以发现之前用过的一些组合子也是可以直接被组合出来的，例如 `>>` 这个可以将左侧 parser 的成功结果直接舍弃再使用右侧 parser 来解析输入字符串的组合子：

```scala 
// defined inside Parser[A] trait

def skipLeft[B](pr: Parser[B]): Parser[B] =
  for {
    _ <- this
    r <- pr
  } yield r

def skipRight(pr: Parser[Any]): Parser[A] =
  for {
    l <- this
    _ <- pr
  } yield l
  
def >>[B](pr: Parser[B]): Parser[B] = skipLeft(pr)
def <<(pr: Parser[Any]): Parser[A] = skipRight(pr)
```

再考虑一下使用场景，我们还可以构建更多一些简单而常用的组合子，例如解析出单个空格字符的 parser，或者是之前使用过的 `between` 等等：

```scala 
// defined inside Parser object

val space: Parser[Char] = item.filter(_.isSpaceChar)

// defined inside Parser[A] trait

def between(pl: Parser[Any], pr: Parser[Any]): Parser[A] = pl >> this << pr
```

在我们不断构想新的组合子的时候就会发现目前设计的不足，例如，当我们想要解析出多个空格字符的时候就发现这个需求难以实现，所以我们需要添加一个 `some` 方法，用于将一个 parser 应用一次或多次。`some` 应该如何实现？将一个 parser 应用一次或多次的意思就是先将 parser 应用一次，再应用零次或多次，所以我们需要一个 `many` 方法用于将一个 parser 应用零次或多次。`many` 又要如何实现？将一个 parser 应用零次到多次的实现可以是：如果能够应用一次到多次，我们就直接使用 `some`，如果失败，就直接返回空列表。所以我们可以这样写：

```scala 
// defined inside Parser[A] trait 

def some: Parser[List[A]] = for {
  a <- this
  as <- many
} yield a::as

def many: Parser[List[A]] = some.attempt | success(Nil)

def + : Parser[List[A]] = some
def * : Parser[List[A]] = many
```

现在我们可以这样写了：

```
>>> ((space+) >> string("hello")).parse("hello")
None
>>> ((space+) >> string("hello")).parse("   hello")
Some(hello)
```

这个实现很简单，但是效率很低，原因就在于 `some` 和 `many` 之间的相互递归调用，这个过程 Scala 编译器无法优化。我们可以作出一些改进，与其将 `some` 和 `many` 互相实现，不如将其中一个作为基础的组合子，使用更高效的便于编译器优化的实现方式，这里将 `many` 重写，`some` 保持不变：

```scala
// defined inside Parser[A] trait 

def many: Parser[List[A]] = {
  
  def rec(s: State, acc: List[A], total: Int): (List[A], Int) =
    run(s) match {
      /** 
       * if parsing succeeds, 
       * append the current result at the head of accumulation,
       * then continue to parse the remain input string
       */ 
      case Success(a, c) => rec(s.moveForward(c), a::acc, total + c)
      /** 
       * if parsing fails, stop and return the accumulation result
       * note that we need to reverse the accumulation list
       */ 
      case Failure(_) => (acc.reverse, total)
    }

  Parser(state => {
    val (res, len) = rec(state, Nil, 0)
    Success(res, len)
  })
}
```

`many` 的内部定义了一个辅助函数，这个函数不断使用原 parser 来解析输入字符串，如果解析成功，就将解析结果记录在一个列表里，同时累积了移动的总字符数，当解析失败时就将这个结果返回。因为这个辅助函数是尾递归的，所以它可以被编译器优化成一个循环，这样就不需要担心栈溢出的问题了。

限于篇幅，其他更多的组合子此处不一一列出，大致的思想是可以理解了，接下来可以发挥想象力构建更多的组合子，在这个过程中，又会因为基础组合子不够而需要扩充基础组合子的规模，在扩充基础组合子后，有时又会发现原来的一些基础组合子不再「基础」，此时可以将这些组合子又用新的组合子重构，如此往复迭代。可以参考我放上 [Github](https://github.com/ZhiruiLi/SimpleParserCombinator) 的实例。 

# 面向组合子编程

前面说过，这篇文章介绍 parser combinator 的原因之一是这里面反映了相当有趣的编程思想。一般在使用面向对象的思路编程时，我们设计一个程序的方案一般是自顶向下来进行设计，先思考一个程序的功能需求，再思考一个程序如何根据功能拆分模块，这个过程中还要考虑各种继承关系、依赖关系等等，这是一个树形和块状的结构。这个设计方式针对需求，所以它非常有效，但是又由于它由需求出发又归于需求，所以它的整个设计是比较僵化的，对于变化的适应力不强，良好的架构可以一定程度上缓解这个问题，但是如果要构建另一个系统，往往就很难再复用前一个系统的代码。这常常导致多个系统中存在大量功能类似而不尽相同的模块，结果是到了相当长时间后发现太过混乱，不得不进行大规模重构，抽取相似模块共用。

而面向组合子的设计方式则从一个相反的方向进行设计，它的策略是先不管具体需求，确定大致的需求后就去构建一堆基础的组合子，然后再用基础的组合子组装成一些新的组合子，再用这些组合子再组装更复杂的组合子，慢慢接近需求。相比于自顶向下的设计这个过程更像是一个搭积木的过程，先有一些基础的小积木，然后拼成稍大一点的，然后再继续组合。由于每一次组合产生的组合子都相对独立，所以整个过程产生的不同粒度的组件都是可以复用的。比如 parser combinator 可以先组合出解析整数的 parser、解析出浮点数的 parser 等，然后这些 parser 都可以在其他不同的具体需求中使用。还有一个好处是，每一个组合子的规模都不大且独立，这非常利于测试和 debug，单元测试的划分在写组合子的时候就已经划分好了。

这种思路在函数式语言中非常常用，因为在面向对象的设计中，抽象的单元是对象，每个对象都包含了若干数据和方法，而函数式设计将每一个函数都作为独立的个体，数据被独立出来由函数来操作，这个抽象粒度比对象要小得多。所以每个小函数都难以承载完整的逻辑，一个复杂的逻辑需要由许多组件来组合完成，而这些组件本身又由更小的组件组合而来。

注意到另一个优点是我们通过组合的方式构建新组合子的时候，我们并不需要知道我们所用的组件的具体实现方式，我们构建了一个新的 parser，但我们可能甚至连 parser 的实现都不知道。接触 parser 具体实现的只有最基础的那些组合子而已，除此之外都不需要。这就给程序带来了很大的灵活性，可以很容易变更其实现，例如，我们可以为结果添加错误信息，或者是改变 parser 所接受的输入类型等等，这些改变对于上层的组合子而言是不可见的。

当然，这个设计策略也不是尽善尽美，它的一个显然的缺点在于设计组合子的过程本身是相对自由而不受限或很少受限于需求的，在设计基础组合子的时候你根本不知道这东西到底能不能组成最终的结果，有时可能会设计出一堆组合子后发现根本没有办法接近结果，这就比较麻烦了，尤其是对于需求明确的场合。所以这里只是介绍了其中思想，具体开发策略的选择还是要根据实际情况而定。

# 参考
