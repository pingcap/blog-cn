---
title: 十分钟成为 Contributor 系列 | 重构内建函数进度报告
author: ['徐怀宇']
date: 2017-07-14
summary: 为了方便社区同学更好地参与 TiDB 项目，本文一方面对继上一篇文章发布后参考社区的反馈对表达式计算框架所做的修改进行详细介绍，另一方面对尚未重写的 built-in 函数进行陈列。
tags: ['TiDB', 'Contributor','社区']
---

6 月 22 日，TiDB 发布了一篇如何十分钟成为 TiDB Contributor 系列的[第二篇文章](./reconstruct-built-in-function.md)，向大家介绍如何为 TiDB 重构 built-in 函数。

截止到目前，得到了来自社区的积极支持与热情反馈，TiDB 参考社区 contributors 的建议，对计算框架进行了部分修改以降低社区同学参与的难度。

本文完成以下**2 项工作**，希望帮助社区更好的参与进 TiDB 的项目中来:

1. 对尚未重写的 built-in 函数进行陈列
2. 对继上篇文章后，计算框架所进行的修改，进行详细介绍

### 一. 尚未重写的 built-in 函数陈列如下：

共计 165 个
在 expression 目录下运行 `grep -rn "^\tbaseBuiltinFunc$" -B 1 * | grep "Sig struct {" | awk -F "Sig" '{print $1}' | awk -F "builtin" '{print $3}' > ~/Desktop/func.txt` 命令可以获得所有未实现的 built-in 函数

|       0       |            1             |       2       |        3        | 4 |
|:-------------:|:------------------------:|:-------------:|:---------------:|:-:|
|   Coalesce    |        Uncompress        |     Log10     |     Default     | UnaryOp |
|   Greatest    |    UncompressedLength    |     Rand      |    InetAton     | IsNull |
|     Least     | ValidatePasswordStrength |      Pow      |    InetNtoa     | In |
|   Interval    |         Database         |     Round     |    Inet6Aton    | Row |
|   CaseWhen    |        FoundRows         |     Conv      |    Inet6Ntoa    | SetVar |
|      If       |       CurrentUser        |     CRC32     |   IsFreeLock    | GetVar |
|    IfNull     |           User           |     Sqrt      |     IsIPv4      | Values |
|    NullIf     |       ConnectionID       |  Arithmetic   | IsIPv4Prefixed  | BitCount |
|  AesDecrypt   |       LastInsertID       |     Acos      |     IsIPv6      | Reverse |
|  AesEncrypt   |         Version          |     Asin      |   IsUsedLock    | Convert |
|   Compress    |        Benchmark         |     Atan      |  MasterPosWait  | Substring |
|    Decode     |         Charset          |      Cot      |    NameConst    | SubstringIndex |
|  DesDecrypt   |       Coercibility       |      Exp      | ReleaseAllLocks | Locate |
|  DesEncrypt   |        Collation         |      PI       |      UUID       | Hex |
|    Encode     |         RowCount         |    Radians    |    UUIDShort    | UnHex |
|    Encrypt    |          Regexp          |   Truncate    |     AndAnd      | Trim |
|  OldPassword  |           Abs            |     Sleep     |      OrOr       | LTrim |
|  RandomBytes  |           Ceil           |     Lock      |    LogicXor     | RTrim |
|     SHA1      |          Floor           |  ReleaseLock  |      BitOp      | Rpad |
|     SHA2      |           Log            |   AnyValue    |    IsTrueOp     | BitLength |
|     Char      |          Format          |   FromDays    |    DayOfWeek    | Timestamp |
|  CharLength   |        FromBase64        |     Hour      |    DayOfYear    | AddTime |
|   FindInSet   |        InsertFunc        |    Minute     |      Week       | ConvertTz |
|     Field     |          Instr           |    Second     |     WeekDay     | MakeTime |
|    MakeSet    |         LoadFile         |  MicroSecond  |   WeekOfYear    | PeriodAdd |
|      Oct      |           Lpad           |     Month     |      Year       | PeriodDiff |
|     Quote     |           Date           |   MonthName   |    YearWeek     | Quarter |
|      Bin      |         DateDiff         |      Now      |  FromUnixTime   | SecToTime |
|      Elt      |         TimeDiff         |    DayName    |    GetFormat    | SubTime |
|   ExportSet   |        DateFormat        |  DayOfMonth   |    StrToDate    | TimeFormat |
|    UTCTim     |        ToSeconds         | TimestampDiff |    DateArith    | Extract |
| UnixTimestamp |       UTCTimestamp       |    UTCDate    |      Time       | CurrentTime |
|    ToDays     |       TimestampAdd       |   TimeToSec   |   CurrentDate   | SysDate |

