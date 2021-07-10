---
layout: post
title: Rust 提升安全性的方式
excerpt: "通过和 C++ 进行对比，谈谈 Rust 如何通过编译器和程序语义避免程序员犯错。"
categories: articles
author: zrl
date: 2017-03-16
modified: 2017-03-16
tags:
  - Rust
  - C++
comments: true
share: true
---

# Rust 的起源与目的

Rust [^rust-lang] 是 Mozilla 公司开发的编程语言，它在 2010 才开始发布第一个版本，可以说是一个非常年轻的语言了。在提出一个新的编程语言的时候，设计者必须要回答的一个问题是「为什么要设计这样一个编程语言？」。对于 Rust 来说，他的目的就是要在保证安全的基础上不失对底层的控制力。

[^rust-lang]: [Rust Programming Language](https://www.rust-lang.org)

注意这里所指的「安全」不是说防止黑客攻击服务器，而是内存安全。拿 Rust 的主要竞争对手 C++ 为例，下面这段代码是安全的吗？

```c++
int foo(Bar* pBar) {
    if (pBar == nullptr) {
        return -1;
    } else {
        return pBar->baz();
    }
}
```

显然不是，尽管在 `foo` 函数中对 `pBar` 进行了非空的判断，但 `pBar` 可能指向了一块已经被释放掉了的内存，也就是所谓的「dangling pointer」错误 [^dangling-pointer]，此时程序的行为是未定义的。在 Java 等跑在虚拟机里的语言中，一般会将指针操作隐藏起来，同时由于有 GC 的存在，避免了程序员手动去释放内存，当一个对象不可达的时候，虚拟机会帮程序员去释放掉其占用的内存，所以，这段代码在 Java 中是安全的：

[^dangling-pointer]: [Dangling pointer - Wikipedia](https://en.wikipedia.org/wiki/Dangling_pointer)

```java
int foo(Bar bar) {
    if (bar == null) {
        return -1;
    } else {
        return bar.baz();
    }
}
```

Java 对内存安全的解决方案的问题在于，用户额外增加了虚拟机运行的开销，而且其模型无法做到 C++ 引以为傲的「zero overhead abstraction」。什么叫「zero overhead abstraction」？考虑如下的 C++ 代码：

```c++
class Foo { int i; ... };
class Bar { int j; Foo foo; ... };
// ...
void func() {
    Bar b;
    b.foo.i;
}
```

以及类似的 Java 代码：

```java
class Foo { int i; ... }
class Bar { int j; Foo foo; ... }
// ...
void func() {
    Bar b = new Bar();
    b.foo.i;
}
```

这两段代码很相似，但它们是不同的，在 C++ 的代码中，我们在栈上分配了一个 `Bar` 类型的对象 `bar`，然后直接获取了其中 `Foo` 部分的成员 `i`。在编译之后的代码中，`bar` 对象以两个整形变量的形式紧密排布在栈上。而在 Java 的代码中，我们做的事情则是在栈上分配了一个指向 `Bar` 类型对象的指针，堆上的 `Bar` 类型对象所占用的内存里有一个指向 `Foo` 类型对象的指针，也就是说，`b.foo.i` 这个调用在 Java 中需要两次指针访问。另外，在 `func` 函数结束的时候，C++ 版本的代码将会释放全部资源，而 Java 版本的代码仅仅释放了栈上的指针，堆上分配的内存则等待 GC 回收，这些事实意味着在 Java 中，我们进行抽象的时候是要付出额外开销的，这样的代码就是不如在栈上直接分配 `int i` 和 `int j` 高效。除此之外，C++ 的非虚成员函数、模板等的实现也体现了「zero overhead abstraction」这一原则，这里就不展开讲了。而 Rust 就是要跳出这个局限，既要保证内存安全，也要保证对底层的操控力，使得用户不需要在不必要的地方付出代价，拿刚刚的例子来说：

```rust
struct Foo { i: i32, ... }
struct Bar { j: i32, foo: Foo, ... }
// ...
fn func() {
    let bar = Bar { ... };
    bar.foo.i;
}
```

Rust 也保证了这样的代码在内存中是紧密排布的，不会有额外的指针开销，同时也保证在 `func` 执行结束的时候，`bar` 对象所占用的内存被释放，不需要 GC 的存在。了解完 Rust 的设计目标之后，下面就开始说明 Rust 是如何实现这个目标的。

# Rust 的所有权管理

Rust 实现内存安全的第一件事就是严格的所有权管理 [^rust-ownership]。对于 Java 程序员来说，这个概念可能比较陌生，毕竟内存交给了 JVM 来进行管理，但对于 C++ 程序员来说，所有权这个概念是必然要接触的，尤其是配合上智能指针之后，所有权就更加清晰了 [^smart-pointer]。

[^rust-ownership]: [Ownership - The Rust Programming Language](https://doc.rust-lang.org/book/ownership.html)

[^smart-pointer]: [Smart pointer - Wikipedia](https://en.wikipedia.org/wiki/Smart_pointer)

例如在 C++ 98 里我们常常会需要写类似这样的代码：

```c++
void f1(Foo* pFoo) { ... }
void f2(Foo* pFoo) { ... }

int main() {
    Foo* p = new Foo;
    f1(p);
    f2(p);
    // ...
}
```

这段代码让人非常怀疑其可靠性，因为它做了两个假设，第一是 `f1` 只使用 `pFoo` 而不去释放它所指向的内存，第二是 `f2` 负责释放 `pFoo` 所指向的内存。如果此处 `f1` 释放了内存，则会导致之前提到的「dangling pointer」错误，如果 `f2` 没有释放内存，则会导致内存泄漏。这段代码的可疑点正是在于所有权的不清晰。在智能指针的帮助下，这段代码可以可靠很多：

```c++
auto f1(Foo* pFoo) { ... }
auto f2(unique_ptr<Foo> pFoo) { ... }

auto main() -> int {
    auto p = make_unique<Foo>{};
    f1(p.get());
    // error: call to implicitly-deleted copy constructor of 'unique_ptr<Foo>'
    // f2(p);
    f2(std::move(p));
    // ...
}
```

利用 C++ 11 后出现的移动语义 [^cpp-11]，我们可以明确表达所有权的移动。在 `f1` 里，资源只被使用，但其所在的内存不会被释放，在 `f2` 里，`pFoo` 所指向的内存就会被释放，而且不需要手动调用 `delete`，由智能指针的析构函数进行释放操作，这是所谓的 RAII 范式 [^raii]。`unique_ptr` 表达了独占的所有权，如果我们尝试复制指针则会造成编译错误，需要用 `std::move` 来表达所有权的移动。但是，即便是有了这个移动语义，代码还是可能会出现未定义的行为。假设我们在调用完 `f2` 之后又一次使用了 `p` 会出现什么情况？由于资源已经被移动了，所以我们不应该对 `p` 进行操作，但编译器并不会制止我们的这一行为（虽然一般会有警告），其原因在于，`std::move` 并没有移动资源，它做的事情仅仅是对类型进行了转换，通过重载决议使得 `unique_ptr` 调用了移动构造函数这一特定的重载版本而已。而在 Rust 里，如果我们做了类似的事情，编译器会直接报错（`Box` 相当于 C++ 的 `unique_ptr`）：

[^cpp-11]: [C++ 11 - Wikipedia](https://en.wikipedia.org/wiki/C%2B%2B11)

[^raii]: [Resource acquisition is initialization - Wikipedia](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization)

```rust
fn func(foo: Box<Foo>) { ... }

fn main() {
    let p = Box::new(Foo::new());
    func(p);
    p.i;  // error: use of moved value: `p.i`
}
```

Rust 默认采用了移动语义，如果需要拷贝需要另外声明，而 C++ 出于历史的兼容性考虑，其默认语义是拷贝的，如果要禁止拷贝或者实现移动语义则是需要另外声明。并且，Rust 的编译器在发现一个变量被移动后又被继续使用时，会直接拒绝编译，这个安全保证直接嵌进了语言中，防止出现 C++ 中使用已移动资源的未定义行为。

# Rust 的借用检查

如果仅仅是实现了对重新使用已移动对象的检查，似乎还不是太有说服力，尤其是对于 Java 程序员来说，他们根本就不需要考虑这个问题。但 Rust 在保证内存安全的方面不仅做了这件事，例如下面说的这个问题是连 Java 这样依赖虚拟机管理内存的语言也没法解决的问题。考虑如下 C++ 代码：

```c++
auto vec = vector<int>{ 1, 2, 3 };
for (auto i : vec) {
    for (auto j = 0; j < i * 10; ++j) {
        vec.push_back(i);
    }
}
```

这段代码的结果是未定义的，原因是 `vector` 的内部是用动态数组实现的，这段代码通过 `vector` 的迭代器来遍历了 `vec`。当我们获取了迭代器之后，我们又去向 `vec` 添加了元素，使得 `vec` 内部原本的动态数组被释放，换了新的更大的动态数组来存放元素。这是经典的「迭代器失效」错误，在 Java 中，编译器也没法检测这一错误，取而代之的是一个运行时的 `ConcurrentModificationException` 异常。

这个问题的出现是 GC 无法解决的，而 Rust 的设计者发现了这其实不是单个特定的问题，而是一类问题，这类问题的存在是因为两件事的同时发生，一是「aliasing」（多于一个指针指向同一块内存），二是「mutation」（在此例中为向 `vector` 添加元素）[^rust-talk]。在如 Haskell 这样的函数式语言里，采用了更高级别的抽象，直接要求所有变量都是不可变的，所以多个别名总是安全的。但在 Rust 中，编译器不是去阻止任何可变的变量，而是去阻止 aliasing 和 mutation 这两件事的同时发生。

[^rust-talk]: [Guaranteeing memory safety in Rust](https://air.mozilla.org/guaranteeing-memory-safety-in-rust/)

之前提到，Rust 采用了默认的移动语义，也就是说，`fn foo(v: Vec<i32>)` 这样的声明会导致资源被移动进 `foo` 中，由 `foo` 独占 vector `v` 的所有权，如果我们要实现 alasing，我们则需要把声明改成这样：`fn foo(v: &Vec<i32>)`。这在 Rust 中被称为「借用」（Borrowing）[^rust-borrowing]。和 C++ 不同，Rust 中默认是不可变的，这包括了变量默认不可变，借用也是默认不可变的，所以以下代码是非法的：

[^rust-borrowing]: [Referneces and Borrowing - The Rust Programming Language](https://doc.rust-lang.org/book/references-and-borrowing.html)

```rust
fn foo(v: &Vec<i32>) {
    // error: cannot borrow immutable borrowed content `*v` as mutable
    v.push(0)
}
```

在 Rust 中，`&` 表达了借用语义，另外，与 C++ 需要手动增加 `const` 来表示不可变不同，在 Rust 中，我们需要手动添加 `mut` 关键字才能表达可变，这包括了变量声明和借用声明的地方，所以下面的代码可以编译通过：

```rust
fn foo(v: &mut Vec<i32>) {
    v.push(0)
}

fn main() {
    let mut v = vec![1, 2, 3];
    foo(&mut v)
}
```

正如前文所述，aliasing 和 mutation 同时存在的时候，程序就很可能出现问题，而多个不可变借用和单个可变借用都是安全的，所以，Rust 允许多个不可变的借用的存在，例如，下面这段代码是合法的：

```rust
fn add(i1: &i32, i2: &i32) -> i32 {
    *i1 + *i2
}

fn main() {
    let x = 1;
    println!("{}", add(&x, &x))
}
```

上面这段代码中，`i1` 和 `i2` 都被标记为不可变的借用，所以，对变量 `x` 同时进行这两个借用是合法的。另外，Rust 也允许单个可变借用的存在，例如，如下代码也是合法的：

```rust
fn add1(i: &mut i32) {
    *i += 1
}

fn main() {
    let mut x = 1;
    add1(&mut x);
    println!("{}", x)
}
```

在这里，`add1` 的参数 `i` 的类型标记里通过将 `&` 改为 `&mut` 将其声明为可变借用，在声明变量 `x` 的时候，通过添加关键字 `mut` 也将其声明为可变的，借用 `x` 的时候，需要用 `&mut x` 的形式，这样就表示了对可变量的可变借用。

当我们想对一个变量进行可变借用的同时进行其他借用的情况下，编译就无法通过，例如下面这样的代码就是不合法的：

```rust
fn foo(i1: &mut i32, i2: &mut i32) -> i32 { ... }
fn bar(i1: &mut i32, i2: &i32) -> i32 { ... }

fn main() {
    let mut x = 1;
    // error: cannot borrow `x` as mutable more than once at a time
    println!("{}", foo(&mut x, &mut x));
    // error: cannot borrow `x` as immutable
    // because it is also borrowed as mutable
    println!("{}", bar(&mut x, &x))
}
```

上面那段代码尽管在逻辑上没什么大问题，但是编译器还是拒绝了这段代码，这约束似乎有点过强了。这样有什么意义呢？现在回到之前的迭代器失效的问题上，考虑一下如果我们在 Rust 里写类似之前用 C++ 写出的代码会出现什么结果？例如：

```rust
fn main() {
    let mut vec = vec![1, 2, 3];
    for i in &vec {
        // error: cannot borrow `vec` as mutable
        // because it is also borrowed as immutable
        vec.push(4);
    }
}
```

编译器报错了，错误很明确，由于在我们对 `vec` 进行迭代访问操作的时候对 `vec` 进行了不可变的借用，而在 `for` 代码块中又尝试对其进行可变的借用，所以编译就出错了。Rust 的做法从根源上直接防止了这个错误的出现。如果你认为这个例子太显然了，大家都懂得避免，那下面举个不太显然的例子：

```c++
template <typename T>
auto pushMany(const T& t, const unsigned int n, vector<T>& vec) {
    for (auto i = 0u; i < n; ++i) {
        vec.push_back(t);
    }
}
```

这段代码有什么问题？看起来似乎没什么问题，但是如果我这样调用呢？

```c++
auto vec = vector<int>{ 1, 2, 3 };
pushMany(vec[0], 100, vec);
```

这里又是一个「dangling pointer」错误，相当隐蔽，对于不知晓 `pushMany` 实现细节的用户而言，上面这段调用是很正常的，我希望向 `vec` 中添加 100 个 `vec` 的第一个元素，但是由于 `pushMany` 的实现使用了引用，且用户在传参数的时候对同一个 `vector` 同时进行了可变的引用（ `vec` ）和不可变的引用（ `vec[0]` ）这导致了潜在的错误，而且这个错误还不一定会发生，例如写 `pushMany(vec[0], 1, vec)` 的时候就很可能不会出错，这导致了错误排查的困难。如果在 Rust 中，这个错误则直接可以被 Borrow Checker 发现，它将禁止用户同时对 `vec` 进行可变和不可变的借用。

# Rust 的生命周期管理

如果 Borrow Checker 只能做到之前的说到的保障那还不足够，我们还是可能出现「dangling pointer」这类错误，考虑如下 C++ 代码：

```c++
auto get0() -> int& {
    auto i = 0;
    return i;
}

auto main() -> int {
    auto& x = get0();
    x = 3;
}
```

这段代码即便放到 Rust 中也没有违背 Aliasing 和 Mutation 不能同时存在的原则，但它还是造成了一个未定义行为。原因是 `get0` 函数返回了存在于栈上的变量的引用，当 `get0` 结束后，`i` 已经被销毁，而 `main` 函数中却尝试去修改这个值。为了避免这类问题 Rust 还有一个生命周期的检查。Lifetime 是 Rust 中另一个重要的概念 [^rust-lifetime]，一个变量从初始化到最终销毁构成了其生命周期。而对一个变量的借用是不允许超出其生命周期的，Borrow Checker 通过对生命周期的覆盖检查确保借用是有效的。如果在 Rust 中写下如下代码：

[^rust-lifetime]: [Lifetimes - The Rust Programming Language](https://doc.rust-lang.org/book/lifetimes.html)

```rust
// error: missing lifetime specifier
fn get0() -> &i32 {
    let i = 0;
    &i
}
```

编译器提示我们，这个地方缺少了生命周期标记。生命周期标记指明了变量存活的范围。在许多情况下，生命周期可以由编译器隐式推导出来，例如：

```rust
fn foo(i: &i32) -> &i32 {
    i
}
```

上面这段代码其实相当于：

```rust
fn foo<'a>(i: &'a i32) -> &'a i32 {
    i
}
```

这里的 `'a` 就是一个生命周期标记，它被编译器自动推导得出，在这里，`i` 具有 `'a` 的生命周期，而返回值也标注了 `'a`，这意味着返回值的生命周期至少要能覆盖 `'a` 的长度。现在回到刚刚 `get0` 的例子里，编译器没有能推导出其生命周期，那我们可以给它加上生命周期标记：

```rust
// info: borrowed value mut be valid for the lifetime 'a
fn get0<'a>() -> &'a i32 {
    let i = 0;
    // error: `i` does not live long enough
    &i
}
```

编译器提示了错误，`i` 的生命周期在 `get0` 返回的时候就结束了，而返回值对 `i` 的借用已经超出了它的生命周期，所以这段代码无法编译通过。如果我们将代码改成这样，编译就可以通过了：

```rust
fn get0<'a>() -> &'a i32 {
    static I: i32 = 0;
    &I
}
```

原因是现在 `I` 具有了 `'static` 的生命周期，标记为 `'static` 的生命周期是特殊的，它意味着一个变量的存在范围是整个程序。所以 `I` 的生命周期覆盖了 `'a`，所以可以安全地将 `&I` 返回。在这里，Rust 的编译器又一次阻止了潜在错误的发生。

放到多线程的环境下，这个生命周期检查也是非常有价值的，例如编译器会阻止我们写如下的函数：

```rust
fn print_async(i: i32) {
    // error: closure may outlive the current function,
    // but it borrows `i`, which is owned by the current function
    thread::spawn(|| println!("the number is: {}", i));
}
```

这错误提示简直不能更清晰，由于我们创建了一个线程，然后传入的 closure 以借用的形式捕获了局部的变量，由于这个 closure 的生命周期已经超出了 `i` 的生命周期，所以这段代码是危险的，可能会使得线程捕获了一个已经被释放的 `i`。此时我们可以通过对象的移动来解决这个问题，直接将所有权交给 closure，这样 `i` 的生命周期就不会随着 `print_async` 的结束而结束了。

```rust
fn print_async(i: i32) {
    thread::spawn(move || println!("the number is: {}", i));
}
```

# 对 Rust 的质疑

有人会指出，这些错误都是很显然的，只要你认真看了《Effective C++》、《More Effective C++》、《Effective Modern C++》等等等等，并且严格按照标准的做法去做，注意不去犯这些错误就可以了，如果一个人会在 C++ 中犯这些错误，他即便换到 Rust 中也不会有什么好转。但事实上，即便是专业的程序员，在面对一个大型系统的时候，也难免出现这样那样的错误，一个语言提供的保障可以在很大程度上防止错误的发生。在 Rust 中，一个被判定为不安全的操作需要一个 `unsafe` 块来包裹才能编译通过，这个明确的界限使得程序的 debug 更加方便。

另外的一个质疑点是 Rust 的做法会使得一些本来合理的代码出现编译错误，使得用户需要用很扭曲的方式实现 C++ 这类语言中很轻易实现的功能。其实这一点，在 C++ 发明人 Bjarne Stroustrup 的一个演讲里也提到，他说他听到过许多「Language Myths」，其中一个就是「We want a language for writing reliable code」，他认为大多数人并不想要这样的语言，因为这样的语言需要学习更多的概念，而且要同时兼顾性能与安全的时候，开发会变得困难 [^bjarne-talk]。这个质疑其实很像动态语言的拥趸对静态语言的质疑，他们的其中一个质疑点就是静态语言的编译器会拒绝一些合理的代码，编译器只能提供非常弱的保障，更多的保障还是需要测试来实现，与其依赖编译器，不如完全依赖测试。但无论如何，静态强类型的语言确实能在编译期间发现许多问题，防止了程序员犯低级错误，为此付出的一点代价是值得的。举例来说，JavaScript 的设计者在设计这个语言时就认为人们只会用它来完成一些简单的工作，不应该让他们去学习类型的概念，所以 JavaScript 被设计为动态弱类型语言，当然现在 JavaScript 因为其在浏览器的独占性而逐渐侵蚀了许多领域，使得 JavaScript 承担了许多本来不应该由它来承担的非常复杂的逻辑，这使得各种语言支持了「compile to JS」[^compile-to-js]，也有许多改良的语言出现，例如 TypeScript、PureScript 等等，他们其实主要解决的问题就是 JavaScript 的类型系统实在是太容易让人犯错了，在一个复杂的工程里，我们需要编译器来协助减少一些错误。所以说，在工程规模足够小的时候，例如写一个日常辅助的小脚本，许多概念可以不必要存在，灵活性可能要放在首位，让人能快速完成工作，但是，一旦工程规模增大，一个足够严格的语义和足够强大的编译器能够给开发者提供非常多的保障，防止许多错误的发生，而 Rust 的杀手级优势也正是在于此。

[^bjarne-talk]: [Bjarne Stroustrup - What – if anything – have we learned from C++?](https://youtu.be/2egL4y_VpYg?t=32m42s)

[^compile-to-js]: [List of languages that compile to JS](https://github.com/jashkenas/coffeescript/wiki/List-of-languages-that-compile-to-JS)

# 参考
