---
title: TiKV 源码解析系列文章（三）Prometheus（上）
author: ['Breezewish']
date: 2019-03-08
summary: 本篇将为大家介绍 rust-prometheus 的基础知识以及最基本的几个指标的内部工作机制。
tags: ['TiKV 源码解析','Prometheus','社区']
---

> 本文为 TiKV 源码解析系列的第三篇，继续为大家介绍 TiKV 依赖的周边库 [rust-prometheus]，本篇主要介绍基础知识以及最基本的几个指标的内部工作机制，下篇会介绍一些高级功能的实现原理。
>
> [rust-prometheus] 是监控系统 [Prometheus] 的 Rust 客户端库，由 TiKV 团队实现。TiKV 使用 [rust-prometheus] 收集各种指标（metric）到 Prometheus 中，从而后续能再利用 [Grafana] 等可视化工具将其展示出来作为仪表盘监控面板。这些监控指标对于了解 TiKV 当前或历史的状态具有非常关键的作用。TiKV 提供了丰富的监控指标数据，并且代码中也到处穿插了监控指标的收集片段，因此了解 [rust-prometheus] 很有必要。
>
> 感兴趣的小伙伴还可以观看我司同学在 [FOSDEM 2019] 会议上关于 rust-prometheus 的[技术分享][Share @ FOSDEM 2019]。

## 基础知识

### 指标类别

[Prometheus] 支持四种指标：Counter、Gauge、Histogram、Summary。[rust-prometheus] 库目前还只实现了前三种。TiKV 大部分指标都是 Counter 和 Histogram，少部分是 Gauge。

#### Counter

[Counter] 是最简单、常用的指标，适用于各种计数、累计的指标，要求单调递增。Counter 指标提供基本的 [`inc()`][`Counter::inc()`] 或 [`inc_by(x)`][`Counter::inc_by()`] 接口，代表增加计数值。

在可视化的时候，此类指标一般会展示为各个时间内增加了多少，而不是各个时间计数器值是多少。例如 TiKV 收到的请求数量就是一种 Counter 指标，在监控上展示为 TiKV 每时每刻收到的请求数量图表（QPS）。

#### Gauge

[Gauge] 适用于上下波动的指标。Gauge 指标提供 [`inc()`][`Gauge::inc()`]、[`dec()`][`Gauge::dec()`]、[`add(x)`][`Gauge::add()`]、[`sub(x)`][`Gauge::sub()`] 和 [`set(x)`][`Gauge::set()`] 接口，都是用于更新指标值。

这类指标可视化的时候，一般就是直接按照时间展示它的值，从而展示出这个指标按时间是如何变化的。例如 TiKV 占用的 CPU 率是一种 Gauge 指标，在监控上所展示的直接就是 CPU 率的上下波动图表。

#### Histogram

[Histogram] 即直方图，是一种相对复杂但同时也很强大的指标。Histogram 除了基本的计数以外，还能计算分位数。Histogram 指标提供 [`observe(x)`][`Histogram::observe()`] 接口，代表观测到了某个值。

举例来说，TiKV 收到请求后处理的耗时就是一种 Histogram 指标，通过 Histogram 类型指标，监控上可以观察 99%、99.9%、平均请求耗时等。这里显然不能用一个 Counter 存储耗时指标，否则展示出来的只是每时每刻中 TiKV 一共花了多久处理，而非单个请求处理的耗时情况。当然，机智的你可能想到了可以另外开一个 Counter 存储请求数量指标，这样累计请求处理时间除以请求数量就是各个时刻平均请求耗时了。

实际上，这也正是 Prometheus 中 Histogram 的内部工作原理。Histogram 指标实际上最终会提供一系列时序数据：

- 观测值落在各个桶（bucket）上的累计数量，如落在 `(-∞, 0.1]`、`(-∞, 0.2]`、`(-∞, 0.4]`、`(-∞, 0.8]`、`(-∞, 1.6]`、`(-∞, +∞)` 各个区间上的数量。
- 观测值的累积和。
- 观测值的个数。

bucket 是 Prometheus 对于 Histogram 观测值的一种简化处理方式。Prometheus 并不会具体记录下每个观测值，而是只记录落在配置的各个 bucket 区间上的观测值的数量，这样以牺牲一部分精度的代价大大提高了效率。

#### Summary

