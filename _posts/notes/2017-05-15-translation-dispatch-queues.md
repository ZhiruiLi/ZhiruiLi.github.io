---
layout: post
title: "[译] Apple 官方指南 - Dispatch Queues"
link: https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html
excerpt: "Apple 官方 Grand Central Dispatch（GCD）文档的翻译。"
date: 2017-05-17
modified: 2017-05-17
categories: notes
author: zrl
tags:
  - Objective-C
  - Concurrency
comments: false
share: true
---

Grand Central Dispatch（GCD）分派队列（dispatch queues）是一个用于处理任务（tasks）的强大工具。分派队列让你能够异步（asynchronously）或同步地（synchronously）执行任意的代码块（blocks of code）。你可以使用分派队列来处理几乎所有的可放在不同线程中处理的任务。使用分派队列的优点在于它们相对于直接使用线程来说要更加易用且更加高效。

本章将介绍分派队列，并提供了关于如何在自己的应用程序中用它们来执行一般任务的参考。如果你希望将当前直接使用线程的代码改为使用分派队列，你可以在 [Migrating Aray from Threads](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ThreadMigration/ThreadMigration.html) 里找到一些额外的提示。

* TOC
{:toc}
# 关于分派队列

分派队列能简化异步并发（concurrently）处理任务的过程。所谓任务就是指你的应用程序中需要处理的一些工作。例如定义一个任务用来处理一些计算、创建或修改一个数据结构、从一个文件中读取数据或者做其他的事情。定义一个任务的方式是将相应的代码放进一个函数（function）或者一个块对象（block object）中并将其添加进一个分派队列。

分派队列是一个类似于对象的结构，它负责管理你向它提交的任务。所有的分派队列都是一个先入先出的数据结构。所以，你添加进队列的任务的开始顺序都和添加顺序一样。GCD 自动提供了一些分派队列，你也可以根据特定的需求创建其他的分派队列。表 1 列出了你能在应用程序中获取到的分派队列以及你使用它们的方式。

**表 1**：分派队列的类型

| 类型                               | 描述                                       |
| :------------------------------- | ---------------------------------------- |
| 串行<br/>（Serial）                  | 串行队列（又被称为私有分派队列（private dispatch queues））在同一时间内只会执行一个任务，并且执行的顺序是你向该队列添加任务的顺序。当前正在执行的任务运行于一个特定的线程中（不同任务可能会运行于不同的线程中），该过程由分派队列进行管理。串行队列常常被用来同步对特定资源的访问。<br/>你可以根据你的需要创建任意数量的串行队列，每一个串行队列的操作是与其他队列并发进行的。换句话说，如果你创建了四个串行队列，每一个队列在同一时间内只执行一个任务，但仍然可能最多有四个任务并发执行，每一个任务都来自单独的一个队列。参看「创建串行分派队列」一节以获取关于如何创建串行队列的信息。 |
| 并发<br/>（Concurrent）              | 并发队列（又被称为一种全局分派队列（a type of global dispatch queue））并发地执行一个或多个任务，但是任务仍然按照它们被添加进队列的顺序来启动。并发执行的任务运行于不同的线程中，该过程由分派队列来进行管理。在每一个时间点执行的具体任务数是可变的，具体数量取决于系统的状况。<br/>在 iOS 5 或之后的版本中，你可以通过将队列类型设置为 `DISPATCH_QUEUE_CONCURRENT` 来创建自己的并发分派队列。不仅如此，系统还提供了四个预定义的全局并发队列供你的应用程序使用。参看「获取全局并发分派队列」一节以获取更多关于如何获取全局并发队列的信息。 |
| 主分派队列<br/>（ Main dispatch queue） | 主分派队列是一个全局的串行分派队列，它在应用程序的主线程中执行任务。该队列和应用程序的运行循环（run loop)（如果有的话）协同工作，将排队中的任务和其他的事件源交错放入运行循环中。因为主队列在你的应用程序的主线程中执行任务，所以它常常被用作一个应用程序的关键同步点。<br/>尽管你不需要创建一个主分派队列，然而你需要确保你的应用程序合理地声明了（drains）它。参看「在主线程中处理任务」一节以获取关于该队列是如何被管理的信息。 |
{: .table}

当涉及到提高一个应用程序并发性的时候，分派队列相对于线程来说有几个优势。最直接的优势就是工作队列编程模型较为简单。使用线程的时候，你需要同时写两部分的代码，一个是待处理工作的代码，另一个是创建和管理线程的代码。分派队列则让你专注于你要处理的工作，系统帮助你处理所有线程的创建和管理工作，使你不需要担心线程的创建和管理。这里有一个优势在于系统可以比单个应用程序更加高效地管理线程，系统能够根据可用资源和当前系统的状态动态地增减线程数量。不仅如此，系统还常常能比你自己创建线程时更加快速地开始运行你的任务。

