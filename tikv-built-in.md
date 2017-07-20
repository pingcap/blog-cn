---
title: 十分钟成为 Contributor 系列 | 为 TiKV 添加 built-in 函数
author: 温文鎏
date: 2017-07-20
summary: SQL 语句发送到 TiDB 后经过 parser 生成 AST （抽象语法树）, 再经过 Query Optimizer 生成执行计划，执行计划切分成很多子任务，这些子任务以表达式的方式最后下推到底层的各个 TiKV 来执行。
tags: TiKV Contributor
---

##背景知识

SQL 语句发送到 TiDB 后经过 parser 生成 AST （抽象语法树）, 再经过 Query Optimizer 生成执行计划，执行计划切分成很多子任务，这些子任务以表达式的方式最后下推到底层的各个 TiKV 来执行。

在此之前 TiDB 只会向 TiKV 下推一小部分简单的表达式，比如取出某一个列的某个数据类型的值，简单数据类型的比较操作，算术运算等。为了充分利用分布式集群的资源，进一步提升 SQL 在整个集群的执行速度，后面需要将更多种类的表达式下推到 TiKV 来运行，其中的一大类就是 MySQL built-in 函数。

在 TiDB 与 TiKV 的下推框架中，每个 built-in 函数被认为是一个表达式。由于 TiDB 在其类型推导阶段已经推导出每个 built-in 函数调用参数的参数类型，这个信息可以用来帮助实现下推，因此每个 built-in 函数根据调用时的函数名字和参数类型，生成一个函数签名，通过将函数签名与参数下发到 TiKV 进行求值。

总体而言，上述流程对于不熟悉 TiDB 与 TiKV 下推框架的朋友来说比较复杂，我们对这部分做了一些工作，将其中一部分流程性，较为繁琐的工作做了统一处理，目前已经将大多数未实现的 built-in 函数的函数签名及下推的接口部分定义好，但是函数实现部分留空。

换句话说，只要找到留空的函数实现，将其补充完整，即可作为一个 PR。

##添加 built-in 函数整体流程

1.找到未实现的函数，在 TiKV 源码的 coprocessor/xeval 目录下搜索 ERROR\_UNIMPLEMENTED, 即可找到所有未实现的函数，从中选择一个感兴趣的函数，比如 abs_int 函数：

```
pub fn abs_int(&mut self, ctx: &EvalContext, expr: &Expr) -> Result<Datum> {
    // TODO add impl
    return Err(Error::Eval(ERROR_UNIMPLEMENTED.to_owned()))
}
```

这个函数对应的函数签名是 AbsInt ，即用 int 参数来调用 MySQL 的 built-in abs() 函数。

2.实现函数签名
	
接下来要做的事情就是实现这个 abs_int 函数，函数的功能请参考 MySQL 文档，在这个例子中，abs() 函数的功能是取绝对值。int 数据类型包括有符号和无符号两种，在 Datum 的定义(coprocessor/codec/mysql/datum.rs)中，我们可以看到
	
```
#[derive(PartialEq, Clone)]
pub enum Datum {
    Null,
    I64(i64),
    U64(u64),
    F64(f64),
    Dur(Duration),
    Bytes(Vec<u8>),
    Dec(Decimal),
    Time(Time),
    Json(Json),
    Min,
    Max,
}
```

其中 I64 和 U64 是 int 类型。实现 abs 只需要对负值的 I64 转成正值。具体代码如下

```
	pub fn abs_int(&mut self, ctx: &EvalContext, expr: &Expr) -> Result<Datum> {
        let child = try!(self.get_one_child(expr));
        let d = try!(self.eval(ctx, child));
        match d {
            Datum::I64(i) => {
                if i >= 0 {
                    Ok(Datum::I64(i))
                } else {
                    Ok(Datum::I64(-i))
                }
            }
            Datum::U64(_) => Ok(d),
            _ => invalid_type_error(&d, TYPE_INT),
        }
    }
```

3.写单元测试
	
在同一个文件中，添加对 abs_int 这个函数的单元测试。如下
	
```
	test_eval!(test_abs_int,
               vec![(build_expr_with_sig(vec![Datum::I64(-1)],
                                         ExprType::ScalarFunc,
                                         ScalarFuncSig::AbsInt),
                     Datum::I64(1)),
                    (build_expr_with_sig(vec![Datum::I64(1)],
                                         ExprType::ScalarFunc,
                                         ScalarFuncSig::AbsInt),
                     Datum::I64(1)),
                    (build_expr_with_sig(vec![Datum::U64(1)],
                                         ExprType::ScalarFunc,
                                         ScalarFuncSig::AbsInt),
                     Datum::U64(1))]);
```

其中 test\_eval 是一个宏，第一个参数 test\_abs_int 是要生成的函数名字，后面的第二个参数是测试用例的数据。上述例子的所有代码可以参考这个 PR [https://github.com/pingcap/tikv/pull/2033](https://github.com/pingcap/tikv/pull/2033)

4.运行 make dev，确保所有的 test case 都能跑过

完成以上几个步骤之后，就可以给 TiKV 项目提 PR 啦，是不是很简单呢？欢迎大家有空多给 TiKV 贡献代码。




