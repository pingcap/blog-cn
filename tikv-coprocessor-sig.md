---
title: TiKV 项目首个 SIG 成立，一起走上 Contributor 进阶之路吧！ 
author: ['Long Heng']
date: 2019-10-24
summary: 今天是 1024 程序员节，我们正式成立 TiKV 项目的首个 SIG —— Coprocessor SIG，希望对 TiKV 项目感兴趣的小伙伴们都能加入进来，探索硬核的前沿技术，交流切磋，一起走上 Contributor 的进阶之路！
image: /images/blog-cn/tikv-coprocessor-sig/1.jpg
tags: ['社区','社区动态']
---

社区是一个开源项目的灵魂，随着 TiDB/TiKV [新的社区架构升级](https://pingcap.com/blog-cn/tidb-community-upgrade/)， TiKV 社区也计划逐步成立更多个 Special Interest Group（SIG ）吸引更多社区力量，一起来改进和完善 TiKV 项目。SIG  将围绕着特定的模块进行开发和维护工作，并对该模块代码的质量负责。

今天是 1024 程序员节，我们正式成立 TiKV 项目的首个 SIG —— Coprocessor SIG，希望对 [TiKV 项目](https://github.com/tikv/tikv) 感兴趣的小伙伴们都能加入进来，探索硬核的前沿技术，交流切磋，一起走上 Contributor 的进阶之路！

## Coprocessor 模块是什么？

为了提升数据库的整体性能，TiDB 会将部分计算下推到 TiKV 执行，即 TiKV 的 Coprocessor 模块。本次成立的 Coprocessor SIG 就聚焦在 TiKV 项目 Coprocessor 模块。本 SIG 的主要职责是对 Coprocessor 模块进行未来发展的讨论、规划、开发和维护。

## 如何加入 Coprocessor SIG？

**社区的 Reviewer 或更高级的贡献者（Committer，Maintainer）将提名 Active Contributor 加入 Coprocessor SIG。Active Contributor 是对于 TiKV Coprocessor 模块或者 TiKV 项目有浓厚兴趣的贡献者，在过去 1 年为 TiKV 项目贡献超过 8 个 PR。**

加入 SIG 后，Coprocessor SIG Tech Lead 将指导成员完成目标任务。在此过程中，成员可以从 Active Contributor 逐步晋升为 Reviewer、Committer 角色，解锁更多角色权利&义务。

+ Reviewer：从 Active Contributor 中诞生，当 Active Contributor 对 Coprocessor 模块拥有比较深度的贡献，并且得到 2 个或 2 个以上 Committer 的提名时，将被邀请成为该模块的 Reviewer，主要权利&义务：
    - 参与 Coprocessor PR Review 与质量控制；
    - 对 Coprocessor 模块 PR 具有有效的 Approve / Request Change 权限；
    - 参与项目设计决策。
+ Committer：资深的社区开发者，从 Reviewer 中诞生。当 Reviewer 对 Coprocessor  模块拥有非常深度的贡献，或者在保持 Coprocessor  模块 Reviewer 角色的同时，也在别的模块深度贡献成为了 Reviewer，这时他就在深度或者广度上具备了成为 Committer 的条件，只要再得到 2 个或 2 个以上 Maintainer 的提名时，即可成为 Committer，主要权利及义务：
    - 拥有 Reviewer 具有的权利与义务；
    - 整体把控项目的代码质量；
    - 指导 Contributor 与 Reviewer。

## 工作内容有哪些？

1. 完善测试

	* 为了进一步提高 Coprocessor 的集成测试覆盖率，TiKV 社区开源了 copr-test 集成测试框架（github.com/tikv/copr-test），便于社区为 Coprocessor 添加更多集成测试；
	
	* 从 TiDB port 的函数需要同时 port 单元测试，如果 TiDB 的单元测试没有覆盖所有的分支，需要补全单元测试；
	
	* Expression 的集成测试需要构造使用这个 Expression 的算子进行测试。

2. 提升代码质量

	* Framework: 计算框架改进，包括表达式计算框架、算子执行框架等；
	
	* Executor: 改进现有算子、与 TiDB 协作研发新算子；
	
	* Function: 维护现有的 UDF / AggrFn 实现或从  TiDB port 新的 UDF / AggrFn 实现；
	
	* 代码位置：[https://github.com/tikv/tikv/tree/master/components/tidb_query](https://github.com/tikv/tikv/tree/master/components/tidb_query)

3. 设计与演进 Proposal

4. Review 相关项目代码

## 如何协同工作？

1. 为了协同效率，我们要求 SIG 成员遵守一致的代码风格、提交规范、PR Description 等规定。具体请参考 [文档](https://github.com/tikv/tikv/blob/master/CONTRIBUTING.md)。


2. 任务分配方式

	* SIG Tech Lead 在 github.com/tikv/community 维护公开的成员列表与任务列表链接；
	
	* 新加入的 SIG 成员可有 2 周时间了解各个任务详情并认领一个任务，或参与一个现有任务的开发或推动。若未能在该时间内认领任务则会被移除 SIG；
	
	* SIG 成员需维持每个月参与开发任务，或参与关于现有功能或未来规划的设计与讨论。若连续一个季度不参与开发与讨论，视为不活跃状态，将会被移除 SIG。作为 acknowledgment，仍会处于成员列表的「Former Member」中。

3. 定期同步进度，定期周会

	* 每 2 周以文档形式同步一次当前各个项目的开发进度；
	
	* 每 2 周召开一次全组进度会议，时间依据参会人员可用时间另行协商。目前没有项目正在开发的成员可选择性参加以便了解各个项目进度。若参与开发的成员不能参加，需提前请假且提前将自己的月度进度更新至文档；
	
	* 每次会议由一名成员进行会议记录，在会议结束 24 小时内完成会议记录并公开。会议记录由小组成员轮流执行；
	
	*  Slack：[https://tikv-wg.slack.com](https://tikv-wg.slack.com/join/shared_invite/enQtNTUyODE4ODU2MzI0LWVlMWMzMDkyNWE5ZjY1ODAzMWUwZGVhNGNhYTc3MzJhYWE0Y2FjYjliYzY1OWJlYTc4OWVjZWM1NDkwN2QxNDE)（Channel #copr-sig-china）
	
4. 通过更多线上、线下成员的活动进行交流合作。

## Coprocessor SIG 运营制度

1. 考核 & 晋升制度

	a. Coprocessor SIG Tech Lead 以月为单位对小组成员进行考核，决定成员是否可由 Active Contributor 晋升为 Reviewer：
	
    + 熟悉代码库；
    + 获得至少 2 位 TiKV Committer 的提名；
    + PR 贡献满足以下任意一点：
        - Merge Coprocessor PR 总数超过 10 个；
        - Merge Coprocessor PR 总行数超过 1000 行；
        - 已完成一项难度为 Medium 或以上的任务；
        - 提出设计想法并得到采纳成为可执行任务超过 3 个。
	
	b. Coprocessor SIG Tech Lead 和 TiKV Maintainer 以季度为单位对小组成员进行考核，决定成员是否可由 Reviewer 晋升为 Committer：
	
	+ 表现出良好的技术判断力；
	+ 在 TiKV / PingCAP 至少两个子项目中是 Reviewer；
	+ 获得至少 2 位 TiKV Maintainer 的提名；
	+ 至少完成两项难度为 Medium 的任务，或一项难度为 High 的任务；
	+ PR 贡献满足以下至少两点：
	    - 半年内 Merge Coprocessor PR 总行数超过 1500 行；
	    - 有效 Review Coprocessor PR 总数超过 10 个；
	    - 有效 Review Coprocessor PR 总行数超过 1000 行。

2. 退出制度

	a. SIG 成员在以下情况中会被移除 SIG，但保留相应的 Active Contributor / Reviewer / Committer 身份：
	
	   + 作为新成员未在指定时间内认领任务；
	   + 连续一个季度处于不活跃状态。
	
	b. Reviewer 满足以下条件之一会被取消 Reviewer 身份且收回权限（后续重新考核后可恢复）：
	
	   + 超过一个季度没有 review 任何 Coprocessor 相关的 PR；
	   + 有 2 位以上 Committer 认为 Reviewer 能力不足或活跃度不足。

3. Tech Lead 额外承担的职责

   + SIG 成员提出的问题需要在 2 个工作日给出回复；
   + 及时 Review 代码；
   + 定时发布任务（如果 SIG 成员退出后，未完成的任务需要重新分配）。

## 小结

通过上文相信大家对于 Coprocessor SIG 的工作内容、范围、方式以及运营制度有了初步的了解。如果你是一个开源爱好者，想要参与到一个工业级的开源项目中来，或者想了解社区的运行机制，想了解你的代码是如何从一个想法最终发布到生产环境中运行，那么加入 Coprocessor SIG 就是一个绝佳的机会！

**如果你仍对 SIG 有些疑问或者想要了解更多学习资料，欢迎加入 [tikv-wg.slack.com](https://tikv-wg.slack.com/join/shared_invite/enQtNTUyODE4ODU2MzI0LWVlMWMzMDkyNWE5ZjY1ODAzMWUwZGVhNGNhYTc3MzJhYWE0Y2FjYjliYzY1OWJlYTc4OWVjZWM1NDkwN2QxNDE) 哦～**