[Summary] 与 [Histogram] 类似，针对观测值进行采样，但分位数是在客户端进行计算。该类型的指标目前在 [rust-prometheus] 中没有实现，因此这里不作进一步详细介绍。大家可以阅读 Prometheus 官方文档中的[介绍][Summary]了解详细情况。感兴趣的同学也可以参考其他语言 Client Library 的实现为 [rust-prometheus] 贡献代码。

### 标签

Prometheus 的每个指标支持定义和指定若干组标签（[Label]），指标的每个标签值独立计数，表现了指标的不同维度。例如，对于一个统计 HTTP 服务请求耗时的 Histogram 指标来说，可以定义并指定诸如 HTTP Method（GET / POST / PUT / ...）、服务 URL、客户端 IP 等标签。这样可以轻易满足以下类型的查询：

- 查询 Method 分别为 POST、PUT、GET 的 99.9% 耗时（利用单一 Label）
- 查询 POST /api 的平均耗时（利用多个 Label 组合）

普通的查询诸如所有请求 99.9% 耗时也能正常工作。

需要注意的是，不同标签值都是一个独立计数的时间序列，因此应当避免标签值或标签数量过多，否则实际上客户端会向 Prometheus 服务端传递大量指标，影响效率。

与 Prometheus [Golang client] 类似，在 [rust-prometheus] 中，具有标签的指标被称为 Metric Vector。例如 Histogram 指标对应的数据类型是 [`Histogram`]，而具有标签的 Histogram 指标对应的数据类型是 [`HistogramVec`]。对于一个 [`HistogramVec`]，提供它的各个标签取值后，可获得一个 [`Histogram`] 实例。不同标签取值会获得不同的 [`Histogram`] 实例，各个 [`Histogram`] 实例独立计数。

## 基本用法

本节主要介绍如何在项目中使用 [rust-prometheus] 进行各种指标收集。使用基本分为三步：

1. 定义想要收集的指标。

2. 在代码特定位置调用指标提供的接口收集记录指标值。

3. 实现 HTTP Pull Service 使得 Prometheus 可以定期访问收集到的指标，或使用 rust-prometheus 提供的 Push 功能定期将收集到的指标上传到 [Pushgateway]。

> 注意，以下样例代码都是基于本文发布时最新的 rust-prometheus 0.5 版本 API。我们目前正在设计并实现 1.0 版本，使用上会进一步简化，但以下样例代码可能在 1.0 版本发布后过时、不再工作，届时请读者参考最新的文档。

### 定义指标

为了简化使用，一般将指标声明为一个全局可访问的变量，从而能在代码各处自由地操纵它。rust-prometheus 提供的各个指标（包括 Metric Vector）都满足 `Send + Sync`，可以被安全地全局共享。

以下样例代码借助 [lazy_static] 库定义了一个全局的 Histogram 指标，该指标代表 HTTP 请求耗时，并且具有一个标签名为 `method`：

```rust
#[macro_use]
extern crate prometheus;

lazy_static! {
   static ref REQUEST_DURATION: HistogramVec = register_histogram_vec!(
       "http_requests_duration",
       "Histogram of HTTP request duration in seconds",
       &["method"],
       exponential_buckets(0.005, 2.0, 20).unwrap()
   ).unwrap();
}
```

### 记录指标值

有了一个全局可访问的指标变量后，就可以在代码中通过它提供的接口记录指标值了。在“基础知识”中介绍过，[`Histogram`] 最主要的接口是 [`observe(x)`][`Histogram::observe()`]，可以记录一个观测值。若想了解 [`Histogram`] 其他接口或其他类型指标提供的接口，可以参阅 [rust-prometheus 文档]。

以下样例在上段代码基础上展示了如何记录指标值。代码模拟了一些随机值用作指标，装作是用户产生的。在实际程序中，这些当然得改成真实数据 :)

```rust
fn thread_simulate_requests() {
   let mut rng = rand::thread_rng();
   loop {
       // Simulate duration 0s ~ 2s
       let duration = rng.gen_range(0f64, 2f64);
       // Simulate HTTP method
       let method = ["GET", "POST", "PUT", "DELETE"].choose(&mut rng).unwrap();
       // Record metrics
       REQUEST_DURATION.with_label_values(&[method]).observe(duration);
       // One request per second
       std::thread::sleep(std::time::Duration::from_secs(1));
   }
}
```

### Push / Pull

