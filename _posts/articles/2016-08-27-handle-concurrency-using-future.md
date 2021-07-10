---
layout: post
title: 使用 Future 进行并发编程
excerpt: "Future 能够将计算任务的并发化和计算最终的执行方式分离，通过一套设计良好的 API 使得对数据的操作变得简单。"
categories: articles
author: zrl
date: 2016-08-27
modified: 2016-08-28
tags:
  - Asynchrony
  - Concurrency
  - Java
  - Scala
comments: true
share: true
---

# Future 的概念

在编程的时候，常常会遇到需要并行处理一些代码，最原始的做法就是创建不同的线程进行处理，但是线程之间的同步处理非常麻烦而且容易出错，如果要同时得到几个线程的结果并且通过这些结果进行进一步的计算，则需要共享变量或者进行线程间通信，无论如何都非常难以处理。另外，直接使用线程也使得代码灵活性不高，比如在双核机器上可能只希望使用两个线程执行代码，到了四核机器上就希望最多能有四个线程了。Future 能够提供一个高层的抽象，将计算任务的并发化和计算最终的执行方式分离，使得这类处理更为方便。Future 作为一个代理对象代表一个可能完成也可能未完成的值 [^wiki-future]，通过对 future 进行操作，能够获取内部的计算是否已经完成，是否出现异常，计算结果是什么等信息。

