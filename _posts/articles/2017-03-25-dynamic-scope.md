---
layout: post
title: 静态作用域和动态作用域
excerpt: "解释静态作用域和动态作用域的区别，并对其进行实现与模拟。"
categories: articles
author: zrl
date: 2017-03-25
modified: 2017-03-26
tags:
  - C++
  - Theory
  - Haskell
  - Lisp
comments: true
share: true
---

# 静态作用域和动态作用域

所谓作用域规则就是程序解析名字的方法。如果一个变量的名称不在当前作用域内，则这样的变量称为 unbound variable，例如有一个函数 `(lambda () (+ a a))`，`a` 就是一个 unbound variable，在当前作用域内我们无法找到这个变量。那么调用这个函数的求值结果是什么呢？显然要根据 context 来确定，对于 unbound variables 的解析，从解析的时机来划分，有两种规则，一种是「静态作用域（static scope）」也被称为「词法作用域（lexical scope）」，另一种是「动态作用域（dynamic scope）」[^scope-wiki]。

对于现在流行的大多数语言来说，其作用域规则都是静态作用域规则，例如 Java、C++ 等，其特点根据函数定义处的环境解析里面用到的 unbound variables。仅有少数语言使用动态作用域规则，例如 Emacs Lisp，其函数内的 unbound variables 的解析是根据函数被调用时的环境来决定的。举例而言，对如下的表达式求值：

```scheme
(let ((a 1))
  (let ((doubleA (lambda () (+ a a))))
    (let ((a 2))
      (doubleA))))
```

如果采用静态作用域规则，这个表达式的值为 `2`，而如果采用动态作用域规则，其值则为 `4`。原因是当 `doubleA` 被定义时，可以在外层作用域找到 `a = 1`。而对于采用动态作用域的语言来说，`a` 的查找并不是在 `doubleA` 被定义的时候，而是在 `doubleA` 被调用的地方，此时 `a = 2`。当然，采用动态作用域规则的语言也会不断向外层作用域寻找名字，所以对下面这个表达式求值，无论是采用静态作用域规则还是动态作用域规则，其结果都是 `2`：

```scheme
(let ((a 1))
  (let ((doubleA (lambda () (+ a a))))
    (doubleA)))
```

那这两种规则哪种比较好呢？看被语言所采用的比例就知道，显然是静态作用域规则更好。其原因是在采用静态作用域规则的时候，对于函数的定义者来说，他可以通过阅读自己的代码很容易地知道他所使用到的变量当前绑定的具体实体是什么，而在使用采用动态作用域的语言时，则需要考虑这个函数被调用的时候该变量所对应的具体实体，这事实上是一种破坏封装的行为。举个例子，假设我们需要写几个对传入参数加一个数字的函数，例如 `(lambda (n) (+ n 1))`，那我们可能会希望对这组函数进行一个抽象，构建一个 `createAddN` 函数：

```scheme
(define createAddN
  (lambda (n)
    (lambda (x) (+ n x))))
```

那么我们现在可以这样定义和使用 `add1` 函数了：

```scheme
(let ((add1 (createAddN 1)))
  (add1 2))
```

这个表达式会如我们所愿地返回 `3`，现在似乎没什么问题，但如果我这样使用呢：

```scheme
(let ((add1 (createAddN 1)))
  (let ((n 2))
    (add1 n)))
```

显然我们还是希望这个表达式求值的结果为 `3`，这对采用静态作用域的语言来说仍然如此，但对于采用动态作用域语言的使用者来说这就有问题了，因为这个表达式将会返回 `4`。所以，对于函数的实现者来说他需要担心函数被使用的位置会出现重名造成的问题，对于函数的调用者来说他也要担心这个问题，结果就是在 Emacs Lisp 这样的语言里，函数的实现者往往会使用一个非常长的名字来命名变量，这显然不是什么好事。也许有时你确实需要这样的效果，但是在大多数情况下，动态作用域的规则是很容易造成大量难以排查的 bug 的，况且如果真的需要在静态作用域语言里实现这样的效果，传入参数是一个更加可靠且可控的做法。

# 分别实现两种作用域

下面要谈到的是对于一个解释器来说，这两种作用域应该分别怎么实现 [^eopl]，当然，刚刚也说了，动态作用域其实没什么好处，这么做其实纯粹是为了好玩。

首先说明需要用到的 ADT，先说程序里面用到的值和表达式应该如何表示，这里只列出我们关心的部分：

```haskell
data Val = IntVal Int
         | Closure ...
         | ...
         
data Expr = Variable String
          | LetBinding [(String, Expr)] Expr
          | Call Expr [Expr]
          | Lambda [String] Expr
          | ...
```

