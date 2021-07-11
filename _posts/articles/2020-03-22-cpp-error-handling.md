---
layout: post
title: 另一种 C++ 程序错误处理方式
excerpt: >-
  C++ 是一个很灵活的语言，这把双刃剑一方面使得 C++ 有很强大的表达能力，但也使得其编程风格相当混乱，就连错误处理到底是使用错误码还是异常都常常争论不休。在这篇文章中，我将聊一下 C++ 错误处理的方式优劣，以及我们团队是如何进行 C++ 错误处理的。
categories: articles
author: zrl
date: 2020-03-22
modified: 2020-03-22
tags:
  - C++
  - Go
comments: true
share: true
---

C++ 是一个很灵活的语言，这把双刃剑一方面使得 C++ 有很强大的表达能力，但也使得其编程风格相当混乱，就连错误处理到底是使用错误码还是异常都常常争论不休。例如在 C 中我们默认用错误码处理错误，而在 Python、Java 中， 则默认用异常来处理错误。而在 C++ 中，使用这两种形式的错误处理形式都有，而目前来看，在我所在的团队中，除非是外部库，否则基本都是使用错误码。在这篇文章中，我将聊一下 C++ 错误处理的方式优劣，以及我们团队是如何进行 C++ 错误处理的。

## 错误码的问题

在我们的工程实践中，错误码首先带来的问题是代码中充斥着大量的 `-1`、`-2`、`-10000` 这样的错误码，这样错误码在日志中出现总是让人头痛，在代码中一搜就出来数不清的匹配项，根本无法定位问题。当然，你可能会说，这个主要是开发者水平参差不齐和开发规范不够严明的问题，我们可以通过全局统一错误码来解决问题。这当然是一个合理的反对意见，但问题是，即便确定要全局统一错误码，但这个全局统一错误码应该做到什么层级呢？

例如，在我们的后台采用了微服务架构，那么一个很显然的处理方案就是全局统一错误码是在服务级别的，A 服务调用 B 服务的时候，通过错误码来获知调用过程中出了什么错误。但是，这事实上并没有解决问题，例如我们现在发现 B 服务给 A 服务返回了 `12345` 这个错误码，然后我们尝试查看 B 服务的代码，看看为什么会导致这样的错误：

```c++
enum Errors {
  kErrFailToCallSomeFunction = 12345,
};

int Handle(Req const &req, Rsp *rsp) {
  int ret = SomeFunction();
  if (ret) {
    ERRORLOG("call SomeFunction fail: %d\n", ret);
    return kErrFailToCallSomeFunction;
  }
  // ...
}
```

因为错误码统一，我们很快就发现了是调用 `SomeFunction` 失败导致了这个错误，然后我们知道了应该找 `call SomeFunction fail: ` 这条日志，可以我们一查阅这条日志就发现，`call SomeFunction fail: -1`，结果我们又回到了之前的问题，尤其是我们可能在 `SomeFunction` 中看到这样的实现：

```c++
int SomeFunction() {
  if (xxx) {
    return -1;
  }
  int ret = AnotherFunction();
  if (ret) {
    return ret;
  }
  return 0;
}
```

这就更麻烦了，我们根本不知道到底是 `SomeFunction` 返回了 `-1` 还是 `AnotherFunction` 返回了 `-1` 然后被向上传递。

如果要解决这个问题，那就只能让所有的错误码都统一了。这样，每一层都可以自由地向上传递错误码。不过这个做法也有很明显的缺陷。首先，要全局统一错误码到各个函数中，这件事情是很麻烦的，而太过麻烦的事情只会导致大家都不愿意去做，推广困难。另外，在很多时候，太过底层的错误码并没有太大意义，举个例子，请求超时，这样的错误直接向上传递就没什么意义，因为每一个远程调用的路径上都可能请求超时，如果随意向上传递，结果就是我们拿到了请求超时的错误码却无法定位是什么地方导致的这个错误，结果就是我们依然需要将错误码进行转换。

另外，使用错误码还会导致一个问题，就是错误码仅仅是一个数字，它可以表示错误发生的位置，却无法说明错误发生的原因和发生的环境，于是我们为了知道具体发生的错误，就需要在出错时打印一些错误日志记录一些当时的环境信息，但同时，由于在多级的调用中， 每一级信息总是不全的，这导致我们经常需要逐级调用都打印日志，来记录那一级才知道的信息。这不但使得错误日志经常出现大量冗余打印的情况，而且使得多框架环境下开发库变得困难，因为不同框架的日志库的接口可能不太一样，结果可能又需要搞一个 log 的 adapter 层来处理这种麻烦。

