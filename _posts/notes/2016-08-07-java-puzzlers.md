---
layout: post
title: Java Puzzlers
excerpt: "看 Google 的 Java Puzzlers 技术视频的笔记"
date: 2016-08-07
modified: 2016-08-07
categories: notes
author: zrl
tags:
  - Java
comments: false
share: true
---

* TOC
{:toc}

# The Joy of Sets

```java
public class ShortSet {
  public static void main(String args[]) {
    Set<Short> s = new HashSet<Short>();
    for (short i = 0; i < 100; i++) {
      s.add(i);
      s.remove(i - 1);
    }
  }
  System.out.println(s.size());
}
```

可能会认为这段代码输出的结果是 `1`，但实际上这段代码输出的结果是 `100`。

原因在于当调用 `i - 1` 时，这个计算结果不是 `short` 类型，而是 `int` 类型，当调用 `s.remove(i - 1)` 时，`i - 1` 的结果会被自动装箱，成为一个 `Integer` 类型的对象，而不是 `Short` 类型的对象，这导致 `Set<Short> s` 中根本没有要删除的对象，所以会出现 `100` 这个结果。

但为什么当用户想从一个存放 `Short` 类型对象的 `Set` 中移除一个 `Integer` 类型的对象而编译器并没有报错呢？原因是 `Set<T>` 提供的接口声明如下：

```java
public interface Set<E> extends Collection<E> {
  public abstract boolean add(E e);
  public abstract boolean remove(Ojbect o);
  ...
}
```

也就是说，当向一个 `Set<Short>` 对象中添加元素时，编译器会要求用户只能向其中添加 `Short` 类型的对象，但是移除元素的方法则是可以对任意类型的元素有效，并没有限定是 `Short` 类型（这个行为是类型安全的）。类似地，`Map` 的 `get` 方法声明也是 `Object` 类型，所以也可以传递任意类型的对象进去。

**启示**

- `Collection<E>.remove` 要求 `Object` 类型的参数而非 `E`
  - 同样地，`Collection.contains`，`Map.get` 也是
  - 要注意泛型类的方法的签名
- Integral arithmetic 总是返回 `int` 或是 `long`
  - 永远不会返回 `byte`、`short` 或是 `char`
- 避免混用类型
- 避免使用 `short`，最好用 `int` 和 `long`
  - 只有 `short` 类型的数组才是 `short` 类型唯一有意义的用例

# More Joy of Sets

```java
import java.net.*;
public class UrlSet {
  private static final String[] URL_NAMES = {
    "http://javapuzzlers.com",
    "http://apache2-snort.skybar.dreamhost.com",
    "http://www.google.com",
    "http://javapuzzlers.com",
    "http://findbugs.sourceforge.net",
    "http://www.cs.umd.edu"
  };
  public static void main(String[] args)
    throws HalformedURLException {
      Set<URL> favorites = new HashSet();
      for (String urlName : URL_NAMES) {
        favorites.add(new URL(urlName));
      }
      System.out.println(favorites.size());
    }
}
```

可能会认为这段代码输出的结果是 `5`，但实际上这段代码一般情况下输出的结果是 `4` 也有可能会出现其他结果。

原因是 `URL` 类的 `equals` 和 `hashcode` 有设计缺陷。

在刚刚的代码中，出现了两次 `"http://javapuzzlers.com"`，这两个地址是一样的，而它们又和 `"http://apache2-snort.skybar.dreamhost.com"` 具有同样的 IP 地址。在 `URL` 类的文档中有提到：

> - **如果两个 `URL`** 有同样的协议、**引用同样的主机**、有同样的端口号且在同一个文件的同一片段上，**那么他们是相等的**
> - **如果两个主机名能够解析出同一个 IP 地址，那么这两个主机就是相等的**。如果两个主机都无法解析，那么它们是相等的，或者如果两个主机名都是 `null` 那他们也是相等的。

这个做法违反了 `equals` 的协定，因为 `equals` 应该要与环境无关，而此处 `equals` 的结果依赖于主机解析，于是，当出现虚拟主机时，这段代码的结果就难以预知。而且 `URL` 的 `equals` 的调用还会造成网络 IO，导致阻塞。

