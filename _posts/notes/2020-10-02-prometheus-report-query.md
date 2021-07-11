---
layout: post
title: "Prometheus 上报和查询"
excerpt: "Prometheus 是一个功能极为强大的监控告警系统，为了实现其灵活性，Prometheus 难免增加了复杂度，造成了一定的理解成本，本文记录了一下学习内容。"
date: 2020-10-02
modified: 2020-10-02
categories: notes
author: zrl
tags:
  - Prometheus
comments: false
share: true
---

# 基本概念

## 采样样本

Prometheus 会定期去对数据进行采集，每一次采集的结果都是一次采样的样本（sample），这些数据会被存储为时间序列，也就是带有时间戳的 value stream，这些 value stream 归属于自己的监控指标。

这里采集样本包括了三部分：

- 监控指标（metric）
- 毫秒时间戳（timestamp）
- 样本的值（value）

## 监控指标

一个监控指标被表示为下面的格式：

```
metric_name { label_name_1=label_value_1, label_name_2=label_value_2, ... }
```

这里的 `metric_name` 用于指明监控的内容，`label_value_x` 则用于声明这个监控内容中不同维度的值。用我们常见的二维坐标系举例，下面有一个二维坐标系，名称为「xxx 坐标系」，其中，有 `X`，`Y` 两个轴，上面有两个点，分别是 `A` 和 `B`，它们的坐标分别为 `(1, 3)` 和 `(2, 1)`：

```
xxx坐标系

Y
  ^
  │   . A (1, 3)
  │
  │     . B (2, 1)
  v
    <-----------------> X
```

对应于 Prometheus，这里的 `metric_name` 就是 「xxx 坐标系」，`label_name_1` 就是 `X`，`label_name_2` 就是 `Y`。需要注意的是，这里的 `A` 和 `B` 两个点并不代表采样点，而是监控指标。我们可以想象在这个图中还存在一条虚拟的时间轴，分别从 `A` `B` 两点从屏幕外垂直屏幕进去，在这两条虚拟的时间轴上，每一个点就是一个采样点，采样点上会带一个毫秒时间戳和一个值，这个值就是样本的值。在 Prometheus 中，样本的值必须为 `float64` 类型的值。

对于 Prometheus 而言，这里存在两个时间序列，分别为：

```
xxx坐标系{"X"="1","Y"="3"}
xxx坐标系{"X"="2","Y"="1"}
```

说到这里，我们不难意识到，当我们上报数据的时候，这里的标签值不可以用一个数量非常多的值（例如用户 ID），否则会造成时间序列数量的极度膨胀。

# 数据上报

