---
title: TiKV Rust Client 迁移记 - futures 0.1 至 0.3
author: ['Nick Cameron']
date: 2019-09-25
summary: 最近我将一个中小型的 crate 从 futures 库的 0.1 迁移至了 0.3 版本。过程本身不是特别麻烦，但还是有些地方或是微妙棘手，或是没有很好的文档说明。这篇文章里，我会把迁移经验总结分享给大家。 
tags: ['Rust','TiKV','futures']
---

>作者介绍：Nick Cameron，PingCAP 研发工程师，Rust core team 成员，专注于分布式系统、数据库领域和 Rust 语言的进展。

最近我将一个中小型的 crate 从 futures 库的 0.1 迁移至了 0.3 版本。过程本身不是特别麻烦，但还是有些地方或是微妙棘手，或是没有很好的文档说明。这篇文章里，我会把迁移经验总结分享给大家。 

我所迁移的 crate 是 TiKV 的 [Rust Client](https://github.com/tikv/client-rust)。该 crate 的规模约为 5500 行左右代码，通过 gRPC 与 TiKV 交互，采用异步接口实现。因此，对于 futures 库的使用颇为重度。 

异步编程是 Rust 语言中影响广泛的一块领域，已有几年发展时间，其核心部分就是 [futures](https://github.com/rust-lang-nursery/futures-rs) 库。作为一个标准 Rust 库，futures 库为使用 futures 编程提供所需数据类型以及功能。虽然它是异步编程的关键，但并非你所需要的一切 - 你仍然需要可以推进事件循环 (event loop) 以及与操作系统交互的其他库。

`futures` 库在这几年中变化很大。最新的版本为 0.3（crates.io 发布的 `futures` 预览版）。然而，有许多早期代码是 futures 0.1 系列版本，且一直没有更新。这样的分裂事出有因 - 0.1 和 0.3 版本之间变化太大。0.1 版本相对稳定，而 0.3 版本一直处于快速变化中。长远来看，0.3 版本最终会演进为 1.0。有一部分代码会进入 Rust 标准库，其中的第一部分已在最近发布了稳定版，也就是 `Future` trait。

为了让 Rust Client 跑在稳定的编译器上，我们将核心库限制为仅使用稳定或即将稳定的特性。我们在文档和示例中确实使用了 async/await，因为 async/await 更符合工程学要求，而且将来也一定会成为使用 Rust 进行异步编程的推荐方法。除了在核心库中避免使用 async/await，我们对使用 futures 0.1 的 crate 也有依赖，这也意味着我们需要经常用到兼容层。从这个角度说，我们这次迁移其实并不够典型。

我不是异步编程领域的专家，或许有其他方法能让我们这次迁移（以及所涉及的代码）更符合大家的使用习惯。如果您有好的建议，可以在 [Twitter](https://twitter.com/nick_r_cameron) 上联系我。如果您想要贡献 PR 就更赞了，我们期待越来越多的力量加入到 [TiKV Client](https://github.com/tikv/client-rust) 项目里。

## 机械性变化

此类变化是指那些 “查询替换类” ，或其他无需复杂思考的变化。

这一类别中最大的变化莫过于 0.1 版本的 `Future` 签名中包含了一个 `Error` 关联类型，而且 `poll` 总是会返回一个 `Result`。0.3 版本里该错误类型已被移除，对于错误需要显式处理。为了保持行为上的一致性，我们需要将代码里所有  `Future<Item=Foo, Error=Bar>` 替换为 `Future<Output=Result<Foo, Bar>>`（留意 `Item` 到 `Output` 的名称变化）。替换后，  `poll` 就可以返回和以前一样的类型，这样在使用 futures 的时候无需任何变化。

如果你定义了自己的 futures，那就需要根据是否需要处理错误的需求更新 futures 的定义。 

futures 0.3 中支持 `TryFuture` 类型，基本上可以看作 `Future<Output=Result<...>>` 的替代。使用这个类型，意味着你需要在 `Future` 与 `TryFuture` 之间转换，因此最好还是尽量避免吧。`TryFuture` 类型包含了一个 blanket implementation，这使它可以通过 `TryFutureEx` trait 轻松将某些函数应用于此类 futures。

futures 0.3 中，`Future::poll` 方法会接受一个新的上下文参数。这基本上只需要调用 `poll` 方法即可完成传递（偶尔也会忽略）。 

我们的依赖包依然使用了 futures 0.1，所以我们必须在两个版本的库之间转换。0.3 版本包含了一些兼容层以及其他实用工具（例如 `Compat01As03`）。我们在调用依赖关系时会用到这些。
 
`wait` 方法已被从 `Future` trait 中移除。这是让人拍手称快的变化，因为该方法确实够反人性，而且本身可以用 `.await` 或 `executor::block_on` 代替（需要注意的是后者可能会阻断整个进程，而并不只是当前执行的 future）。

## Pin

futures 0.3 中， [`Pin`](https://doc.rust-lang.org/nightly/std/pin/index.html) 是一个频繁使用的类型， `Future::poll` 方法签名的 `self` 类型对其尤为青睐。除了对这些签名进行一些机械性的处理之外，我还得借助于 `Pin::get_unchecked_mut` 与 `Pin::new_unchecked` 这两种方法（均为不安全方法）对 futures 的项目字段做一些变更。

指针定位（pinning）是一个微妙又复杂的概念，我至今也不敢说自己已经掌握了多少。我能提供的最好的参考是 [std::pin docs](https://doc.rust-lang.org/nightly/std/pin/index.html)。下面是我整理的一些要点（有一些重要的细节此处不会涉及，这里本意也并非提供一个关于指针定位的教程）。

* `Pin` 作为一个类型构造，只有用于指针类型（如 `Pin<Box<_>>`）时才会生效。

* Pin 本身是一种“标识/封装”类型（有一点像 [`NonNull`](https://doc.rust-lang.org/nightly/std/ptr/struct.NonNull.html)），并不是指针类型。

* 如果一个指针类型被“定位”了，意味着指针指向的值不可移动（当一个非拷贝对象通过数值传入，或者调用  `mem::swap` 时会发生移动）。需要注意的移动只能发生在指针被定位之前，而非之后。

* 如果某个类型使用了 `Unpin` trait，这意味着无论此类型移动与否都不会有任何影响。换句话说，即使指向该类型的指针没有被定位，我们也可以放心把它当作被定位的。

* `Pin` 与 `Unpin` 并没有置入 Rust 语言，虽然某些特性会对指针定位有间接依赖。指针定位由编译器强制执行，但编译器本身却不自知（这点非常酷，也体现了 Rust 特性系统对此类处理的强大之处）。它是这样工作的：`Pin<P<T>>` 只允许对于 `P` 的安全访问，禁止移动 `P` 指向的任何数值，除非 `T` 应用了 `Unpin`（代码编写者已宣称 `T` 并不在意是否被移动）。任何允许删除没有执行 `Unpin` 数值的操作（可变访问）都是 `unsafe` 的，且应该由程序编写者决定是否要移动任何数值，并保证之后的安全代码中不可删除任何数值。

让我们回到 futures 迁移的话题上。如果你对 `Pin` 使用了不安全的方法，你就需要考虑上面的要点，以保证指针定位的稳定。[std::pin docs](https://doc.rust-lang.org/nightly/std/pin/index.html) 提供了更多的解释。我在许多地方通过字段投射的方式为另外一个 future 调用  `poll` 方法（有时是间接的），为了达到这个目的，你需要一个已定位的指针，这也意味着能你需要结构性指针定位。如，你可以将 `Pin<&mut T>` 字段投射至 `Pin<&mut FieldType>`。

## 函数

迁移中比较让人不爽的一点是 futures 库里有许多函数（与类型）的名称改变了。有的名称和标准库里的通用名重复，这让用自动化的手段处理变更的难度变大。比如，`Async` 变成了 `Poll`，`Ok` 变成了 `ready`，`for_each` 变成 `then`，`then` 变成 `map`，`Either::A` 变成 `Either::Left`。

有时名称没有变化，但其代表的功能语义变了（或者两方面都变了）。一个较为普遍的变化就是 closure 函数现在会返回可以使用 `T` 类型生成数值的 future，而不会直接返回数值本身。 

有许多组合子函数从 `Future` trait 移至扩展 crate 里。这个问题本身不难修复，只是有时候不容易从错误信息中判定。

## LoopFn

0.1 版本的 futures 库包含了 `LoopFn` 这个 future 构造，用于处理多次执行某动作的 futures。`LoopFn` 在 0.3 版本中被移除，这样做的原因个人认为可能是 `for` 循环本身是 `async` 的函数，或者 streams 才是长远看来的更佳解决方案。为了让我们的迁移过程简单化，我为 futures 0.3 写了我们自己版本的 `LoopFn` future，其实大部分也都是复制粘贴的工作，加上一些调整（如处理指针定位投射）：[code](https://github.com/tikv/client-rust/pull/41/commits/6353dbcfe391d66714686aafab9a49e593259dfb#diff-eeffc045326f81d4c46c22f225d3df90R28)。后来我将几处 `LoopFn` 用法转换为 streams，对代码似乎有一定改进。


## Sink::send_all

我们在项目中几个地方使用了 sink。我发现对于它们对迁移和 futures 相比要有难度不少，其中最麻烦的问题就是 `Sink::send_all` 结构变了。0.1 版本里，`Sink::send_all` 会获取 stream 的所有权，并在确定所有 future 都完成后返回 sink 以及 stream。0.3 版本里， `Sink::send_all` 会接受一个对 stream 的可变引用，不返回任何值。我自己写了一个 [兼容层](https://github.com/tikv/client-rust/pull/41/commits/6353dbcfe391d66714686aafab9a49e593259dfb#diff-eeffc045326f81d4c46c22f225d3df90R68) 在  futures 0.3  里模拟 0.1 版本的 sink。这不是很难，但也许有更好的方式来做这件事。

大家可以在 [这个 PR](https://github.com/tikv/client-rust/pull/41) 里看到整个迁移的细节。本文最初发表在 [www.ncameron.org](https://www.ncameron.org/blog/migrating-a-crate-from-futures-0-1-to-0-3/)。


