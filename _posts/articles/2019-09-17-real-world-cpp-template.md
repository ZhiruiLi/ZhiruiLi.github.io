---
layout: post
title: 实际工程中的 C++ 模板
excerpt: >-
  C++ 的模板是 C++ 的一个重要的语言特性，在这篇文章里，我将聊一下最近实际工程中的一些模板的应用，希望可以让更多人了解到模板并不是一个可怕的存在，以及一些常见的使用方式。
categories: articles
author: zrl
date: 2019-09-17
modified: 2019-09-17
tags:
  - C++
comments: true
share: true
---

C++ 的模板是 C++ 的一个重要的语言特性，我们使用的 STL 就是 Standard Template Library 的缩写，但是在很多情况下，开发者都对其敬而远之，有些团队甚至是直接在工程中禁用模板。模板常被当作洪水猛兽的一个原因是许多人提起模板就要提 C++ 模板图灵完备，甚至还要再秀一段编译期排序，这种表现模板强大的方式不仅不会让人觉得模板有用，反而让人觉得模板难以理解而且不应该使用。在这篇文章里，我将聊一下最近实际工程中的一些模板的应用，希望可以让更多人了解到模板并不是一个可怕的存在，以及一些常见的使用方式。

# 按版本号过滤配置

我所在的项目组前后台的复杂配置现在都用 protobuf 进行承载，然后生成 Excel 进行配置，生成 C++ 代码进行加载。例如这样的 message：

```protobuf
message ConfigItem1 {
  int32 id = 1;
  string text = 2;
}

message Config {
  repeated ConfigItem1 items1 = 1;
}
```

这里的 `Config` 会被映射为一个 Excel，里面有一个表 `items1`，其中，这个表有两列，一列 `id`，一列 `text`。这个表的每一行都是一个具体的配置项。也就是我们可以这样获取配置：

```c++
cout << cfg.items1(0).id() << ": " << cfg.items1(0).text();
```

现在有个需求是这样的，在加载某些配置的时候，前台需要根据版本号进行配置的过滤，部分配置项只会在某些版本中可见，例如这样：

```protobuf
message VersionRange {
  int32 lo = 1;
  int32 hi = 2;
}

message ConfigItem2 {
  repeated VersionRange version_ranges = 1;
  int32 id = 2;
  int32 value = 3;
}

message Config {
  repeated ConfigItem1 items1 = 1;
  repeated ConfigItem2 items2 = 2;
}
```

加载的时候大概有这样的代码：

```c++
// 加载配置时进行过滤
for (auto iter = cfg.items2().begin(); iter != cfg.items2().end();) {
  if (!IsAvailableVersion(*iter, ver)) {
    iter = cfg.mutable_items2()->erase(iter);
  } else {
    iter++;
  } 
}
```

这个 `IsAvailableVersion` 要怎么实现呢？我们当然可以对每个配置项类型都实现一个函数重载，但是我们也可以使用函数模板来生成这些代码，非常简单：

```c++
template<class CfgItem>
bool IsAvailableVersion(CfgItem const &item, int ver) {
  auto const &ranges = item.version_ranges();
  if (std::begin(ranges) == std::end(ranges)) {
    return true; // 如果 version range 列表为空，默认返回 true
  }
  for (auto const& range : ranges) {
    if (ver >= range.lo() && ver <= range.hi()) {
      return true; // 如果当前版本在范围内，就返回 true
    }
  }
  return false; // 如果 version range 列表不为空，默认返回 false
}
```

但这里有个问题，不是每一个配置项的类型里都有 `version_range` 字段，例如 `ConfigItem1` 就没有。这就导致了 `IsAvailableVersion` 不能对所有的配置项对象进行使用，这不利于我们统一 code gen 上面加载配置时进行过滤的代码。

这时候，我们想要做的，是对 `IsAvailableVersion` 的类型参数进行限制，根据这个类型是否带有 `version_range` 字段来决定是否进行过滤：

```c++
template<class CfgWithVerRange>
bool IsAvailableVersion(CfgWithVerRange const &item, int ver) { /* 实现同上 */ }

template<class CfgWithoutVerRange>
bool IsAvailableVersion(CfgWithoutVerRange const &item, int ver) { return true; }
```

可惜编译器没法通过类型参数的名字明白我们的意图，因此我们需要用一些技巧达到我们的目的：