## 使用异常？

异常是 C++ 的一个重要特性，当年就是因为要加入这个特性，Bjarne Stroustrup 才放弃了将 C++ 转换为 C 再交给 C 编译器进行编译的处理方式，改为直接编译。也许我们没有意识到，但是异常其实遍布在我们的代码之中，只要我们使用了 STL，那么就必然接触过异常。例如我们常用的 `std::vector`，如果调用 `at` 函数越界了，那么就会抛出 `std::out_of_range` 异常。而 [ISOCPP](https://isocpp.org/wiki/faq/exceptions) 也是建议我们使用异常而不是错误码来处理错误。

在上述的 ISOCPP FAQ 中，指出了使用异常而不是错误码原因主要是[让代码简洁以及不容易忽视错误](https://isocpp.org/wiki/faq/exceptions#why-exceptions)，具体而言，有如下的几个原因：

- [相比 if-else 判断而言能提高代码质量、提高开发效率](https://isocpp.org/wiki/faq/exceptions#exceptions-eliminate-ifs)
- [在实际的复杂工程中能大幅度简化错误传递](https://isocpp.org/wiki/faq/exceptions#exceptions-avoid-spreading-out-error-logic)
- [区分 Good Path 和 Bad Path](https://isocpp.org/wiki/faq/exceptions#exceptions-separate-good-and-bad-path)
- [简化参数类型和返回类型](https://isocpp.org/wiki/faq/exceptions#exceptions-avoid-two-return-types)

上述的每一点确实都是返回码的不足之处，但是，异常真如这篇 FAQ 说的这么好吗？让我们一条条来看。

第一点，文章认为 if 语句很容易出错，而且增加测试的复杂度。但事实上，在使用错误码的时候，由于都是判定 `if (ret != 0)` 或是写为 `if (ret)`，因此这个判断写错的可能性并不是很高。而且即便是抛出异常，测试的时候难道就不需要测试异常分支了么？事实上也是需要的，至于文章所说的测试覆盖度的问题，下面会再讨论到。

第二点，文章认为简单情况下异常虽不能体现其优势，但是复杂情况下，异常能大幅度减化错误传递。在文章给出的例子中，`f1` 到 `f10` 层层调用，最终 `f10` 抛出的异常直接被 `f1` 捕获，代码比错误码的形式简化了非常多。这看起来很美好，但是，我们此时应该记得一句话：

> For every complex problem there is an answer that is clear, simple, and wrong.
>
> —— H. L. Mencken

对于复杂业务而言，这样处理错误的方式往往反而是不合适的，而游戏正是一个业务逻辑相当复杂的场景，很少有「纯粹」的东西。正如前文所述，在每一层的函数调用中，往往会存在那一层调用才存在的一些环境变量，例如下面这个例子：

```c++
void F1() {
  try {
    F2();
  } catch (SomeException &e) {
    ...
  }
}

void F2() {
  auto ret1 = Fx();
  auto ret2 = Fy(ret1);
  F3(ret2);
}

void F3(X in) {
  if (...) { throw SomeException(); }
}
```

当 `F1` 捕获到异常时，我们会希望知道导致这个异常的原因是什么，此时由于异常的抛出点是在 `F3` 中， 因此它并不知道 `F2` 是如何计算出它的输入参数的。而 `F2` 中的 `ret1` 却很有可能是查问题的关键。这意味着，我们可能在使用异常的情况下需要层层捕获异常，逐层添加错误信息，以便查找问题。实际操作起来，这反倒使得代码变得啰嗦了。

第三点和第二点其实是类似的，在一个复杂的业务中，想要 Good Path 直接走完其实是不现实的，比如下面这个简单例子：

```c++
void F() {
  try {
    GResult gg = G(); // 1
    HResult hh = H(gg);
    IResult ii = I(hh);
    JResult jj = J(ii); // 2
  } catch (FooError &e) {
    ...
  } catch (BarError &e) {
  	...
  }
}
```

这里有两个问题，首先，和刚刚类似，我们在某一个异常出现的时候希望打印环境信息，例如 `J(ii)` 这个调用其实和 `ii`、`hh`、`gg` 都有一定的关系，因此，我们会希望在 `J` 抛出异常的时候，将前面这几个变量打印出来，而在 Good Path 和 Bad Path 分离的状况下，我们做不到这一点。第二，`J` 可能本身调用了 `G`，而 `G` 会抛出 `FooError`，那么我们捕捉到 `FooError` 的情况下，就并不清楚代码究竟执行到 1 处还是 2 处。为了解决这两个问题，结果就是我们可能需要写出层层嵌套的 try-catch 代码，这使得使用异常的代码甚至比 if-else 的 early return 更加难读。而且当异常出现多层的嵌套的时候，异常分支会和 if-else 分支一样多，测试代码覆盖的难度并不会因为使用异常而下降。

至于第四点，返回错误码确实无能为力，这一点在本文后面会提到我们的解决方法。

## 我们应该如何选择方案

在思考我们团队的 C++ 错误处理改进方案的时候，除了要考虑方案本身的优劣，还需要面对团队自身存在的问题，我相信我们团队遇到的问题许多团队也或多或少会遇到。

首先，我们团队的代码历史悠久，C++ 曾被仅仅当作 C with class 来使用，其中存在许多不是异常安全的代码，贸然使用异常很可能会造成修改存量代码时开发新功能时引入很多难以排查的问题。

另外，我们的工程使用了自研的 C++ 协程框架，而这个框架事实上是很难和异常特性融合使用的。

而且，不管是对于存量代码的风格还是团队成员的开发习惯而言，使用错误码传递错误都是一个默认的做法。如果使用异常，那么会导致代码中长期两种错误处理风格混用，不但让人不知如何处理错误，而且也不太会受到团队成员的支持。

所以，我们在选择改进方案的时候，至少要考虑如下几点限制：

- 要能和现有的处理方式比较好地融合，能平滑过渡
- 不要过分挑战已有的开发习惯，要让团队成员易于接受新的方案
- 最好能用起来比较简单，不要反而降低了开发效率

而这个方案希望能够解决如下几个问题：

- 能够避免代码中处处打日志记录当时环境信息的情况
- 在下层发生错误时，上层能够在转换成别的错误的同时，保留下层的错误信息
- 上层除了能够判定有错误发生，还要能判定发生了哪一个错误

最近我们团队开始试水使用 Golang 来开发一些服务，在这个过程中，发现 Golang 风格的错误处理虽然各种被喷，但是它也有其可取之处，尤其是它相当符合我们团队的开发习惯，所以我们尝试在 C++ 中实现类似的处理方式。

## 我们的解决方案

我们参考 Golang 的错误处理模式用 C++ 实现了一个简单易用 [GErr](https://github.com/ZhiruiLi/GErr) 库，使用了这个库，之前我们这样的函数：

```c++
int Function1(int i) {
  if (i < 0) {
    return -1;
  }
  int ret = api.Call(i);
  if (ret != 0) {
    ERRORLOG("call api fail, i = %d, ret = %d\n", i, ret);
    return -2;
  }
  return 0;
}
```

可以变为这样：

```c++
gerr::Error Function1(int i) {
  if (i < 0) {
    return gerr::New("i = {} < 0 is illegal", i); // 基于 fmt 库格式化字符串
  }
  int ret = api.Call(i);
  if (ret != 0) {
    return gerr::New(ret, "call api fail, i = {}", i); // 携带错误信息和错误码
  }
  return nullptr; // 返回 nullptr 代表没有错误发生
}
```

这里，对于简单的情况，我们可以通过 `gerr::New` 来创建一个匿名的错误，携带一段错误信息。对于调用者，可以从 `if (ret != 0) {...}` 改为 `if (err != nullptr) {...}`，来判断是否出现错误，如果你习惯写 `if (ret) {...}` 这样的形式，那么你甚至不需要修改，直接写 `if (err) {...}` 即可，代码非常类似。注意到，`int ret = api.Call(i);` 是一个旧有的 API，返回了一个错误码，而 `gerr::New` 可以简单封装了底层返回的错误码的同时附带了错误信息。

如果底层返回了一个 `gerr::Error`，那么，上层可以在判断是否有错误的基础上，包装当时的一些环境信息：

```c++
gerr::Error MyFunction2(int i) {
  auto err = MyFunction1(i);
  if (err != nullptr) {
    return gerr::Wrap(err, "fail to call 1, i = {}", i); // 仅附加错误信息
  }
  err = MyFunction1(i + 1);
  if (err != nullptr) {
    return gerr::Wrap(err, 123456, "call fail, i = {}", i); // 附加错误码和信息
  }
  return nullptr;
}
```

这样一来，我们就不需要每一层打印错误日志，直接在上层统一打印即可，甚至如果框架支持，直接由框架来打印，都不是问题。

处理好错误的传递之后，我们需要知道如何判定底层返回的特定错误，例如我们在拉取用户数据的时候，会判定一个特定的错误码，即用户数据不存在。如果错误码为 0 或者是代表用户数据不存在的错误码，那么我们就不认为这是一个错误，可以继续往下处理。在 `GErr` 中，我们可以用 `DEFINE_ERROR` 和 `DEFINE_CODE_ERROR` 来定义错误：

```c++
// 定义一个错误类型 MyError1，附带错误信息
DEFINE_ERROR(MyError1, "my error message1"); 
// 定义一个错误类型 MyError2，附带错误码和错误信息
DEFINE_CODE_ERROR(MyError2, 1000001, "my error message2"); 

// 使用的时候需要用静态函数 E 来获取错误值
gerr::Error MyFunction3(int i) {
  if (i < 0) { return MyError1::E(); }
  auto err = MyFunction1(i);
  if (err != nullptr) {
    // 自定义的错误类型也可以 wrap 父错误
    return MyError2::E(err);
  }
  return nullptr;
}
```

可以看到，定义一个错误非常方便，相对于过往使用 `enum` 定义错误码的方式，这里并不会让开发者多写多少代码。而且由于这里定义的错误是带具体类型的，调用者也可以很容易进行错误类型的判定：

```c++
void MyFunction4(unsigned uin, int i) {
  auto err = SomeOtherFunction(uin, i)
  // 判定一个错误链条上是不是包含 MyError1 类型的错误，
  // 这里不需要先判定 err 是否为空，Is 函数是 nullptr 安全的，
  // 如果一个 err 是 nullptr，那么 Is 函数总会返回 false。
  if (gerr::Is<MyError1>(err)) {
    std::cerr << "error 1";
    return;
  }
  // 其他未知错误，直接打印一下
  if (err != nullptr) {
    // String 函数也是 nullptr 安全的，nullptr 会被格式化为 <nil>
    std::cerr << gerr::String(err); 
    return;
  }
}
```

前面提到了使用异常可以[简化参数类型和返回类型](https://isocpp.org/wiki/faq/exceptions#exceptions-avoid-two-return-types)，而在这里，我们的返回值被 `gerr::Error` 占据了，那我们要如何表达一个可能出错的返回值呢？在 Golang 中，可以通过多返回值来实现，这也是 Golang 的惯用法。C++ 不支持这样的语法，虽然我们可以直接用 `std::pair` 或 `std::tuple` 模拟多返回值，但是使用起来非常不便，到了 C++ 17 我们将可以用上 [Structure Binding](https://en.cppreference.com/w/cpp/language/structured_binding) 语法，能够使多返回值的使用更加方便。但目前看来，我们将长期被锁在不完整的 C++ 11 版本上，因此在 `GErr` 的例子中，给出了一个 [`Try` 模板](https://github.com/ZhiruiLi/GErr/blob/master/examples/simpletry/try.hpp)的简单实现，使得我们可以这样写代码：

```c++
Try<int> SafeDiv(int a, int b) {
  if (b == 0) {
    return gerr::New("div 0");
  }
  return a / b;
}

// Usage
auto const t = SafeDiv(a, b);
if (t) { // 转为 bool 判断是否成功
  std::cout << "No error, result = " << t.Value(); // Value 获取值
} else {
  std::cout << "Error occurs: " << gerr::String(t.Error()); // Error 获取错误
}
```

更多关于 `GErr` 的用法，可以参考 [README](https://github.com/ZhiruiLi/GErr/blob/master/README.md) 和其中的 examples。

## 总结

在这篇文章中，我们讨论了 C++ 的两种主要错误处理方式，以及我们团队遇到的问题，并提出了一个简单可行的解决方案。通过使用 `GErr`，我们可以较为平滑地从旧的基于错误码的错误传递方式切换到新的方式上，在一定程度上解决我们现有的问题，既保证了使用心智负担足够小，又足够简洁易用，且避免了到处打印错误日志的情况，上层也能很轻松地对下层的错误进行判断和错误的打印。
