---
title: 分布式系统测试那些事儿 - 理念
author: ['刘奇']
date: 2016-11-01
summary: 本话题系列文章整理自 PingCAP Infra Meetup 第 26 期刘奇分享的《深度探索分布式系统测试》议题现场实录。文章较长，为方便大家阅读，会分为上中下三篇，本文为上篇。
tags: ['TiDB', '分布式系统测试', '自动化测试']
meetup_type: memoir
---

> 本话题系列文章整理自 PingCAP NewSQL Meetup 第 26 期刘奇分享的《深度探索分布式系统测试》议题现场实录。文章较长，为方便大家阅读，会分为上中下三篇，本文为上篇。

今天主要是介绍分布式系统测试。对于 PingCAP 目前的现状来说，我们是觉得做好分布式系统测试比做一个分布式系统更难。就是你把它写出来不是最难的，把它测好才是最难的。大家肯定会觉得有这么夸张吗？那我们先从一个最简单的、每个人都会写的 Hello world  开始。

### A simple “Hello world” is a miracle

We should walk through all of the bugs in:

+ Compiler
+ Linker
+ VM (maybe)
+ OS

其实这个 Hello world 能够每次都正确运行已经是一个奇迹了，为什么呢？首先，编译器得没 bug，链接器得没 bug ；然后我们可能跑在 VM 上，那 VM 还得没 bug；并且 Hello world 那还有一个 syscall，那我们还得保证操作系统没有 bug；到这还不算吧，我们还得要硬件没有 bug。所以一个最简单程序它能正常运行起来，我们要穿越巨长的一条路径，然后这个路径里面所有的东西都不能出问题，我们才能看到一个最简单的 Hello world。

但是分布式系统里面呢，就更加复杂了。比如大家现在用的很典型的微服务。假设你提供了一个微服务，然后在微服务提供的功能就是输出一个 Hello world  ，然后让别人来 Call。

### A RPC “Hello world” is a miracle

We should walk through all of the bugs in:

+ Coordinator (zookeeper, etcd)
+ RPC implementation
+ Network stack
+ Encoding/Decoding library
+ Compiler for programming languages or [protocol buffers, avro, msgpack, capn]

那么我们可以看一下它的路径。我们起码需要依赖 Coordinator 去做这种服务发现，比如用 zookeeper，etcd ，大家会感觉是这东西应该很稳定了吧？但大家可以去查一下他们每一次  release notes，里边说我们 fix 了哪些 bug，就是所有大家印象中非常稳定的这些东西，一直都在升级，每一次升级都会有 bug fix。但换个思路来看，其实我们也很幸运，因为大部分时候我们没有碰到那个 bug，然后 RPC 的这个实现不能有问题。当然如果大家深度使用 RPC，比如说 gRPC，你会发现其实 bug 还是挺多的，用的深一点，基本上就会发现它有 bug。还有系统网络协议栈，去年 TCP 被爆出有一个 checksum 问题，就是 Linux 的 TCP 协议栈，这都是印象中永远不会出问题的。再有，编解码，大家如果有 Go 的经验的话，可以看一下 Go 的 JSON 历史上从发布以来更新的记录，也会发现一些 bug。还有更多的大家喜欢的编解码，比如说你用 Protocol buffers、Avro、Msgpack、Cap'n 等等，那它们本身还需要 compiler 去生成一个代码，然后我们还需要那个 compiler 生成的代码是没有 bug 的。然后这一整套下来，我们这个程序差不多是能运行的，当然我们没有考虑硬件本身的 bug。

其实一个正确的运行程序从概率上来讲（不考虑宇宙射线什么的这种），已经是非常幸运的了。当然每一个系统都不是完善的，那通常情况下，为什么我们这个就运行的顺利呢？因为我们的测试永远都测到了正确的路径，我们跑一个简单的测试一定是把正确的路径测到了，但是这中间有很多错误路径其实我们都没有碰到。然后我不知道大家有没有印象，如果写 Go 程序的时候，错误处理通常写成 if err != nil，然后 return error ，不知道大家写了多少。那其它程序、其它的语言里就是 try.catch，然后里面各种 error 处理。就是一个真正完善的系统，最终的错误处理代码实际上通常会比你写正常逻辑代码还要多的，但是我们的测试通常 cover 的是正确的逻辑，就是实际上我们测试的 cover 是一小部分。

那先纠正几个观念，关于测试的。就是到底怎么样才能得到一个好的、高质量的程序，或者说得到一个高质量的系统？

### Who is the tester ?

+ Quality comes from solid engineering.
+ Stop talking and go build things.
+ Don’t hire too many testers.
	- Testing is owned by the entire team.  It is a culture, not a process.
+ Are testers software engineers? Yes.
+ Hiring good people is the first step.  And then keep them challenged.