```c++
template<class CfgItem, class = void>
struct IsAvailableVersionHelper {
  static bool Check(CfgItem const&, int) { return true; }
};

template<class CfgItem>
struct IsAvailableVersionHelper<
  CfgItem,
  lib::void_t<
    decltype(std::begin(std::declval<CfgItem>().version_ranges())->lo()),
    decltype(std::begin(std::declval<CfgItem>().version_ranges())->hi())
  >
> { /* 实现同上 */ };

template<class CfgItem>
bool IsAvailableVersion(CfgItem const &item, int ver) {
  return IsAvailableVersionHelper<CfgItem>::Check(item, ver);
}
```

这时候我们就可以放心地写 `IsAvailableVersion(*iter, ver)` 了，如果传入的是 `ConfigItem1`，则使用的是上面原始的实现，而 `ConfigItem2` 则使用的是下面特化的实现。

这是如何做到的呢？我们知道，C++ 的模板有个规则是 SFINAE，这不是一个单词，而是 Substitution Failure Is Not An Error 的缩写，也就是说，编译器在基于模板生成代码时，如果将模板的类型参数置换为给定的类型时，如果失败，编译器不会报错，而是将这个结果从可选的集合里丢弃，并从剩下的中进行选择。

当我们将 `ConfigItem1` 放入时，上面的版本能够正确替换，而下面的版本则因为 `ConfigItem1` 没有 `version_range` 字段而失败，此时，编译器会将这个失败的版本抛弃，由于只剩下原始版本了，因此选择了原始版本。

这里的 `lib::void_t` 是什么？`std::void_t` 是 C++ 17 之后才在 STL 中提供的模板，它很简单也非常有用，功能是将任意的类型序列映射到 `void` 上，也就是忽略掉这些类型。由于我们在使用 C++ 11，因此需要自己实现一下：

```c++
// C++11 中这样简单实现可能会有 bug，参考 en.cppreference.com/w/cpp/types/void_t
// template<class...>
// using void_t = void;
template<class... Ts> struct make_void { using type = void; };
template<class... Ts> using void_t = typename make_void<Ts...>::type;
```

这里使用 `void_t` 将多个类型声明忽略掉以适应 `template<class CfgItem, class = void>` 中的第二个类型参数：

```c++
decltype(std::begin(std::declval<CfgItem>().version_ranges())->lo()),
decltype(std::begin(std::declval<CfgItem>().version_ranges())->hi())
```

虽然说这两个类型声明被忽略了，但是它们还是会参与替换，`decltype` 可以根据括号里的表达式计算出其类型，而 `std::declval<T>()` 则相反，给定一个类型，它可以获得该类型的值，虽然这个值并不是有效的，但是在这个类型声明里我们可以用它来填写表达式。如果一个类型没有带 `version_ranges` 字段，则 `std::declval<CfgItem>().version_ranges()` 会失败，如果这个 `version_range()` 返回的对象不支持 `std::begin`，则 `std::begin(...)` 会失败，若这个 `std::begin` 计算出来的迭代器不支持 `lo` 函数，则 `std::begin(...)->lo()` 会失败，这里的结合就确保了 `CfgItem` 类型必须有 `version_range`，且每一个 `version_range` 都是可迭代的，且每一个 range 都有 `lo` 成员。下面的 `hi` 也是类似的。当然，我们还可以通过 `std::is_same` 之类的 type trait 进一步确保 `lo` 和 `hi` 返回的类型，这个就不在此演示了，对于我们的需求而言，这样就足够了。

那说回来，如果我们填入的是 `ConfigItem2` 会怎样？在这个时候，两个类型替换都会成功，但由于原始版本中，第二个类型参数是默认值 `void`，而特化版本中，则填入了自定义的一个类型 `lib::void_t...`，虽然这个类型最后计算出来的类型还是 `void`，但它依然是比原始版本更「特殊」的版本，因此编译器会选择这个版本，这就达到了我们的目的。

# Data blob 操作辅助类

在公司中，我们有自己的 NoSQL 数据库服务，我们在使用的过程中常常有这样的模式：

```c++
MyDataBlob data{};
data.key1 = ...;
data.key2 = ...;
DbApi api(...);
int const res = api.Get("tablename-x", &data, sizeof(data));
if (res == RSP_ERROR && api.GetDbErr() == NOT_EXIST) {
  LOGDBG(...); // 数据不存在，打印调试日志
} else if (res != 0) {
  LOGERR(...); // 其他错误，打印错误日志，返回错误
  return ERR_DB_GET_FAIL;
} 
// 正常逻辑，使用 data ...
```