其中，`data Val` 就是在这个小语言中用到的值，由于只用到了整形和函数，所以这里只列了两个构造器：表示整形数的 `IntVal` 和表示 closure 的 `Closure`。对于表达式来说，用 `data Expr` 表示，这里就列了四个，它们分别是变量 `Variable`，let 表达式 `LetBinding`，函数调用 `Call` 以及 lambda 表达式 `Lambda`。求值的环境 `Env` 是作用域 `Scope` 的列表，而 `Scope` 本身则是表示为一堆名字与值的绑定列表：

```haskell
type Scope = [(String, Val)]
type Env = [Scope]
```

那么我们的求值函数就是：

```haskell
type TryVal = Either String Val

eval :: Expr -> TryVal
eval expr = eval' expr baseEnv

eval' :: Expr -> Env -> TryVal
eval' expr env = ...

-- basic functions such as `+`
baseEnv :: Env
baseEnv = [("+", ...), ...]
```

这里返回的结果是 `Either String Val` 类型的值，如果执行正常，那么返回值就是 `Right val` 的形式，如果出现异常情况，返回值就会是 `Left errorMessage` 的形式。现在来实现 `eval'` 这个函数，这个函数是整个求值器的核心。对于变量，求值方式是很显然的，就是在环境中找这个变量，如果找不到就返回错误信息：

```haskell
eval' (Variable name) env = evalVar name env

evalVar :: String -> Env -> TryVal
evalVar name [] = Left $ "Unbound variable: " `mappend` name
evalVar name (scope:scopes) = 
  maybe (evalVar name scopes) Right (lookup name scope)
```

而对 let 表达式的求值就是分别对名字绑定进行求值并添加进环境中，然后再对其 body 部分进行求值：

```haskell
eval' (LetBinding bindings body) env = evalLet bindings body env []

evalLet :: [(String, Expr)] -> Expr -> Env -> Scope -> TryVal
evalLet [] body env scope = eval' body (scope:env)
evalLet ((name, expr):bindings) body env scope = do
  val <- eval' expr env
  evalLet bindings body env ((name, val):scope)
```

对于静态和动态作用域而言，这两个表达式的求值方式都是相同的。它们的主要区别在于对函数调用的求值方式，前面描述了这个小语言中值的表示，但是没说 `Closure` 是如何表示的，我们在将一个 lambda 表达式求值为一个 closure 的时候不可以仅仅保留其参数列表和函数体，为了计算其中的 unbound variables，我们还需要捕获当前的环境，所以我们可以将其表示如下：

```haskell
data Val = ...
         | Closure [String] Expr Env
         | ...
         
eval' (Lambda params body) env = Right $ Closure params body env
```

静态作用域应该如何实现呢？如前所述，静态作用域的 unbound variables 的名字查找是在函数定义的地方进行的，所以对于调用表达式的求值我们需要这样做：

```haskell
eval' (Call (Closure params body capture) args) env = 
  evalCall params args body capture env []
eval' _ _ = Left "Calling a non-function value"

evalCall :: [String] -> [Expr] -> Expr -> Env -> Env -> Scope -> TryVal
evalCall [] [] body capture _ scope = eval' body (scope:capture)
evalCall (name:params) (expr:args) body capture env scope = do
  val <- eval' expr env
  evalCall params args body capture env ((name, val):scope)
evalCall _ _ _ _ _ _ = Left "Mismatched parameters and arguments"
```

在这里，我们先对传入的参数列表进行求值，并将其与对应的参数名进行绑定，这些绑定形成一个作用域 `scope`，如果形式参数和实际参数的数量不匹配就会返回错误。注意到与前面两种表达式的求值不同，现在求值有两个环境，一个是 `env`，另一个是 `capture`，其中，`env` 是程序运行到调用表达式时的环境，我们在这个环境中求出参数的值，`capture` 是 lambda 表达式在定义时捕获的外部环境，我们在这个环境中求 closure 的 body 的值，当然，参数绑定形成的作用域要被放在 `capture` 环境的开头。通过这个方式，我们就可以实现静态作用域了。当我们在当前作用域中找不到一个变量的绑定时，我们就会在捕获到的环境中向外查找，直到找到或是没有更外层的作用域为止。

动态作用域的解析则不同，我们虽然捕获了函数定义处的环境，但是我们需要先在函数被调用处的环境中进行名字查找，所以此时计算的方式需要改成这样：

```haskell
evalCall :: [String] -> [Expr] -> Expr -> Env -> Env -> Scope -> TryVal
evalCall [] [] body capture env scope = 
  eval' body ((scope:env) `mappend` capture)
evalCall (name:params) (expr:args) body capture env scope = do
  val <- eval' expr env
  evalCall params args body capture env ((name, val):scope)
evalCall _ _ _ _ _ _ = Left "Mismatched parameters and arguments"
```