正确的做法是，不要使用 `URL` 类，尽可能使用 `URI` 类，仅在必要的时候才通过调用 `URI` 中的方法将其转换为 `URL` 类的对象。`URI` 类的 `equals` 和 `hashcode` 的实现是正确的，它只依赖于字面地址。换用 `URI` 之后结果就是 `5` 了。

**启示**

- 不要使用 `URL` 作为 `Set` 的元素和 `Map` 的键
  - `equals` 和 `hashcode` 的实现有错误
  - 它们没有遵从一般的协议
- 使用 `URI`
  - 在需要的时候将 `URI` 转为 `URL`
- 在设计 API 时，要注意 `equals` 方法不应该依赖于环境

# Racy Little Number

```java
public class Test extends junit.framework.TestCase {
  int number;

  public void test() throws InterruptedException {
    number = 0;
    Thread t = new Thread(new Runnable() {
      public void run() {
        assertEquals(2, number);
      }
    });
    number = 1;
    t.start();
    number++;
    t.join();
  }
}
```

可能会认为这个测试有时成功有时失败，但实际上这个测试将永远都通过（但没有告诉我们任何信息）。

原因在于 JUnit 没有机会去看那个 `assertEquals(2, number)` 的断言是正确的还是错误的。JUnit 只会知道这个方法正常地返回了，所以它会认为这个方法没有出现错误。

解决方案比较麻烦，首先，要定义多一个方法：

```java
volatile Exception exception;
volatile Error error;

public void tearDown() throws Exception {
  if (error != null) throw error;
  if (exception != null) throw exception;
}
```

然后在测试的多线程内部这样写：

```java
Thread t = new Thread(new Runnable() {
  public void run() {
    try {
      assertEquals(e, number);
    } catch(Error e) {
      error = e;
    } catch(Exception e) {
      exception = e;
    }
  }
});
```

最后再去调用这个 `tearDown` 方法即可。

**启示**

- JUnit 不支持并发
- 在使用 JUnit 时，你需要自己提供相应的东西去支持并发

# Elvis Lives Again

```java
public class Elvis {
  // Singleton
  public static final Elvis ELVIS = new Elvis();

  private Elvis() { }

  private static final Boolean LIVING = true;

  private final Boolean alive = LIVING;

  public final Boolean lives()  { return alive; }

  public static void main(String[] args) {
    System.out.println(ELVIS.lives() ? "Hound Dog" : "Heartbreak Hotel");
  }
}
```

可能会认为这段代码输出的结果是 `Hound Dog`，但实际上这段代码会抛出一个 `NullPointerException` 异常。

这个错误的来源是类初始化的过程和自动拆箱。

首先，`public static final Elvis ELVIS = new Elvis();` 是一个递归的类初始化，然而，`private static final Boolean LIVING = true;` 这句的执行太晚了，以至于后面得到的只是一个 `null`。所以当执行 `ELVIS.lives() ? "Hound Dog" : "Heartbreak Hotel"` 这个自动拆箱的过程时，会抛出异常。

当初始化 `Elvis` 类时，需要创建 `Elvis` 类的对象 `ELVIS`，而创建 `Elvis` 类的对象又需要初始化 `Elvis` 类，这时 `Elvis` 类已经在初始化了，为了防止出现无穷的递归情况，编译器会将类后面的静态域初始化过程忽略而直接去初始化实例域，在这里，实例域只有一个，就是 `private final Boolean alive`。这个实例域的值是静态常量 `LIVING`，此时这个静态常量是什么呢？此时由于类还没初始化完成，还没执行到初始化 `LIVING` 这一句，又因为除了 `String` 类型对象之外，所有的引用类型对象都不是编译期的常量，所以它的值是 `null`。在初始化 `ELVIS` 之后才执行了 `LIVING` 的初始化，但此时已经太晚了，`ELIVIS` 的初始化已经结束，`null` 值已经被复制到了 `ELVIS` 的实例域 `alive` 中。

解决方案是交换语句的顺序，使得在 `ELVIS` 初始化之前先初始化 `LIVING` 即可：