到目前为止，代码还仅仅是将指标记录了下来。最后还需要让 Prometheus 服务端能获取到记录下来的指标数据。这里一般有两种方式，分别是 Push 和 Pull。

- Pull 是 Prometheus 标准的获取指标方式，Prometheus Server 通过定期访问应用程序提供的 HTTP 接口获取指标数据。
- Push 是基于 Prometheus [Pushgateway] 服务提供的另一种获取指标方式，指标数据由应用程序主动定期推送给 [Pushgateway]，然后 Prometheus 再定期从 Pushgateway 获取。这种方式主要适用于应用程序不方便开端口或应用程序生命周期比较短的场景。

以下样例代码基于 [hyper] HTTP 库实现了一个可以供 Prometheus Server pull 指标数据的接口，核心是使用 [rust-prometheus] 提供的 [`TextEncoder`] 将所有指标数据序列化供 Prometheus 解析：

```rust
fn metric_service(_req: Request<Body>) -> Response<Body> {
   let encoder = TextEncoder::new();
   let mut buffer = vec![];
   let mf = prometheus::gather();
   encoder.encode(&mf, &mut buffer).unwrap();
   Response::builder()
       .header(hyper::header::CONTENT_TYPE, encoder.format_type())
       .body(Body::from(buffer))
       .unwrap()
}
```

