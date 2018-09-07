---
title: 使用 TiKV 构建分布式类 Redis 服务
author: ['唐刘']
date: 2018-09-07
summary: 本文将介绍 Redis 的特性与不足，以及使用 TiKV 构建分布式类 Redis 服务的
tags: ['Redis', 'TiKV']
---


## 什么是 Redis

[Redis](https://redis.io/) 是一个开源的，高性能的，支持多种数据结构的内存数据库，已经被广泛用于数据库，缓存，消息队列等领域。它有着丰富的数据结构支持，譬如 String，Hash，Set 和 Sorted Set，用户通过它们能构建自己的高性能应用。

Redis 非常快，没准是世界上最快的数据库了，它虽然使用内存，但也提供了一些持久化机制以及异步复制机制来保证数据的安全。 

## Redis 的不足

Redis 非常酷，但它也有一些问题：

1. 内存很贵，而且并不是无限容量的，所以我们不可能将大量的数据存放到一台机器。
2. 异步复制并不能保证 Redis 的数据安全。
3. Redis 提供了 transaction mode，但其实并不满足 ACID 特性。
4. Redis 提供了集群支持，但也不能支持跨多个节点的分布式事务。

所以有时候，我们需要一个更强大的数据库，虽然在延迟上面可能赶不上 Redis，但也有足够多的特性，譬如：

1. 丰富的数据结构
2. 高吞吐，能接受的延迟
3. 强数据一致
4. 水平扩展
5. 分布式事务

## 为什么选择 TiKV

大约 4 年前，我开始解决上面提到的 Redis 遇到的一些问题。为了让数据持久化，最直观的做法就是将数据保存到硬盘上面，而不是在内存里面。所以我开发了 [LedisDB](https://github.com/reborndb/reborn)，一个使用 Redis 协议，提供丰富数据结构，但将数据放在 RocksDB 的数据库。LedisDB 并不是完全兼容 Redis，所以后来，我和其他同事继续创建了 [RebornDB](https://github.com/reborndb/reborn)，一个完全兼容 Redis 的数据库。
无论是 LedisDB 还是 RebornDB，因为他们都是将数据放在硬盘，所以能存储更大量的数据。但它们仍然不能提供 ACID 的支持，另外，虽然我们可以通过 [codis](https://github.com/CodisLabs/codis) 去提供集群的支持，我们也不能很好的支持全局的分布式事务。

所以我们需要另一种方式，幸运的是，我们有 [TiKV](https://github.com/tikv/tikv)。

TiKV 是一个高性能，支持分布式事务的 key-value 数据库。虽然它仅仅提供了简单的 key-value API，但基于 key-value，我们可以构造自己的逻辑去创建更强大的应用。譬如，我们就构建了 [TiDB](https://github.com/pingcap/tidb) ，一个基于 TiKV 的，兼容 MySQL 的分布式关系型数据库。TiDB 通过将 database 的 schema 映射到 key-value 来支持了相关 SQL 特性。所以对于 Redis，我们也可以采用同样的办法 - 构建一个支持 Redis 协议的服务，将 Redis 的数据结构映射到 key-value 上面。

## 如何开始

![](https://upload-images.jianshu.io/upload_images/542677-444fff797845a591.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

整个架构非常简单，我们仅仅需要做的就是构建一个 Redis 的 Proxy，这个 Proxy 会解析 Redis 协议，然后将 Redis 的数据结构映射到 key-value 上面。

### Redis Protocol

Redis 协议被叫做 [RESP](https://redis.io/topics/protocol)（Redis Serialization Protocol），它是文本类型的，可读性比较好，并且易于解析。它使用 “rn” 作为每行的分隔符并且用不同的前缀来代表不同的类型。例如，对于简单的 String，第一个字节是 “+”，所以一个 “OK” 行就是 “+OKrn”。
大多数时候，客户端会使用最通用的 Request-Response 模型用于跟 Redis 进行交互。客户端会首先发送一个请求，然后等待 Redis返回结果。请求是一个 Array，Array 里面元素都是 bulk strings，而返回值则可能是任意的 RESP 类型。Redis 同样支持其他通讯方式：

Pipeline - 这种模式下面客户端会持续的给 Redis 发送多个请求，然后等待 Redis 返回一个结果。
Push - 客户端会在 Redis 上面订阅一个 channel，然后客户端就会从这个 channel 上面持续受到 Redis push 的数据。

下面是一个简单的客户端发送 `LLEN mylist` 命令到 Redis 的例子：

```
C: *2\r\n
C: $4\r\n
C: LLEN\r\n
C: $6\r\n
C: mylist\r\n

S: :48293\r\n
```

客户端会发送一个带有两个 bulk string 的 array，第一个 bulk string 的长度是 4，而第二个则是 6。Redis 会返回一个 48293 整数。正如你所见，RESP 非常简单，自然而然的，写一个 RESP 的解析器也是非常容易的。

作者创建了一个 Go 的库 [goredis](https://github.com/siddontang/goredis)，基于这个库，我们能非常容易的从连接上面解析出 RESP，一个简单的例子：

```
// Create a buffer IO from the connection.
br := bufio.NewReaderSize(conn, 4096)
// Create a RESP reader.
r := goredis.NewRespReader(br)
// Parse the Request
req := r.ParseRequest()
```

函数 `ParseRequest` 返回一个解析好的 request，它是一个 `[][]byte` 类型，第一个字段是函数名字，譬如 “LLEN”，然后后面的字段则是这个命令的参数。

### TiKV 事务 API

在我们开始之前，作者将会给一个简单实用 TiKV 事务 API 的例子，我们调用 Begin 开始一个事务：

```
txn, err := db.Begin()
```

函数 `Begin` 创建一个事务，如果出错了，我们需要判断 err，不过后面作者都会忽略 err 的处理。

当我们开始了一个事务之后，我们就可以干很多操作了：

```
value, err := txn.Get([]byte(“key”))
// Do something with value and then update the newValue to the key.
txn.Put([]byte(“key”), newValue)
```

上面我们得到了一个 key 的值，并且将其更新为新的值。TiKV 使用乐观事务模型，它会将所有的改动都先缓存到本地，然后在一起提交给 Server。

```
// Commit the transaction
txn.Commit(context.TODO())
```

跟其他事务处理一样，我们也可以回滚这个事务：

```
txn.Rollback()
```

如果两个事务操作了相同的 key，它们就会冲突。一个事务会提交成功，而另一个事务会出错并且回滚。

### 映射 Data structure 到 TiKV

现在我们知道了如何解析 Redis 协议，如何在一个事务里面做操作，下一步就是支持 Redis 的数据结构了。Redis 主要有 4 中数据结构：String，Hash，Set 和 Sorted Set，但是对于 TiKV 来说，它只支持 key-value，所以我们需要将这些数据结构映射到 key-value。

首先，我们需要区分不同的数据结构，一个非常容易的方式就是在 key 的后面加上 Type flag。例如，我们可以将 ’s’ 添加到 String，所以一个 String key “abc” 在 TiKV 里面其实就是 “abcs”。

对于其他类型，我们可能需要考虑更多，譬如对于 Hash 类型，我们需要支持如下操作：

```
HSET key field1 value1
HSET key field2 value2
HLEN key
```

一个 Hash 会有很多 fields，我有时候想知道整个 Hash 的个数，所以对于 TiKV，我们不光需要将 Hash 的 key 和 field 合在一起变成 TiKV 的一个 key，也同时需要用另一个 key 来保存整个 Hash 的长度，所以整个 Hash 的布局类似：

```
key + ‘h’ -> length
key + ‘f’ + field1 -> value
key + ‘f’ + field2 -> value 
```

如果我们不保存 length，那么如果我们想知道 Hash 的 length，每次都需要去扫整个 Hash 得到所有的 fields，这个其实并不高效。但如果我们用另一个 key 来保存 length，任何时候，当我们加入一个新的 field，我们都需要去更新这个 length 的值，这也是一个开销。对于我来说，我倾向于使用另一个 key 来保存 length，因为 `HLEN` 是一个高频的操作。

## 例子

作者构建了一个非常简单的例子 [example](https://github.com/siddontang/redis-tikv-example) ，里面只支持 String 和 Hash 的一些操作，我们可以 clone 下来并编译：

```
git clone https://github.com/siddontang/redis-tikv-example.git $GOPATH/src/github.com/siddontang/redis-tikv-example

cd $GOPATH/src/github.com/siddontang/redis-tikv-example
go build
```

在运行之前，我们需要启动 TiKV，可以参考 [instruction](https://github.com/tikv/tikv#deploying-to-production)，然后执行：

```
./redis-tikv-example
```

这个例子会监听端口 6380，然后我们可以用任意的 Redis 客户端，譬如 `redis-cli` 去连接：

```
redis-cli -p 6380
127.0.0.1:6380> set k1 a
OK
127.0.0.1:6380> get k1
"a"
127.0.0.1:6380> hset k2 f1 a
(integer) 1
127.0.0.1:6380> hget k2 f1
"a"
```

## 尾声

现在已经有一些公司基于 TiKV 来构建了他们自己的 Redis Server，并且也有一个开源的项目 [tidis](https://github.com/yongman/tidis) 做了相同的事情。`tidis` 已经比较完善，如果你想替换自己的 Redis，可以尝试一下。
正如同你所见，TiKV 其实算是一个基础的组件，我们可以在它的上面构建很多其他的应用。如果你对我们现在做的事情感兴趣，欢迎联系我：tl@pingcap.com。