```java
public class Elvis {
  private static final Boolean LIVING = true;
  // Singleton
  public static final Elvis ELVIS = new Elvis();
  ...
}
```

另一个解决方案是不要使用 `Boolean` 直接使用 `boolean`，因为基本类型有字面常量，不需要等待那个语句的执行。

**启示**

- 包装后的基本类型不再是基本类型
  - 尽可能使用基本类型而非包装后的基本类型
- 自动拆箱会在最不经意的时候发生
  - 而且可能导致 `NullPointerException`
- 永远不要将 `Boolean` 对象用于 3 值返回的情况（即：`true`、`false` 和 `null`）
- 注意环状的类初始化
  - 静态实例要在其他类最后进行初始化，不要在其他静态域之前

# Mind the Gap

```java
import java.io.*;
public class Gap {
  privagte static final int GAP_SIZE = 10 * 1024;
  public static void main(String args[]) throws IOException {
    File tmp = File.createTempFile("gap", ".txt");
    FileOutputStream out = new FileOutputStream(tmp);
    out.write(1);
    out.write(new byte[GAP_SIZE]);
    out.write(2);
    out.close();
    InputStream in = new BufferedInputStream(new FileInputStream(tmp));
    int first = in.read();
    in.skip(GAP_SIZE);
    int last = in.read();
    System.out.println(first + last);
  }
}
```

可能会认为这段代码输出的结果是 `3`，但实际上这段代码在实际情况下输出的结果是 `1`，而在理论上，它的结果每次运行都会改变。

错误的原因在于 `skip` 方法。这个方法接受一个 `long` 类型的值，用于指明需要忽略多少个字节的数据，并且会返回一个 `long` 值用于说明实际略过了都少个字节。这个返回值很重要，如果随便忽略这个返回值，将很容易导致程序出错，因为它可能没有略过用户要求的字节数。它的 API 只保证 `skip` 会略过 0 到输入值之间的某个数量的数据，对于 `BufferedInputStream` 来说，如果缓冲区有数据，它最多只会去略过缓冲区的那些字节，并不往下执行以保证效率。

因此，上面的程序中的 `last` 并没有读出最后一个数据而是中间的空白，也就是 `0`。解决方案是定义自己的略过方法：

```java
static void skipFully(InputStream in, long nBytes) throws IOException {
  long remaining = nBytes;
  while (remaining != 0) {
    long skipped = in.skip(remaining);
    if (skipped == 0)
      throw new EOFException();
    remaining -= skipped;
  }
}
```

**启示**

- `skip` 方法难以使用而且容易产生错误
- 用自己的略过方法而不要直接使用 `skip`
- 一般来说，如果一个 API 出错了，就将它包装一下来使用
- 对于 API 的设计者来说
  - 不要违背**最小惊讶原则**
  - 要使简单的事情足够易于完成

# Histogram Mystery

```java
public class Histogram {
  private static final String[] words =
    { "I", "recommend", "polygene", "lubricants" };
  public static void main(String[] args) {
    int[] histogram = new int[5];
    for (Strign word1 : words) {
      for (String word2 : words) {
        String pair = word1 + word2;
        int bucket = Math.abs(pair.hashCode()) % historgram.length;
        histogram[bucket] ++;
      }
    }
    int pairCount = 0;
    for (int freq : histogram)
      pairCount += freq;
    System.out.println(freq);
  }
}
```

可能会认为这段代码输出的结果是 `16`，但实际上这段代码会抛出 `ArrayOutOfBoundsException` 异常。

原因在于 `Math.abs(int)` 结果可能为负数，且 `%` 运算符也能产生负数的结果。

此处的字符串是特殊的，因为 `"polygenelubricants".hashCode()` 的值是 `Integer.MIN_VALUE`。而且 `Math.abs(Integer.MIN_VALUE)` 的结果就是 `Integer.MIN_VALUE`，这是个负数。

解决方法是先进行取余运算，再取绝对值：

```java
int bucket = Math.abc(pair.hashCode() % histogram.length);
```

**启示**