这里先创建一个空白的数据对象，填入它的 key 值，然后调用 API 拉取数据。由于 DB 会将拉取不存在的数据这种情况也认为是一个错误，而数据不存在对于业务而言又往往不是一个错误，因此我们一般是要对这种情况单独进行处理。

这种重复的工作显然可以抽象一个更加方便的 API 类型出来，希望能更轻松地进行使用。一个简单的想法是这样的：

```c++
template<class Db>
struct Result {
  int code{};
  int subCode{};
  Db data{};
  
  bool IsError() { return code != 0 && subCode != NOT_EXIST; }
  bool NoExist() { return code != 0 && subCode == NOT_EXIST; }
}

template<class Db>
struct NewDbApi {
  Result Get() {
    DbApi api(...);
    Result res{};
    res.data.SetKey(???); // 1
    res.code = api.Get(res.data.TableName(), &res.data, sizeof(res.data)); // 2
    res.subCode = api.GetDbErr();
    if (res.IsError()) {
      LOGERR(...);
    }
    return res;
  }
}
```

这里我们碰到了一点麻烦的问题，首先，在 1 处，这个 `data.SetKey()` 我们不知道应该怎么填。当然，我们可以像原先一样在外部自行设置 key，然后再将 `data` 传进来，但是我们更加希望能够免去这一个步骤，直接通过 `Get` 函数的参数传入对应的 key，然后转交给 `data`。但我们又不知道这个 `Db` 类型的 key 是什么，那我们该怎么办呢？也许我们可以这样做：

```c++
template<class ... Args>
Result Get(Args &&...args) {
  ...
  res.data.SetKey(std::forward<Args>(args)...); // 1
  ...
}
```

呃……这确实可以实现我们要的效果，但是这个实现方法并不好，它带来了不必要的复杂度。最让人难受的一点是，我们丢失了 `data.SetKey` 所需参数的类型信息，这让调用者完全不知道这里应该填什么数据。为了解决这个问题，我们可以添加一层抽象，让 `Db` 类型告诉我们 key 的类型是什么：

```c++
Result Get(typename Db::key_type const &key) {
  res.data.SetKey(key); // 1
  ...
}
```

这样简单多了，`Get` 函数的调用者可以获知对应的 key 的类型。

另外一个问题是，1 和 2 处我们直接调用了 `data` 的 `SetKey` 和 `TableName` 成员函数，但是我们的 `MyDataBlob` 是一个用另外一个工具基于 XML 描述生成出来的代码，主要实现的是序列化和反序列化功能，我们没法去通过修改这个工具来添加新的接口。所以我们只能使用 adapter 模式解决这个问题：

```c++
struct MyDataBlobAdapter {
  using key_type = ...;
  void SetKey(key_type const &) { ... }
  std::string TableName() const { ... }
  MyDataBlob myDataBlob{};
}
```

这就可以解决上面提到的问题了，给 `Get` 函数的实现提供了 `SetKey` 和 `TableName`。我们可以发现，`Result` 里的数据类型不再是 `MyDataBlob` 了，而是 `MyDataBlobAdapter`，使用者拿到了这个对象后，使用的方法不再是 `res.data` 而是 `res.data.myDataBlob` 了。这在大多数情况下不是什么大问题，但是，如果一个数据不只是被这一个接口操作，而是被多个接口操作那该怎么办呢？这时候，为了适配多个接口，我们可能需要多个 adapter。例如：

```c++
NewDbApi<MyDataBlobAdapter> api1{};
OtherAPI<OtherAdapter> api2{};

Result res = api1.Get(...);
OtherAdapter other(res.data.myDataBlob);
api2.Modify(&other);
res.data.myDataBlob = other.myDataBlob;
api1.Put(res.data);
```

这不仅麻烦，而且会造成比较大的开销，在这里，为了适配两个接口，我们不得不进行两次数据的复制。我们能否做得更好呢？

首先注意到 `TableName` 这个函数其实和对象无关，我们可以实现为一个静态的函数：

```c++
struct MyDataBlobAdapter {
  static std::string TableName() const { ... }
  ...
}
```

所以上面 2 处的代码可以改为：

```c++
res.code = api.Get(Db::TableName(), ...); // 2
```