### 二. 计算框架进行的修改:

此处依然使用 Length 函数( expression/builtin_string.go )为例进行说明，与前文采取相同目录结构:

**1. expression/builtin_string.go**

（1）`lengthFunctionClass.getFunction()` 方法: **简化类型推导实现**

getFunction 方法用来生成 built-in 函数对应的函数签名，在[构造 ScalarFunction](https://github.com/pingcap/tidb/blob/master/expression/scalar_function.go#L76) 时被调用

``` go
func (c *lengthFunctionClass) getFunction(args []Expression, ctx context.Context) (builtinFunc, error) {
    // 此处简化类型推导过程，对 newBaseBuiltinFuncWithTp() 实现进行修改，新的实现中，传入 Length 返回值类型 tpInt 表示返回值类型为 int，参数类型 tpString 表示返回值类型为 string
  bf, err := newBaseBuiltinFuncWithTp(args, ctx, tpInt, tpString)
  if err != nil {
    return nil, errors.Trace(err)
  }
  // 此处参考 MySQL 实现，设置返回值长度为 10(character length)
  // 对于 int/double/decimal/time/duration 类型返回值，已在 newBaseBuiltinFuncWithTp() 中默认调用 types.setBinChsClnFlag() 方法，此处无需再进行设置
  bf.tp.Flen = 10
  sig := &builtinLengthSig{baseIntBuiltinFunc{bf}}
  return sig.setSelf(sig), errors.Trace(c.verifyArgs(args))
}
```

***注：***

- 对于**返回值类型为 string** 的函数，需要，注意参考 MySQL 行为设置

    `bf.tp.[charset | collate | flag]`
查看 MySQL 行为可以通过在终端启动

    ` $ mysql -uroot \-\-column-type-info`，这样对于每一个查询语句，可以查看每一列详细的 metadata
对于返回值类型为 string 的函数，以 [concat](https://github.com/pingcap/tidb/blob/master/expression/builtin_string.go#L204) 为例，当存在类型为 string 且包含 binary flag 的参数时，其返回值也应设置 binary flag

- 对于**返回值类型为 Time** 的函数，需要注意，根据函数行为，设置

    `bf.tp.Tp = [ TypeDate | TypeDatetime | TypeTimestamp ]` ，
  若为 `TypeDate/ TypeDatetime`，还需注意推导 `bf.tp.Decimal` (即小数位数)

- 不确定性的函数：

|   0    |      1       |      2       |      3      |     4      |    5     |
|:------:|:------------:|:------------:|:-----------:|:----------:|:--------:|
|  Rand  | ConnectionID | CurrentUser  |    User     |  Database  | RowCount |
| Schema |  FoundRows   | LastInsertId |   Version   |   Sleep    |   UUID   |
| GetVar |    SetVar    |    Values    | SessionUser | SystemUser |          |

（2）实现 `builtinLengthSig.evalInt()` 方法：保持不变，此处请注意修改该函数的注释 (s/ eval/ evalXXX)

**2. expression/builtin_string_test.go**

```go
func (s *testEvaluatorSuite) TestLength(c *C) {
   defer testleak.AfterTest(c)()
   cases := []struct {
      args     interface{}
      expected int64
      isNil    bool
      getErr   bool
   }{
     ......
   }

   for _, t := range cases {
      f, err := newFunctionForTest(s.ctx, ast.Length, primitiveValsToConstants([]interface{}{t.args})...)
      c.Assert(err, IsNil)
      d, err := f.Eval(nil)
     // 注意此处不再对 LENGTH 函数的返回值类型进行测试，相应测试被移动到 plan/typeinfer_test.go/TestInferType 函数中，(注意不是expression/typeinferer_test.go）
      if t.getErr {
         c.Assert(err, NotNil)
      } else {
         c.Assert(err, IsNil)
         if t.isNil {
            c.Assert(d.Kind(), Equals, types.KindNull)
         } else {
            c.Assert(d.GetInt64(), Equals, t.expected)
         }
      }
   }

   // 测试函数是否具有确定性
   // 在 review 社区的 PRs 过程中发现，这个测试经常会被遗漏，烦请留意
   f, err := funcs[ast.Length].getFunction([]Expression{Zero}, s.ctx)
   c.Assert(err, IsNil)
   c.Assert(f.isDeterministic(), IsTrue)
}
```

**3. executor/executor_test.go**

与上一篇文章保持不变，需要注意的是，为了保证可读性， `TestStringBuiltin()` 方法仅对 `expression/builtin_string.go` 文件中的 built-in 函数进行测试。如果 `executor_test.go` 文件中不存在对应的 `TestXXXBuiltin()` 方法，可以新建一个对应的测试函数。

**4. plan/typeinfer_test.go**

```go
func (s *testPlanSuite) TestInferType(c *C) {
  ....
   tests := []struct {
      sql     string
      tp      byte
      chs     string
      flag    byte
      flen    int
      decimal int
   }{
     ...
     // 此处添加对 length 函数返回值类型的测试
     // 此处注意，对于返回值类型、长度等受参数影响的函数，此处测试请尽量覆盖全面
      {"length(c_char, c_char)", mysql.TypeLonglong, charset.CharsetBin, mysql.BinaryFlag, 10, 0},
     ...
   }
   for _, tt := range tests {
      ...
   }
}
```

***注：***

当有多个 PR 同时在该文件中添加测试时，若有别的 contributor 的 PR 先于自己的 PR merge 进 master，有可能会发生冲突，此时在本地 merge 一下 master 分支，解决一下再 push 一下即可。

----------------

**成为 New Contributor 赠送限量版马克杯**的活动还在继续中，任何一个新加入集体的小伙伴都将收到我们充满了诚意的礼物，很荣幸能够认识你，也很高兴能和你一起坚定地走得更远。

#### 成为 New Contributor 获赠限量版马克杯，马克杯获取流程如下：

1. 提交 PR
2. PR提交之后，请耐心等待维护者进行 Review。
目前一般在一到两个工作日内都会进行 Review，如果当前的 PR 堆积数量较多可能回复会比较慢。
代码提交后 CI 会执行我们内部的测试，你需要保证所有的单元测试是可以通过的。期间可能有其它的提交会与当前 PR 冲突，这时需要修复冲突。
维护者在 Review 过程中可能会提出一些修改意见。修改完成之后如果 reviewer 认为没问题了，你会收到 LGTM(looks good to me) 的回复。当收到两个及以上的 LGTM 后，该 PR 将会被合并。
3. 合并 PR 后自动成为 Contributor，会收到来自 PingCAP Team 的感谢邮件，请查收邮件并填写领取表单

 - 表单填写地址：[http://cn.mikecrm.com/01wE8tX](http://cn.mikecrm.com/01wE8tX)

4. 后台 AI 核查 GitHub ID 及资料信息，确认无误后随即便快递寄出属于你的限量版马克杯
5. 期待你分享自己参与开源项目的感想和经验，TiDB Contributor Club 将和你一起分享开源的力量


了解更多关于 TiDB 的资料请登陆我们的官方网站：[https://pingcap.com](https://pingcap.com)

加入 TiDB Contributor Club 请添加我们的 AI 微信：

![](media/tidb-robot.jpg "tidb_rpbot")
