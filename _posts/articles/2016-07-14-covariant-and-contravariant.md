---
layout: post
title: 协变、逆变与不变
excerpt: "类型系统中的协变、逆变、不变分别是什么概念，以及 Java 对此的处理有何问题？"
categories: articles
author: zrl
date: 2016-07-14
modified: 2016-07-20
tags:
  - Object-oriented Programming
  - Type System
  - Scala
  - Java
comments: true
share: true
---

# 型变

型变（variance）是类型系统里的概念，包括协变（covariance）、逆变（contravariance）和不变（invariance）。这组术语的目的是描述泛型情况下类型参数的父子类关系如何影响参数化类型的父子类关系。也就是说，假设有一个接收一个类型参数的参数化类型 `T` 和两个类 `A`，`B`，且 `B` 是 `A` 的子类，那么 `T[A]` 与 `T[B]` 的关系是什么？如果 `T[B]` 是 `T[A]` 的子类，那么这种型变就是「协变」，因为参数化类型 `T` 的父子类关系与其类型参数的父子类关系是「同一个方向的」。如果 `T[A]` 是 `T[B]` 的子类，则这种关系是「逆变」，因为参数化类型 `T` 的父子类关系与类型参数的父子类关系是「相反方向的」。类似地，如果 `T[A]` 和 `T[B]` 之间不存在父子类关系，那么这种型变就是「不变」[^variance]。

[^variance]: [Covariance and contravariance (computer science) - Wikipedia](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science))

# 协变

在 Java 中，数组是协变的，也就是说，假设有一个基类 `Person` 和一个 `Person` 的子类 `Student`。因为 `Student` 类型是 `Person` 类型的子类，所以 `Student[]` 类型是 `Person[]` 类型的子类，这个设计似乎相当符合直觉，一个学生（`Student`）是一个人（`Person`），那一个存放着学生的数组当然也应该是一个存放着人的数组了。

然而这是错误的。

假设 `Person` 有另一个子类 `Teacher`，考虑如下代码：

```java
Student[] students = { new Student() }
students[0].study();
Person[] persons = students;
persons[0] = new Teacher();
students[0].study();  // Oops!
```

这段代码显然错了，看一下刚刚做了什么。我们在 `Student` 数组里存放了一个 `Student` 实例，紧接着调用了这个对象的 `study` 方法，这个显然没错；然后将这个数组赋值给一个 `Person` 数组，由于数组是协变的，所以这步没问题；然后，向 `Person` 数组里添加一个 `Teacher` 的实例，这步也没问题，因为一个 `Teacher` 是一个 `Person`；接下来是获取 `Student` 数组里的对象，调用 `Student` 类的 `study` 方法，这似乎也是合理的。那问题在哪呢？

事实上，这段代码可以编译通过，Java 并不会因此报编译错误，而是在运行 `persons[0] = new Teacher();` 时抛出一个 `java.lang.ArrayStoreException`。也就是说，给协变的数组的单元赋值的时候出错了。这个错误本来应该由编译器发现并指出，但 Java 将对这一错误的防止延后到了运行时期，错过了编译期的检查。编译器没有做正确的事情，这显然是一个设计错误，但这个错误是有其历史原因的 [^origin_scala]。

