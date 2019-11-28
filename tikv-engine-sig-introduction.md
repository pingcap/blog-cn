---
title: TiKV Engine SIG 成立，硬核玩家们看过来！
author: ['Yi Wu']
date: 2019-11-28
summary: TiKV Engine SIG 主要职责是对 TiKV 的存储引擎的未来发展进行讨论和规划，并进行相关开发和维护。期待社区伙伴们的支持和贡献～
tags: ['TiKV','社区','存储引擎']
---

TiKV 是一个开源项目，我们一直都欢迎和感激开源社区对 TiKV 所作出的贡献。但我们之前对开源社区的合作主要是在代码审阅和散落在各种社交媒体的线下讨论，开发者并没有合适的途径去了解和影响 TiKV 的开发计划。怎么才能更好的帮助大家找到组织，更好地参与到 TiKV 的开发中来呢？我们的设想是搭建公开的平台，邀请对 TiKV 中特定领域感兴趣的开发者加入其中，与我们一起探讨和推进相应工作。Special Interest Group（SIG）就是这样的平台。

TiKV Engine SIG 是继 [Coprocessor SIG](https://pingcap.com/blog-cn/tikv-coprocessor-sig/) 之后成立的第二个 TiKV SIG 社区组织，主要职责是对 TiKV 的存储引擎的未来发展进行讨论和规划，并进行相关开发和维护。

目前 TiKV 仅支持默认存储引擎 RocksDB，但是通过扩展接口，希望未来 TiKV 可以支持更多的存储引擎，我们也期待这部分工作可以得到社区的支持，在社区的讨论和贡献中得到更好的完善。此外，Engine SIG 也会对已有的存储引擎进行相关的开发和完善工作。

Engine SIG 的工作主要涉及的模块包括：

*   Engine Trait： TiKV 中存储引擎的抽象层。
*   RocksDB：包括维护 TiKV 所使用的 RocksDB 分支，以及 rust-rocksdb 封装。
*   Titan：提供 KV 分离支持的 RocksDB 存储引擎插件。
*   未来 TiKV 对其它存储引擎的支持。

## 如何加入 Engine SIG

无论你是数据库开发新手，希望通过实战了解存储开发相关知识；​还是 TiKV 资深用户，希望扩展 TiKV 的能力以应用到生产环境，Engine SIG 都欢迎你的加入！

有兴趣的开发者可以浏览 Engine SIG 文档并加入 Engine SIG 的 Slack 频道。Engine SIG 希望能够帮助 Contributor 逐渐成长为 Reviewer，Committer 乃至 TiKV 的 Maintaner。

* Engine SIG 主页：[https://github.com/tikv/community/tree/master/sig/engine](https://github.com/tikv/community/tree/master/sig/engine)

* Engine SIG 章程：[https://github.com/tikv/community/blob/master/sig/engine/constitution-zh_CN.md](https://github.com/tikv/community/blob/master/sig/engine/constitution-zh_CN.md)

* Engine SIG Slack：加入 tikv-wg.slack.com 并进入 [#engine-sig](https://tikv-wg.slack.com/?redir=%2Fmessages%2Fengine-sig) 频道。

## 近期工作计划

近期 Engine SIG 工作会围绕在对 TiKV 已有存储引擎的改进上面，但我们会尽量选取一些对以后引入其它存储引擎也有意义的工作。具体有以下几方面：

*   使用 [Bindgen](https://rust-lang.github.io/rust-bindgen/) 对 [rust-rocksdb](https://github.com/tikv/rust-rocksdb) 进行重构，减少新增存储引擎接口的开发复杂度。
*   扩展 [failpoint](https://pingcap.com/blog-cn/tikv-source-code-reading-5/) 接口，允许为不同的存储引擎开发相应的插件，使得 TiKV 测试能够对存储引擎内部进行错误注入。
*   [Titan](https://github.com/pingcap/titan) 存储引擎插件的性能和功能的改进。

详细任务列表见：[https://github.com/tikv/tikv/projects/22](https://github.com/tikv/tikv/projects/22)。

## 未来工作计划

未来 Engine SIG 会更多关注于为 TiKV 引入新的存储引擎。这上面可以做的工作很多。比如说，我们可以考虑为 TiKV 引入针对不同硬件（纯内存、持久化内存、云盘等）的存储引擎，不同数据结构的存储引擎（B-Tree 引擎等），针对特殊场景的存储引擎（全文搜索等），或者单纯是不一样的存储引擎实现（LevelDB 等）。这些工作非常需要社区的参与。我们希望这些工作未来能够扩展 TiKV 的领域和可能。目前 TiKV 正在加紧对存储引擎抽象 Engine Trait 进行开发，使以上的设想成为可能。

期待社区伙伴们的加入！欢迎在 Slack [#engine-sig](https://tikv-wg.slack.com/?redir=%2Fmessages%2Fengine-sig) 中与我们交流！如果对于流程或技术细节有任何疑问，都可在 channel 中讨论～