- `Math.abs` 不保证结果为非负数
  - `Integer.MIN_VALUE == -Integer.MIN_VALUE`
- `%` 运算符是取余数的运算符而非取模运算符，它也可能得出负数结果
- 将一个带符号的哈希值映射到桶中时，可以采取如下几种方法：
  - `Math.abs(hashVal % buckets.length)`  
  - `(hashVal >>> 1) % buckets.length`
  - `(hashVal & 0x7fffffff) % buckets.length`
  - 或使用长度为 2 的指数的数组，这样就可以直接写：`(hashVal & (buckets.length - 1))` 了

# A Sea of Troubles

```java
public class Hamlet {
  public static void main(String[] args) {
    Random rnd = new Random();
    boolean toBe = rnd.nextBoolean();
    Number result = (toBe || !toBe) ? new Integer(3) : new Float(1);
    System.out.println(result);
  }
}
```

可能会认为这段代码输出的结果是 `3`，但实际上这段代码输出的结果是 `3.0`。

这个错误产生于 `?:` 操作符在应用于类型不匹配的包装类型时的奇怪行为。当 `(toBe || !toBe) ? new Integer(3) : new Float(1)` 这个表达式并不是产生一个包裹着 `3` 的 `Integer`，而是产生一个包裹着 `3` 的 `Float`。

解决方案是如果不确定 `?:` 操作符的行为，就不要用它，将代码改为：

```java
Number result;
if (toBe || !toBe) {
  result = new Integer(3);
} else {
  result = new Float(1);
}
```

**启示**

- 避免混用类型
- `?:` 操作符在操作两个不同的包装类型时会有违反直觉的语义
- 如果一定要在两个包装类型的值之间进行选择，不要用 `?:` 操作符，而应该使用 `if-else`

# Ground Round

```java
public class Round {
  public static void main(String[] args) {
    Random rnd = new Random();
    int i = rnd.nextInt();
    if (Math.round(i) != i)
      System.out.println("Ground Round");
  }
}
```

可能会认为这个程序永远都不会打印出 `Ground Round`，但事实上，这个程序有非常高的概率（大约 96.7%）会打印这个字符串。

原因在于，`round` 没有在 `int` 类型的参数上进行重载，它只有两个重载版本：

```java
public static int round(float);
public static long round(double);
```

当传入 `int` 类型的值时，参数类型为 `float` 的版本将会被调用。由于 `float` 有 8 bits 的指数位，所以这个转换过程损失了精度。这是一个语言设计的错误，一般来说，不应该在这种有损精度的隐式转换下不报错或不提示警告。

解决方案是将其强制转换为 `double` 使得 `double` 的重载版本被调用，由于 `double` 是 64 bits 的，所以这个转换过程并不会损失精度。

```java
Math.round((double) i) != i
```

**启示**

- 从 `int` 到 `float` 的静默**放宽**转换是有损的且危险的
  - 类似地，从 `long` 到 `double` 也是
- `float` 类型很少被使用，尽可能用 `double`。除非有一个很大的 `float` 数组
- 方法的重载是危险的，尤其是出现隐式的**放宽**时

# Size Matters

```java
public class Size {
  private enum Sex { MALE, FEMALE }

  public static void main(String[] args) {
    System.out.print(size(new HashMap<Sex, Sex>()) + " ");
    System.out.print(size(new EnumMap<Sex, Sex>(Sex.class)));
  }

  private static int size(Map<Sex, Sex> map) {
    map.put(Sex.MALE, Sex.FEMALE);
    map.put(Sex.FEMALE, Sex.MALE);
    map.put(Sex.MALE, Sex.MALE);
    map.put(Sex.FEMALE, Sex.FEMALE);
    Set<Map.Entry<Sex, Sex>> set =
      new HashSet<Map.Entry<Sex, Sex>>(map.entrySet());
    return set.size();
  }
}
```

可能会认为这个程序将打印出 `2 2` 但它将打印出 `2 1`。

原因在于 `EnumMap` 的 `entrySet` 方法总返回同一个 `Entry` 类型的对象。所以，解决方案是将 entries 复制出来并手动插入：