我们的观念是说先有 solid engineering 。我觉得这个几乎是勿庸置疑的吧，不知道大家的经验是什么？然后还有一个就是不扯淡，尽快去把东西 build 起来，然后让东西去运转起来。我前一段时间也写了一个段子，就是：“你是写 Rust 的，他是写 Java 的，你们这聊了这么久，人家 Rust （编译速度慢） 的程序已经编译过了，你 Java 还没开始写。”原版是这样的:“你是砍柴的，他是放羊的，你们聊了一天，他的羊吃饱了，你的柴呢？”然后最近还有一个特别有争议的话题：CTO 应该干嘛。就是 CTO 到底该不该写代码，这个也是众说纷纭。因为每一个人都受到自己环境的局限，所以每个人的看法都是不一样的。那我觉得有点像，就是同样是聊天，然后不同人有不同的看法。

### Test automation

+ Allow developers to get a unit test results immediately.
+ Allow developers to run all unit tests in one go.
+ Allow code coverage calculations.
+ Show the testing evolution on the dashboards.
+ Automate everything.

我们现在很有意思的一个事情是，迄今为止 PingCAP 没有一个测试人员，这是在所有的公司看来可能都是觉得不可思议的事情，那为什么我们要这么干？因为我们现在的测试已经不可能由人去测了。究竟复杂到什么程度呢？我说几个基本数字大家感受一下：我们现在有六百多万个 Test，这是完全自动化去跑的。然后我们还有大量从社区收集到的各种 ORM Test，一会我会提到这一点。就是这么多 Test 已经不可能是由人写出来的了，以前的概念里面是 Test 是由人写的，但实际上 Test 不一定是人写的，Test 也是可以由机器生成的。举个例子，如果给你一个合法的语法树，你按照这个语法树去做一个输出，比如说你可以更换变量名，可以更换它的表达式等等，你可以生成很多的这种 SQL 出来。

Google Spanner 就用到这个特性，它会有专门的程序自动生成符合 SQL 语法的语句，然后再交给系统去执行。如果执行过程中 crash 了，那说明这个系统肯定有 bug。但是这地方又蹦出另外一个问题，就是你生成了合法的 SQL 语句，但是你不知道它语句执行的结构，那你怎么去判断它是不是对的？当然业界有很聪明的人。我把它扔给几个数据库同时跑一下，然后取几个大家一致的结果，那我就认为这个结果基本上是对的。如果一个语句过来，然后在我这边执行的结果和另外几个都不一样，那说明我这边肯定错了。就算你是对的，可能也是错的，因为别人执行下来都是这个结果，你不一样，那大家都会认为你是错的。

所以说在测试的时候，怎么去自动生成测试很重要。去年，在美国那边开始流行一个新的说法，叫做 “怎么在你睡觉的时候发现 bug”。那么实际上测试干的很重要的事情就是这个，就是自动化测试是可以在你睡觉的时候发现 bug。好像刚才我们还提到 fault injection ，好像还有 fuzz testing。然后所有测试的人都是工程师，因为只有这样你才不会甩锅。

这是我们现在坚信的一个事情，就是所有的测试必须要高度的自动化，完全不由人去干预。然后很重要的一个就是雇最优秀的人才，同时给他们挑战，就是如果没有挑战，这些人才会很闲，精力分散，然后很难合力出成绩。因为以现在这个社会而言，很重要一个特性是什么？就是对于复杂性工程需要大量的优秀人才，如果优秀的人才力不往一处使力的话，这个复杂性工程是做不出来的。我今天看了一下龙芯做了十年了，差不多是做到英特尔凌动处理器的水平。他们肯定是有很优秀的人才，但是目前还得承认，我们在硬件上面和国外的差距还比较大，其实软件上面的差距也比较大，比如说我们和 Spanner 起码差了七年，2012 年 Spanner 就已经大规模在 Google 使用了，对这些优秀的作品，我们一直心存敬仰。

我刚才已经反复强调过自动化这个事情。不知道大家平时写代码 cover 已经到多少了？如果 cover 一直低于 50%，那就是说你有一半的代码没有被测到，那它在线上什么时候都有可能出现问题。当然我们还需要更好的方法去在上线之前能够把线上的 case 回放。理论上你对线上这个回放的越久你就越安全，但是前提是线上代码永远不更新，如果业务方更新了，那就又相当于埋下了一个定时炸弹。比如说你在上面跑两个月，然后业务现在有一点修改，然而那两个又没有 cover 住修改，那这时候可能有新的问题。所以要把所有的一切都自动化，包括刚才的监控。比如说你一个系统一过去，然后自动发现有哪些项需要监控，然后自动设置报警。大家觉得这事是不是很神奇？其实这在 Google 里面是司空见惯的事情，PingCAP 现在也正在做。

### Well… still not enough ?

+ Each layer can be tested independently.
+ Make sure you are building the right tests.
+ Don’t bother great people unless the testing fails.
+ Write unit tests for every bug.

这么多还是不够的，就是对于整个系统测试来讲，你可以分成很多层、分成很多模块，然后一个一个的去测。还有很重要的一点，就是早期的时候我们发现一个很有意思的事情。就是我们 build 了大量 Test，然后我们的程序都轻松的 pass 了大量的 Test，后来发现我们一个 Test 是错的，那意味着什么？意味着我们的程序一直是错的，因为 Test 会把你这个 cover 住。所以直到后来我们有一次觉得自己写了一个正确的代码，但是跑出来的结果不对，我们这时候再去查，发现以前有一个 Test 写错了。所以一个正确的 Test 是非常重要的，否则你永远被埋在错误里面，然后埋在错误里面感觉还特别好，因为它告诉你是正确的。

