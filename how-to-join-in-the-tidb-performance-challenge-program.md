---
title: 如何玩转 TiDB 性能挑战赛？本文教你 30 分钟快速上手拿积分！
author: ['Wish']
date: 2019-11-11
summary: 本文以 TiKV 性能挑战赛 Easy 级别任务“PCP：Migrate functions from TiDB”为例，教大家如何快速又正确地完成这个任务。
tags: ['TiKV','社区','性能挑战赛']
---

上周我们正式宣布了 [TiDB 性能挑战赛](https://pingcap.com/community-cn/tidb-performance-challenge/)。在赛季内，通过向 TiDB、TiKV、PD 贡献代码完成指定类别任务的方式，你可以获得相应的积分，最终你可以使用积分兑换礼品或奖金。在性能挑战赛中，你首先需要完成几道 Easy 的题目，积累一定量积分后，才能开始挑战 Medium / Hard 难度的题目。

活动发布后，大家向我们反馈 TiKV 任务的资料比较少，上手难度比较高。因此本文以 TiKV 性能挑战赛 Easy 级别任务 [PCP: Migrate functions from TiDB](https://github.com/tikv/tikv/issues/5751) 为例，教大家如何快速又正确地完成这个任务，从而玩转“TiDB 性能挑战赛”。这个任务中每项完成后均可以获得 50 分，是积累分数从而挑战更高难度任务的好机会。既能改进 TiKV 为性能提升添砖加瓦、又能参与比赛得到积分，还能成为 Contributor，感兴趣的小伙伴们一起来“打怪”吧！

## 背景知识

TiKV Coprocessor（协处理）模块为 TiDB 提供了在存储引擎侧直接进行部分 SQL 计算的功能，支持按表达式进行过滤、聚合等，这样不仅利用起了 TiKV 机器的 CPU 资源，还能显著减少网络传输及相应的 RPC 开销，显著提升性能。大家可以阅读 [《TiKV 源码解析系列文章（十四）Coprocessor 概览》](https://pingcap.com/blog-cn/tikv-source-code-reading-14/)一文进一步了解 Coprocessor 模块。

表达式计算是 Coprocessor 非常重要的一个功能，例如用户输入了这样的 SQL：

```sql
SELECT * FROM t WHERE  sqrt(col_area) > 10;
```

TiKV Coprocessor 使用表达式 `sqrt(col_area) > 10` 对每一行进行求值，并根据结果对数据进行过滤，最后将过滤后的结果返回给 TiDB。为了能计算这个表达式，TiKV 必须实现与 TiDB 行为一致的 `Sqrt` 函数，当然 `>` 运算符也要提供对应的实现，这些统称为内置函数（built-in function）。

TiDB 和 MySQL 有非常多的内置函数，但 TiKV 目前只实现了一部分，只有当用户输入的表达式完全被 TiKV 支持并已经进行充分测试时，对应的表达式才会被下推到 Coprocessor 执行，否则 TiDB 只能从 TiKV 捞完整数据上来，达不到加速目的。

另外，TiKV 从 3.0 版本开始就包含两套 Coprocessor 执行框架，一套是老的框架，基于火山模型（推荐阅读 paper： [Volcano - An Extensible and Parallel Query Evaluation System](https://paperhub.s3.amazonaws.com/dace52a42c07f7f8348b08dc2b186061.pdf)）实现，另一套是 3.0 的新框架，基于向量化模型（推荐阅读 paper：[MonetDB/X100: Hyper-Pipelining Query Execution](http://cidrdb.org/cidr2005/papers/P19.pdf)）实现。火山模型中每个算子和函数都按行一个一个计算，向量化模型中则按列批量计算。由于在向量化模型中一个批次进行的处理操作是一样的，因此它可以规避条件分支，且能更好地利用流水线与缓存，从而具有更高的计算效率，差距可达 10 倍以上。

既然两个模型中函数处理的数据单位是不一样的，它们自然也有不一样的函数签名及实现，因此还有一大批内置函数虽然在 TiKV 侧已经实现了，但只有火山模型的实现，而没有向量化模型的实现。这类函数虽然 TiDB 已下推计算，但 TiKV 会回退到使用火山模型而不是向量化模型，无法达成最优计算效率。

综上，TiDB 内置函数在 TiKV 侧有几种实现状态：

1.  完全没有实现，如 `FromDays` 函数。

2.  已有火山模型的实现，没有向量化模型的实现，如 `BitLength` 函数。

3.  火山模型和向量化都已实现，如 `LTReal` 函数。

[PCP: Migrate functions from TiDB](https://github.com/tikv/tikv/issues/5751) 这个任务就是希望大家能帮助我们在 TiKV 侧实现更多 TiDB 所支持的内置函数，并支持向量化计算。这个 issue 中 Non-Vectorize 打钩意味着函数已有火山模型的实现，Vectorized 打钩意味着函数已有向量化模型的实现。因此你可以：

*   选择一个完全没有实现的函数，如 `FromDays`，从 TiDB 侧迁移它的代码到 TiKV 并实现在火山模型（Non-Vectorize）上，提个 PR +50 积分，再迁移到向量化模型（Vectorize）上，从而再提个 PR +50 积分。

*   或选择一个已有火山模型但没有向量化实现的函数，如 `BitLength` 函数，为它适配向量化模型（Vectorize）接口，提个 PR +50 积分。

**实现一个完全没有在 TiKV 侧实现的内置函数一般来说具有更高难度，因此能获得更高回报！**

## 如何从 TiDB 迁移内置函数在火山模型上实现

这部分在 [《三十分钟成为 Contributor | 为 TiKV 添加 built-in 函数》](https://pingcap.com/blog-cn/30mins-become-contributor-of-tikv) 中有所介绍，大家可以照着这个教程来，这里就不再赘述。

>注：由于 Coprocessor 框架实现的是 Fallback 机制，不允许函数只有向量化实现而没有火山模型实现。因此，若一个内置函数完全没有在 TiKV 侧实现，请先将它在火山模型上进行实现，再迁移至向量化模型。

## 如何为函数适配向量化模型接口

**以下本文的重点！**

如果一个内置函数在 TiKV 中已经有了火山模型的实现，但没有向量化模型的实现，则可以迁移它。以 LogicalXor 内置函数为例，它之前并没有向量化的实现（当然现在 [有了](https://github.com/tikv/tikv/pull/5826)）。可以遵循以下步骤：

### 1. 找到火山模型的实现

在 `components/tidb_query/src/expr/scalar_function.rs` 中搜索 `LogicalXor`，可以发现这个函数的实现位于 `logical_xor` 函数：

```rust
LogicalXor => logical_xor,
```

接下来搜索 `fn logical_xor` 就可以定位到函数具体内容，位于 `builtin_op.rs`（PS：不同内置函数会在不同文件中，不要照搬）：

```rust
pub fn logical_xor(&self, ctx: &mut EvalContext, row: &[Datum]) -> Result<Option<i64>> {
    let arg0 = try_opt!(self.children[0].eval_int(ctx, row));
    let arg1 = try_opt!(self.children[1].eval_int(ctx, row));
    Ok(Some(((arg0 == 0) ^ (arg1 == 0)) as i64))
}
```

### 2. 翻译为向量化实现

阅读理解上面的代码，可知 `LogicalXor` 是一个二元内置函数。其中，第一个参数 `children[0]` 和第二个参数 `children[1]` 都是通过 `eval_int` 方式访问的，因此 `LogicalXor` 接受的两个参数都是 int 类型。最后，这个函数返回值是 `Result<Option<i64>>` 代表它计算结果也是 int 类型。可以由这些信息翻译为以下向量化计算代码，实现在 `components/tidb_query/src/rpn_expr/impl_op.rs` 文件中：

 
```rust
#[rpn_fn]
#[inline]
pub fn logical_xor(arg0: &Option<Int>, arg1: &Option<Int>) -> Result<Option<Int>> {
    // TODO
}
```


> 注：`Int` 是 `i64` 的 Type Alias。你既可以写 `Int` 也可以写 `i64`，不过更推荐 `Int` 一些。你可以从[这里](https://github.com/tikv/tikv/blob/d019ccecefc260ff760a53b7b8742fb84ffca9b5/components/tidb_query/src/codec/data_type/mod.rs#L10)找到所有的 Type Alias。`eval_xxx` 函数与类型的对应关系如下表所示。

| 火山模型函数名 | 对应参数类型 | 参数类型别名 |
|:-- |:-- |:----- | 
| `eval_int` | `Int` | `i64`|
| `eval_real` | `Real` | `ordered_float::NotNan<f64>`|
|`eval_decimal` | `Decimal` |  
|`eval_bytes` | `Bytes` | `Vec<u8>`|
|`eval_time` | `DateTime` |  
|`eval_duration` | `Duration` |  
|`eval_json` | `Json` |  

换句话说就是：向量化版本的 `logical_xor` 是一个接受两个参数且两个参数都是 Int 类型的函数，返回 Int，是不是非常直观呢？另外我们使用 `None` 来代表 SQL 中的 `NULL` 值，因此函数参数及返回值都是 `Option<Int>` 类型。

最后照搬原来的内部实现（注意处理好 `None` / `Some` 的情况），这个函数就算完成了：

```rust
#[rpn_fn]
#[inline]
pub fn logical_xor(arg0: &Option<Int>, arg1: &Option<Int>) -> Result<Option<Int>> {
    Ok(match (arg0, arg1) {
        (Some(arg0), Some(arg1)) => Some(((*arg0 == 0) ^ (*arg1 == 0)) as i64),
        _ => None,
    })
}
```

你可能会问，不是说好了向量化计算是批量计算的吗，为什么向量化计算版本的代码没有接受数组，而只是接受单个值呢？原因在于 TiKV 向量化计算框架会自动基于你的这个基本实现，在编译期生成向量化计算版本，伪代码类似于这样：

```rust
fn logical_xor_vector_scalar(arg0: []Int, arg1: Int) -> []Int {
  let r = vec![];
  for i in 0..n {
    r.push( logical_xor(arg0[i], arg1) );
  }
  return r;
}
 
fn logical_xor_scalar_vector(arg0: Int, arg1: []Int) -> []Int {
  let r = vec![];
  for i in 0..n {
    r.push( logical_xor(arg0, arg1[i]) );
  }
  return r;
}
 
fn logical_xor_vector_vector(arg0: []Int, arg1: []Int) -> []Int {
  let r = vec![];
  for i in 0..n {
    r.push( logical_xor(arg0[i], arg1[i]) );
  }
  return r;
}
 
fn logical_xor_scalar_scalar(arg0: Int, arg1: Int) -> []Int {
  let r = vec![];
  for i in 0..n {
    r.push( logical_xor(arg0, arg1) );
  }
  return r;
}
```


你只需要关注内置函数本身的逻辑实现，其他的全部自动搞定！这些所有的奥秘都隐藏在了 `#[rpn_fn]` 过程宏中。

当然，上面的伪代码只是便于你进行理解。这个过程宏的实际实现并不是像上面这样粗暴地组装代码。它巧妙地利用了 Rust 的泛型机制，让编译器去生成不同个数参数情况下的最优实现。这里有点偏题就不继续展开细说了，我们后续的源码阅读文章对这个机制会有进一步分析，感兴趣的同学可以阅读代码自行学习。

### 3. 增加函数入口

目前只是提供了向量化版本的函数实现，但还需要告诉向量化计算框架，在遇到 LogicalXor 这个内置函数的时候，使用上向量化版本 `logical_xor` 的实现。这一步很简单，修改 `components/tidb_query/src/rpn_expr/mod.rs` 文件中的 `map_expr_node_to_rpn_func` 函数，增加一个对应关系即可：
 
```rust
ScalarFuncSig::LogicalXor => logical_xor_fn_meta(),
```

注意，此处要为函数名加上 `_fn_meta` 后缀，从而用上 `#[rpn_fn]` 过程宏自动生成的向量化版本函数实现。不要问为什么，问就是约定 :D 

### 4. 撰写单元测试

搜索 `ScalarFuncSig::LogicalXor` 可以找到火山模型下的该函数单元测试：

 
```rust
#[test]
fn test_logic_op() {
    let tests = vec![
        ...
        (
            ScalarFuncSig::LogicalXor,
            Datum::I64(1),
            Datum::I64(1),
            Some(0),
        ),
        (
            ScalarFuncSig::LogicalXor,
            Datum::I64(1),
            Datum::I64(0),
            Some(1),
        ),
        (
            ScalarFuncSig::LogicalXor,
            Datum::I64(0),
            Datum::I64(0),
            Some(0),
        ),
        (
            ScalarFuncSig::LogicalXor,
            Datum::I64(2),
            Datum::I64(-1),
            Some(0),
        ),
        (ScalarFuncSig::LogicalXor, Datum::I64(0), Datum::Null, None),
        (ScalarFuncSig::LogicalXor, Datum::Null, Datum::I64(1), None),
    ];
    let mut ctx = EvalContext::default();
    for (op, lhs, rhs, exp) in tests {
        let arg1 = datum_expr(lhs);
        let arg2 = datum_expr(rhs);
        ……
    }
}
```


这个测试覆盖挺完备的，因此可以直接拿样例来复用，作为向量化版本的单元测试。向量化版本单元测试中不再使用 Datum 等结构，而是可以直接用最原始的基础数据结构 `Option<Int>`，配上 `RpnFnScalarEvaluator` 进行执行，代码如下：

 
```rust
#[test]
fn test_logical_xor() {
    let test_cases = vec![
        (Some(1), Some(1), Some(0)),
        (Some(1), Some(0), Some(1)),
        (Some(0), Some(0), Some(0)),
        (Some(2), Some(-1), Some(0)),
        (Some(0), None, None),
        (None, Some(1), None),
    ];
    for (arg0, arg1, expect_output) in test_cases {
        let output = RpnFnScalarEvaluator::new()
            .push_param(arg0)
            .push_param(arg1)
            .evaluate(ScalarFuncSig::LogicalXor)
            .unwrap();
        assert_eq!(output, expect_output);
    }
}
```

如果原来火山模型实现的单元测试不完备，那么请在你的向量化实现中的单元测试中补充更多测试样例，尽可能覆盖所有分支条件。你也可以从 TiDB 的实现中迁移测试样例。注意，测试的目标是要检测实现是否符合预期，预期的是 TiKV 实现与 TiDB 实现能输出一样的结果，因此 TiDB 的输出是标准输出，不能由你自己来决定这个函数的标准输出。

不过，有些情况下 TiDB 的输出可能与 MySQL 不一致，你可以选择与 TiDB 行为保持一致，也可以选择与 MySQL 行为保持一致，但都需要在 TiDB 中开 issue 汇报这个行为不一致情况。

### 5. 运行测试

至此，这个函数已经可以工作起来了，可以运行单元测试看一下：

```
make dev
```

或者干脆只跑刚才写的这个测试：

```
EXTRA_CARGO_ARGS="test_logical_xor" make dev
```

**测试通过就可以提 PR 了。注意要在 PR 的开头写上 `PCP #5751` 指明这个 PR 对应的性能挑战赛题目，不然合了是得不到积分的。另外我们鼓励每个 PR 都专注于做一件事情，所以请尽量不要在同一个 PR 内迁移或实现多个内置函数，否则只能得到一次 50 积分。**

### 6. 运行下推测试

众所周知，手工编写的测试样例往往会遗漏一些考虑欠缺的边缘情况，并且可能由于犯了一些错误，测试的预期输出实际与 TiDB 不一致。为了能覆盖这些边缘情况，进一步确保 TiKV 中的内置函数实现与 TiDB 的实现一致，我们有一批使用 [randgen](https://github.com/MariaDB/randgen) 自动生成的下推测试，位于 [https://github.com/tikv/copr-test](https://github.com/tikv/copr-test)。不管你是在 TiKV 中引入一个新的函数实现，还是迁移一个现有实现，都需要确保能跑过这个测试。流程如下：

1.  需要确保你新实现的函数在 [copr-test](https://github.com/tikv/copr-test) 项目的 [push-down-test/functions.txt](https://github.com/tikv/copr-test/blob/master/push-down-test/functions.txt) 文件中，如果没有的话需要往 [copr-test](https://github.com/tikv/copr-test) 项目提 PR 将函数加入测试列表中。你需要将 SQL 里的函数名追加在文件中，或者可以参考 [all_functions_reference.txt](https://github.com/tikv/copr-test/blob/master/push-down-test/all_functions_reference.txt) 文件，这个文件里列出了所有可以写的函数名，从中挑出你的那个函数名，加入 [push-down-test/functions.txt](https://github.com/tikv/copr-test/blob/master/push-down-test/functions.txt)。

2.  假设 [copr-test](https://github.com/tikv/copr-test) 中提的 PR 是 #10，则在你之前提的 TiKV PR 中回复 `@sre-bot /run-integration-copr-test copr-test=pr/10` 运行下推测试。如果你的函数之前已经在 [push-down-test/functions.txt](https://github.com/tikv/copr-test/blob/master/push-down-test/functions.txt) 列表中了，可以直接回复 `@sre-bot /run-integration-copr-test` 运行下推测试。

当然，我们更推荐你能直接往 [copr-test](https://github.com/tikv/copr-test) 中添加人工编写的测试，更准确地覆盖边缘情况，具体方式参见 [copr-test](https://github.com/tikv/copr-test) 的 README。

### 7. 在 TiDB 中增添签名映射

如果上一步 copr-test 的测试挂了，一般来说有两种情况，一种情况是内置函数的实现有问题，被 copr-test 测了出来，另一种情况是你新实现的内置函数在 TiDB 侧还未建立函数签名与下推枚举签名 `ScalarFuncSig` 之间的映射关系。后者会在测试中产生 “unspecified PbCode” 错误，非常容易辨别。如果出现了这种情况，大家可以参考 [https://github.com/pingcap/tidb/pull/12864](https://github.com/pingcap/tidb/pull/12864) 的做法，为 TiDB 提 PR 增添相应内置函数的 PbCode 映射。添加完毕之后，可以在 TiKV PR 中回复 `@sre-bot /run-integration-copr-test copr-test=pr/X tidb=pr/Y`（其中 `X` 是你提的 copr-test PR 号，`Y` 是你提的 TiDB PR 号）进行联合测试。

## 完成！

至此，你新实现的内置函数有了单元测试，也有了与 TiDB 的集成下推测试，是一个合格的 PR 了，可以接受我们的 review。在 merge 后，你就能拿到相应的积分，积分可以在赛季结束后兑换 [TiDB 限量周边礼品](https://pingcap.com/community-cn/tidb-performance-challenge/)！

最后欢迎大家加入 [TiDB Community Slack Workspace](https://join.slack.com/t/tidbcommunity/shared_invite/enQtNzc0MzI4ODExMDc4LWYwYmIzMjZkYzJiNDUxMmZlN2FiMGJkZjAyMzQ5NGU0NGY0NzI3NTYwMjAyNGQ1N2I2ZjAxNzc1OGUwYWM0NzE) 和 [tikv-wg Slack Workspace](http://tikv.org/chat)，参赛过程中遇到任何问题都可以直接通过 **#performance-challenge-program** channel 与我们取得联系。