---
title: TiDB 开源社区指南（上）
author: ['申砾']
date: 2018-11-09
summary: 本系列文章旨在帮助社区开发者了解 TiDB 项目的全貌，更好的参与 TiDB 项目开发。上篇会聚焦在社区参与者的角度，描述如何更好的参与 TiDB 项目开发。
tags: ['TiDB','社区']
---


本系列文章旨在帮助社区开发者了解 TiDB 项目的全貌，更好的参与 TiDB 项目开发。大致会分两个角度进行描述：

* 从社区参与者的角度描述如何更好的参与 TiDB 项目开发；

* 从 PingCAP 内部团队的角度展示 TiDB 的开发流程，包括版本规划、开发流程、Roadmap 制定等。

希望通过一内一外两条线的描述，读者能在技术之外对 TiDB 有更全面的了解。本篇将聚焦在社区参与者的角度进行描述，也就是“外线”。

## 了解 TiDB

参与一个开源项目第一步总是了解它，特别是对 TiDB 这样一个大型的项目，了解的难度比较高，这里列出一些相关资料，帮助 newcomers 从架构设计到工程实现细节都能有所了解：

* [Overview](https://github.com/pingcap/docs#tidb-introduction)
* [How we build TiDB](https://pingcap.com/blog/2016-10-17-how-we-build-tidb/)
* [TiDB 源码阅读系列文章](https://pingcap.com/blog-cn/#%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB)
* [Deep Dive TiKV (Work-In-Process)](https://github.com/tikv/deep-dive-tikv/)

当然，最高效地熟悉 TiDB 的方式还是使用它，在某些场景下遇到了问题或者是想要新的 feature，去跟踪代码，找到相关的代码逻辑，在这个过程中很容易对相关模块有了解，不少 Contributor 就是这样完成了第一次贡献。 

我们还有一系列的 Infra Meetup，大约两周一次，如果方便到现场的同学可以听到这些高质量的 Talk。除了北京之外，其他的城市（上海、广州、成都、杭州）也开始组织 Meetup，方便更多的同学到现场来面基。

## 发现可以参与的事情

对 TiDB 有基本的了解之后，就可以选一个入手点。在 TiDB repo 中我们给一些简单的 issue 标记了 [for-new-contributors](https://github.com/pingcap/tidb/issues?q=is%3Aissue+is%3Aopen+label%3A%22for+new+contributors%22) 标签，这些 issue 都是我们评估过容易上手的事情，可以以此为切入点。另外我们也会定期举行一些活动，把框架搭好，教程写好，新 Contributor 按照固定的模式即可完成某一特性开发。

当然除了那些标记为 for-new-contributors 的 issue 之外，也可以考虑其他的 issue，标记为 [help-wanted](https://github.com/pingcap/tidb/issues?q=is%3Aissue+is%3Aopen+label%3A%22help+wanted%22) 标签的 issue 可以优先考虑。除此之外的 issue 可能会比较难解决，需要对 TiDB 有较深入的了解或者是对完成时间有较高的要求，不适合第一次参与的同学。

当然除了现有的 issue 之外，也欢迎将自己发现的问题或者是想要的特性提为新的 issue，然后自投自抢 :) 。 

当你已经对 TiDB 有了深入的了解，那么可以尝试从 [Roadmap](https://github.com/pingcap/docs/blob/master/ROADMAP.md) 上找到感兴趣的事项，和我们讨论一下如何参与。

## 讨论方案

找到一个感兴趣的点之后，可以在 issue 中进行讨论，如果是一个小的 bug-fix 或者是小的功能点，可以简单讨论之后开工。即使再简单的问题，也建议先进行讨论，以免出现解决方案有问题或者是对问题的理解出了偏差，做了无用功。

但是如果要做的事情比较大，可以先写一个详细的设计文档，提交到 [docs/design](https://github.com/pingcap/tidb/tree/master/docs/design) 目录下面，这个目录下有设计模板以及一些已有的设计方案供你参考。一篇好的设计方案要写清楚以下几点：

*   背景知识

*   解决什么问题

*   方案详细设计

*   对方案的解释说明，证明正确性和可行性

*   和现有系统的兼容性

*   方案的具体实现 

用一句话来总结就是写清楚“你做了什么，为什么要做这个，怎么做的，为什么要这样做”。如果对自己的方案不太确定，可以先写一个 Google Doc，share 给我们简单评估一下，再提交 PR。

## 提交 PR

按照方案完成代码编写后，就可以提交 PR。当然如果开发尚未完成，在某些情况下也可以先提交 PR，比如希望先让社区看一下大致的解决方案，这个时候请将 PR 标记为 WIP。

对于 PR 我们有一些要求：

1. 需要能通过 `make dev` 的测试，跑过基本的单元测试；

2. 必须有测试，除非只是改动文档或者是依赖包，其他情况需要有充足的理由说明没有测试的原因；

3. 代码以及注释的质量需要足够高，[这里](https://github.com/pingcap/community/blob/master/CONTRIBUTING.md#code-style) 有一些关于编码风格和 commit message 的 guide；

4. 请尽可能详细的填写 PR 的描述，并打上合适的 label。

**对于 PR 的描述，我们提供了一个模板，希望大家能够认真填写，一个好的描述能够加速 PR 的 review 过程。通过这个模板能够向 reviewers 以及社区讲明白：**

* 这个PR 解决什么问题：相关的问题描述或者是 issue 链接；

* 如何解决：具体的解决方法，reviewers 会根据这里的描述去看代码变动，所以请将这一段写的尽可能详细且有帮助；

* 测试的情况；

* 其他相关信息（如果需要）：benchmark 结果、兼容性问题、是否需要更新文档。

最后再说几句测试，正确性是数据库安身立命之本，怎么强调测试都不为过。PR 中的测试不但需要充足，覆盖到所做的变动，还需要足够清晰，通过代码或者注释来表达测试的目的，帮助 reviewer 以及今后可能变动/破坏相关逻辑的人能够容易的理解这段测试。一段完善且清晰的测试也有利于让 reviewer 相信这个 Patch 是正确的。

## PR review

PR review 的过程就是 reviewer 不断地提出 comment，PR 作者持续解决 comment 的过程。

**每个 PR 在合并之前都需要至少得到两个 Committer/Maintainer 的 LGTM，一些重要的 PR 需要得到三个，比如对于 DDL 模块的修改，默认都需要得到三个 LGTM。**

#### Tips：

* 提了PR 之后，可以 at 一下相关的同学来 review；

* Address comment 之后可以 at 一下之前提过 comment 的同学，标准做法是 comment 一下 “**PTAL @xxx**”，这样我们内部的 Slack 中可以得到通知，相关的同学会受到提醒，让整个流程更紧凑高效。

## 与项目维护者之间的交流

**目前标准的交流渠道是 GitHub issue**，请大家优先使用这个渠道，我们有专门的同学来维护这个渠道，其他渠道不能保证得到研发同学的及时回复。这也是开源项目的标准做法。

无论是遇到 bug、讨论具体某一功能如何做、提一些建议、产品使用中的疑惑，都可以来提 issue。在开发过程中遇到了问题，也可以在相关的 issue 中进行讨论，包括方案的设计、具体实现过程中遇到的问题等等。

**最后请大家注意一点，除了 pingcap/docs-cn 这个 repo 之外，请大家使用英文。**

## 更进一步

当你完成上面这些步骤的之后，恭喜你已经跨过第一个门槛，正式进入了 TiDB 开源社区，开始参与 TiDB 项目开发，成为 TiDB Contributor。

如果想更进一步，深入了解 TiDB 的内部机制，掌握一个分布式数据库的核心模块，并能做出改进，那么可以了解更多的模块，提更多的 PR，进一步向 Committer 发展（[这里](https://github.com/pingcap/community/blob/master/become-a-committer.md) 解释了什么是 Committer）。目前 TiDB 社区的 Committer 还非常少，我们希望今后能出现更多的 Committer 甚至是 Maintainer。

从 Contributor 到 Committer 的门槛比较高，比如今年的新晋 Committer 杜川同学，在成为 Committer 的道路上给 tidb/tikv 项目提交了大约 80 个 PR，并且对一些模块有非常深入的了解。当然，成为 Committer 之后，会有一定的权利，比如对一些 PR 点 LGTM 的权利，参加 PingCAP 内部的技术事项、开发规划讨论的权利，参加定期举办的 TechDay/DevCon 的权利。目前社区中还有几位贡献者正走在从 Contributor 到 Committer 的道路上。
