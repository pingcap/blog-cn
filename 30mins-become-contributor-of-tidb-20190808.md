---
title: 三十分钟成为 Contributor | 提升 TiDB Parser 对 MySQL 8.0 语法的兼容性
author: ['谢腾进','赵一霖']
date: 2019-08-08
summary: 本次活动聚焦于语法兼容，提升 TiDB SQL Parser 对 MySQL 8.0 的语法支持。对于新的贡献者而言，除了能将理论知识运用到实践上以外，还可以从中体验参与一个开源项目的整体流程与规范。
tags: ['TiDB','社区','Contributor']
---

TiDB 的一大特性就是和 MySQL 高度兼容，目标是让用户能够无需修改代码即可从 MySQL 迁移至 TiDB。要达成这个目标，需要完成两个提升兼容性的任务，分别是「语法兼容」和「功能行为兼容」。

**本次活动聚焦于语法兼容，提升 TiDB SQL Parser 对 MySQL 8.0 的语法支持。对于新的贡献者而言，除了能将理论知识运用到实践上以外，还可以从中体验参与一个开源项目的整体流程与规范。**

我们把 TiDB Parser 整体看作一个函数，输入是 SQL 字符串，输出是用于描述 SQL 语句的抽象语法树（AST）。在这个转换的过程中涉及到的组件有两个：一个是 lexer，负责将字符流变成 Token，并赋予它们类别标识，这个过程叫「Tokenize」；另一个是 parser，负责将 Token 转为树状结构，便于将来遍历和转换，这个过程叫「Parse」；TiDB 使用了 parser 生成工具 goyacc，它能够根据 `parser.y` 生成 `parser.go`，其中包含了名称为 `Parse` 的函数接口，供 TiDB 直接使用。更多关于 TiDB Parser 以及 Lex & Yacc 的信息请参考 [TiDB SQL Parser 的实现](https://www.pingcap.com/blog-cn/tidb-source-code-reading-5/)。

## 参与流程

参与流程分为 7 步：**领任务  -> 写 test case -> make test -> coding -> 补充 test case -> make test -> 提 PR**。

### 1. 从 Issue 领取任务

我们会在 [Improve parser compatibility](https://github.com/pingcap/tidb/issues/11486) 周期性发布不兼容的 Bad SQL Case 组，每组 Case 都会构成一个 Issue，Contributor 选择某个 Issue，在它的下方评论：**Let me fix it**。在我们将 Contributor 的 Github ID 添加到 Index Issue 中后，即完成任务的领取。

### 2. 编写测试用例

根据 Issue 描述的情况在 [`parser_test.go`](https://github.com/pingcap/parser/blob/master/parser_test.go) 中编写测试用例，除了 Issue 中提到的 Case 以外，可以适当添加更多的 Case。保证目标 SQL Case 语句能够通过 Parser 解析，并且通过 Restore 还原为预期的 SQL。

### 3. 执行所有测试

parser 根目录下运行 `make test`，确保第一次测试失败，并且失败的 Case 是第 2 步编写的。

### 4. 编码

Contributor 修改文法规则。对于涉及到语义层面的规则变动，需要同步修改 AST 节点的数据结构（AST 节点定义在 `parser/ast` 中）。TiDB 通过 AST 树生成执行计划，修改 AST 节点可能会影响 TiDB 的行为，因此应尽量保持原有结构，不改变原有的属性，如果确实有修改 AST 树必要，我们会帮助 Contributor 检查 TiDB 的行为是否正常。另外，AST 节点其中的两个接口方法是 `Accept` 和 `Restore`，分别用于遍历子树和通过 AST 树还原 SQL 字符串。应确保它们的行为都符合预期。

另外，还要检查新加的规则是否存在冲突问题。「冲突」可以被理解为当 parser 读到某个 token 时，有两种或以上的方式来构造语法树，从而导致歧义。goyacc 所生成的 parser 采用了 `LALR(1)` 方法进行解析，冲突有两类：一类是两条规则都能被用于归约，称为 `reduce-reduce` 冲突。另一类是既能使用一条规则归约，又能按照另一条规则移进下一个 token，称为 `shift-reduce` 冲突。可以通过指定优先级的方式消除冲突，具体可以参考 yacc 的 [%precedence 和 %prec 指示](https://www.gnu.org/software/bison/manual/html_node/Precedence.html#Precedence)。

编码完成后在项目根目录下运行 `make parser`，这时会执行 goyacc 重新生成 `parser.go` 文件。

### 5. 补充 test case

根据实际情况尽可能提升测试覆盖率。

### 6. 执行所有测试

parser 根目录下运行 `make test`，确保测试通过。

### 7. 提交 PR
	
提交 PR 之前请先阅读 [contributing 指南](https://github.com/pingcap/tidb/blob/master/CONTRIBUTING.md)。下面是 PR 的模板，逐项填写即可。

```	
### What problem does this PR solve?

#### [ Put the subtask title here ]

Issue: [ put the subtask issue link here ]

#### MySQL Syntax:

[ describe MySQL syntax here ]

#### Bad SQL Case:

[ give a SQL statement example that passes MySQL but fails TiDB parser ]

[ give a SQL statement example that passes MySQL but fails TiDB parser ]
	
...
	
### Check List
	
Tests

- Unit test
```

## 示例

**下面以添加 「REMOVE PARTITIONING」 语法支持为例解释说明整个过程**。

### 1. 从 Issue 领取任务

从 [这里](https://github.com/pingcap/tidb/issues/11486) 找到 `REMOVE PARTITIONING` 子任务。[子任务 Issue](https://github.com/pingcap/parser/issues/402) 中有若干不兼容的 Case。

首先，手动测试任一 Case，发现在 MySQL 下输出：

```
Error 1505: Partition management on a not partitioned table is not possible
```

而在 TiDB 下输出：

```
Error 1064: You have an error in your SQL syntax; check the manual that corresponds to your TiDB version for the right syntax to use line 1 column 20 near "remove partitioning"
```

这意味着 parser 无法识别 remove 关键字。

确认了问题存在后，到 [该 Issue](https://github.com/pingcap/parser/issues/402) 下评论：**Let me fix it**，完成任务领取。

### 2. 编写测试用例

在 `parser_test.go` 下寻找合适位置添加测试用例，这里我们选择在 `func (s *testParserSuite) TestDDL(c *C)` 的 [末尾](https://github.com/pingcap/parser/pull/396/files#diff-688912c3f38a8a80d6bdc16c02088d69R2172) 添加：

```
// for remove partitioning
{"alter table t remove partitioning", true, "ALTER TABLE `t` REMOVE PARTITIONING"},
{"alter table db.ident remove partitioning", true, "ALTER TABLE `db`.`ident` REMOVE PARTITIONING"},
{"alter table t lock = default remove partitioning", true, "ALTER TABLE `t` LOCK = DEFAULT REMOVE PARTITIONING"},
```

这里每一个 test case 分成了三部分，第一列是用于测试的 SQL 语句，第二列是「是否期望第一列的语句 parse 通过」，第三列是「从语法树 restore 后期望的 SQL 语句」。具体可以参考 [`TestDDL.RunTest()`](https://github.com/pingcap/parser/blob/53c769c5836485c83d5f339faab97ef5d853d560/parser_test.go#L308)。

### 3. 执行所有测试

parser 根目录下运行 `make test`，确保第一次测试失败，并且 fail 的 case 是第 2 步编写的。

```
FAIL: parser_test.go:1664: testParserSuite.TestDDL

parser_test.go:2148:
    s.RunTest(c, table)
parser_test.go:318:
    c.Assert(err, IsNil, comment)
... value *errors.withStack = line 1 column 20 near "remove partitioning"  ("line 1 column 20 near \"remove partitioning\" ")
... source alter table t remove partitioning
```

错误信息和期望的一致，则可以开始进行下一步。

### 4. 编码

#### 4.1 修改 `parser.y` 文件

首先从 [MySQL 文法](https://github.com/mysql/mysql-server/blob/8.0/sql/sql_yacc.yy) 中找到 remove partitioning 的定义：

```
alter_table_stmt: ALTER TABLE_SYM table_ident opt_alter_table_actions
| ALTER TABLE_SYM table_ident standalone_alter_table_action

opt_alter_table_actions: opt_alter_command_list
| opt_alter_command_list alter_table_partition_options

alter_table_partition_options: partition_clause
| REMOVE_SYM PARTITIONING_SYM
```

经过分析可得该语法只能出现在 SQL 语句的最后一个部分，并且只能出现一次。

在 `parser.y` 中找到 `AlterTableStmt`，其中一条规则是：

```
"ALTER" IgnoreOptional "TABLE" TableName AlterTableSpecListOpt PartitionOpt
```

其中最后一个符号 PartitionOpt 和 MySQL 中 `partition_clause` 非常相似，为了支持 remove partitioning，容易想到引入一条规则：

```
AlterTablePartitionOpt: PartitionOpt | "REMOVE" "PARTITIONING"
```

将 `PartitionOpt` 的语义动作移到 `AlterTablePartitionOpt` 中，`REMOVE PARTITIONING` 部分先返回 `nil`，并添加 parser 警告，表示目前 parser 能够解析但 TiDB 尚未支持该功能。修改后的规则如下：

```
AlterTablePartitionOpt:
	PartitionOpt
	{
		if $1 != nil {
			$$ = &ast.AlterTableSpec{
				Tp: ast.AlterTablePartition,
				Partition: $1.(*ast.PartitionOptions),
			}
		} else {
			$$ = nil
		}
	}
|	"REMOVE" "PARTITIONING"
	{
		$$ = nil
		yylex.AppendError(yylex.Errorf("The REMOVE PARTITIONING clause is parsed but ignored by all storage engines."))
		parser.lastErrorAsWarn()
	}
```

由于 `REMOVE` 和 `PARTITIONING` 都是新添加的关键字，如果不做任何处理，lexer 扫描的时候只会将它们看作普通的标识符。于是需要在 `parser.y` 的 `%token` 字段上补充声明，其中一个目的是为该字符串产生一个 `tokenID`（一个整数），供 lexer 标识。另外 `goyacc` 也会对 `parser.y` 中所有的字符串常量进行检查，如果没有相应的 `token` 声明，会报 `Undefined symbol` 的错误。

为支持这两个关键字，我们在文件开头的 `token` 字段添加声明。由于 `REMOVE` 和 `PARTITIONING` 都是非保留关键字，它们应被添加在含有 `The following tokens belong to UnReservedKeyword` [注释](https://github.com/pingcap/parser/blob/53c769c5836485c83d5f339faab97ef5d853d560/parser.y#L274) 的下方。此外，非保留字说明它们能够作为标识符 `Identifier`，因此在 `Identifier` 规则下的 [`UnRerservedKeyword`](https://github.com/pingcap/parser/blob/53c769c5836485c83d5f339faab97ef5d853d560/parser.y#L3717) 中也应加上 `REMOVE` 和 `PARTITIONING`。

关于如何确定一个关键字是保留的还是非保留的，可以参考 [MySQL 文档](https://dev.mysql.com/doc/refman/8.0/en/keywords.html)。

#### 4.2 增加「关键字-`tokenID`」映射

前文提到，添加声明是为了让 lexer 能够识别关键字并赋予对应的 `tokenID`，对于 lexer 而言，它需要一个从关键字字符串到 `tokenID` 的映射关系。在 TiDB parser 中，这个映射关系就是 [`misc.go`](https://github.com/pingcap/parser/blob/53c769c5836485c83d5f339faab97ef5d853d560/misc.go) 中的 `tokenMap` 结构。

在这个例子中，我们往 `tokenMap` 中添加 `remove` 和 `partitioning`（如果不添加，会使关键字一致性的检查测试失败）。

#### 4.3 修改 AST 节点

到目前为止，我们已经让 `goyacc` 生成的 parser 能够解析 remove partitioning 语法。但是，解析完成后并没有返回有效的数据结构（4.1 中我们返回了 `nil`），这意味着 parser 不能够根据语法树重新生成原 SQL 语句。

所以，要修改现有的 AST 节点，使它们能够以某种形式保存 remove partitioning 信息。回顾目前规则层面的修改：

```
AlterTableStmt:
"ALTER" IgnoreOptional "TABLE" TableName AlterTableSpecListOpt PartitionOpt
```

已改为：

```
AlterTableStmt:
"ALTER" IgnoreOptional "TABLE" TableName AlterTableSpecListOpt AlterTablePartitionOpt
AlterTablePartitionOpt:
      PartitionOpt | "REMOVE" "PARTITIONING"
```

其中几个非终结符对应的数据结构如下：

```
AlterTableSpecListOpt -> []AlterTableSpec
PartitionOpt -> PartitionOptions
```

关于修改节点，有几种方案可以选择：

1. 增加一个新的节点 struct，表示 `AlterTablePartitionOpt`。其中包含 `PartitionOptions` 和一个 bool 值，表示是否为 remove partitioning。

2. 将 remove partitioning 看作 `PartitionOptions`，在内部添加一个 bool 成员 `isRemovePartitioning` 以做区分。

3. 将 `PartitionOpt` 和 remove partitioning 都看作 `AlterTableSpec`，为 `AlterTableSpec` 的添加一个类型，单独表示 remove partitioning。

经过比较和分析，我们发现第一个方案会引入新的数据结构，有较大的概率会引起现有的 TiDB 测试不过，为此可能要修改 TiDB 方面的代码，工作量大的同时提高了 parser 的复杂度，因此作为备选方案；第二个方案没有引入新的数据结构，但是修改了现有的数据结构（加了个 bool 成员）；第三个方案即没有添加也没有修改数据结构，并且能够以较少的代码完成任务，作为首选方案。

**在以上的选择中，我们遵循「尽量不修改 AST 节点结构」的原则。**

按照方案三，观察 [`AlterTableSpec`](https://github.com/pingcap/parser/pull/396/files#diff-688d51c34d61e5d538b15582305c9a8dL1768)，其定义如下：

```
type AlterTableSpec struct {
  node
  ...
  Tp              AlterTableType
  Name            string
  ...
}
```

其中一个成员是 `Tp`，它所属的类型包含了 `AlterTable` 的许多操作，例如 `AddColumn`，`AddConstraint`，`DropColumn` 等。我们的任务是添加一个 [类似的 Type](https://github.com/pingcap/parser/pull/396/files#diff-688d51c34d61e5d538b15582305c9a8dR1708)，让它能够表示 remove partitioning。

到这里，解析完 SQL 语句生成的 AST 树已经包含 remove partitioning 的信息了。接下来要处理 Restore，让它能够从 AST 树还原出 SQL 语句。`AlterTableSpec` 的 Restore 十分简单，加一个 case 即可，这里不再赘述。

#### 4.4 完善 `parser.y`

第一次修改 `parser.y` 的时候我们在新加规则的语义动作中返回了 `nil`，原因是尚未确定 AST 是否需要修改，以及如何修改。而到这一步，这些都已经确定下来了，把 remove partitioning 看作 `AlterTableSpec` 类型：

```
|       "REMOVE" "PARTITIONING"
      {
              $$ = &ast.AlterTableSpec{
                     Tp: ast.AlterTableRemovePartitioning,
              }
             yylex.AppendError(yylex.Errorf("The REMOVE PARTITIONING clause is parsed but ignored by all storage engines."))
             parser.lastErrorAsWarn()
      }
```

注意，这里必须抛出警告，表示虽然目前 parser 能够解析该语法，但实际上 TiDB 并未支持相应的功能。

#### 5. 补充 test case

这里，所有的代码修改引入的分支结构都能够被现有的测试覆盖，因此在提升覆盖率上没有需求。当然，如果想要测试更多类似的 case，可以将它们添加到前面提到的 `TestDDL` 函数中。

#### 6. 执行所有测试（`make test`）

```
gofmt (simplify)
sh test.sh
ok      github.com/pingcap/parser       4.294s  coverage: 62.1% of statements in ./...
ok      github.com/pingcap/parser/ast   2.090s  coverage: 42.3% of statements in ./...
ok      github.com/pingcap/parser/auth  1.155s  coverage: 1.3% of statements in ./... [no tests to run]
ok      github.com/pingcap/parser/charset       1.094s  coverage: 2.0% of statements in ./...
ok      github.com/pingcap/parser/format        1.114s  coverage: 2.5% of statements in ./...
?       github.com/pingcap/parser/goyacc        [no test files]
ok      github.com/pingcap/parser/model 1.100s  coverage: 3.5% of statements in ./...
ok      github.com/pingcap/parser/mysql 1.102s  coverage: 1.3% of statements in ./... [no tests to run]
ok      github.com/pingcap/parser/opcode        1.099s  coverage: 1.4% of statements in ./...
ok      github.com/pingcap/parser/terror        1.091s  coverage: 2.3% of statements in ./...
ok      github.com/pingcap/parser/types 1.106s  coverage: 7.0% of statements in ./...

```

**确保所有的 test 都是 ok 的状态。**

#### 7. 提交 PR

按照 PR 模板逐项填写。

```
### What problem does this PR solve?

#### Fix compatibility problem about keyword REMOVE PARTITIONING

Issue: pingcap/parser#402

#### MySQL Syntax:

alter_specification:
...
  | REMOVE PARTITIONING

### Bad SQL Case:

alter table t remove partitioning;
alter table t lock = default, remove partitioning;
alter table t drop check c, remove partitioning;

### Check List

Tests
- Unit test

```

**需要特别指出的是，我们鼓励各位 Contributor 多使用 `make test`。当不知道从何处入手或者失去目标时，`make test` 输出的错误信息或许能够引导大家进行思考和探索**。

>Tips: [完整的 PR 示例](https://github.com/pingcap/parser/pull/396)

## FAQ

以下是在增加 remove partitioning 语法支持时遇到的问题和解决方法。

**Q1. 为什么不在 `PartitionOpt` 中直接添加规则？** 

**A1**：`PartitionOpt` 用于匹配含有 `partition by` 的 SQL 语句，除了 `Alter Table` 语句以外，它还被 `Create Table` 使用，而 `remove partitioning` 只存在于 `alter table` 语句中，因此不能在 `PartitionOpt` 中添加规则。

**Q2. 执行 make test 时报错：** 

```
parser.y:1100:1: undefined symbol "PARTITIONING"
parser.y:1100:1: undefined symbol "REMOVE"
make[1]: *** [Makefile:19: parser] Error 1
```

**A2**：在 yacc 中，出现在规则中的字符串，要么是 `token`（终结符），要么是非终结符。这里引用一段 yacc 的规范：

```
Names refer to either tokens or nonterminal symbols.
Names representing tokens must be declared; this is most simply done by writing
%token   name1 name2 . . .
```

所以，修复方法是在 `parser.y` 的 `%token` 字段上添加 `PARTITIONING` 和 `REMOVE` 的声明。

**Q3. 执行 make test 时报错：** 

```
 c.Assert(len(tokenMap)-len(aliases), Equals, keywordCount-len(windowFuncTokenMap))
... obtained int = 454
... expected int = 456
```

**A3**：这是关键字的一致性检查出了问题，解决方案是补充 `tokenMap`（它是关键字到 `token ID` 的映射，被 scanner 用来判断某个字符串是否为关键字）。

**Q4. 执行 make test 时报错：** 

```
FAIL: parser_test.go:1666: testParserSuite.TestDDL
parser_test.go:2166:
    s.RunTest(c, table)
parser_test.go:351:
    c.Assert(restoreSQLs, Equals, expectSQLs, comment)
... obtained string = "ALTER TABLE `t`"
... expected string = "ALTER TABLE `t` REMOVE PARTITIONING"
... restore ALTER TABLE `t`; expect ALTER TABLE `t` REMOVE PARTITIONING
```

**A4**：这个错误说明 parser 已经解析通过，但不能从语法树中恢复原 SQL 语句的 remove partitioning 部分。此时应检查相应 AST 节点的 Restore 方法是否正确处理了 `REMOVE PARTITIONING`。

***最后欢迎大家加入 [TiDB Contributor Club](https://pingcap.com/community-cn/)，无门槛参与开源项目，改变世界从这里开始吧！***
