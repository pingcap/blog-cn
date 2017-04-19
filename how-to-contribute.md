---
title: 如何从零开始参与大型开源项目
author: TiDB Contributor
date: 2017-03-27
summary: 上世纪 70 年代，IBM 发明了关系型数据库。但是随着现在移动互联网的发展，接入设备越来越多，数据量越来越大，业务越来越复杂，传统的数据库显然已经不能满足海量数据存储的需求。虽然目前市场上也不乏分布式数据库模型，但没有品位的文艺青年不是好工程师，我们觉得，不，这些方案都不是我们想要的，它们不够美，鲜少能够把分布式事务与弹性扩展做到完美。
tags: TiDB
---


## 写在前面的话

上世纪 70 年代，IBM 发明了关系型数据库。但是随着现在移动互联网的发展，接入设备越来越多，数据量越来越大，业务越来越复杂，传统的数据库显然已经不能满足海量数据存储的需求。虽然目前市场上也不乏分布式数据库模型，但没有品位的文艺青年不是好工程师，我们觉得，不，这些方案都不是我们想要的，它们不够美，鲜少能够把分布式事务与弹性扩展做到完美。

受 Google Spanner/F1 的启发，一款从一开始就选择了开源道路的 TiDB 诞生了。 它是一款代表未来的新型分布式 NewSQL 数据库，它可以随着数据增长而无缝水平扩展，只需要通过增加更多的机器来满足业务增长需求，应用层可以不用关心存储的容量和吞吐，用东旭的话说就是「他自己会生长」。

在开源的世界里，TiDB 和 TiKV 吸引了更多的具有极客气质的开发者，目前已经拥有超过 9000 个 star 和 100 个 contributor，这已然是一个世界顶级开源项目的水准。而成就了这一切的，则是来自社区的力量。

最近我们收到了很多封这样的邮件和留言，大家说：

- 谢谢你们，使得旁人也能接触大型开源项目。本身自己是DBA，对数据库方面较干兴趣，也希望自己能逐步深入数据库领域，深入TiDB，为 TiDB 社区贡献更多、更有价值的力量。

- 我是一个在校学生，刚刚收到邮件说我成为了 TiDB 的 Contributor，这让我觉得当初没听父母的话坚持了自己喜欢的计算机技术，是个正确的选择，但我还需要更多的历练，直到能完整地展现、表达我的思维。

这让我感触颇多，因为，应该是我们感谢你们才是啊，没有社区，一个开源项目就成不了一股清泉甚至一汪海洋。
公司的小姑娘说，她觉得还有很多的人想要参与进来的，可工程师团队欠缺平易近人的表达，这个得改。

于是便有了这篇文章以及未来的多篇文章和活动，我们欢迎所有的具有气质的开发者能和 TiDB 一起成长，一起见证数据库领域的革新，改变世界这事儿有时候也不那么难。

我要重点感谢今天这篇文章的作者，来自社区的朱武（GitHub ID:viile ）、小卢（GitHub ID:lwhhhh ）和杨文（GitHub ID: yangwenmai），当在 TiDB Contributor Club 里提到想要做这件事的时候，是他们踊跃地加入了 TiDB Tech Writer 的队伍，高效又专业地完成了下文的编辑，谢谢你们。


## 一个典型的开源项目是由什么组成的

### The Community（社区） 

- 一个项目经常会有一个围绕着它的社区,这个社区由各个承担不同角色的用户组成.

- **项目的拥有者**: 在他们账号中创建项目并拥有它的用户或者组织.

- **维护者和合作者**: 主要做项目相关的工作和推动项目发展,通常情况下拥有者和维护者是同一个人,他们拥有仓库的写入权限.

- **贡献者**: 发起拉取请求 (pull request) 并且被合并到项目里面的人.

- **社区成员**: 对项目非常关心,并且在关于项目的特性以及 pull requests 的讨论中非常活跃的人.

### The Docs（文档）
    
项目中经常出现的文件有:

- **Readme**：几乎所有的Github项目都包含一个README\.md文件,readme文件提供了一些项目的详细信息,包括如何使用,如何构建.有时候也会告诉你如何成为贡献者.
    - TiDB Readme https://github.com/pingcap/tidb/blob/master/README.md 

- **Contributing**: 项目以及项目的维护者各式各样,所以参与贡献的最佳方式也不尽相同.如果你想成为贡献者的话,那么你要先阅读那些有CONTRIBUTING标签的文档.Contributing文档会详细介绍了项目的维护者希望得到哪些补丁或者是新增的特性.
    文件里也可以包含需要写哪些测试,代码风格,或者是哪些地方需要增加补丁之类的内容.
    - TiDB Contributing 文档 https://github.com/pingcap/tidb/blob/master/CONTRIBUTING.md

- **License**: LICENSE文件就是这个开源项目的许可证.一个开源项目会告知用户他们可以做什么,不可做什么(比如:使用,修改,重新分发),以及贡献者允许其他人做哪些事.开源许可证有多种,你可以在[认识各种开源协议及其关系](http://blog.jasonding.top/2015/05/11/Git/%E3%80%90Git%E3%80%91%E8%AE%A4%E8%AF%86%E5%90%84%E7%A7%8D%E5%BC%80%E6%BA%90%E5%8D%8F%E8%AE%AE%E5%8F%8A%E5%85%B6%E5%85%B3%E7%B3%BB/)了解更多关于开源许可证的信息.
    - TiDB 遵循 Apache-2.0 Lincense https://github.com/pingcap/tidb/tree/master/LICENSES 
    - TiKV 遵循 Apache-2.0 Lincense https://github.com/pingcap/tikv/blob/master/LICENSE

-  **Documentation and Wikis**:许多大型项目不会只通过自述文件去引导用户如何使用.在这些项目中你经常可以找到通往其他文件的超链接,或者是在仓库中找到一个叫做docs的文件夹.
    - TiDB Docs https://github.com/pingcap/tidb/tree/master/docs  

    ![][1]

    有些项目也会把文档写在Github wiki里
    - TiKV Wiki https://github.com/pingcap/tikv/wiki

    ![][2]


## 起步走成为 Contributor
###Create an Issue
如果你在使用项目中发现了一个bug,而且你不知道怎么解决这个bug.或者使用文档时遇到了麻烦.或者有关于这个项目的问题.你可以创建一个issue.
不管你有什么bug,你提出bug后,会对那些和你有同样bug的人提供帮助.
更多关于issue如何工作的信息,请点击[Issues guide](http://guides.github.com/features/issues).

#### Issues Pro Tips
* **检查你的问题是否已经存在**  重复的问题会浪费大家的时间,所以请先搜索打开和已经关闭的问题,来确认你的问题是否已经提交过了.
* **清楚描述你的问题** 你预期的结果是什么?实际执行结果是什么?如何复现这个问题?
* **给出你的代码链接** 使用像 [JSFiddle](http://jsfiddle.net/) 或者[CodePen](http://codepen.io/)等工具,贴出你的代码,好帮助别人复现你的问题
* **详细的系统环境介绍** 例如使用什么版本的浏览器,什么版本的库,什么版本的操作系统等其他你运行环境的介绍.
* **详细的错误输出或者日志** 使用[Gist](http://gist.github.com/)贴出你的错误日志. 如果你在issue中附带错误日志,请使用` ``` `来标记你的日志.以便更好的显示.

### Pull Request
如果你能解决这个bug,或者你能够添加其他的功能.并且知道如何成为贡献者,理解license,已经签过[Contributor Licence Agreement](https://en.wikipedia.org/wiki/Contributor_License_Agreement) (CLA) 后,请发起Pull Request.这样维护人员可以将你的分支与现有分支进行比较，来决定是否合并你的更改。

#### Pull Request Pro Tips
* **[Fork](http://guides.github.com/activities/forking/)代码并且clone到你本地** 通过将项目的地址添加为一个remote,并且经常从remote合并更改来保持你的代码最新,以便在提交你的pull请求时,尽可能少的发生冲突。.详情请参阅[这里](https://help.github.com/articles/syncing-a-fork).
* **创建[branch](http://guides.github.com/introduction/flow/)** 来修改你的代码
* **描述清楚你的问题** 方便其他人能够复现.或者说明你添加的功能有什么作用,并且清楚描述你做了哪些更改.
* **最好有测试**. 如果项目中已经有测试代码,修改或者新增测试代码.不论测试是否存在,保证新增的代码不会影响项目现有功能
* **包含截图** 如果您的更改包含HTML/CSS中的差异,请添加前后的屏幕截图.将图像拖放到您的pull request的正文中.
* **保持良好的代码风格**这意味着使用与你自己的代码风格中不同的缩进,分号或注释,但是使维护者更容易合并,其他人将来更容易理解和维护.

### Open Pull Requests
一旦你新增一个pull request,讨论将围绕你的更改开始.其他贡献者和用户可能会进入讨论,但最终决定是由维护者决定的.你可能会被要求对你的pull request进行一些更改,如果是这样,请向你的branch添加更多代码并推送它们,它们将自动进入现有的pull request.

![pr convo](media/pr-convo.png)

如果你的pull request被合并,这会非常棒.如果没有被合并,不要灰心.也许你的更改不是项目维护者需要的.或者更改已经存在了.发生这种情况时,我们建议你根据收到的任何反馈来修改代码,并再次提出pull request.或创建自己的开源项目.

### TiDB 合并流程
PR提交之后，请耐心等待维护者进行Review。
目前一般在一到两个工作日内都会进行Review，如果当前的PR堆积数量较多可能回复会比较慢。
代码提交后CI会执行我们内部的测试，你需要保证所有的单元测试是可以通过的。期间可能有其它的提交会与当前PR冲突，这时需要修复冲突。
维护者在Review过程中可能会提出一些修改意见。修改完成之后如果reviewer认为没问题了，你会收到LGTM(looks good to me)的回复。当收到两个及以上的LGTM后，该PR将会被合并。


**标注: 本文「一个典型的开源项目是由什么组成的」及「起步走成为 Contributor」参考自英文 GitHub Guide，由社区成员朱武（GitHub ID: viile）、小卢（GitHub ID:lwhhhh）着手翻译并替换部分原文中的截图。GitHub Guides：如何参与一个 GitHub 开源项目英文原文地址： https://guides.github.com/activities/contributing-to-open-source/**

## 加入 TiDB Contributor Club

为更好地促进 Contributor 间的交流，便于随时提出好的想法和反馈，我们创建了一个 Contributor Club 微信群，对成为 TiDB Contributor 有兴趣的同学可以添加 TiDB Robot 微信号，它会在后台和你打招呼，并积极招募你成为开源社区的一员。

![tidb_rpbot](media/tidb-robot.jpg)

欢迎加入 TiDB Tech Writer 计划，让我们一起用文字的力量推动开源项目的发展。

## 衍生阅读
- PingCAP 唐刘：重度开源爱好者眼中的 “ 开源精神 ” http://www.tuicool.com/articles/mYfuq26
- How do we build TiDB http://mp.weixin.qq.com/s?__biz=MzI3NDIxNTQyOQ==&mid=2247483954&idx=1&sn=433caeb1aa021700b223047c50d87696&scene=19#wechat_redirect
- TiDB 源码剖析 https://github.com/ngaut/builddatabase/blob/master/tidb/sourcecode.md
- 解析 TiDB 在线数据同步工具 Syncer http://www.pingcap.com/blog-tidb-syncer.html
- PingCAP 将出席 Percona Live Amsterdam 2016 http://www.oschina.net/news/77574/pingcap-percona-live-amsterdam-2016

## 更多资料
- 官方网站: PingCAP.com
- 官方文档: pingcap.com/docs
- 官方博客: pingcap.com/blog-add-a-built-in-function-zh.html
- TiDB Weekly：weekly.pingcap.com
- 微信公众号：PingCAP

  [1]: http://on51si7u9.bkt.clouddn.com/meitu%20%281%29.jpg
  [2]: https://download.pingcap.org/images/tidb-convo.png