类似地，对于 `SetKey`，我们也可以进行类似的改造，虽然它需要操作自己的成员变量，但是，我们可以将 `this` 指针手动传递一下，也就是这样：

```c++
struct MyDataBlobAdapter {
  static void SetKey(key_type const &key, MyDataBlobAdapter *adapter) { ... }
  ...
}
```

对于使用者，1 处的代码可以改为：

```c++
Db::SetKey(key, &res.data);
```

这个时候，我们可以发现，这里 `SetKey` 的第二个参数根本不需要是 `MyDataBlobAdapter*`，我们可以直接将其换为 `MyDataBlob*`！同时，类似于 `key_type`，为了能告诉使用者这个数据的类型，我们加一个 `type` 类型声明，结果是这样：

```c++
struct MyDataBlobAdapter {
  using type = MyDataBlob;
  static void SetKey(key_type const &key, type *blob) { ... }
  ...
}
```

这时我们重新回来看一下 `NewDbApi` 的实现：

```c++
template<class Db>
struct Result {
  typename Db::type data{};
  ...
}

template<class Db>
struct NewDbApi {
  Result Get(typename Db::key_type const &k, typename Db::type *db) {
    ...
    Db::SetKey(k, db); // 1
    res.code = api.Get(Db::TableName(), &res.data, sizeof(res.data)); // 2
    ...
  }
}
```

使用的时候，只需要这样写：

```c++
NewDbApi<MyDataBlobAdapter> api1{};
OtherAPI<OtherAdapter> api2{};

Result res = api1.Get(...);
api2.Modify(&res.data);
api1.Put(res.data);
```

这样一来，我们就实现了既能直接操作数据，又避免给原始类型添加新接口，而且还做到了足够的泛化灵活，甚至不需要用到虚函数导致性能损失。

不过，这种形式的实现有个小缺点，这里的 `Db` 类型的约束非常不明确，对于使用者而言，可能会碰到非常难读的编译错误，这可能是许多人害怕模板的另一个原因。到 C++ 20，我们才能用上 Concept，能够直接指名模板参数的约束，但现实情况是，我们可能将长期被锁在 C++ 11 里，在这种情况下，我们也可以尽力去给使用者清晰的提示：

```c++
// 示例:
// struct LegalDb {
//   struct type;
//   struct key_type;
//   static void SetKey(key_type const &key, type *db);
//   static std::string TableName();
// }
template<class Db>
struct NewDbApi {
  ...
  static_assert(IsLegalDb<Db>::value,
                "Db must match requirements of LegalDb, see comments above");
}
```

这样一来，一旦使用者填入了不合法的类型，编译期立刻就能收到上面的提示，并且可以基于示例来了解应该如何实现这个类型。这个 `IsLegalDb` 的实现也用到了 SFINAE，大致可以实现为这样：

```c++
template<class T, class = void>
struct IsLegalDb: std::false_type {}; // 3

template<class T>
struct IsLegalDb<T,
  lib::void_t<
    typename T::type,
    typename T::key_type,
    decltype(T::SetKey(std::declval<typename T::key_type const &>(),
                       std::declval<typename T::type *>())),
    typename std::enable_if<
      std::is_convertible<decltype(T::TableName()), std::string>::value>
    >::type // 4
>: std::true_type {}; // 5
```

这里也用到了前面实现的 `void_t`，总体思路是类似的，也是基于类型声明来让编译器选择我们想要的模板实现，这里可能和上一个例子不太一样的有两点。第一是我们这里的类型在 3 和 5 处继承了 `std::true_type` 和 `std::false_type`，这两个类型可以认为是类型级别的 `true` 和 `false`，在头文件 `<type_traits>` 里有很多 `is_` 开头的模板就是基于这两个类的，如果一个类型符合它的约束，它就是 `true_type` 否则就是 `false_type`。这里用到的 `std::is_convertible` 就是这样的 type trait，它判定的是第一个类型参数能被转换为第二个类型参数。我们可以用 `value` 成员的来获得它们对应的 `bool` 值。这里用到了另一个基础工具是 `std::enable_if`，它可以接受一个编译期计算出来的 `bool` 值，如果这个值为 `true`，那么我们就能获得其 `type` 成员类型，否则就获取不到，可能直接用一个简单实现来说明更加方便：

```c++
template<bool B, class T = void>
struct enable_if {};
 
template<class T>
struct enable_if<true, T> { using type = T; };
```

