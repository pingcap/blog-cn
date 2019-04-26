---
title: TiKV 源码解析系列文章（五）fail-rs 介绍
author: ['张博康']
date: 2019-03-29
summary: 本文为 TiKV 源码解析系列的第五篇，为大家介绍 TiKV 在测试中使用的周边库 fail-rs。
tags: ['TiKV 源码解析','社区']
---

本文为 TiKV 源码解析系列的第五篇，为大家介绍 TiKV 在测试中使用的周边库 [fail-rs](https://github.com/pingcap/fail-rs)。

fail-rs 的设计启发于 FreeBSD 的 [failpoints](https://www.freebsd.org/cgi/man.cgi?query=fail)，由 Rust 实现。通过代码或者环境变量，其允许程序在特定的地方动态地注入错误或者其他行为。在 TiKV 中通常在测试中使用 fail point 来构建异常的情况，是一个非常方便的测试工具。

## Fail point 需求

在我们的集成测试中，都是简单的构建一个 KV 实例，然后发送请求，检查返回值和状态的改变。这样的测试可以较为完整地测试功能，但是对于一些需要精细化控制的测试就鞭长莫及了。我们当然可以通过 mock 网络层提供网络的精细模拟控制，但是对于诸如磁盘 IO、系统调度等方面的控制就没办法做到了。

同时，在分布式系统中时序的关系是非常关键的，可能两个操作的执行顺行相反，就导致了迥然不同的结果。尤其对于数据库来说，保证数据的一致性是至关重要的，因此需要去做一些相关的测试。

基于以上原因，我们就需要使用 fail point 来复现一些 corner case，比如模拟数据落盘特别慢、raftstore 繁忙、特殊的操作处理顺序、错误 panic 等等。

## 基本用法

### 示例

在详细介绍之前，先举一个简单的例子给大家一个直观的认识。

还是那个老生常谈的 Hello World：

```rust
#[macro_use]
extern crate fail;

fn say_hello() {
    fail_point!(“before_print”);
    println!(“Hello World~”);
}

fn main() {
    say_hello();
    fail::cfg("before_print", "panic");
    say_hello();
}
```

运行结果如下：

```text
Hello World~
thread 'main' panicked at 'failpoint before_print panic' ...
```

可以看到最终只打印出一个 `Hello World～`，而在打印第二个之前就 panic 了。这是因为我们在第一次打印完后才指定了这个 fail point 行为是 panic，因此第一次在 fail point 不做任何事情之后正常输出，而第二次在执行到 fail point 时就会根据配置的行为 panic 掉！

### Fail point 行为

当然 fail point 不仅仅能注入 panic，还可以是其他的操作，并且可以按照一定的概率出现。描述行为的格式如下：

```
[<pct>%][<cnt>*]<type>[(args...)][-><more terms>]
```

+ pct：行为被执行时有百分之 pct 的机率触发
+ cnt：行为总共能被触发的次数
+ type：行为类型
    - off：不做任何事
    - return(arg)：提前返回，需要 fail point 定义时指定 expr，arg 会作为字符串传给 expr 计算返回值
    - sleep(arg)：使当前线程睡眠 arg 毫秒
    - panic(arg)：使当前线程崩溃，崩溃消息为 arg
    - print(arg)：打印出 arg
    - pause：暂停当前线程，直到该 fail point 设置为其他行为为止
    - yield：使当前线程放弃剩余时间片
    - delay(arg)：和 sleep 类似，但是让 CPU 空转 arg 毫秒
+ args：行为的参数

比如我们想在 `before_print` 处先 sleep 1s 然后有 1% 的机率 panic，那么就可以这么写：

```text
"sleep(1000)->1%panic"
```

### 定义 fail point

只需要使用宏 `fail_point!` 就可以在相应代码中提前定义好 fail point，而具体的行为在之后动态注入。

```rust
fail_point!("failpoint_name");
fail_point!("failpoint_name", |_| { // 指定生成自定义返回值的闭包，只有当 fail point 的行为为 return 时，才会调用该闭包并返回结果
    return Error
});
fail_point!("failpoint_name", a == b, |_| { // 当满足条件时，fail point 才被触发
    return Error
})
```

### 动态注入

#### 环境变量

通过设置环境变量指定相应 fail point 的行为：

```shell
FAILPOINTS="<failpoint_name1>=<action>;<failpoint_name2>=<action>;..."
```

注意，在实际运行的代码需要先使用 `fail::setup()` 以环境变量去设置相应 fail point，否则 `FAILPOINTS` 并不会起作用。

```rust
#[macro_use]
extern crate fail;

fn main() {
    fail::setup(); // 初始化 fail point 设置
    do_fallible_work();
    fail::teardown(); // 清除所有 fail point 设置，并且恢复所有被 fail point 暂停的线程
}
```

#### 代码控制

不同于环境变量方式，代码控制更加灵活，可以在程序中根据情况动态调整 fail point 的行为。这种方式主要应用于集成测试，以此可以很轻松地构建出各种异常情况。

```rust
fail::cfg("failpoint_name", "actions"); // 设置相应的 fail point 的行为
fail::remove("failpoint_name"); // 解除相应的 fail point 的行为
```

## 内部实现

以下我们将以 fail-rs v0.2.1 版本代码为基础，从 API 出发来看看其背后的具体实现。

fail-rs 的实现非常简单，总的来说，就是内部维护了一个全局 map，其保存着相应 fail point 所对应的行为。当程序执行到某个 fail point 时，获取并执行该全局 map 中所保存的相应的行为。

全局 map 其具体定义在 [FailPointRegistry](https://github.com/pingcap/fail-rs/blob/v0.2.1/src/lib.rs#L602)。

```rust
struct FailPointRegistry {
    registry: RwLock<HashMap<String, Arc<FailPoint>>>,
}
```

其中 [FailPoint](https://github.com/pingcap/fail-rs/blob/v0.2.1/src/lib.rs#L518) 的定义如下：

```rust
struct FailPoint {
    pause: Mutex<bool>,
    pause_notifier: Condvar,
    actions: RwLock<Vec<Action>>,
    actions_str: RwLock<String>,
}
```

`pause` 和 `pause_notifier` 是用于实现线程的暂停和恢复，感兴趣的同学可以去看看代码，太过细节在此不展开了；`actions_str` 保存着描述行为的字符串，用于输出；而 `actions` 就是保存着 failpoint 的行为，包括概率、次数、以及具体行为。`Action` 实现了 `FromStr` 的 trait，可以将满足格式要求的字符串转换成 `Action`。这样各个 API 的操作也就显而易见了，实际上就是对于这个全局 map 的增删查改：

+ [fail::setup()](https://github.com/pingcap/fail-rs/blob/v0.2.1/src/lib.rs#L628) 读取环境变量 `FAILPOINTS` 的值，以 `;` 分割，解析出多个 `failpoint name` 和相应的 `actions` 并保存在 `registry` 中。
+ [fail::teardown()](https://github.com/pingcap/fail-rs/blob/v0.2.1/src/lib.rs#L729) 设置 `registry` 中所有 fail point 对应的 `actions` 为空。
+ [fail::cfg(name, actions)](https://github.com/pingcap/fail-rs/blob/v0.2.1/src/lib.rs#L729) 将 `name` 和对应解析出的 `actions` 保存在 `registry` 中。
+ [fail::remove(name)](https://github.com/pingcap/fail-rs/blob/v0.2.1/src/lib.rs#L729) 设置 `registry` 中 `name` 对应的 `actions` 为空。

而代码到执行到 fail point 的时候到底发生了什么呢，我们可以展开 [fail_point!](https://github.com/pingcap/fail-rs/blob/v0.2.1/src/lib.rs#L817) 宏定义看一下：

```rust
macro_rules! fail_point {
    ($name:expr) => {{
        $crate::eval($name, |_| {
            panic!("Return is not supported for the fail point \"{}\"", $name);
        });
    }};
    ($name:expr, $e:expr) => {{
        if let Some(res) = $crate::eval($name, $e) {
            return res;
        }
    }};
    ($name:expr, $cond:expr, $e:expr) => {{
        if $cond {
            fail_point!($name, $e);
        }
    }};
}
```

现在一切都变得豁然开朗了，实际上就是对于 `eval` 函数的调用，当函数返回值为 `Some` 时则提前返回。而 `eval` 就是从全局 map 中获取相应的行为，在 `p.eval(name)` 中执行相应的动作，比如输出、等待亦或者 panic。而对于 `return` 行为的情况会特殊一些，在 `p.eval(name)` 中并不做实际的动作，而是返回 `Some(arg)` 并通过 `.map(f)` 传参给闭包产生自定义的返回值。

```rust
pub fn eval<R, F: FnOnce(Option<String>) -> R>(name: &str, f: F) -> Option<R> {
    let p = {
        let registry = REGISTRY.registry.read().unwrap();
        match registry.get(name) {
            None => return None,
            Some(p) => p.clone(),
        }
    };
    p.eval(name).map(f)
}
```

## 小结

至此，关于 fail-rs 背后的秘密也就清清楚楚了。关于在 TiKV 中使用 fail point 的测试详见 [github.com/tikv/tikv/tree/master/tests/failpoints](https://github.com/tikv/tikv/tree/master/tests/failpoints)，大家感兴趣可以看看在 TiKV 中是如何来构建异常情况的。

同时，fail-rs 计划支持 HTTP API，欢迎感兴趣的小伙伴提交 PR。