[^wiki-future]: [Futures and promises - Wikipedia](https://en.wikipedia.org/wiki/Futures_and_promises)

# Java 中的 `Future`

Java 很早就提供了 `Future` 接口 [^java-future]，使用起来大概是这样的：

[^java-future]: [Future - JavaSE 8 References](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html)

```java
interface ArchiveSearcher { String search(String target); }
class App {
  ExecutorService executor = ... ;  // init executor service
  ArchiveSearcher searcher = ... ;  // init searcher
  void showSearch(final String target) throws InterruptedException {
    Future<String> future = executor.submit(new Callable<String>() {
        public String call() {
            return searcher.search(target);
        }
    });
    displayOtherThings();  // do other things while searching
    try {
      displayText(future.get());  // use future
    } catch (ExecutionException ex) { 
      cleanup(); 
      return; 
    }
  }
}
```

这段代码是一个搜索与展示搜索结果的代码，`showSearch` 方法接收待搜索字符串，并调用 `executor` 的 `submit` 方法来提交一个搜索任务。`executor` 的类型是 `ExecutorService`，用于将任务的提交和任务执行的具体实现机制解绑 [^java-executor]，一般用线程池实现 [^java-executor-service]。当提交的任务是具有返回值的时候，`submit` 返回的不是这个任务完成后的值（例如这里需要返回的搜索结果是 `String` 类型），因为此时这个任务尚未执行完成。为了能在之后获取到这个值，此时需要返回一个占位对象，或者说是一个对结果值的代理对象，这个对象就是一个 future。这个 future 是立刻返回的，不会阻塞当前线程，这样，future 内的任务就能和 future 外的任务并发地进行计算，例如此处调用 `displayOtherThings` 来显示其他内容。当 `displayOtherThings` 执行完成，就去执行 `displayText` 来显示搜索结果，搜索结果的获取需要调用 `Future` 的 `get` 方法，此处 `future` 的类型是 `Future<String>`，其 `get` 方法的返回值类型就为 `String`。注意如果 future 的结果尚未计算出来，那么 `Future` 的 `get` 方法将阻塞当前线程。

[^java-executor]: [Executor - JavaSE 8 References](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executor.html)
[^java-executor-service]: [ExecutorService - JavaSE 8 References](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ExecutorService.html)

在 Java 8 里提供了 Lambda 表达式，上面的例子可以简化为：

```java
// ...
class App {
  // ...
  void showSearch(final String target) throws InterruptedException {
    Future<String> future = executor.submit(() -> searcher.search(target));
    displayOtherThings();  // do other things while searching
    try {
      displayText(future.get());  // use future
    } catch (ExecutionException ex) { 
      // ...
    }
  }
}
```

Java 提供了一些对 future 的监控和操作手段：

```java
interface Future<V> {
  boolean cancel(boolean mayInterruptIfRunning);
  V get() throws InterruptedException, ExecutionException;
  V get(long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException;
  boolean isCancelled();
  boolean isDone();
}
```

从 API 可以看出，`Future` 里的值是只能获取不能设置的，因为 future 从概念上来说就是对某一值的一个只读的占位符，这个值可能暂时没有计算出来，也可能永远无法计算出来。对于这个值在计算过程中出现异常而无法获取的情况，在 Java 中使用 `get` 方法抛出的异常来表示，`get` 方法会抛出如下 3 个异常：

> `CancellationException` - if the computation was cancelled<br/>
> `InterruptedException` - if the current thread was interrupted while waiting<br/>
> `ExecutionException` - if the computation threw an exception

但是，这套 `Future` 的 API 存在一些问题，首先，要获取一个 future 的计算结果必须要同步获取，这就对灵活性产生了很多限制，另外，这套 API 没有提供 future 间的组合方式，复杂的组合变得困难。考虑如下情况：

```java
interface SearcherServiceFetcher { String fetch(String type); }
interface ArchiveSearcher { 
  String search(String service, String config, String target); 
}
class App {
  ExecutorService executor = ... ;  // init executor service
  SearcherServiceFetcher serviceFetcher = ... ;  // init searcher service fetcher
  ArchiveSearcher searcher = ... ;  // init searcher
  void showSearch(final String type, final String target) throws InterruptedException {
    Future<String> resultFuture  = executor.submit(() -> {
      Future<String> serviceFuture =  // future for fetching searcher service
        executor.submit(() -> serviceFetcher.fetch(type));
      Future<String> configFuture =   // future for reading config
        executor.submit(() -> readConfig());
      try {  // handle failure of fetching service
        String service = serviceFuture.get();  // get service
        String config = "";
        try {  // nested try-catch block, for reading config
          config = configFuture.get();  // get config
        } catch (ExecutionException ex) {
          config = getDefaultConfig();  // if fail to read config, use default
        }
        return searcher.search(service, config, target);  // search
      } catch (ExecutionException ex) {
        throw ex.getCause();  // if fail to fetch service, throw exception
      }
    });
    displayOtherThings();  // do other things while searching
    try {
      String textForDisplay = render(resultFuture.get());  // render result
      displayText(textForDisplay);  // display result
    } catch (ExecutionException ex) { 
      cleanup(); 
      return; 
    }
  }
}
```

在这个例子里，搜索不但依赖于传入的搜索内容，还要依赖于根据搜索类型决定的搜索服务提供者以及搜索配置，由于获取搜索服务提供者和读取配置的过程都是需要费时的，所以此处将这两个任务都提交给 `executor` 处理，获得两个 future 后，我们首先查看搜索服务提供者是否成功被获取到了，如果获取失败，就直接抛出一个异常。如果服务提供者获取成功了，就去查看配置是否读取成功，由于读取配置的过程也可能出错，所以这里还要进行错误处理，如果配置读取不到，就使用默认的配置。获取到服务提供者和配置后再进行搜索并返回结果。在 `displayOtherThings` 结束后，调用 `resultFuture` 的 `get` 方法获取搜索结果，然后渲染获得的内容，再进行展示。这段代码虽然不长，但也足够难读了，从这段代码可以发现，由于每一次的 `get` 操作都可能抛出异常，我们需要进行很多异常处理，再加上嵌套的 future，使得主要的代码逻辑非常混乱，不但难以阅读，且调试相当困难，最后的阻塞调用也可能会对性能造成很大影响。

# 对 Java `Future` API 的改进

要改善 Java 的 `Future` API，首先要提供接口让用户从阻塞调用变为非阻塞调用，也就是使用回调函数（使用 Scala 表示）：

```scala 
trait Future[T] {
  def onComplete(callback: T => Unit): Unit
  def cancel(mayInterruptIfRunning: Boolean): Boolean
  def isCancelled: Boolean
  def isDone: Boolean
}
```

使用的时候大概是这样：

```scala
future.onComplete(res => consume(res))
```

使用回调函数之后，在 `onComplete` 处就不会阻塞线程，当 future 所代理的值被计算出来后，通过 `onComplete` 注册的回调函数就会被调用，从而执行所需的代码。但很快可以发现，由于整个过程是异步的，所以这样无法直接使用 try-catch 块来捕获异常，如前所述，Java 的 `Future` 的 `get` 方法的完整声明其实是这样的：

```java 
V get() throws InterruptedException, ExecutionException;
V get(long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException;
```

所以，`get` 的声明其实不是 `() => T` 而是 `() => Try[T]`，对应 `get` 改为异步后的 `onComplete` 不应该是 `T => Unit`，而应该是 `Try[T] => Unit`。

```scala 
trait Future[T] {
  def onComplete(callback: Try[T] => Unit): Unit
  def onSuccess(callback: T => Unit): Unit
  def onFailure(callback: Throwable => Unit): Unit
  // ...
}
```

例如前面的例子，用异步回调的方式写出来大概是这样的：

```scala 
trait SearcherServiceFetcher { def fetch(type: String): String }
trait ArchiveSearcher { 
  def search(service: String, config: String, target: String): String 
}
class App {
  val executor: ExecutorService = ...
  val serviceFetcher: SearcherServiceFetcher = ...
  val searcher: ArchiveSearcher = ...
  def showSearch(type: String, target: String): Unit = {
    val resultFuture: Future[String] = executor.submit { _ => 
      val serviceFuture: Future[String] = 
        executor.submit { _ => serviceFetcher.fetch(type) }
      val configFuture: Future[String] = 
        executor.submit { _ => readConfig() }
      serviceFuture.onComplete {  // callback for fetching service
        case Success(service) =>
          configFuture.onComplete {  // nested callback for reading config
            case Success(config) => 
              searcher.search(service, config, target)
            case Failure(thw) =>  // if fail to read config, use default
              searcher.search(service, getDefaultConfig(), target)
          }
        case Failure(thw) => throw thw  // if fail to get service, throw exception
      }
    }
    resultFuture.onComplete {
      case Success(result) => 
        val textForDisplay: String = render(result)
        displayText(textForDisplay)
      case Failure(thw) => cleanup()
    }
    displayOtherThings()  // do other things while searching
  }
}
```

上面的代码为了和 Java 版本的进行对照，所以使用了类似的调用，但由于是使用 Scala，上述的代码的类型签名会不太一样（例如 Scala 使用 `ExecutionContext` 而非 `ExecutorService`，但作用是类似的），另外可以更加简练，例如用隐式的参数传递 `executor`，使用类型推导减少类型声明等，实际写出来大概是这样的：

```scala 
// ...
class App {
  implicit val context = ...  // init execution context 
  val serviceFetcher = ...
  val searcher = ...
  def showSearch(type: String, target: String): Unit = {
    val resultFuture = Future {
      val serviceFuture = Future { serviceFetcher.fetch(type) }
      val configFuture = Future { readConfig() }
      serviceFuture.onComplete {
        case Success(service) =>
          configFuture.onComplete {
            case Success(config) => 
              searcher.search(service, config, target)
            case Failure(thw) => 
              searcher.search(service, getDefaultConfig(), target)
          }
        case Failure(thw) => throw thw
      }
    }
    resultFuture.onComplete {
      case Success(result) => 
        val textForDisplay = render(result)
        displayText(textForDisplay)
      case Failure(thw) => cleanup()
    }
    displayOtherThings()
  }
}
```

这个使用回调函数而非阻塞获取结果的版本的改进之处在于它使得主线程不会因为这些计算而阻塞，但是从代码逻辑上看，即便是依靠语法的简洁减少了一些代码的书写，整段代码还是比较难读。显然，使用回调函数实现的这个版本也是难以组合的，操作起来甚至比直接使用阻塞的 `get` 调用还要复杂，很容易就陷入 JavaScript 程序常常遇到的「Callback Hell」[^callback-hell]。由于 `onComplete` 返回的是 `Unit`，所以整个回调过程完全是通过副作用的形式产生效果的。嵌套的回调代码比顺序执行的 `get` 调用更为混乱。所以现在的实现目标是：尽管最后的回调完全是副作用过程，但在进行 future 间组合时不让用户去关心这些副作用，也就是希望能将计算中的组合和最终的计算实现分离。

[^callback-hell]: [Callback Hell - A guide to writing asynchronous JavaScript programs](http://callbackhell.com/)

首先，一个常用的组合子就是 `map` 了（真实 API 会有隐式传递的 `ExecutionContext` 参数，这里省去，不影响表意）：

```scala 
trait Future[T] {
  def map[R](f: T => R): Future[R]
  // ...
}
```

`map` 方法产生一个新的 future，如果原 future 成功计算出了结果，那么新的 future 的结果就是将 `f` 作用于原 future 所代理的值上所得出的结果，如果原 future 出现了异常导致失败，或者 `f` 的调用过程出现异常，那么新的 future 将会失败。

比如，上面的代码中获得结果后需要对结果进行渲染，然后再显示，使用 `map` 就可以写成：

```scala 
resultFuture.map(render).onComplete {
  case Success(textForDisplay) => 
    displayText(textForDisplay)
  case Failure(thw) => cleanup()
}
```

但为了能处理某一 future 的构建依赖于前一个 future 的结果的情况（例如 config 和 service 的获取），光有 `map` 这个组合子还不够，我们还需要有一个组合子能够去处理上下文相关的情景，这个组合子就是 `flatMap`：

```scala 
trait Future[T] {
  def flatMap[R](f: T => Future[R]): Future[R]
  // ...
}
```

`flatMap` 方法会根据原 future 的计算结果来产生一个新的 future，如果原 future 成功计算出了结果，那么新的 future 就是将 `f` 作用于原 future 所代理的值上所得出的 future，如果原 future 出现了异常导致失败，或者 `f` 的调用过程出现异常，又或者新的 future 自身出现了异常，那么新的 future 将会失败。有了这个组合子，配合 Scala 的 for-comprehension，就可以这样写：

```scala 
val resultFuture = for {
  service <- Future { serviceFetcher.fetch(type) }
  config <- Future { readConfig() }
} yield searcher.search(service, config, target)

resultFuture.map(render).onComplete {
  case Success(textForDisplay) => 
    displayText(textForDisplay)
  case Failure(thw) => cleanup()
}
```

这段代码将被翻译为对 `flatMap` 和 `map` 的调用，但即便不懂 Scala 的语法，上面这段代码的目的也非常清晰，先取得搜索服务提供者，并命名为 `service`，取得配置，并命名为 `config`，然后通过这两个信息进行搜索。之后将搜索结果进行渲染，再注册回调函数，在整个过程完成后进行展示。

上面的代码没有进行错误处理，除了 `map` 和 `flatMap` 之外，Scala 的 `Future` 还提供了更多的组合子，例如用于从异常中恢复的 `recover`，用于筛选结果的 `filter`，用于进行副作用处理的 `foreach` 等 [^scala-doc-future] [^scala-api-future]。配合之下，前面的代码将变成这样：

[^scala-doc-future]: [Futures and Promises - Scala Documentation](http://docs.scala-lang.org/overviews/core/futures.html)
[^scala-api-future]: [Future - Scala API References](http://www.scala-lang.org/files/archive/nightly/docs/library/scala/concurrent/Future.html)

```scala 
class App {
  implicit val context = ...
  val serviceFetcher = ...
  val searcher = ...
  
  def showSearch(type: String, target: String): Unit = { 
    
    val displayTextFuture = for {
      service <- Future { serviceFetcher.fetch(type) }
      config <- (Future { readConfig() }).recover(_ => getDefaultConfig)
      result <- Future { searcher.search(service, config, target) }
    } yield render(result)
    
    displayTextFuture.onComplete {
      case Success(textForDisplay) => displayText(textForDisplay)
      case Failure(thw) => cleanup()
    }
    
    displayOtherThings()
  }
}
```

相比起之前的版本，这一版本可以说是惊人的简洁，虽然最后的回调是有副作用的，但是前面的组合根本不需要考虑这些副作用，可以将不同的 future 进行纯的组合，只有在最后才会碰到一次副作用的回调函数注册，而且展现出来的代码非常扁平，没有难读的嵌套。

虽然 Scala 的这一套 API 很优雅，但是受限于 Java 的语法，这个设计在 Java 上却无法直接照搬，例如上面那段代码中的 for-comprehension 部分将被翻译成：

```scala 
val displayTextFuture = Future { serviceFetcher.fetch(type) } flatMap { 
  service => (Future { readConfig() }).recover(_ => getDefaultConfig) flatMap {
    config => Future { searcher.search(service, config, target) } map {
      result => render(result)
    }
  }
}
```

由于 Java 没有类似 for-comprehension 的语法，如果直接照搬 Scala 的 API 设计，那就必须在 Java 的代码中写这样的嵌套处理了。这样的嵌套处理非常难读难写，所以，Java 8 设计了另外一套 API，实现在 `CompletableFuture` 中 [^java-completable-future]，举例而言：

[^java-completable-future]: [CompletableFuture - JavaSE 8 References](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)

```java
class CompletableFuture<T> extends Object 
  implements Future<T>, CompletionStage<T> {
  
  public <U> CompletableFuture<U> thenApplyAsync
    (Function<? super T, ? extends U> fn) { ... }
      
  public <U,V> CompletableFuture<V> thenCombineAsync
    (CompletionStage<? extends U> other, 
     BiFunction<? super T, ? super U, ? extends V> fn) { ... }
       
  public CompletableFuture<Void> thenAcceptAsync
    (Consumer<? super T> action) { ... }
  
  public static <U> CompletableFuture<U> supplyAsync
    (Supplier<U> supplier) { ... }
    
  public CompletableFuture<T> whenCompleteAsync
    (BiConsumer<? super T, ? super Throwable> action) { ... }
    
  public <U> CompletableFuture<U> handleAsync
    (BiFunction<? super T, Throwable, ? extends U> fn) { ... }
    
  // ...
}
```

正如之前的在 [协变、逆变与不变](/articles/covariant-and-contravariant) 一文中提到的一样，Java 的型变是在使用的地方进行限制的，所以这里的几个方法签名都非常难看，但对于使用者来说，其实并不太需要理会复杂的声明，将上面的声明看作这样就可以了：

```java
class CompletableFuture<T> extends Object 
  implements Future<T>, CompletionStage<T> {
  
  public <U> CompletableFuture<U> 
  thenApplyAsync((T -> U) fn) { ... }
      
  public <U,V> CompletableFuture<V> 
  thenCombineAsync(CompletableFuture<U> other, ((T, U) -> V) fn) { ... }
       
  public CompletableFuture<Void> 
  thenAcceptAsync((T -> Void) action) { ... }
  
  public static <U> CompletableFuture<U> 
  supplyAsync((() -> U) supplier) { ... }
    
  public CompletableFuture<T> 
  whenCompleteAsync(((T, Throwable) -> Void) action) { ... }
    
  public <U> CompletableFuture<U> 
  handleAsync(((T, Throwable) -> U) fn) { ... }
    
  // ...
}
```

这一套 API 其实光看方法签名就能大概猜到它们的作用了，例如 `thenApply` 和 `thenApplyAsync` 就相当于 `map`，`whenComplete` 和 `whenCompleteAsync` 就相当于 `onComplete` 等。在配合 Java 8 的 Lambda 表达式之后，使用时写出的代码也是相当清晰的，例如之前的代码可以写成：

```java 
CompletableFuture<String> serviceFuture =  // fetch searcher service
  CompletableFuture.supplyAsync(  // use supplyAsync to construct future
    () -> serviceFetcher.fetch(type));  
  
CompletableFuture<String> configFuture =  // read config
  CompletableFuture
    .supplyAsync(() -> readConfig())
    .handleAsync((config, ex) -> {  // use handleAsync to handle result
      if (ex == null) return config;
      else return getDefaultConfig();  // if fail to read config, use default
    });
    
CompletableFuture<String> textForDisplayFuture =  // search and render
  serviceFuture
    .thenCombineAsync(  // use thenCombineAsync to combine result
      configFuture, 
      (service, config) -> searcher.search(service, config, target))
    .thenApplyAsync((result) -> render(result));  // render result
      
textForDisplayFuture.whenCompleteAsync((text, ex) -> { 
  if (ex == null) cleanup(); else displayText(text); 
});
  
displayOtherThings();
```

注意到在这套 API 中，为了避免使用类似 `flatMap` 这样的函数导致嵌套调用，Java 使用 `thenCombine` 和 `thenCombineAsync` 来承担 Scala 中 `flatMap` 的作用，处理上下文相关的场景，但这个组合子并没有 `flatMap` 那么强大。为了说明这个问题，考虑一个计算依赖于三个 future 的计算结果的场景。假设现在需要用 a，b，c 三个字符串构建一个新的字符串，而这三个字符串分别由 `futureA`，`futureB`，`futureC` 代理，那么可能就要写出如下的代码：

```java
class Pair<X, Y> {
  public X x;
  public Y y;
  public Pair(X x, Y y) {
    this.x = x; this.y = y;
  }
}
CompletableFuture<String> stringFuture =
  futureA
    .thenCombineAsync(futureB, (a, b) -> new Pair<String, String>(a, b))
    .thenCombineAsync(futureC, (pair, b) -> buildString(pair.x, pair.y, c));
```

而 Scala 只需要这样写：

```scala 
val stringFuture = for {
  a <- futureA
  b <- futureB
  c <- futureC
} yield buildString(a, b, c)
```

显然，这个组合子只能方便地处理两个 future 的结合，没有 `flatMap` 强大，事实上，可以用 `flatMap` 来很容易地实现 `thenCombine`，一般称为 `map2`：

```scala 
trait Future[T] {
  def map2(other: Future[U], f: (T, U) => V): Future[V] = for {
    t <- this
    u <- other
  } yield f(t, u)
}
```

虽然 Java 的这个实现没有 Scala 版本的代码优雅，但是在大多数情况下也够用，尤其是在受到 Java 的语法局限的情况下，这个已经是一个比较好的处理了。从获取搜索结果并显示的例子中可以看出，使用这套 API 的关键优点在于这个版本的代码也做到了在异步回调避免阻塞主线程的情况下，加强了 future 间的组合性，避免出现最初版本的难读代码。

总之，在 Java 8 之后，应该使用新的 API 来操作 future，以便能更加简便地处理并发和异步代码。另外，对于 API 设计而言，要尽可能加强组件与组件之间的可组合性，将无法组合的部分抽离，只有在最后才调用，以使得 API 更加易用。

# 参考