尽管你可能认为将你的代码重写为使用分派队列的形式会比较困难，但事实上写使用分派队列的代码经常比写使用线程的代码简单。写这样的代码的关键在于设计自包含的（self-contained）能异步执行的独立任务。（事实上无论是使用分派队列还是直接使用线程，你都应该这样设计。）分派队列的一个优势是它的可预测性。如果你有两个运行于不同线程的任务访问同一个资源，其中任意一个线程可能先修改该资源，此时你会需要用一个锁来确保这两个任务不会同时修改该资源。如果你使用了分派队列来实现这一逻辑，你可以将这两个任务添加到一个串行队列中以确保在任意给定时间内，只有一个任务在修改该资源。这种基于分派队列的同步比使用锁要更加高效，因为锁无论是在争用还是在无争用的情况下都需要进行一个代价高昂的内核陷阱中断，而分派队列则主要工作在应用程序的进程空间里，只有当必须要陷入内核的时候才会陷入内核。

你也许会指出，两个运行于一个串行队列中的任务并没有并发地运行，尽管这是对的，但你要记住，如果两个线程在争用一个锁，那么任何线程提供的并发性都会失去或是大幅减少。更重要的是，线程编程模型需要创建两个线程，这需要申请内核和用户空间的内存。分派队列则不需要付出这种创建线程的内存代价，它们使用的线程总是处于占用状态并且不会阻塞。

关于分派队列，你需要记住一些关键点：