所以说，4 处的代码的实现了如果 `std::is_convertible` 判定为 `true`，那么 `std::enable_if` 里就会有 `type`，那么模板的类型置换就会成功，否则则是失败，这就实现了我们想要的判定 `T::TableName()` 返回类型可以转换为 `std::string` 的效果。

`IsLegalDb` 的实现相对而言可能会有点麻烦，但是它可以带来清晰的错误提示，是一个很好的文档，因此对于一个有特定约束的模板类型参数，尤其是无法从名字上直接看出来约束内容的模板类型参数，最好配套加上这样一个检查，配合注释说明，给使用者明确的约束，以方便使用者实现合法的类型。

# 强类型别名

我们经常会碰到一个函数带有几个类型相同的参数的情况。以扑克牌举例，一种表示方式是基于花色和数字的表示，使用一个 `uint8_t` 表示花色，同时一个 `uint8_t` 表示数字，另一种是直接基于牌编码的方式，也就是将牌从 0 编号到 54，只需要一个 `uint8_t` 就能实现。那么，如果不同地方使用到了不同的表示方式，就需要有类似这样的转换函数：

```c++
uint8_t ConvertCardToCode(uint8_t shape, uint8_t number);
```

这个函数本身是没什么问题，但在使用的时候经常一不小心就写歪了：

```c++
auto const num = uint8_t(13);
auto const shp = uint8_t(2);
auto const code = ConvertCardToCode(num, shp); // num 和 shp 的位置写反了
```

我们可以通过类型别名声明来使得函数类型更加明晰：

```c++
using CardCode = uint8_t;
using Shape = uint8_t;
using Number = uint8_t;

CardCode ConvertCardToCode(Shape shape, Number number);
```

这个写法看起来很不错，但是在函数调用处，我们仍然无法避免出现这种情况：

```c++
auto const num = Number(13);
auto const shp = Shape(2);
auto const code = ConvertCardToCode(num, shp); // 仍然能正常编译
```

虽然我们声明了类型别名，但是这个类型别名的本质上还是原来的类型，我们仍然无法避免出现前面的错误。在 Go 语言中，「type alias」（`type T = xxx`）和「type definition」（`type T xxx`）是两种不同的语法，如果我们使用前者，则依然会遇到上面说的这个问题，但如果我们使用后者，则可以让编译器帮我们避免它：

```go
type CardCode uint8;
type Shape uint8;
type Number uint8;

func ConvertCardToCode(shape Shape, number Number) CardCode { /*...*/ }
```

```go
num := Number(13)
shp := Shape(2)
code := ConvertCardToCode(num, shp); // 编译失败
```

这样的强类型别名非常好，使得函数签名本来就成为了注释的一部分，想要在 C++ 中实现类似的效果，我们可以不是用 `using` 起别名而是直接将类型包裹一层：

```c++
struct Shape {
  Shape() = default;
  explicit Shape(uint8_t val): v{val} {}
  uint8_t v;
};

struct Number {
  Number() = default;
  explicit Number(uint8_t val): v{val} {}
  uint8_t v{};
};

using CardCode = uint8_t;

CardCode ConvertCardToCode(Shape shape, Number number);
```

此时，函数调用者如果传错了参数，就完全没法编译通过了：

```c++
auto const num = Number(13);
auto const shp = Shape(2);
auto const code = ConvertCardToCode(num, shp); // 编译出错
```

可以发现这两个类型是很类似的，我们会考虑用模板来使得这个过程更加便利：

```c++
template<class T>
struct StrongAlias {
  StrongAlias() = default;
  explicit StrongAlias(T val): v{std::move(val)} {}
  T v{};
};

using Shape = StrongAlias<uint8_t>;
using Number = StrongAlias<uint8_t>;
using CardCode = uint8_t;
CardCode ConvertCardToCode(Shape shape, Number number);
```

但很可惜的是，这样并不能达到我们想要的效果，因为 `StrongAlias<uint8_t>` 和 `StrongAlias<uint8_t>` 是同一个类型，所以使用 `using` 来声明的 `Shape` 和 `Number` 也依然是同一个类型。因此我们需要用另一个标记将两个类型完全区分开来，我们可以在类型参数列表里加多一个类型参数来做到这一点，这个类型参数的唯一作用就是用来实现类型的区分：