在 Prometheus 内部，所有的采样样本都是以时间序列的形式保存在时序数据库中，但为了方便理解和使用，Prometheus 定义了 4 种数据上报的类型，用户可以根据上报的数据内容选择合适的接口。下面以 [Go 的接口](https://godoc.org/github.com/prometheus/client_golang/prometheus)为例说明这几种类型的区别和应用场景。

## 计数器 Counter

和一般理解的计数器一样，Prometheus 的 counter 也是一个只增不减的值，Go 语言中的接口如下：

```go
type Counter interface {
    Metric
    Collector

    // Inc increments the counter by 1. Use Add to increment it by arbitrary
    // non-negative values.
    Inc()
    // Add adds the given value to the counter. It panics if the value is <
    // 0.
    Add(float64)
}
```

用户可以调用 `Inc` 接口进行上报数据 +1，也可以调用 `Add` 接口增加任意的值（必须为非负数）。

如前所述，Prometheus 将数据拆分为不同监控指标名和不同的维度，我们上报的值具体属于哪个监控指标要如何指定呢？下面是官方的 example：

```go
httpReqs := prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "How many HTTP requests processed, partitioned by status code and HTTP method.",
    },
    []string{"code", "method"},
)
prometheus.MustRegister(httpReqs)

httpReqs.WithLabelValues("404", "POST").Add(42)
```

这里，我们指定的 `metric_name` 是 `http_requests_total`（[官方建议](https://prometheus.io/docs/practices/naming/) counter 的 metric name 用 `_total` 结尾），分成两个维度，`code` 和 `method`，我们在 `(404, POST)` 这个维度上上报了一个 `42`。

Counter 非常常见，也非常容易理解，常被用来监控类似「请求量」、「失败量」、「错误码出现次数」等场景。由于 counter 只增不减，所以我们不能用它来监控可能增可能减的数值（例如 goroutine 的数量），如果要监控这种数量，就应该用下面的 gauge。

## 测量仪表 Gauge

Gauge 的监控值可增可减，在 Go 语言中的接口如下：

```go
type Gauge interface {
    Metric
    Collector

    // Set sets the Gauge to an arbitrary value.
    Set(float64)
    // Inc increments the Gauge by 1. Use Add to increment it by arbitrary
    // values.
    Inc()
    // Dec decrements the Gauge by 1. Use Sub to decrement it by arbitrary
    // values.
    Dec()
    // Add adds the given value to the Gauge. (The value can be negative,
    // resulting in a decrease of the Gauge.)
    Add(float64)
    // Sub subtracts the given value from the Gauge. (The value can be
    // negative, resulting in an increase of the Gauge.)
    Sub(float64)

    // SetToCurrentTime sets the Gauge to the current Unix time in seconds.
    SetToCurrentTime()
}
```

可以看到，相比起 `Counter` 这里的 `Gauge` 增加了 `Dec` 和 `Sub` 这样的减少数值的接口，同时提供了 `Set` 和 `SetToCurrentTime` 这样的直接设置数值的接口。相比起 counter 而言，gauge 的数值要更加灵活通用。我们可以用它来监控前面提到的「goroutine 的数量」或者是其他可增可减的值，例如「CPU 使用率」、「内存使用率」等。

## 直方图 Histogram

尽管我们能够通过 gauge 监控可增可减的值，并可以在查询时求出其一段时间内的平均值，但是对于一些场景而言，这个能力还是存在相当大的局限性。典型的场景是请求时延、响应数据量大小等，在这些场景中，平均值可能并不能很好地反映问题。

举例而言，我们现在需要统计请求时延的长度，如果大部分时延都只有 100 毫秒，而少量有几秒，那么单纯的平均时延就会让我们难以确定实际的时延情况，我们并不是想知道平均时延是几百毫秒，而是想知道多大比例的请求时延是 100 毫秒，多大比例的是几百毫秒，多大比例的是超过一秒。

Histogram 可以帮我们解决这个问题，它并不是记录一个值的变化情况，而是将被观测到的值划分进某一个区间中，这里称为桶（bucket）。下面是 Go 版本的接口，可以看到 `Histogram` 只有一个 `Observe` 方法：

```go
type Histogram interface {
    Metric
    Collector

    // Observe adds a single observation to the histogram.
    Observe(float64)
}
```

与前面提到的两个上报模式不同，在 counter 中，一个 counter 对应了一个时间序列，我们创建一个 counter 然后用这个 counter 上报数据，它影响的时间序列是确定的，gauge 也是类似。而 Histogram 则会帮我们创建多个时间序列，当我们调用 `Observe` 的时候，被观测到的值会被放进预先划分好的桶中，每一个桶中并不记录被观测的值，而是对其进行计数。例如前面提到的情况，我们可能会给 0 ～ 100ms 划分一个桶，100ms ～ 1s 划分一个桶，1s 以上划分一个桶，那么，当我们上报一个值的时候，这三个桶中符合条件的桶的计数值就会增加。不过我们最后看到的并不是每一个桶的具体值，而是每个桶和前面所有桶的总和值，这个之后会再提到。这里先来看看如何声明桶的参数。桶的划分是在我们创建 `Histogram` 时指定的，官方例子如下：

```go
temps := prometheus.NewHistogram(prometheus.HistogramOpts{
    Name:    "pond_temperature_celsius",
    Help:    "The temperature of the frog pond.", // Sorry, we can't measure how badly it smells.
    Buckets: prometheus.LinearBuckets(20, 5, 5),  // 5 buckets, each 5 centigrade wide.
})

// Simulate some observations.
for i := 0; i < 1000; i++ {
    temps.Observe(30 + math.Floor(120*math.Sin(float64(i)*0.1))/10)
}

// Just for demonstration, let's check the state of the histogram by
// (ab)using its Write method (which is usually only used by Prometheus
// internally).
metric := &dto.Metric{}
temps.Write(metric)
fmt.Println(proto.MarshalTextString(metric))
```

通过这样的方法，我们可以描绘出被观测值的分布情况。通过这个分布数据，我们还能算出类似「99% 的请求时延在多少以内」这样的数据。不过，当我们需要这样的数据时，需要对被观测数据有一定的先验知识才能真正使得计算结果比较准确。还是用刚刚提到的时延观测来举个极端一点的例子，假设我们将桶划分为 1ms、10ms、10s，那么我们得出的结果可能是 0% 的请求时延在 1ms 以内，0% 的请求时延在 10ms 以内，100% 的请求在 10s 以内，0% 的请求在 10s 以上，这样的结论显然没有什么意义，只有当我们对时延的长度本身有一个基本概念，并正确划分桶的大小时，我们才能更准确地计算出我们想要的结果。那假设我们就是对一个数据没有什么先验知识，那我们是否有更准确的方式计算出这个数据呢？Prometheus 给出的方法就是用 Summary。

## 概要 Summary

Summary 和 histogram 类似，也是用来观测数据的分布情况的，它的接口也和 histogram 一样：

```go
type Summary interface {
    Metric
    Collector

    // Observe adds a single observation to the summary.
    Observe(float64)
}
```

但我们在创建一个 `Summary` 时，并不是像创建 `Histogram` 那样划分桶，而是直接划分我们所要计算的分位数区间，例如：

```go
temps := prometheus.NewSummary(prometheus.SummaryOpts{
    Name:       "pond_temperature_celsius",
    Help:       "The temperature of the frog pond.",
    Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
})

// Simulate some observations.
for i := 0; i < 1000; i++ {
    temps.Observe(30 + math.Floor(120*math.Sin(float64(i)*0.1))/10)
}

// Just for demonstration, let's check the state of the summary by
// (ab)using its Write method (which is usually only used by Prometheus
// internally).
metric := &dto.Metric{}
temps.Write(metric)
fmt.Println(proto.MarshalTextString(metric))
```

通过上面的 summary 上报，我们能够更加准确地获知 50% 的观测值，90% 的观测值以及 99% 的观测值，避免了前面 histogram 的问题。但 summary 的数据计算是由客户端进行的，会造成一定的性能损耗。

更多 histogram 和 summary 的对比可以参考[这一篇文章](https://prometheus.io/docs/practices/histograms/)。

# 数据查询

Prometheus 定义了一个名为 PromQL 的 DSL 用来进行数据查询。常用的 Prometheus 数据可视化工具 Grafana 里面的面板就是通过 PromQL 来进行数据查询的。

## 瞬时向量 Instant Vector

假设我们有一个对 HTTP 请求量的 counter 名为 `http_requests_total`，那么我们只需要在 Grafana 面板的 query editor 中输入 `http_requests_total` 就可以看到数据了。这个 `http_requests_total` 是一个瞬时向量，也就是说，这个数据是 Prometheus 采集数据的那一刻数据的值。对于 counter 数据，我们看到的会是一条不断增长的线（采集到的值只增不减）。

之前提到了，我们可以通过标签来给一个监控指标划分维度，在上面绘制出的图里，所有的 标签的值会交叉成多个时间序列，也就是说，假设有一个取值为 `200` 或 `404` 的标签 `code` 和取值为 `GET` 或 `POST` 的标签 `method`，那么，图中会有四条线，分别为：

```promql
http_requests_total { code=200, method=GET }
http_requests_total { code=200, method=GET }
http_requests_total { code=404, method=POST }
http_requests_total { code=404, method=POST }
```

这里重申一下，当我们上报数据的时候，这里的标签值不可以用一个数量非常多的值（例如用户 ID），否则会造成时间序列数量的极度膨胀。

假如我们只想看 `code` 为 `404` 的数据，那么我们可以用标签来对数据进行过滤：

```promql
http_requests_total { code="404" }
```

显然，我们一开始写的 `http_requests_total` 就相当于 `http_requests_total {}`。这里的 `=` 表示完整匹配，对应地，不匹配用 `!=`，如果需要正则匹配，则用 `=~` 和 `!~`。我们这里也可以将其写为 Prometheus 的底层表示形式：

```promql
{ __name__="http_requests_total", code="404" }
```

这个写法比较少用，但是有时候不得不用，例如我们的监控指标名是 PromQL 的关键字的时候：

```promql
on {}              # 错误，on 是关键字
{ __name__="on" }  # OK
```

另外，只使用标签而不使用监控指标名来进行匹配也是合法的，但如果不使用监控指标名，那么至少要存在一个标签匹配是不能匹配到空字符串的。

```promql
{job=~".*"}               # 错误，没有监控指标名，且会匹配到空字符串

{job=~".+"}               # OK，不会匹配到空字符串
{job=~".*",method="get"}  # OK，虽然 job 会匹配到空字符串，但 method 不会
```

前面提到，histogram 和 summary 会产生多个时间序列，那么它们的时间序列要如何进行查询呢？事实上，Prometheus 会根据一定的规则来给这些时间序列命名，以 histgram 为例，假设存在一个监控 `mymetric`，设置了 1, 2, 3 bucket ，且采集到了如下数据：

| buckets | observe | write | values             |
| ------- | ------- | ----- | ------------------ |
| 1       | 2       | 2     | 0.2, 0.6           |
| 2       | 3       | 5     | 1.3, 1.5, 1.5      |
| 3       | 4       | 9     | 2.4, 2.6, 2.8, 2.9 |

那么，可以得到这样的结果（注意 bucket 的结果向下包含）：

```
mymetric_bucket { le="1" } = 2
mymetric_bucket { le="2" } = 5
mymetric_bucket { le="3" } = 9
mymetric_bucket { le="+Inf" } = 9
mymetric_count = 9
mymetric_sum = 15.8
```

如前所述，histogram 并没有存储数据采样点的值，只保留了总和和每一个区间的 counter。我们可以在 PromQL 中用 `histogram_quantile()` 函数来计算其值的分位数。由于 histogram 的分位数是在 PromQL 中指定的，因此它的灵活性比 summary 高（summary 只能获取上报时定下来的分位数）。

## 范围向量 Range Vector

前面提到我们用 `http_requests_total` 获取到单调递增的 HTTP 请求总数图像，但是这个对于我们而言并没有太大意义，我们希望看到的是类似「每分钟请求数量」这样的数据，PromQL 允许我们获取一段时间的数据，这个被称为范围向量。它的获取方式是在瞬时向量后面加一个中括号，里面填入需要的时间段长度，例如：

```promql
http_requests_total [1m]
```

这里获取的就是过去一分钟的 HTTP 请求总数。但是，当我们将这个语句输入 query editor 后，我们会看到一个错误

> invalid expression type "range vector" for range query, must be Scalar or instant Vector

也就是说，范围向量是无法被直接绘制的。我们需要用内置的函数将其转换为一个瞬时向量后才能被绘制出来。例如我们想看到每 1 分钟的 HTTP 请求量，这个需求事实上是「查看一分钟范围内的变化量」，那么我们可以用 `increase` 函数：

```promql
increase(http_requests_total [1m])
```

更多完整的函数介绍可以在[这里](https://prometheus.io/docs/prometheus/latest/querying/functions/)找到。

前面提到，假设有一个取值为 `200` 或 `404` 的标签 `code` 和取值为 `GET` 或 `POST` 的标签 `method`，图中会有四条线，分别为：

```promql
http_requests_total { code=200, method=GET }
http_requests_total { code=200, method=GET }
http_requests_total { code=404, method=POST }
http_requests_total { code=404, method=POST }
```

但我们希望的是查看每分钟的请求总量，此时，我们可以用 `sum` 操作符（注意这不是一个函数而是一个聚合操作符）将数据聚合起来：

```promql
sum(increase(http_requests_total [1m]))
```

上面的表达式可以将 4 条曲线合并为一条，让我们更清晰地看到请求总量的变化情况。但我们有时候还希望能够按照返回码来对数据进行归类，这个需求在希望观察错误码变化量的情况下非常常见。所有聚合操作符都可以跟 `by (labels)` 或 `without (labels)` 这样的后缀，用于针对某些标签进行操作。例如上面的需求就可以用这个表达式实现：

```promql
sum by (code) (increase(http_requests_total [1m]))
```

这等价于：

```promql
sum without (method) (increase(http_requests_total [1m]))
```

后缀可以紧跟 `sum`，也可以放到被操作的表达式后面：

```promql
sum (increase(http_requests_total [1m])) by (code)
```

更多对操作符的介绍可以在[这里](https://prometheus.io/docs/prometheus/latest/querying/operators)找到。另外，也可以在[这里](https://prometheus.io/docs/prometheus/latest/querying/examples/)找到更多的使用例子。

# 总结

Prometheus 基于时序数据库的查询实现了丰富复杂的语义，让用户能够灵活实现各种监控需求，为了能更好地表达自己的查询逻辑，我们需要先了解其中的基本语义，本文仅进行了较为简略的总结，更详细的可以参考[官方文档](https://prometheus.io/docs/prometheus/latest/getting_started/)和[官方最佳实践](https://prometheus.io/docs/practices/naming/)。