对于如何使用 Push 感兴趣的同学可以自行参考 rust-prometheus 代码内提供的 [Push 示例](https://github.com/pingcap/rust-prometheus/blob/89ca69913691d9d1609c78cc043fca9c3faa1a78/examples/example_push.rs#L1)，这里限于篇幅就不详细介绍了。

可以查看上述三段样例的 [完整代码](https://gist.github.com/breeswish/bb10bccd13a7fe332ef534ff0306ceb5) 了解更多内容。

## 内部实现

以下内部实现都基于本文发布时最新的 rust-prometheus 0.5 版本代码，该版本主干 API 的设计和实现 port 自 Prometheus [Golang client]，但为 Rust 的使用习惯进行了一些修改，因此接口上与 Golang client 比较接近。

目前我们正在开发 1.0 版本，API 设计上不再主要参考 Golang client，而是力求提供对 Rust 使用者最友好、简洁的 API。实现上为了效率考虑也会和这里讲解的略微有一些出入，且会去除一些目前已被抛弃的特性支持，简化实现，因此请读者注意甄别。

### Counter / Gauge

Counter 与 Gauge 是非常简单的指标，只要支持线程安全的数值更新即可。读者可以简单地认为 Counter 和 Gauge 的核心实现都是 `Arc<Atomic>`。但由于 Prometheus 官方规定指标数值需要支持浮点数，因此我们基于 [`std::sync::atomic::AtomicU64`] 和 CAS 操作实现了 [`AtomicF64`]，其具体实现位于 [src/atomic64/nightly.rs](https://github.com/pingcap/rust-prometheus/blob/89ca69913691d9d1609c78cc043fca9c3faa1a78/src/atomic64/nightly.rs)。核心片段如下：

```rust
impl Atomic for AtomicF64 {
   type T = f64;

   // Some functions are omitted.

   fn inc_by(&self, delta: Self::T) {
       loop {
           let current = self.inner.load(Ordering::Acquire);
           let new = u64_to_f64(current) + delta;
           let swapped = self
               .inner
               .compare_and_swap(current, f64_to_u64(new), Ordering::Release);
           if swapped == current {
               return;
           }
       }
   }
}
```

另外由于 0.5 版本发布时 [`AtomicU64`] 仍然是一个 nightly 特性，因此为了支持 Stable Rust，我们还基于自旋锁提供了 [`AtomicF64`] 的 fallback，位于 [src/atomic64/fallback.rs](https://github.com/pingcap/rust-prometheus/blob/89ca69913691d9d1609c78cc043fca9c3faa1a78/src/atomic64/fallback.rs)。

> 注：[`AtomicU64`] 所需的 [integer_atomics](https://github.com/rust-lang/rust/issues/32976) 特性最近已在 rustc 1.34.0 stabilize。我们将在 rustc 1.34.0 发布后为 Stable Rust 也使用上原生的原子操作从而提高效率。

### Histogram

根据 Prometheus 的要求，Histogram 需要进行的操作是在获得一个观测值以后，为观测值处在的桶增加计数值。另外还有总观测值、观测值数量需要累加。

注意，Prometheus 中的 Histogram 是[累积直方图](https://en.wikipedia.org/wiki/Histogram#Cumulative_histogram)，其每个桶的含义是 `(-∞, x]`，因此对于每个观测值都可能要更新多个连续的桶。例如，假设用户定义了 5 个桶边界，分别是 0.1、0.2、0.4、0.8、1.6，则每个桶对应的数值范围是 `(-∞, 0.1]`、`(-∞, 0.2]`、`(-∞, 0.4]`、`(-∞, 0.8]`、`(-∞, 1.6]`、`(-∞, +∞)`，对于观测值 0.4 来说需要更新`(-∞, 0.4]`、`(-∞, 0.8]`、`(-∞, 1.6]`、`(-∞, +∞)` 四个桶。

一般来说 [`observe(x)`][`Histogram::observe()`] 会被频繁地调用，而将收集到的数据反馈给 Prometheus 则是个相对很低频率的操作，因此用数组实现“桶”的时候，我们并不将各个桶与数组元素直接对应，而将数组元素定义为非累积的桶，如 `(-∞, 0.1)`、`[0.1, 0.2)`、`[0.2, 0.4)`、`[0.4, 0.8)`、`[0.8, 1.6)`、`[1.6, +∞)`，这样就大大减少了需要频繁更新的数据量；最后在上报数据给 Prometheus 的时候将数组元素累积，得到累积直方图，这样就得到了 Prometheus 所需要的桶的数据。

当然，由此可见，如果给定的观测值超出了桶的范围，则最终记录下的最大值只有桶的上界了，然而这并不是实际的最大值，因此使用的时候需要多加注意。

[`Histogram`] 的核心实现见 [src/histogram.rs](https://github.com/pingcap/rust-prometheus/blob/89ca69913691d9d1609c78cc043fca9c3faa1a78/src/histogram.rs)：

```rust
pub struct HistogramCore {
   // Some fields are omitted.
   sum: AtomicF64,
   count: AtomicU64,
   upper_bounds: Vec<f64>,
   counts: Vec<AtomicU64>,
}

impl HistogramCore {
   // Some functions are omitted.

   pub fn observe(&self, v: f64) {
       // Try find the bucket.
       let mut iter = self
           .upper_bounds
           .iter()
           .enumerate()
           .filter(|&(_, f)| v <= *f);
       if let Some((i, _)) = iter.next() {
           self.counts[i].inc_by(1);
       }

       self.count.inc_by(1);
       self.sum.inc_by(v);
   }
}

#[derive(Clone)]
pub struct Histogram {
   core: Arc<HistogramCore>,
}
```

[`Histogram`] 还提供了一个辅助结构 [`HistogramTimer`]，它会记录从它创建直到被 Drop 的时候的耗时，将这个耗时作为 [`Histogram::observe()`] 接口的观测值记录下来，这样很多时候在想要记录 Duration / Elapsed Time 的场景中，就可以使用这个简便的结构来记录时间：

```rust
#[must_use]
pub struct HistogramTimer {
   histogram: Histogram,
   start: Instant,
}

impl HistogramTimer {
   // Some functions are omitted.

   pub fn observe_duration(self) {
       drop(self);
   }

   fn observe(&mut self) {
       let v = duration_to_seconds(self.start.elapsed());
       self.histogram.observe(v)
   }
}

impl Drop for HistogramTimer {
   fn drop(&mut self) {
       self.observe();
   }
}
```

[`HistogramTimer`] 被标记为了 [`must_use`]，原因很简单，作为一个记录流逝时间的结构，它应该被存在某个变量里，从而记录这个变量所处作用域的耗时（或稍后直接调用相关函数提前记录耗时），而不应该作为一个未使用的临时变量被立即 Drop。标记为 `must_use` 可以在编译期杜绝这种明显的使用错误。

[Prometheus]: https://prometheus.io
[Grafana]: https://grafana.com/
[rust-prometheus]: https://github.com/pingcap/rust-prometheus
[Golang client]: https://github.com/prometheus/client_golang
[Share @ FOSDEM 2019]: https://fosdem.org/2019/schedule/event/rust_prometheus/
[Counter]: https://prometheus.io/docs/concepts/metric_types/#counter
[Gauge]: https://prometheus.io/docs/concepts/metric_types/#gauge
[Histogram]: https://prometheus.io/docs/concepts/metric_types/#histogram
[Summary]: https://prometheus.io/docs/concepts/metric_types/#summary
[Label]: https://prometheus.io/docs/concepts/data_model/#metric-names-and-labels
[FOSDEM 2019]: https://fosdem.org/2019/
[Pushgateway]: https://github.com/prometheus/pushgateway
[lazy_static]: https://docs.rs/lazy_static
[`Counter`]: https://docs.rs/prometheus/0.5.0/prometheus/core/struct.GenericCounter.html
[`Counter::inc()`]: https://docs.rs/prometheus/0.5.0/prometheus/core/struct.GenericCounter.html#method.inc
[`Counter::inc_by()`]: https://docs.rs/prometheus/0.5.0/prometheus/core/struct.GenericCounter.html#method.inc_by
[`Gauge`]: https://docs.rs/prometheus/0.5.0/prometheus/core/struct.GenericGauge.html
[`Gauge::inc()`]: https://docs.rs/prometheus/0.5.0/prometheus/core/struct.GenericGauge.html#method.inc
[`Gauge::dec()`]: https://docs.rs/prometheus/0.5.0/prometheus/core/struct.GenericGauge.html#method.dec
[`Gauge::add()`]: https://docs.rs/prometheus/0.5.0/prometheus/core/struct.GenericGauge.html#method.add
[`Gauge::sub()`]: https://docs.rs/prometheus/0.5.0/prometheus/core/struct.GenericGauge.html#method.sub
[`Gauge::set()`]: https://docs.rs/prometheus/0.5.0/prometheus/core/struct.GenericGauge.html#method.set
[`Histogram`]: https://docs.rs/prometheus/0.5.0/prometheus/struct.Histogram.html
[`Histogram::observe()`]: https://docs.rs/prometheus/0.5.0/prometheus/struct.Histogram.html#method.observe
[`HistogramVec`]: https://docs.rs/prometheus/0.5.0/prometheus/type.HistogramVec.html
[rust-prometheus 文档]: https://docs.rs/prometheus
[hyper]: https://docs.rs/hyper/0.12.23/hyper/
[`TextEncoder`]: https://docs.rs/prometheus/0.5.0/prometheus/struct.TextEncoder.html
[`std::sync::atomic::AtomicU64`]: https://doc.rust-lang.org/std/sync/atomic/struct.AtomicU64.html
[`AtomicU64`]: https://doc.rust-lang.org/std/sync/atomic/struct.AtomicU64.html
[`AtomicI64`]: https://doc.rust-lang.org/std/sync/atomic/struct.AtomicI64.html
[`AtomicF64`]: https://docs.rs/prometheus/0.5.0/prometheus/core/type.AtomicF64.html
[`Atomic`]: https://docs.rs/prometheus/0.5.0/prometheus/core/trait.Atomic.html
[`HistogramTimer`]: https://docs.rs/prometheus/0.5.0/prometheus/struct.HistogramTimer.html
[`must_use`]: https://doc.rust-lang.org/reference/attributes.html#must_use
[`CounterVec`]: https://docs.rs/prometheus/0.5.0/prometheus/type.CounterVec.html
[`GaugeVec`]: https://docs.rs/prometheus/0.5.0/prometheus/type.GaugeVec.html
[`HistogramVec`]: https://docs.rs/prometheus/0.5.0/prometheus/type.HistogramVec.html
[`HistogramVec::with_label_values`]: https://docs.rs/prometheus/0.5.0/prometheus/core/struct.MetricVec.html#method.with_label_values
[`MetricVecBuilder`]: https://docs.rs/prometheus/0.5.0/prometheus/core/trait.MetricVecBuilder.html
[`LocalCounter::flush()`]: https://docs.rs/prometheus/0.5.0/prometheus/core/struct.GenericLocalCounter.html#method.flush
[`LocalHistogram::observe()`]: https://docs.rs/prometheus/0.5.0/prometheus/local/struct.LocalHistogram.html#method.observe
[`LocalHistogram::flush()`]: https://docs.rs/prometheus/0.5.0/prometheus/local/struct.LocalHistogram.html#method.flush
[`Histogram::local()`]: https://docs.rs/prometheus/0.5.0/prometheus/struct.Histogram.html#method.local
[`LocalCounter`]: https://docs.rs/prometheus/0.5.0/prometheus/core/struct.GenericLocalCounter.html
[`LocalHistogram`]: https://docs.rs/prometheus/0.5.0/prometheus/local/struct.LocalHistogram.html

