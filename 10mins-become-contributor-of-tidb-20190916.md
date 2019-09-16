---
title: 十分钟成为 Contributor 系列 | 助力 TiDB 表达式计算性能提升 10 倍
author: ['Yuanjia Zhang']
date: 2019-09-16
summary: 最近我们扩展了 TiDB 表达式计算框架，增加了向量化计算接口，初期的性能测试显示，多数表达式计算性能可大幅提升，部分甚至可提升 1~2 个数量级。为了让所有的表达式都能受益，我们需要为所有内建函数实现向量化计算。
tags: ['TiDB','社区','Contributor']
---

最近我们扩展了 TiDB 表达式计算框架，增加了向量化计算接口，初期的性能测试显示，多数表达式计算性能可大幅提升，部分甚至可提升 1~2 个数量级。为了让所有的表达式都能受益，我们需要为所有内建函数实现向量化计算。

TiDB 的向量化计算是在经典 Volcano 模型上的进行改进，尽可能利用 CPU Cache，SIMD Instructions，Pipeline，Branch Predicatation 等硬件特性提升计算性能，同时降低执行框架的迭代开销，这里提供一些参考文献，供感兴趣的同学阅读和研究：

1.  [MonetDB/X100: Hyper-Pipelining Query Execution](http://cidrdb.org/cidr2005/papers/P19.pdf)

2.  [Balancing Vectorized Query Execution with Bandwidth-Optimized Storage](https://dare.uva.nl/search?identifier=5ccbb60a-38b8-4eeb-858a-e7735dd37487)

3.  [The Design and Implementation of Modern Column-Oriented Database Systems](https://www.nowpublishers.com/article/DownloadSummary/DBS-024)

在这篇文章中，我们将描述：

1.  如何在计算框架下实现某个函数的向量化计算；

2.  如何在测试框架下做正确性和性能测试；

3.  如何参与进来成为 TiDB Contributor。

## 表达式向量化

### 1. 如何访问和修改一个向量

在 TiDB 中，数据按列在内存中连续存在 Column 内，Column 详细介绍请看：[TiDB 源码阅读系列文章（十）Chunk 和执行框架简介](https://pingcap.com/blog-cn/tidb-source-code-reading-10/)。本文所指的向量，其数据正是存储在 Column 中。

我们把数据类型分为两种：

1.  定长类型：`Int64`、`Uint64`、`Float32`、`Float64`、`Decimal`、`Time`、`Duration`；

2.  变长类型：`String`、`Bytes`、`JSON`、`Set`、`Enum`。

定长类型和变长类型数据在 Column 中有不同的组织方式，这使得他们有如下的特点：

1.  定长类型的 Column 可以随机读写任意元素；

2.  变长类型的 Column 可以随机读，但更改中间某元素后，可能需要移动该元素后续所有元素，导致随机写性能很差。

对于定长类型（如 `int64`），我们在计算时会将其转成 Golang Slice（如 `[]int64`），然后直接读写这个 Slice。相比于调用 Column 的接口，需要的 CPU 指令更少，性能更好。同时，转换后的 Slice 仍然引用着 Column 中的内存，修改后不用将数据从 Slice 拷贝到 Column 中，开销降到了最低。

对于变长类型，元素长度不固定，且为了保证元素在内存中连续存放，所以不能直接用 Slice 的方式随机读写。我们规定变长类型数据以追加写（`append`）的方式更新，用 Column 的 `Get()` 接口进行读取。

总的来说，变长和定长类型的读写方式如下：

1. 定长类型（以 `int64` 为例)

    a. `ResizeInt64s(size, isNull)`：预分配 size 个元素的空间，并把所有位置的 `null` 标记都设置为 `isNull`；
    
    b.  `Int64s()`：返回一个 `[]int64` 的 Slice，用于直接读写数据；
    
    c.  `SetNull(rowID, isNull)`：标记第 `rowID` 行为 `isNull`。

2. 变长类型（以 `string` 为例）
    
    a. `ReserveString(size)`：预估 size 个元素的空间，并预先分配内存；
    
    b. `AppendString(string)`: 追加一个 string 到向量末尾；
    
    c.  `AppendNull()`：追加一个 `null` 到向量末尾；
    
    d.  `GetString(rowID)`：读取下标为 `rowID` 的 string 数据。

当然还有些其他的方法如 `IsNull(rowID)`，`MergeNulls(cols)` 等，就交给大家自己去探索了，后面会有这些方法的使用例子。

### 2. 表达式向量化计算框架

向量化的计算接口大概如下（[完整的定义在这里](https://github.com/pingcap/tidb/blob/master/expression/builtin.go#L340)）：

```
vectorized() bool
vecEvalXType(input *Chunk, result *Column) error
```

*   `XType` 可能表示 `Int`, `String` 等，不同的函数需要实现不同的接口；

*   `input` 表示输入数据，类型为 `*Chunk`；

*   `result` 用来存放结果数据。

外部执行算子（如 Projection，Selection 等算子），在调用表达式接口进行计算前，会通过 `vectorized()` 来判断此表达式是否支持向量化计算，如果支持，则调用向量化接口，否则就走行式接口。

对于任意表达式，只有当其中所有函数都支持向量化后，才认为这个表达式是支持向量化的。

比如 `(2+6)*3`，只有当 `MultiplyInt` 和 `PlusInt` 函数都向量化后，它才能被向量化执行。

## 为函数实现向量化接口

要实现函数向量化，还需要为其实现 `vecEvalXType()` 和 `vectorized()` 接口。

* 在 `vectorized()` 接口中返回 `true` ，表示该函数已经实现向量化计算；

* 在 `vecEvalXType()` 实现此函数的计算逻辑。

**尚未向量化的函数在 [issue/12058](https://github.com/pingcap/tidb/issues/12058) 中，欢迎感兴趣的同学加入我们一起完成这项宏大的工程。**

向量化代码需放到以 `_vec.go` 结尾的文件中，如果还没有这样的文件，欢迎新建一个，注意在文件头部加上 licence 说明。

这里是一个简单的例子 [PR/12012](https://github.com/pingcap/tidb/pull/12012)，以 `builtinLog10Sig` 为例：

1.  这个函数在 `expression/builtin_math.go` 文件中，则向量化实现需放到文件 `expression/builtin_math_vec.go` 中；

2.  `builtinLog10Sig` 原始的非向量化计算接口为 `evalReal()`，那么我们需要为其实现对应的向量化接口为 `vecEvalReal()`；

3.  实现完成后请根据后续的说明添加测试。

下面为大家介绍在实现向量化计算过程中需要注意的问题。

### 1. 如何获取和释放中间结果向量

存储表达式计算中间结果的向量可通过表达式内部对象 `bufAllocator` 的 `get()` 和 `put()` 来获取和释放，参考 [PR/12014](https://github.com/pingcap/tidb/pull/12014)，以 `builtinRepeatSig` 的向量化实现为例：

```
buf2, err := b.bufAllocator.get(types.ETInt, n)
if err != nil {
    return err
}
defer b.bufAllocator.put(buf2) // 注意释放之前申请的内存
```

### 2. 如何更新定长类型的结果

如前文所说，我们需要使用 `ResizeXType()` 和 `XTypes()` 来初始化和获取用于存储定长类型数据的 Golang Slice，直接读写这个 Slice 来完成数据操作，另外也可以使用 `SetNull()` 来设置某个元素为 `NULL`。代码参考 [PR/12012](https://github.com/pingcap/tidb/pull/12012)，以 `builtinLog10Sig` 的向量化实现为例：

```
f64s := result.Float64s()
for i := 0; i < n; i++ {
    if isNull {
        result.SetNull(i, true)
    } else {
        f64s[i] = math.Log10(f64s[i])
    }
}
```

### 3. 如何更新变长类型的结果

如前文所说，我们需要使用 `ReserveXType()` 来为变长类型预分配一段内存（降低 Golang runtime.growslice() 的开销），使用 `AppendXType()` 来追加一个变长类型的元素，使用 `GetXType()` 来读取一个变长类型的元素。代码参考 [PR/12014](https://github.com/pingcap/tidb/pull/12014)，以 `builtinRepeatSig` 的向量化实现为例：

```
result.ReserveString(n)
...
for i := 0; i < n; i++ {
    str := buf.GetString(i)
    if isNull {
        result.AppendNull()
    } else {
    result.AppendString(strings.Repeat(str, int(num)))
    }
}
```

### 4. 如何处理 Error

所有受 SQL Mode 控制的 Error，都利用对应的错误处理函数在函数内就地处理。部分 Error 可能会被转换成 Warn 而不需要立即抛出。

这个比较杂，需要查看对应的非向量化接口了解具体行为。代码参考 [PR/12042](https://github.com/pingcap/tidb/pull/12042)，以 `builtinCastIntAsDurationSig` 的向量化实现为例：

```
for i := 0; i < n; i++ {
    ...
    dur, err := types.NumberToDuration(i64s[i], int8(b.tp.Decimal))
    if err != nil {
       if types.ErrOverflow.Equal(err) {
          err = b.ctx.GetSessionVars().StmtCtx.HandleOverflow(err, err) // 就地利用对应处理函数处理错误
       }
       if err != nil { // 如果处理不掉就抛出
          return err
       }
       result.SetNull(i, true)
       continue
    }
    ...
}
```

### 5. 如何添加测试

我们做了一个简易的测试框架，可避免大家测试时做一些重复工作。

该测试框架的代码在 `expression/bench_test.go` 文件中，被实现在 `testVectorizedBuiltinFunc` 和 `benchmarkVectorizedBuiltinFunc` 两个函数中。

我们为每一个 `builtin_XX_vec.go` 文件增加了 `builtin_XX_vec_test.go` 测试文件。当我们为一个函数实现向量化后，需要在对应测试文件内的 `vecBuiltinXXCases` 变量中，增加一个或多个测试 case。下面我们为 log10 添加一个测试 case：

```
var vecBuiltinMathCases = map[string][]vecExprBenchCase {
    ast.Log10: {
        {types.ETReal, []types.EvalType{types.ETReal}, nil},
    },
}
```

具体来说，上面结构体中的三个字段分别表示:

1.  该函数的返回值类型；

2.  该函数所有参数的类型；

3.  是否使用自定义的数据生成方法（dataGener），`nil` 表示使用默认的随机生成方法。

对于某些复杂的函数，你可自己实现 dataGener 来生成数据。目前我们已经实现了几个简单的 dataGener，代码在 `expression/bench_test.go` 中，可直接使用。

添加好 case 后，在 expression 目录下运行测试指令：

```
# 功能测试
GO111MODULE=on go test -check.f TestVectorizedBuiltinMathFunc

# 性能测试
go test -v -benchmem -bench=BenchmarkVectorizedBuiltinMathFunc -run=BenchmarkVectorizedBuiltinMathFunc
```

在你的 PR Description 中，请把性能测试结果附上。不同配置的机器，性能测试结果可能不同，我们对机器配置无任何要求，你只需在 PR 中带上你本地机器的测试结果，让我们对向量化前后的性能有一个对比即可。

## 如何成为 Contributor

**为了推进表达式向量化计算，我们正式成立 Vectorized Expression Working Group，其具体的目标和制度详见[这里](https://github.com/pingcap/community/blob/master/working-groups/wg-vec-expr.md)。与此对应，我们在 [TiDB Community Slack](https://pingcap.com/tidbslack/) 中创建了 [wg-vec-expr channel](https://app.slack.com/client/TH91JCS4W/CMRD79DRR) 供大家交流讨论，不设门槛，欢迎感兴趣的同学加入。**

如何成为 Contributor：

1.  在此 [issue](https://github.com/pingcap/tidb/issues/12058) 内选择感兴趣的函数并告诉大家你会完成它；

2.  为该函数实现 `vecEvalXType()` 和 `vectorized()` 的方法；

3.  在向量化测试框架内添加对该函数的测试；

4.  运行 `make dev`，保证所有 test 都能通过；

5.  发起 Pull Request 并完成 merge 到主分支。

如果贡献突出，可能被提名为 reviewer，reviewer 的介绍请看 [这里](https://github.com/pingcap/community/blob/master/CONTRIBUTING.md#reviewer)。

如果你有任何疑问，也欢迎到 wg-vec-expr channel 中提问和讨论。