```c++
template<class T, class Tag>
struct StrongAlias {
  StrongAlias() = default;
  explicit StrongAlias(T val): v{std::move(val)} {}
  T v{};
};

using Shape = StrongAlias<uint8_t, struct ShapeTag>;
using Number = StrongAlias<uint8_t, struct NumberTag>;
using CardCode = uint8_t;
CardCode ConvertCardToCode(Shape shape, Number number);
```

这个实现已经很实用了，但是我们可以让它更好用一点，目前而言它的不足之处在于，我们包裹的类型往往是一些基础类型，这些基础类型自带了一些操作符，比如我们之前想比较两张牌是否相等的时候可以写：

```c++
if (card1.shape == card2.shape && card1.number == card2.number) { ... }
```

但现在需要写：

```c++
if (card1.shape.v == card2.shape.v && card1.number.v == card2.number.v) { ... }
```

更为麻烦的是，如果我们想将类型别名作为 `std::map` 的 key 时就会直接报错：

```c++
// using Number = uint8_t;
std::map<Number, int> cardNumCount{}; // 编译通过
```

```c++
// using Number = StrongAlias<uint8_t, struct NumberTag>;
std::map<Number, int> cardNumCount{}; // 编译出错
```

这是因为 `std::map` 要求 key 能够使用 `<` 进行比较，而当我们直接使用 `using` 起类型别名时，这个 `<` 就是 `uint8_t` 的 `<`，而 `StrongAlias<uint8_t, struct NumberTag>` 则没有这个运算符。我们当然可以对每一个类型别名都自己实现一次所需的运算符，但我们还可以做得更加简单：

```c++
template<class T, class Tag, template<class> class Op>
struct StrongAliasType: public Op<StrongAliasType<T, Tag, Op>> {
    StrongAliasType() = default;
    explicit StrongAliasType(T const &value): v(value) {}
    T v{};
};

template<class T>
struct Lt { 
  bool operator<(T const &other) const { 
    return static_cast<T const &>(*this).v < other.v; 
  } 
};
```

这里用到了一个 C++ 里的一个惯用法——奇异递归模板模式，这个模式里派生类被作为基类的模板参数，这个声明看着有点吓人，但是它实现的效果是很妙的：

```c++
using Number = StrongAlias<uint8_t, struct NumberTag, Lt>;
```

可以看到 `StrongAlias<uint8_t, struct NumberTag, Lt>` 本身继承了 `Lt<StrongAlias<uint8_t, struct NumberTag, Lt>>`，这意味着 `Number` 就继承了 `Lt` 中的 `<` 运算符，而 `Lt` 的 `<` 实现中，使用了 `T::v` 的 `<` 运算符进行比较，因此 `Number` 就可以使用 `uint8_t` 的 `<` 运算符了。

当然，有时候我们可能不止需要这一个运算符，所以 `Op` 可能不止一个，要想要支持更多运算符，这里可以使用模板参数包来实现，使用 `...` 来标识一个参数包，然后再用 `...` 展开：

```c++
template<class T, class Tag, template<class> class... Ops>
struct StrongAliasType: public Ops<StrongAliasType<T, Tag, Ops...>>... {
    StrongAliasType() = default;
    explicit StrongAliasType(T const &value): v(value) {}
    T v{};
};
```

这里的 `StrongAliasType` 继承了类型参数中的每一个 `Ops`。然后，类似上面 `Lt` 的实现，我们可以实现一组这样的运算符模板：

```c++
template<class T>
struct Eq { bool operator==(T const &other) const { /*...*/ } };

template<class T>
struct Ne { bool operator!=(T const &other) const { /*...*/ } };

template<class T>
struct Lt { bool operator<(T const &other) const { /*...*/ } };

template<class T>
struct Le { bool operator<=(T const &other) const { /*...*/ } };

template<class T>
struct Gt { bool operator>(T const &other) const { /*...*/ } };

template<class T>
struct Ge { bool operator>=(T const &other) const { /*...*/ } };
```

有了这些运算符模板，使用者就可以按需选择自己需要的来进行使用了，例如：

```c++
using Number = StrongAlias<uint8_t, struct NumberTag, Eq, Ne, Lt, Le, Gt, Ge>;
```

这样，我们就拥有了更加好用的强类型别名了。

# 小结

在这篇文章里，我们看到了在实际工程中 C++ 模板的一些应用。很显然，这些功能脱离了模板的能力是非常难以实现的。对于 C++ 开发者而言，不应该盲目地拒绝模板，而是应该将它应用在正确的地方，以获得更好的性能和更清晰可靠的代码。