```java
...
Set<Map.Entry<Sex, Sex>> set = new HashSet<Map.Entry<Sex, Sex>>();
for (Map.Entry<Sex, Sex> e : map.entrySet())
  set.add(new AbstractMap.SimpleImmutableEntry<Sex, Sex>(e));
return set.size();
```

这样处理后，结果就为 `2 2` 了。

**启示**

- 在 `entrySet` 上进行迭代遍历需要小心
  - `Iterator` 前进时，`Entry` 可能失效
  - JDK 6 的 `Map` 实现中，只有 `EnumMap` 和 `IdentityHashMap` 有这样的行为
     - Android 的 `Map` 实现中没有这样的行为
  - `new HashSet<EntryType>(map.entrySet())` 这一惯例将在面对这一行为时失效
- 对于 API 设计者来说
  - 不要违背**最小惊讶原则**
  - 除非别无选择，不要以恶化 API 为代价来提升性能
    - 这可能在当时是一个看起来好的选择，但你迟早会后悔的

# The Match Game

```java
import java.util.regex.*;
public class Match {
  public static void main(String[] args) {
    Pattern p = Pattern.compile("(aa|aab?)+");
    int count = 0;
    for(String s = ""; s.length() < 200; s += "a")
      if (p.matcher(s).matches())
        count++;
    System.out.println(count);
  }
}
```

可能会认为这个程序将打印出 `100` 但它并不会输出任何结果，因为 **catastrophic backtracking** 导致其计算时间超过 10<sup>15</sup> 年。

正则表达式匹配的本质是构建了一个 DFA，而对于这段代码而言，它将产生一个二叉树，对于任意的奇数个 `a` 构成的字符串，匹配过程将会遍历整个二叉树的叶子节点，而其复杂度为 $$O(2^{n/2})$$。这不是 Java 的问题，而是所有支持回溯的正则表达式匹配引擎共有的问题。这个问题没有什么解决方式，只能是在使用正则表达式的时候想清楚其代价。另外，由于正则表达式本身就很容易产生错误，所以尽量少用正则表达式。

# That Sinking Feeling

```java
abstract class Sink<T> {
  abstract void add(T... elements);
  void addUnlessnull(T... elements) {
    for (T element : elements)
      if (element != null)
        add(element)
  }
}

public class StringSink extends Sink<String> {
  private final List<String> list = new ArrayList<String>();
  void add(String... elements) {
    list.addAll(Arrays.asList(elements));
  }
  public String toString() { return list.toString(); }
  public static void main(String[] args) {
    Sink<String> ss = new StringSink();
    ss.addUnlessNull("null", null);
    System.out.println(ss);
  }
}
```

可能会认为这个程序将打印出 `[null]` 但它将抛出一个 `ClassCastException`。

产生的原因有几个，包括可变长参数列表、类型擦除和桥接方法。

当在 `abstract class Sink<T>` 中声明 `abstract void add(T... elements)` 和 `void addUnlessNull(T... elements) { ... }` 时，由于类型擦除以及可变长参数列表的实现，实际上声明的参数类型是 `Object[]`。在 `addUnlessNull` 方法内部，调用了 `add(element)` 方法，这个 `element` 在传入的时候构建的是一个长度为 `1` 的 `Object[]` 对象，并非 `T[]`。

而在 `StringSink` 内部，编译器会生成一个桥接方法：

```java
public class StringSink extends Sink<String> {
  ...
  void add(Object[] a) {
    add((String[]) a);
  }
  ...
}
```

这个 `void add(Object[] a)` 方法覆盖了抽象类中的那个抽象方法 `abstract void add(T... elements)`，并在 `add((String[]) a)` 处对 `a` 进行了强制类型转换。

当程序执行到 `ss.addUnlessNull("null", null)` 这一句时，编译器其实是构建了一个 `String[]` 类型的对象，然后在 `add((String[]) a)` 处因为强制类型转换而出现了 `ClassCastException` 这个异常。

解决方案是避免使用变长参数列表和数组，而应该使用集合。