- 一个分派队列与其他分派队列并发地执行任务。任务的顺序性只限于单个分派队列内。
- 在任意时间执行任务的总数由系统决定。所以，如果一个应用程序将 100 个任务放进 100 个不同的分派队列中，那么这些任务并不一定会并发地执行（除非有 100 个或者更多个可用的核）。
- 系统在选择开始一个新的任务的时候会考虑队列的优先级。参看「向分派队列提供一个清理函数」一节以获取有关如何设置串行队列的优先级的信息。
- 队列中的任务必须在它被添加进队列的时候就要作好被调用的准备。（如果你曾经用过 Cocoa 操作对象（Cocoa operation objects），注意该行为与模型操作不同。）
- 私有分派队列是引用计数的对象。除了在你自己的代码中保持（retain）对队列的引用，你还需要注意分派源也可以被加入到一个队列中，这也会增加其保持计数。所以，你必须确保所有分派源都被取消了（canceled）且每一个保持调用都有一个合适的释放（release）调用与之平衡。参看「分派队列的内存管理」一节以获取更多有关保持和释放队列的信息。参看 [About Dispatch Sources](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/GCDWorkQueues/GCDWorkQueues.html#//apple_ref/doc/uid/TP40008091-CH103-SW12) 以获取更多关于分派源的信息。

参看 [Grand Central Dispatch (GCD) Reference](https://developer.apple.com/reference/dispatch?language=objc)（注：原链接失效，这里替换了另一个链接）以获取更多有关分派队列的操作接口的信息。

# 与分派队列相关的技术

除了分派队列，GCD 还提供了几个相关技术来帮助你使用分派队列管理代码。表 2 列出了这些技术并提供了链接供你获取更多相关信息。

**表 2**：一些使用分派队列的技术

| 技术                                | 描述                                       |
| :-------------------------------- | ---------------------------------------- |
| 分派组<br/>（Dispatch groups）         | 分派组是一个用于监控一组块对象完成的方法。（你可以根据你的需求同步或异步地进行监控。）分派组为那些依赖于其他任务完成的代码提供了一个有用的同步机制。参看「等待排队中的任务组」一节来获取更多有关使用分派组的信息。 |
| 分派信号量<br/>（Dispatch   semaphores） | 分派信号量与传统的信号量类似，但一般来说它会更加高效。分派信号量只有当调用线程由于信号量不可用而被阻塞时才会陷入内核。如果分派信号量可用，那就不会产生内核调用。参看「使用分派信号量来调整有限资源的使用」一节来获取更多有关如何使用分派信号量的例子。 |
| 分派源<br/>（ Dispatch sources）       | 分派源可以生成通知来响应特定类型的系统事件。你可以使用分派源来监控事件的发生，例如，进程通知、信号以及其他的描述符事件（descriptor events）。当一个事件发生时，分派源会将你的任务代码异步地提交到指定的分派队列中进行处理。参看 [Dispatch Sources](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/GCDWorkQueues/GCDWorkQueues.html) 以获取更多关于创建和使用分派源的信息。 |
{: .table}

# 使用块实现任务

块对象是一个基于 C 的语言特性，你可以在你的 C、Objective-C 以及 C++ 代码中使用它。块使得你可以很容易地定义一个自包含的独立工作单元。尽管一个块可能看起来像一个函数指针，但它在底层其实是被表示为一个类似于对象的数据结构并由编译器负责为你构建和管理。编译器将你提供的代码（和相关的数据一起）打包，并将其进行封装，使得它能够在堆上存活，且能够在你的应用程序中被传递。

块的一个关键优势是它们可以使用它们自己的词法作用域（lexical scope）外部的变量。当你在一个函数或方法中定义一个块的时候，这个块将以某种形式充当传统的代码块。例如，一个块可以读取定义于其父作用域中的变量的值。被块访问的变量被复制到堆上的块数据结构中，这使得它们能够在之后被块获取。当一个块被添加进一个分派队列中时，这些值通常被设置为只读的格式。但是，被同步执行的块也可以使用带有 `__block` 关键字的变量来将数据返回到父调用域（parent’s calling scope）中。

你可以使用类似函数指针的语法来在你的代码中内联（inline）声明一个块。块和函数指针在语法上的主要区别在于块名前面是一个插入符（^），而函数指针名前面是一个星号（*）。类似于函数指针，你可以给一个块传入参数，并获取返回值。代码清单 1 展示了如何在你的代码中定义和同步执行一个块。变量 `aBlock` 被声明为一个接受一个整数参数并且不返回任何值的块。一个符合这个块原型（prototype）的实际块被内联声明并赋值给 `aBlock`。最后一行代码立即执行了这个块，向标准输出打印了指定的整数。

**代码清单 1**：块的一个简单例子

```c
int x = 123;
int y = 456;

// Block declaration and assignment
void (^aBlock)(int) = ^(int z) {
    printf("%d %d %d\n", x, y, z);
};

// Execute the block
aBlock(789);   // prints: 123 456 789
```

下面总结了一些在设计你的块时你需要考虑的关键指导方针：

- 对于你计划使用分派队列进行异步处理的块，从父函数或方法捕获标量变量（scalar variables）并在块中使用它们是安全的。然而，你不应该尝试去捕获那些由调用环境（calling context）负责申请和删除的大型数据结构（large structures）或者其他基于指针的变量（pointer-based variables）。在你的块被执行的时候，指针所引用的内存可能会被释放掉。当然，如果你自己申请一块内存（或一个对象）并显式地将其所有权交给块，那么调用就是安全的了。
- 分派队列会复制那些被添加进队列的块，并在块执行结束的时候释放它们。换句话说，你不需要在向队列添加块时显式复制这些块。
- 尽管分派队列能比原始线程更加高效地执行小任务，但创建块和在分派队列中执行它们始终都会存在一定的开销。如果一个块做的工作太少，那么内联地执行它可能比将其分派到一个分派队列中更好。判断一个块做的工作是否太少的方式是用性能工具收集每一个执行路径的性能指标并进行比较。
- 不要缓存与底层线程相关的数据（data relative to the underlying thread）并指望该数据可以被另一个块访问。如果同一个队列中的任务需要共享数据，使用分派队列的上下文指针（context pointer of the dispatch queue）来存储数据。参看「在分派队列内存储自定义的上下文信息」一节以获取更多关于如何获取分派队列的上下文数据的信息。
- 如果你的块创建比较多的 Objective-C 对象，你应该考虑去用 `@autorelease` 块来包裹部分的块代码来对这些对象进行内存管理。尽管 GCD 分派队列有它们自己的自动释放的内存池（autorelease pools)，但它并不保证这些内存池里的对象何时被排空（drained）。如果你的应用程序内存受限，那么构建自己的自动释放的内存池可以使得你更加稳定地（at more regular intervals）释放需要被自动释放的对象（autoreleased objects）。

参看 [Blocks Programming Topics](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Blocks/Articles/00_Introduction.html) 来获取更多关于定义和使用块的信息。参看「向分派队列添加任务」一节来获取关于如何向分派队列中添加块的信息。

# 创建和管理分派队列

在你向一个队列添加任务之前，你需要确定你想使用的队列的类型以及你将如何使用它。分派队列可以顺序或并发地执行任务。除此之外，如果你心中有一个特定用途的分派队列，那么你可以根据你的设想来配置该队列的属性。接下来的几节将向你展示如何创建分派队列并对它们进行配置。

## 获取全局并发分派队列

当你有多个可以并行（parallel）运行的任务时，并发分派队列就非常有用了。并发队列仍然是一个队列，所以其中的任务在出队时仍然遵守先进先出的顺序，然而，并发队列可能在任何先前的任务执行完毕之前就令更多的任务出队。在任一给定的时刻，实际被并发队列执行的任务数是一个变量，它可以动态地根据你的应用程序状态进行改变。许多因素都会影响并发队列同时执行的任务数量，包括可用的核心数量，其他进程正在完成的工作数以及其他串行分派队列的任务数和优先级。

系统向每一个应用程序提供了四个并发分派队列。这些队列在单个应用程序内是全局共享的，它们之间的区别仅在于其优先级。由于它们是全局的，所以你不会去显式创建它们，而是通过 [`dispatch_get_global_queue`](https://developer.apple.com/reference/dispatch/1452927-dispatch_get_global_queue) 来获取其中任意一个队列，如下面的代码所示：

```c
dispatch_queue_t aQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
```

除了获取默认的并发队列之外，你还可以通过分别将常量 [`DISPATCH_QUEUE_PRIORITY_HIGH`](https://developer.apple.com/reference/dispatch/dispatch_queue_priority_high) 和 [`DISPATCH_QUEUE_PRIORITY_LOW`](https://developer.apple.com/reference/dispatch/dispatch_queue_priority_low) 传入该函数来获取高优先级和低优先级的分派队列，或者你还可以通过传入 [`DISPATCH_QUEUE_PRIORITY_BACKGROUND`](https://developer.apple.com/reference/dispatch/dispatch_queue_priority_background) 常量来获取一个后台分派队列。正如你可能预计的一样，高优先级并发队列中的任务在默认和低优先级队列中的任务之前被分派。类似地，默认的队列中的任务在低优先级队列中的任务之前被分派。

> **注意**：`dispatch_get_global_queue` 函数的第二个参数被保留作未来的扩展使用，你应该永远给该参数传递 `0`。

尽管分派队列是引用计数的对象，然而你并不需要保持和释放全局的并发队列。因为它们在你的应用程序中是可以全局访问的，对这些队列的保持和释放调用都会被忽略。所以，你不需要存储对这些队列的引用。你可以在任何你需要获取这些队列的引用的时候直接调用 `dispatch_get_global_queue` 函数来获得它们。

## 创建串行分派队列

当你想要你的任务按照特定的顺序执行时，串行队列就很有用了。串行队列在同一个时间内只会执行一个任务，而且它总会从队列的头部推出新的任务。当需要保护一个共享资源或是一个可变的数据结构时，你应该使用串行队列而非锁。串行队列和锁不同，它能确保任务以一个可预测的顺序被执行。而且，除非你异步地向一个串行队列提交你的任务，串行队列就绝不会发生死锁。

并发队列被系统创建好供你使用，而串行队列则需要你自己去显式地创建和管理。你可以为你的应用程序创建任意数量的串行队列，但你应该避免完全为了尽可能多地同时执行任务而创建大量的串行队列。如果你希望并发地执行大量的任务，那你应该将它们提交到一个全局的并发队列中。当创建串行队列的时候，你应该试着去将每一个队列与一个特定的目的联系起来，例如保护一个资源或是同步一些你的应用程序的关键行为。

代码清单 2 展示了创建一个串行队列所需要的步骤。[`dispatch_queue_create`](https://developer.apple.com/reference/dispatch/1453030-dispatch_queue_create) 函数有两个参数：一个是队列的名称，另一个是队列的一组属性。调试器和性能工具会显示队列的名称来帮你跟踪你的任务是如何被执行的。队列的属性被保留作未来的扩展使用，它应该永远被设置为 `NULL`。

**代码清单 2**：创建一个新的串行队列

```c
dispatch_queue_t queue;
queue = dispatch_queue_create("com.example.MyQueue", NULL);
```

除了你创建的串行队列之外，系统还自动创建了一个串行队列并将其绑定在你的应用程序的主线程上。参看「在运行时获取公用分派队列」一节以获取更多有关获取主线程队列的信息。

## 在运行时获取公用分派队列

GCD 为你提供了一些函数来让你在应用程序中获取几个公用分派队列：

- 使用 [`dispatch_get_current_queue`](https://developer.apple.com/reference/dispatch/1493248-dispatch_get_current_queue) 函数来进行调试或辨识当前的队列。在一个块对象中调用该函数将返回这个块被提交到的队列（这也是它可能正在其上运行的队列）。在块外调用该函数将返回你的应用程序的默认并发队列。
- 使用 `dispatch_get_main_queue` 函数来获取与你的应用程序主线程相关联的串行分派队列。系统会为 Cocoa 应用程序以及调用过 [`dispatch_main`](https://developer.apple.com/reference/dispatch/1452860-dispatchmain) 函数或是在主线程上配置过运行循环（使用 [`CFRunLoopRef`](https://developer.apple.com/reference/corefoundation/cfrunloop) 类型或是 [`NSRunLoop`](https://developer.apple.com/reference/foundation/nsrunloop) 对象）的应用程序自动创建该队列。
- 使用 [`dispatch_get_global_queue`](https://developer.apple.com/reference/dispatch/1452927-dispatch_get_global_queue) 函数来获取任意一个共享并发队列。参见「获取全局并发分派队列」一节以获取更多信息。 

## 分派队列的内存管理

分派队列和其他分派对象（dispatch objects）都是引用计数的数据类型。当你创建一个串行分派队列时，它会有一个初始为 1 的引用计数。你可以使用 [`dispatch_retain`](https://developer.apple.com/reference/dispatch/1496306-dispatch_retain) 和 [`dispatch_release`](https://developer.apple.com/reference/dispatch/1496328-dispatch_release) 函数来按需增减其引用计数。当一个队列的引用计数变为零的时候，系统将会自动地释放掉该队列。

通过保持和释放分派对象，例如分派队列，来确保它们被使用时还依然存在于内存中，是非常重要的。此处的规则和对 Cocoa 对象进行内存管理一样，即你如果计划使用一个被传入你的代码中的队列，那么你应该在使用它前保持它，然后在你不需要它的时候释放它。该基本模式可以确保只要你正在使用某个队列，那它就一定存在于内存之中。

> **注意**：你不需要去保持和释放包括并发分派队列和主分派队列在内的全局分派队列。任何对这些队列的保持和释放的企图都会被忽略。

就算你在实现一个带有垃圾回收的应用程序，你仍然需要保持和释放你的分派队列和其他分派对象。GCD 并不支持对回收内存的垃圾回收模式。

## 在分派队列内存储自定义的上下文信息

所有的分派对象（包括分派队列）都允许你将一个自定义的上下文数据关联到该对象上。通过使用 [`dispatch_set_context`](https://developer.apple.com/reference/dispatch/1452807-dispatch_set_context) 和 [`dispatch_get_context`](https://developer.apple.com/reference/dispatch/1453005-dispatch_get_context) 这两个函数来设置和获取与对象关联的数据。系统不会以任何方式使用你自定义的这些数据，而且这些数据的申请和释放都应该由你自己在合适的时机完成。

对于分派队列来说，你可以使用上下文数据来存储指向一个 Objective-C 对象或者其他数据结构的指针，这能帮助你辨识该队列或是在你的代码中使用这些数据。你可以利用分派队列的终止器（finalizer）函数来在队列被释放之前释放（或取消与队列的关联）上下文数据。代码清单 3 是一个关于如何写一个清理队列的上下文数据的终止器函数的例子。

## 向分派队列提供一个清理函数

在你创建一个串行分派队列之后，你可以给它附上一个终止器函数，以便于在分派队列被释放时完成你自定义的清理工作。分派队列是引用计数的对象，你可以使用 [`dispatch_set_finalizer_f`](https://developer.apple.com/reference/dispatch/1453100-dispatch_set_finalizer_f) 函数来指定一个当你的队列的引用计数变为零的时候执行的函数。该函数被用来清理与队列相关联的上下文数据，而且仅当上下文指针不为 `NULL` 时，该函数才会被调用。

代码清单 3 展示了一个自定义的终止器函数以及一个用于创建分派队列并给分派队列装配该终止器函数的函数。该分派队列使用该终止器函数来释放存储于队列上下文指针中的数据。（代码中所使用的 `myInitializeDataContextFunction` 和 `myCleanUpDataContextFunction` 两个函数是你可能会提供的用于初始化和清理该数据结构本身内容的函数。）被传入终止器的上下文指针包含了与分派队列相关联的数据。

**代码清单 3**：给一个分派队列装配清理函数

```c
void myFinalizerFunction(void *context)
{
    MyDataContext* theData = (MyDataContext*)context;

    // Clean up the contents of the structure
    myCleanUpDataContextFunction(theData);

    // Now release the structure itself.
    free(theData);
}

dispatch_queue_t createMyQueue()
{
    MyDataContext*  data = (MyDataContext*) malloc(sizeof(MyDataContext));
    myInitializeDataContextFunction(data);

    // Create the queue and set the context data.
    dispatch_queue_t serialQueue = dispatch_queue_create("com.example.CriticalTaskQueue", NULL);
    dispatch_set_context(serialQueue, data);
    dispatch_set_finalizer_f(serialQueue, &myFinalizerFunction);

    return serialQueue;
}
```

# 向分派队列添加任务

你需要将一个任务分派给一个合适的分派队列来执行该任务。你可以同步或异步地、单个或成组地将任务分派给分派队列。一旦一个任务进入了分派队列，该队列就将负责尽快地执行你的任务，该过程会受到当前环境的约束以及已存在于队列中的任务的影响。这一节将向你展示一些关于将任务分派给分派队列的技术并将向你阐述它们的优点。

## 向分派队列添加单个任务

有两种方式可以向分派队列添加任务：异步或同步。如果可能的话，使用 [`dispatch_async`](https://developer.apple.com/reference/dispatch/1453057-dispatch_async) 和 [`dispatch_async_f`](https://developer.apple.com/reference/dispatch/1452834-dispatch_async_f) 两个函数来异步地执行任务会比使用同步的方式更好。当你向一个分派队列中添加一个块对象或是一个函数时，你是无法知道你添加的代码什么时候会被执行的。因此，异步地添加块或函数使得你可以调度（schedule）一段代码的执行并调用线程里继续其他工作。当你需要在你的应用程序的主线程中调度一个任务的时候——或许是响应一些用户事件，这显得尤为重要。

尽管你应该尽可能地以异步的方式添加任务，但你有时还是会需要同步地添加任务来防止竞争条件或是其他同步错误。在这些情况下，你可以使用 [`dispatch_sync`](https://developer.apple.com/reference/dispatch/1452870-dispatch_sync) 和 [`dispatch_sync_f`](https://developer.apple.com/reference/dispatch/1453123-dispatch_sync_f) 两个函数来向分派队列添加任务。这两个函数将阻塞当前线程的执行直到指定的任务执行完毕。

> **重要**：永远不要在一个任务中调用 `dispatch_sync` 和 `dispatch_sync_f` 向该任务所属的队列添加新的任务。这对于串行队列来说尤为重要，这一定会导致死锁，而且你也应该避免对并发队列做这件事。

下面的例子展示了如何分别使用同步和异步的方式基于块来派发任务：

```c
dispatch_queue_t myCustomQueue;
myCustomQueue = dispatch_queue_create("com.example.MyCustomQueue", NULL);
 
dispatch_async(myCustomQueue, ^{
    printf("Do some work here.\n");
});
 
printf("The first block may or may not have run.\n");
 
dispatch_sync(myCustomQueue, ^{
    printf("Do some more work here.\n");
});
printf("Both blocks have completed.\n");
```

## 当任务完成时执行一个完成块

分派到队列的任务天生就独立于创建它们的代码来运行。然而，你的应用程序仍可能希望当任务结束的时候获得一个通知，以便于它获取任务执行的结果。在传统的异步编程中你可能会使用回调机制来做这件事，而对于派发队列，你可以使用完成块（completion block）。

一个完成块只不过是另一段你分派给一个队列并添加到原始任务结束处的代码。典型的做法是调用代码在开始一个任务的时候以参数的形式提供一个完成块。所有的任务代码都需要在它完成其工作的时候向指定的分派队列提交指定的块或函数。

代码清单 4 展示了一个用块实现的求平均数的函数。这个求平均数的函数的最后两个参数允许调用者指定一个分派队列和一个块用于回报结果。在这个求平均数的函数计算出其结果之后，它将把结果传递给指定的块，并将其分派到指定的队列中。为了防止分派队列被过早地释放，你应该在一开始先保持该队列并在完成块被分派出去后释放该队列。

**代码清单 4**：在一个任务结束之后执行一个完成回调

```c
void average_async(int *data, size_t len,
   dispatch_queue_t queue, void (^block)(int))
{
   // Retain the queue provided by the user to make
   // sure it does not disappear before the completion
   // block can be called.
   dispatch_retain(queue);
 
   // Do the work on the default concurrent queue and then
   // call the user-provided block with the results.
   dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
      int avg = average(data, len);
      dispatch_async(queue, ^{ block(avg);});
 
      // Release the user-provided queue when done
      dispatch_release(queue);
   });
}
```

## 并发处理循环迭代

一个可能可以通过使用并发分派队列来提升性能的地方是你需要处理有固定迭代次数的循环的时候。例如，假设你有一个 `for` 循环在每次迭代的时候都做一些工作：

```c
for (i = 0; i < count; i++) {
   printf("%u\n",i);
}
```

如果在每次迭代时处理的工作相对于其他迭代来说是独立的，而且循环的每一次迭代完成的顺序不重要的话，你可以将循环替换为调用 [`dispatch_apply`](https://developer.apple.com/reference/dispatch/1453050-dispatch_apply) 或 [`dispatch_apply_f`](https://developer.apple.com/reference/dispatch/1452846-dispatch_apply_f) 函数。这两个函数在每一次循环迭代的时候都将指定的块或函数提交到一个分派队列中。当任务被分派到一个并发队列的时候，就有可能使得多个循环迭代在同一时间进行处理。

在调用 `dispatch_apply` 或 `dispatch_apply_f` 的时候，你可以指定使用的是串行还是并发的分派队列。将并发分派队列传入这两个函数将使你可以同时处理循环的多次迭代，这也是这两个函数最常见的使用方式。尽管你也可以在代码中使用串行分派队列并得到正确的结果，但这相较于直接使用循环来说并不能获得任何性能上的提升。

> **重要**：就像通常的 `for` 循环一样，`dispatch_apply` 和 `dispatch_apply_f` 函数在所有的循环迭代结束之前并不会返回。所以当你需要在已经在队列的上下文中执行的代码里调用这两个函数的时候需要非常小心。如果你将正在执行当前代码的队列作为参数传入这两个函数而且这个队列又是串行队列的话，那该调用将造成队列的死锁。
>
> 由于这两个函数会阻塞当前线程，当在你的主线程里调用这两个函数的时候也需要非常小心，这将阻止你的事件处理循环及时地响应事件。如果你的循环代码需要比较多的处理时间，你应该考虑在另一个线程里调用这两个函数。

代码清单 5 展示了如何将之前的 `for` 循环代码替换为使用 `dispatch_apply` 的代码。你传入 `diapatch_apply` 函数的块必须能接收一个参数用以识别当前的循环。当这个块被执行的时候，对于第一次迭代，该参数的值会为 `0`，对于第二次迭代，该参数的值会为 `1`，以此类推。对于最后一次迭代，该参数的值会为 `count - 1`，其中，`count` 是迭代的总次数。

**代码清单 5**：并发地处理 `for` 循环的迭代

```c
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
 
dispatch_apply(count, queue, ^(size_t i) {
   printf("%u\n",i);
});
```

你应该确保你的任务代码在每次迭代中都做了合理数量的任务，因为你派发到队列的每一个块或函数都会造成调度执行它们的代码的开销。如果你的循环中每一次迭代都只处理了很少量的任务，那么调度代码所造成的开销可能将超过你将其分派到队列来执行所得到的性能提升。如果你在测试中发现上述情况属实，那么你可以考虑以加大步长的方式增加循环的每一次迭代所处理的工作量。你可以将多次迭代合并到一个块里并加大步长，这将相应地减少迭代次数。例如，如果你本来要处理 100 次迭代，而你现在决定要将步长修改为 4，那么你现在就要在每一个块中处理循环中的 4 次迭代，同时，迭代的次数将变为 25。参看 [Improving on Loop Code](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ThreadMigration/ThreadMigration.html#//apple_ref/doc/uid/TP40008091-CH105-SW2) 以查看有关如何修改步长的例子。

## 在主线程处理任务

GCD 提供了一个特殊的串行分派队列使得你可以在你的应用的主线程里执行任务。该队列由系统自动提供给每一个应用程序，而且所有在主线程上设置了运行循环（用 [`CFRunLoopRef`](https://developer.apple.com/reference/corefoundation/cfrunloop) 类型或是 [`NSRunLoop`](https://developer.apple.com/reference/foundation/nsrunloop) 对象进行管理）的应用程序都会自动声明该队列。如果你不是在创建一个 Cocoa 应用程序而且你也不想显式地创建一个运行循环，你就必须要调用 [`dispatch_main`](https://developer.apple.com/reference/dispatch/1452860-dispatchmain) 函数来显式声明主分派队列。即使上述的条件不满足，你仍然可以向该队列中添加任务，但这些任务永远都不会被执行。

你可以通过调用 `dispatch_get_main_queue` 函数来获取你的应用程序主线程的分派队列。被添加进该队列的任务将被串行地在主线程上处理。所以，你可以将该队列用作被你的应用程序中其他部分完成的工作的同步点。

## 在你的任务中使用 Objective-C 的对象

GCD 提供了内置的对 Cocoa 内存管理技术的支持，所以你可以随意地在被提交给分派队列的块中使用 Objective-C 对象。每一个分派队列都会维护自己的自动释放的内存池以确保所有需要被自动释放的对象都被在某些时间点上被释放；分派队列并不保证这些对象的具体释放时机。

如果你的应用程序的内存受限，而且你的块创建了不少需要被自动释放的对象，那么唯一能确保你的对象能被及时地释放掉的方式就是创建你自己的自动释放的内存池。如果你的块创建了成百上千个对象，你应该考虑创建更多的自动释放的内存池或者定期排空你的内存池。

参考 [Advanced Memory Management Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html) 来获取更多关于自动释放的内存池以及 Objective-C 内存管理相关的信息。

# 暂停和恢复分派队列

你可以通过暂停（suspend）一个分派队列来暂时阻止它执行块对象。调用 [`dispatch_suspend`](https://developer.apple.com/reference/dispatch/1452801-dispatch_suspend) 函数来暂停一个分派队列，调用 [`dispatch_resume`](https://developer.apple.com/reference/dispatch/1452929-dispatch_resume) 函数来恢复（resume）该队列。调用 `dispatch_suspend` 会增加队列的暂停引用计数（suspension reference count），调用 `dispatch_resume` 则会减少该引用计数。当该引用计数大于零的时候，这个分派队列就保持暂停状态。所以，为了能够正确恢复处理块的过程，你需要进行和暂停调用数量相同的恢复调用。

> **重要**：暂停和恢复调用是异步的，而且这个调用效果仅仅会在两个块的执行过程之间产生。暂停一个分派队列并不会导致一个正在执行的块停止。

# 使用分派信号量来调整有限资源的使用

如果被提交到分派队列的任务访问了一些有限的资源，你可能希望使用一个分派信号量来控制同时访问该资源的任务数量。分派信号量和一般的信号量相比只有一个区别，当资源可用的时候，获取一个分派信号量的时间比获取一个传统的系统信号量要少。这是因为 GCD 并不会在这种情况下陷入内核调用。仅当资源不可用的时候 GCD 才会陷入内核调用，系统需要将你的线程暂停直到信号量收到信号。

分派信号量的语义如下：

1. 当你创建信号量的时候（使用 [`dispatch_semaphore_create`](https://developer.apple.com/reference/dispatch/dispatchsemaphore/1452955-init) 函数），你可以指定一个正整数来表示可用的资源数。
2. 在每一个任务里，通过调用 [`dispatch_semaphore_wait`](https://developer.apple.com/reference/dispatch/1453087-dispatch_semaphore_wait) 来等待一个信号量。
3. 当等待调用返回时，你就可以申请资源并去做你的工作了。
4. 在你使用完该资源后，你就可以释放资源并调用 [`dispatch_semaphore_signal`](https://developer.apple.com/reference/dispatch/dispatchsemaphore/1452919-signal) 函数来向信号量发出信号。

举一个关于这几个步骤的例子，考虑使用系统的文件描述符的场景。每一个应用程序都只能使用有限个文件描述符。如果你有一个需要处理大量文件的任务，你不会想同时打开如此多的文件以至于文件描述符被耗尽。相反，你可以使用一个信号量来限制你的文件处理程序在任意时间点上同时占用的文件描述符的数量。下面的代码片段是你将会包含进你的任务中的基本代码：

```c
// Create the semaphore, specifying the initial pool size
dispatch_semaphore_t fd_sema = dispatch_semaphore_create(getdtablesize() / 2);
 
// Wait for a free file descriptor
dispatch_semaphore_wait(fd_sema, DISPATCH_TIME_FOREVER);
fd = open("/etc/services", O_RDONLY);
 
// Release the file descriptor when done
close(fd);
dispatch_semaphore_signal(fd_sema);
```

当你创建一个信号量的时候，你需要指定可用资源的数量。这个值将会作为信号量的初始计数值。每当你去等待一个信号量的时候，`dispatch_semaphore_wait` 函数会将该计数值减 1。如果减少后的结果是一个负数，那么该函数会通知内核将你的线程阻塞。另一方面，`dispatch_semaphore_signal` 函数将计数值增 1 以指示资源已经被释放。如果有任务正在阻塞等待一个资源，那么其中一个任务就会被解除阻塞并被允许进行其工作。

# 等待排队中的任务组

分派组是一个用于阻塞等待一个或多个任务执行结束的方法。当下一步的工作需要等待特定任务结束之后才能进行的时候你可以使用这一行为。例如，在分派了多个任务去计算一些数据之后，你可以使用一个组来等待这些任务，然后在它们都执行完毕后处理它们计算的结果。另一个使用分派组的场景是用它取代线程的连接（join）。你可以将多个任务加入一个分派组中并等待整个组的完成，而非开启多个子线程然后将当前线程与每一个线程进行连接。

代码清单 6 展示了设置分派组，向其分派任务以及等待其结果的基本过程。在向队列分派任务时，你不应该使用 [`dispatch_async`](https://developer.apple.com/reference/dispatch/1453057-dispatch_async) 函数而应该使用 [`dispatch_group_async`](https://developer.apple.com/reference/dispatch/1453084-dispatch_group_async) 函数。这个函数将任务和一个组关联起来，并将其排队等待执行。在此之后，你可以通过调用 [`dispatch_group_wait`](https://developer.apple.com/reference/dispatch/1452794-dispatch_group_wait) 函数并传入一个组来等待该组的任务的完成。

**代码清单 6**: 等待异步任务

```c
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();
 
// Add a task to the group
dispatch_group_async(group, queue, ^{
   // Some asynchronous work
});
 
// Do some other work while the tasks execute.
 
// When you cannot make any more forward progress,
// wait on the group to block the current thread.
dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
 
// Release the group when it is no longer needed.
dispatch_release(group);
```

# 分派队列和线程安全

也许在使用分派队列的场景下讨论线程安全看起来很奇怪，但线程安全仍然是一个与之相关的话题。当你想在你的应用中实现并发的时候，你应该知道以下事情：

- 分派队列本身是线程安全的。换句话说，你可以从任何线程向分派队列提交任务而无需事先获取一个锁或是同步访问该队列。
- 不要在一个任务中调用 [`dispatch_sync`](https://developer.apple.com/reference/dispatch/1452870-dispatch_sync) 并向该任务所属的队列添加新的任务。这么做将导致队列的死锁。如果你需要将任务分派到当前队列，就通过调用 [`dispatch_async`](https://developer.apple.com/reference/dispatch/1453057-dispatch_async) 来异步地添加。
- 避免在提交给分派队列的任务中获取锁。尽管在你的任务中使用锁是安全的，但是当你去获取一个锁的时候，如果锁不可用的话，你可能会阻塞整个串行队列。类似地，对于并发队列，等待一个锁可能会阻止其他任务的执行。如果你需要同步你的部分代码，你应该使用串行分派队列而非锁。
- 尽管你可以获取关于正在运行任务的底层线程的信息，但你最好还是不要这么做。参看 [Compatibility with POSIX Threads](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ThreadMigration/ThreadMigration.html#//apple_ref/doc/uid/TP40008091-CH105-SW18) 以获取更多有关分派队列和线程的兼容性信息。

参看 [Migrating Aray from Threads](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ThreadMigration/ThreadMigration.html) 以获取关于如何将当前直接使用线程的代码改为使用分派队列的形式的额外提示。