[^origin_scala]: [The Origins of Scala](https://www.artima.com/scalazine/articles/origins_of_scala.html)

在 Java 的早期版本中，工程师们因为时间紧迫而选择暂时不添加泛型在 Java 的语法中，这导致 Java 的数组没法使用泛型，在这种情况下，如果数组的型变是不变，那么要写一些通用的数组操作方法就变得困难，解决方案就是将数组设计为协变的，这样，就可以用操作 `Object[]` 的方法来操作所有引用类型的数组了。比如你可以写类似这样的方法来对数组里的对象进行排序：

```java
public static void sort(Object[] objs, Comparator cmp) { ... }
```

这显然是一个妥协，后来又因为兼容性的考量，不得不维持了这个设计，这是 Java 的一个原罪。而 Scala 做了正确的事，在 Scala 中，数组的声明和别的类没有什么不同：

```scala
final class Array[T] extends java.io.Serializable with java.lang.Cloneable
```

Scala 中的 Array 的实现就是 Java 的数组。在 Scala 中在类型参数前添加 `+` 代表参数化类型在该类型参数上协变，添加 `-` 则代表逆变，什么都不加就是不变。从 `Array` 的声明中可以看出，Scala 的 `Array` 是不变的，所以，以下代码是非法的：

```scala
val students: Array[Student] = Array(new Student)
// Compiler error:
// Expression of type Array[Student] doesn't conform to expected type Array[Person]
val persons: Array[Person] = students
```

那么，怎样集合类型才是协变的呢？考虑刚刚的数组的例子，将 `Student[]` 类型的实例赋值给 `Person[]` 类型的对象是没错的，当我们去修改 `Person[]` 对象的元素时，错误才产生。也就是说，不可变的集合才是协变的。

在 Scala 的 `scala.collection.immutable` 包中有许多不可变的集合，例如不可变的链表 `List`，它的声明大概如下（原声明很长，此处有所省略）：

```scala
abstract class List[+A] extends AbstractSeq[A] with LinearSeq[A]
```

这个声明表明 `List` 在其类型参数 `A` 上是协变的。也就是说，如下代码是合法的：

```scala
val students: List[Student] = List(new Student)
val persons: List[Person] = students
```

类似于 Scala 的不变语义，在 Java 中，如下的代码也是错误的：

```java
List<Student> students = new ArrayList<>();
students.add(new Student());
// Compiler error:
// Incompatible types,
//   Required: List <test.Person>
//   Found:    List <test.Student>
List<Person> persons = students;
```

这次 Java 的类型系统做了正确的事情，防止了出现刚刚数组那样的问题。那么在 Java 中又该如何表示协变这样的语义呢？

Java 并没有提供类型的协变声明，取而代之的是在使用时的类型限制，形式大概是这样：

```java
List<Student> students = new ArrayList<>();
students.add(new Student());
List<? extends Person> persons = students;
Person person =  persons.get(0);
// Compiler error:
// Wrong 2nd argument type. Found: 'test.Teacher', required: '? extends test.Person'
// set(int, capture<? extends test.Person>) in List cannot be applied
// to (int, test.Teacher)
persons.set(0, new Teacher());
```

这段代码无法编译通过，`List<? extends Person> persons` 是 Java 的协变声明，它大概表达了这样的语义： `List` 的类型参数是 `Person` 的「某个」子类，而具体是什么类型并不知道，既然不知道是什么类型，也就自然无法将其中的元素替换为其他值了。但由于已经知道了其元素类型是 `Person` 的「某个」子类，所以可以将其元素当作 `Person` 类型的对象取出。这就保证了协变集合的要求。也就是说，Java 选择不在参数化类型声明的时候去声明该类型的型变关系，而是选择在这个类型被使用的时候去进行限定。从语义上也可以看出，这个方式掩盖了协变本身的概念，是一个较为工程化的思路。但是，型变应该是一个类型本身的特性，Scala 的处理方式能在类型声明上更加清晰地表意，个人更偏向于 Scala 的处理方式。

# 逆变

相对于协变，逆变显得非常不符合直觉，它表明，如果 `B` 是 `A` 的子类，那么 `T[B]` 反而是 `T[A]` 的父类。很难想象什么地方会出现逆变的情况，而事实上，函数类型相对于其参数类型就是逆变的，Scala 中接受一个参数的函数类型声明如下：

```scala
trait Function1[-T1, +R] extends AnyRef
```

其中，`T1` 是其参数类型，`R` 是其返回值类型，可以看出，函数在其参数类型上是逆变的。这件事仔细想想就会明白这是合理的，考虑如下代码：

```scala
val student = new Student
val getStudentName: (Student => String) => String = (f) => f(student)
val personNameReader: Person => String = (p) => p.name
getStudentName(personNameReader)
```

在 Scala 中 `A => B` 表示一个接受一个 `A` 类型参数的对象，返回一个 `B` 类型的对象的函数类型。这段代码中 `getStudentName` 要求一个 `Student => String` 类型的函数作为参数，而 `personNameReader` 函数的类型是 `Person => String`。由于函数相对于其参数的类型是逆变的，所以可以将 `getStudentName` 应用于 `personNameReader` 上。从这个例子来说，`personNameReader` 要求一个 `Person` 类型的对象作为参数，而当 `getStudentName` 对其进行调用时，传入了一个类型更为「详细」的 `Student` 自然是合法的，由此就能理解为什么函数类型相对于其参数的类型是逆变的了。

在 Java 中，类似于协变，逆变也是在应用的时候才去对其进行约束，例如：

```java
List<Person> persons = new ArrayList<>();
List<? super Student> students = persons;
students.add(new Student());
// Compiler error:
// Incompatible types:
//   Required: test.Person
//   Found:    capture<? super test.Student>
Person p = students.get(0);
// Compiler error:
// Incompatible types:
//   Required: test.Student
//   Found:    capture<? super test.Student>
Student student = students.get(0);
```

也就是说，如果你进行了逆变的约束，那么 Java 将要求你只能向 `List` 里添加元素而不能将其取出来。语义与协变的情况是类似的。

于是，Scala 和 Java 的型变标记可以进行如下总结 [^prog-scala]：

[^prog-scala]: Dean Wampler, Alex Payne - Programming Scala 2nd

| Scala | Java | 解释 |
|:--------|:--------|:--------|
| +T   | ? extends T | 协变（即：X[T<sub>sub</sub>] 是 X[T] 的一个子类） |
| -T   | ? super T   | 逆变（即：Y[T<sup>sup</sup>] 是 Y[T] 的一个子类） |
| T    | T           | 不变（即：无法用 Z[T<sub>sub</sub>] 或 Z[T<sup>sup</sup>] 替代 Z[T]） |
{: .table}

# 规律

现在可以回头再看看之前的讨论，会发现其实只有一个规律，就是函数类型在其返回值的类型上协变，在其参数类型上逆变。

为什么数组是不变的？因为数组上的每个单元都相当于包含了两个方法，当写下 `T value = arr[3]` 这样的代码时，概念上可以理解为 `T value = arr3.get()`。而 `get` 方法的类型显然是 `() => T`。所以从单元中获取元素这个操作上来看，数组在其元素的类型上协变。而当写下 `arr[3] = value` 的时候，概念上则可以理解为 `arr3.set(value)`。而 `set` 方法的类型则为 `T => Unit`（`Unit` 相当于 Java 的 `void`）。所以从给数组单元赋值这个操作上看，数组又在其元素的类型上逆变。因此，数组在其元素类型上不变。

为什么可以写 `val person: Person = new Student` 呢？因为每个对象都可以看作是一个只带有一个方法的对象，相当于 `value.get()`。而 `get` 方法的类型是 `() => T`。所以我们可以写这样的代码，它是协变的。这么说感觉有点怪，但是，在 Scala 的语法糖加持下，这么说其实挺自然的，因为 Scala 允许在函数不需要参数的情况下省略括号，且如果调用的方法是 `apply` 的话，不需要写 `value.apply()` 直接写成 `value()` 即可。也就是 `def t() = new T` 和 `val t = new T` 相比，虽然前者每次都会创建一个新的实例，但是在使用者看来，都可以写为 `t`，并不会有区别。

在 Scala 中，如果进行了协变或者逆变的标记，编译器就会对这个类型参数的使用进行检查，如果它出现在了错误的位置上，编译器就会提示错误，防止了开发者因此而犯错。例如：

```scala
trait Test[+T] {
  def get(): T
  // Compiler error:
  // Covariant type T occurs in contravariant position in type T of value v
  def set(v: T): Unit
}
```

类型声明是很好的文档，更精确的类型声明就是更清晰的文档，Scala 的设计在这方面可以说是更胜一筹。

# 参考