我们将参数列表绑定形成的作用域放在当前求值环境 `env` 的开头，然后，为了使其在找不到名字的时候能在它被定义的作用域里查找，我们还要将捕获的环境 `capture` 接在末尾，这样形成的新环境就是 closure 的 body 的求值环境，此时其作用域就是动态的了。当我们在当前作用域中找不到一个名字时，我们会先查找函数被调用的空间。

# 在 C++ 中模拟动态作用域

上一节讲的是在解释器中实现两种作用域的方式，那如果我们就是想在现有的语言里模拟这个特性呢？这里就试着在 C++ 里模拟出类似的效果 [^cpp-dynamic-scope]。

其实说 C++ 完全是静态作用域语言是不完全正确的，C++ 的宏系统由于是直接展开，所以它是根据展开的位置来判定其值到底是多少，所以本身是类似于动态作用域的，例如：

```c++
#define ADD_N(x) ((x) + n)

auto add1(int x) {
  auto n = 1;
  return ADD_N(x);
}

auto add2(int x) {
  auto n = 2;
  return ADD_N(x);
}
```

但是，其使用是相当受限的，现在想要实现的是一个更加灵活的版本，大概要做到类似这样的效果：

```c++
auto foo() {
  dynamic_val string s;
  cout << s << endl;
}

auto bar() {
  dynamic_bind string s("bbb") {
    foo();       // output bbb
    // dynamic_val string s;   // compiler error: name conflict
    s = "ccc";
    foo();       // output ccc
    dynamic_bind string s("ddd") {
      foo();     // output ddd
    }
  }
}

auto main() -> int {
  dynamic_bind string s(3, 'a') {
    foo();       // output aaa
    bar();
    foo();       // output aaa
  }
  // foo();      // runtime error: no such binding
}
```

从上面的代码中可以看出要实现的效果大概有以下几点：

- 静态类型检查
- 防止重复绑定
- 允许嵌套绑定
- 作用域清晰，能和非动态绑定的代码很好地区分
- 尽可能接近本身的语法

下面就来进行实现。正如前文所述，动态作用域的实现其实是求值环境的动态绑定，要在一个静态作用域的语言中模拟出这个效果，我们可以自己用一个类管理这个环境。对于单一的变量来说，直接使用一个栈就可以了，当进行动态绑定的时候将值入栈，离开动态绑定的作用域时出栈。但我们还需要对名字的绑定，所以我们可以用一个 `map` 保存多个栈，大概类似这样：

```c++
namespace lang {
  template<typename T>
  class DynamicScope {
  public:
    static auto bindVal(const string& name, const T& val) {
      instancesMap[name].push_back(val);
    }
    static auto unbindVal(const string& name) {
      instancesMap[name].pop_back();
    }
    static decltype(auto) getVal(const string& name) {
      return instancesMap[name].back();
    }
  private:
    static std::map<std::string, std::vector<T>> instancesMap{};
  }
}
```

我们可以这样使用这个类：

```c++
auto foo() {
  cout << lang::DynamicScope<string>::getVal("x");
}

auto main() -> int {
  lang::DynamicScope<string>::bindVal("x", "aaa");
  foo();
  lang::DynamicScope<string>::unbindVal("x");
}
```

这种简单的实现有很多问题，例如，这段代码没有检查变量未绑定的情况，而且在绑定结束的时候我们需要手动去将变量解除绑定，这不仅意味着我们在绑定和解绑的时候必须输入完全正确的名字，而且还意味着这段代码不是异常安全的，我们如果在绑定调用和解绑调用之间有未捕获的异常，那么对象的作用域栈就会出错。为了解决这个问题，我们可以使用 RAII 的设计范式，这样实现我们的代码：

```c++
namespace lang {
  template<typename T>
  class DynamicScope {
  public:
    DynamicScope(std::string name, T&& val): name_(std::move(name)) {
      instances()[name_].push_back(std::move(val));
    }
    ~DynamicScope() {
      auto it = instances().find(name_);
      (*it).second.pop_back();
      if ((*it).second.empty()) {
        instances().erase(it);
      }
    }
    DynamicScope(const DynamicScope&) = delete;
    auto operator=(const DynamicScope&) = delete;
    static decltype(auto) instanceOf(const std::string& name) {
      using namespace std;
      auto it = instances().find(name);
      if (it == instances().end()) {
        throw runtime_error("Unbound variable: "s + name);
      }
      return (*it).second.back();
    }

  private:
    std::string name_;
    static decltype(auto) instances() {
      using namespace std;
      static map<string, vector<T>> instancesMap{};
      return (instancesMap);
    }
  };
}
```