```java
abstract class Sink<T> {
  abstract void add(Collection<T> elements);
  void addUnlessnull(Collection<T> elements) {
    for (T element : elements)
      if (element != null)
        add(Collections.singleton(element))
  }
}

public class StringSink extends Sink<String> {
  private final List<String> list = new ArrayList<String>();
  void add(Collection<String> elements) {
    list.addAll(elements);
  }
  public String toString() { return list.toString(); }
  public static void main(String[] args) {
    Sink<String> ss = new StringSink();
    ss.addUnlessNull(Arrays.asList("null", null));
    System.out.println(ss);
  }
}
```

此时程序将不会构建桥接方法，并将输出 `[null]`。

**启示**

- 可变长参数列表提供了一个有漏洞的抽象
- 泛型和数组无法很好地协同工作
  - 所以泛型和可变长参数列表无法很好地协同工作
- 尽量不要使用数组而应该使用集合类
  - 尤其是 API 设计的时候
- 不要忽略编译器的警告
  - 原先有漏洞的代码会产生编译器的警告
  - 理想状况下，尽可能通过改善代码来消除编译器警告，如果做不到的话：
    - 证明实际上不存在问题并将证明写在注释中
    - 局部使用 `@SuppressWarnings` 注解消除警告

# Glommer Pile

```java
public class Glommer<T> {
  String glom(Clooection<?> objs) {
    String result = "";
    for (Object o : objs)
      result += o;
    return result;
  }

  int glom(List<Integer> ints) {
    int result = 0;
    for (int i : ints)
      result += i;
    return result;
  }

  public static void main(String args[]) {
    List<String> strings = Arrays.asList("1", "2", "3");
    System.out.println(new Glommer().glom(strings));
  }
}
```

可能会认为这个程序将打印出 `123` 但它将抛出一个 `ClassCastException`。

原因在于，类声明时虽然声明了一个类型参数 `T`，但实际上并没有在类 `Glommer` 中使用到这个类型参数。在调用 `new Glommer()` 的时候，由于没有指定类型参数，所以实际上创建的是 `Glommer<T>` 的原始类型（也就是 `Glommer`）的对象，而泛型类的原始类型会忽略所有泛型类内部的类型参数信息，即相当于：

```java
public class Glommer {
  String glom(Clooection objs) { ... }

  int glom(List ints) { ... }

  public static void main(String args[]) {
    List strings = Arrays.asList("1", "2", "3");
    System.out.println(new Glommer().glom(strings));
  }
}
```

这导致了 `int glom(List ints)` 的重载版本成为了最为准确的重载版本，所以编译器选择了调用这个版本的 `glom`。

解决方法是在创建 `Glommer<T>` 类型的对象时指定一个类型参数，对于这个例子而言，由于类型参数 `T` 并未在 `Glommer<T>` 中被使用，所以随便指定一个就可以了，例如：

```java
System.out.println(new Glommer<Random>().glom(strings));
```

当然，由于此处根本没用到类型参数，一开始就不要去声明这个类型参数就是一个更好的选择：

```java
public class Glommer { ... }
```

这里还可以进行进一步的优化，由于此处两个方法连实例的域都没有用到，所以可以直接将其声明为静态的：

```java
public class Glommer {
  static String glom(Clooection<?> objs) { ... }

  static int glom(List<Integer> ints) { ... }

  public static void main(String args[]) {
    List<String> strings = Arrays.asList("1", "2", "3");
    System.out.println(glom(strings));
  }
}
```

**启示**

- 永远不要在新的代码中使用泛型类的原始类型
- 泛型类的原始类型将失去**全部**泛型的类型信息
  - 这可能导致重载到意想不到的方法上
- 不要忽视编译器警告，即便它们难以阅读
  - 错误版本的代码会产生一个编译警告
  - 未受检的警告意味着自动生成的转换可能会在运行时失败

<h1>Source Videos</h1>

[Advanced Topics in Programming Languages: Java Puzzlers,...](https://www.youtube.com/watch?v=wDN_EYUvUq0)

[Google I/O 2011: Java Puzzlers - Scraping the Bottom of the Barrel](https://www.youtube.com/watch?v=wbp-3BJWsU8)