还有，为什么要自动化呢？就是你不要去打扰这些聪明人。他们本身很聪明，你没事别去打扰他们，说“来，你过来给我做个测试”，那这时候不断去打扰他们，是影响他们的发挥，影响他们做自己的挑战。

这一条非常重要，所有出现过的 bug，历史上只要出现过一次，你一定要写一个 Test 去 cover 它，那这个法则大家应该已经都清楚了。我看今天所在的人的年龄，应该《圣斗士星矢》是看过的，对吧？这个圣斗士是有一个特点的，所有对他们有效的招数只能用一次，那这个也是一样的，就保证你不会被再次咬到，就不会再次被坑到。我印象中应该有很多人 fix bug 是这样的：有一个 bug 我 fix 了，但没有 Test，后来又出现了，然后这时候就觉得很奇怪，然后积累的越多，最后就被坑的越惨。

这个是目前主流开源社区都在坚持的做法，基本没有例外。就是如果有一个开源社区说我发现一个 bug，我没有 Test 去 cover 它，这个东西以后别人是不敢用的。

### Code review

+ At least two LGTMs (Looks good to me) from the maintainers.
+ Address comments.
+ Squash commit logs.
+ Travis CI/Circle CI for PRs.

简单说一下 code review 的事情，它和 Test 还是有一点关系，为什么？因为在 code review 的时候你会提一个新的 pr，然后这个 pr 一定要通过这个 Test。比如说典型的 Travis CI，或者 CircleCI 的这种 Test。为什么要这样做呢？因为要保证它被 merge 到 master 之前你一定要发现这个问题，如果已经 merge 到 master 了，首先这不好看，因为你要 revert 掉，这个在 commit 记录上是特别不好看的一个事情。另外一个就是它出现问题之前，你就先把它发现其实是最好的，因为有很多工具会根据 master 自动去 build。比如说我们会根据 master 去自动 build docker 镜像，一旦你代码被 commit 到 master，然后 docker 镜像就出来了。那你的用户就发现，你有新的更新，我要马上使用新的，但是如果你之前的 CI 没有过，这时候就麻烦了，所以 CI 没过，一定不能进入到 CD 阶段。

### Who to blame in case of bugs?

The entire team.

另外一个观念纠正一下，就是出现 bug 的时候，责任是谁的？通常我见过的很多人都是这样，就说“这个 bug 跟我没关系，他的模块的 bug”。那 PingCAP 这边的看法不一样，就是一旦出现 bug，这应该是整个 team 的责任，因为你有自己的 code review 机制，至少有两个以上的人会去看它这个代码，然后如果这个还出现问题，那一定不是一个人的问题。

除了刚才说的发现一些 bug，还有一些你很难定义，说这是不是 bug，怎么系统跑的慢，这算不算 bug，怎么对 bug 做界定呢？我们现在的界定方式是用户说了算。虽然我们觉得这不是 bug，这不就慢一点吗，但是用户说了这个东西太慢了，我们不能忍，这就是 bug，你就是该优化的就优化。然后我们团队里面出现过这样的事情，说“我们这个已经跑的很快了，已经够快了”，对不起，用户说慢，用户说慢就得改，你就得去提升。总而言之，标准不能自己定，当然如果你自己去定这个标准，那这个事就变成“我这个很 OK 了，我不需要改了，可以了。”这样是不行的。

### Profiling

+ Profile everything, even on production
	- once-in-a-lifetime chance
+ Bench testing

另外，在 Profile 这个事情上面，我们强调一个，即使是在线上，也需要能做  Profile，其实  Profile 的开销是很小的。然后很有可能是这样的，有一次线上系统特别卡，如果你把那个重启了，你可能再也没有机会复现它了，那么对于这些情况它很可能是一辈子发生一次的，那一次你没有抓住它，你可能再也没有机会抓住它了。当然我们后面会介绍一些方法，可以让这个能复现，但是有一些确实是和业务相关性极强的，那么可能刚好又碰到一个特别的环境才能让它出现，那真的可能是一辈子就那么一次的，你一定要这次抓住它，这次抓不住，你可能永远就抓不住了。因为有些犯罪它一辈子只犯一次，它犯完之后你再也没有机会抓住它了。

### Embed testing to your design

+ Design for testing or Die without good tests
+ Tests may make your code less beautiful

再说测试和设计的关系。测试是一定要融入到你的设计里面，就是在你设计的时候就一定要想这个东西到底应该怎么去测。如果在设计的时候想不到这个东西应该怎么测，那这个东西就是正确性实际上是没法验证的，这是非常恐怖的一件事情。我们把测试的重要程度看成这样的：你要么就设计好的测试，要么就挂了，就没什么其它的容你选择。就是说在这一块我们把它的重要性放到一个最高的程度。

##### 未完待续...