这个版本的实现就相当完备了，这时，我们使用这个类的方式变成这样：

```c++
auto foo() {
  cout << lang::DynamicScope<string>::instanceOf("x");
}

auto main() -> int {
  lang::DynamicScope<string> scope("x", "aaa");
  foo();
}
```

但这样的实现比较暴露实现细节，而且写起来也不是特别自然，尤其是我们平常使用 `x` 来表示一个名字，而这里却需要使用 `"x"`，这看起来就与原本的代码比较割裂，这意味着如果我们需要声明局部的变量需要这样写：

```c++
auto x = lang::DynamicScope<string>::instanceOf("x");
```

这无疑是很难看的，我们还要自己小心确保 `x` 和 `"x"` 这两处的名字相同，以免将变量 `x` 绑定到错误的值上。而且，尽管我们很小心，这个写法还是不小心错了，因为这里我们不应该写 `auto` 而应该写 `auto&`，以便于我们能像对一般的变量赋值一样给动态绑定的变量赋值。也就是说我们应该这样写：

```c++
auto& x = lang::DynamicScope<string>::instanceOf("x");
x = "123";
```

我们可以考虑使用一点 C++ 的宏，来逼近我们一开始想实现的效果，用 `#` 能将 token 转为字符串，而 `##` 则能用于 token 的拼接 [^msdn-cpp-marco]：

```c++
#define DYNAMIC_VAL(_type, _name) \
  auto& _name = lang::DynamicScope<_type>::instanceOf(#_name)

#define BIND_DYNAMIC(_type, _name, ...) \
  do { \
    lang::DynamicScope<_type> \
      ___Dynamic_scope_##_type##_##_name(#_name, __VA_ARGS__); \
    DYNAMIC_VAL(_type, _name);

#define END_BINDING \
  } while(false);

namespace lang {
  template<typename T>
  class DynamicScope {
  public:
    template<typename... Args>
    DynamicScope(std::string name, Args&&... args): name_(std::move(name)) {
      instances()[name_].emplace_back(std::forward<Args>(args)...);
    }
    ~DynamicScope() {
      auto it = instances().find(name_);
      (*it).second.pop_back();
      if ((*it).second.empty()) {
        instances().erase(it);
      }
    }
    DynamicScope(const DynamicScope&) = delete;
    auto operator=(const DynamicScope&) = delete;
    static decltype(auto) instanceOf(const std::string& name) {
      using namespace std;
      auto it = instances().find(name);
      if (it == instances().end()) {
        throw runtime_error("Unbound variable: "s + name);
      }
      return (*it).second.back();
    }

  private:
    std::string name_;
    static decltype(auto) instances() {
      using namespace std;
      static map<string, vector<T>> instancesMap{};
      return (instancesMap);
    }
  };
}
```

注意到这段代码改进了 `DynamicScope` 构造函数，通过 perfect forwarding 来使得我们不需要自己手写构造函数，这使得我们可以写出类似这样的代码来将名字 `"x"` 绑定到值 `"aaa"` 上：

```c++
lang::DynamicScope<string> scope("x", 3, 'a');
```

除了类本身的改动之外，最重要的是上面定义了三个宏，用这三个宏进行 token 的处理，使我们不必手动将 `x` 写成 `"x"`，避免了出错，同时它也在一个 do-while 循环中帮我们创建了 `DynamicScope` 的对象，避免了我们接触实现细节，这使得我们可以写出类似我们想要的代码了：

```c++
auto foo() {
  DYNAMIC_VAL(string, x);
  cout << x << endl;
}

auto bar() {
  BIND_DYNAMIC(string, x, "bbb")
    foo();       // output bbb
    // DYNAMIC_VAL(string, x);   // compiler error: name conflict
    x = "ccc";
    foo();       // output ccc
    BIND_DYNAMIC(string, x, "ddd")
      foo();     // output ddd
    END_BINDING
  END_BINDING
}

auto main() -> int {
  BIND_DYNAMIC(string, x, 3, 'a')
    foo();       // output aaa
    bar();
    foo();       // output aaa
  END_BINDING
  // foo();      // runtime error: no such binding
}
```

# 参考

[^scope-wiki]: [Scope (computer science) - Wikipedia](https://en.wikipedia.org/wiki/Scope_(computer_science))
[^eopl]: Daniel P. Friedman, Mitchell Wand - Essentials of Programming Languages 3rd
[^cpp-dynamic-scope]: [Singleton revisited - Italian C++ Community](http://www.italiancpp.org/2017/03/19/singleton-revisited-eng)
[^msdn-cpp-marco]: [Preprocessor Operators - MSDN C/C++ Preprocessor Reference](https://msdn.microsoft.com/en-us/library/wy090hkc.aspx)
