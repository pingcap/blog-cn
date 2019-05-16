---
title: TiKV 源码解析系列文章（七）gRPC Server 的初始化和启动流程
author: ['屈鹏']
date: 2019-05-16
summary: 本篇 TiKV 源码解析将为大家介绍 TiKV 的另一周边组件—— grpc-rs。grpc-rs 是 PingCAP 实现的一个 gRPC 的 Rust 绑定，其 Server/Client 端的代码框架都基于 Future，事件驱动的 EventLoop 被隐藏在了库的内部，所以非常易于使用。
tags: ['TiKV 源码解析','社区']
---


本篇 TiKV 源码解析将为大家介绍 TiKV 的另一周边组件—— [grpc-rs](https://github.com/pingcap/grpc-rs/pulls)。grpc-rs 是 PingCAP 实现的一个 gRPC 的 Rust 绑定，其 Server/Client 端的代码框架都基于 [Future](https://docs.rs/futures/0.1.26/futures/)，事件驱动的 EventLoop 被隐藏在了库的内部，所以非常易于使用。本文将以一个简单的 gRPC 服务作为例子，展示 grpc-rs 会生成的服务端代码框架和需要服务的实现者填写的内容，然后会深入介绍服务器在启动时如何将后台的事件循环与这个框架挂钩，并在后台线程中运行实现者的代码。

## 基本的代码生成及服务端 API

gRPC 使用 protobuf 定义一个服务，之后调用相关的代码生成工具就可以生成服务端、客户端的代码框架了，这个过程可以参考我们的 [官方文档](https://github.com/pingcap/grpc-rs)。客户端可以直接调用这些生成的代码，向服务端发送请求并接收响应，而服务端则需要服务的实现者自己来定制对请求的处理逻辑，生成响应并发回给客户端。举一个例子：

```rust
#[derive(Clone)]
struct MyHelloService {}
impl Hello for MyHelloService {
    // trait 中的函数签名由 grpc-rs 生成，内部实现需要用户自己填写
    fn hello(&mut self, ctx: RpcContext, req: HelloRequest, sink: UnarySink<HelloResponse>) {
        let mut resp = HelloResponse::new();
        resp.set_to(req.get_from());
        ctx.spawn(
            sink.success(resp)
                .map(|_| println!("send hello response back success"))
                .map_err(|e| println!("send hello response back fail: {}", e))
        );
    }
}
```

我们定义了一个名为 `Hello` 的服务，里面只有一个名为 `hello` 的 RPC。grpc-rs 会为服务生成一个 trait，里面的方法就是这个服务包含的所有 RPC。在这个例子中唯一的 RPC 中，我们从 `HelloRequest` 中拿到客户端的名字，然后再将这个名字放到 `HelloResponse` 中发回去，非常简单，只是展示一下函数签名中各个参数的用法。

然后，我们需要考虑的是如何把这个服务运行起来，监听一个端口，真正能够响应客户端的请求呢？下面的代码片段展示了如何运行这个服务：

```rust
fn main() {
    // 创建一个 Environment，里面包含一个 Completion Queue
    let env = Arc::new(EnvBuilder::new().cq_count(4).build());
    let channel_args = ChannelBuilder::new(env.clone()).build_args();
    let my_service = MyHelloWorldService::new();
    let mut server = ServerBuilder::new(env.clone())
        // 使用 MyHelloWorldService 作为服务端的实现，注册到 gRPC server 中
        .register_service(create_hello(my_service))
        .bind("0.0.0.0", 44444)
        .channel_args(channel_args)
        .build()
        .unwrap();
    server.start();
    thread::park();
}
```

以上代码展示了 grpc-rs 的足够简洁的 API 接口，各行代码的意义如其注释所示。

## Server 的创建和启动

下面我们来看一下这个 gRPC server 是如何接收客户端的请求，并路由到我们实现的服务端代码中进行后续的处理的。

第一步我们初始化一个 Environment，并设置 Completion Queue（完成队列）的个数为 4 个。完成队列是 gRPC 的一个核心概念，grpc-rs 为每一个完成队列创建一个线程，并在线程中运行一个事件循环，类似于 Linux 网络编程中不断地调用 `epoll_wait` 来获取事件，进行处理：

```rust
// event loop
fn poll_queue(cq: Arc<CompletionQueueHandle>) {
    let id = thread::current().id();
    let cq = CompletionQueue::new(cq, id);
    loop {
        let e = cq.next();
        match e.event_type {
            EventType::QueueShutdown => break,
            EventType::QueueTimeout => continue,
            EventType::OpComplete => {}
        }
        let tag: Box<CallTag> = unsafe { Box::from_raw(e.tag as _) };
        tag.resolve(&cq, e.success != 0);
    }
}
```

事件被封装在 Tag 中。我们暂时忽略对事件的具体处理逻辑，目前我们只需要知道，当这个 Environment 被创建好之后，这些后台线程便开始运行了。那么剩下的任务就是监听一个端口，将网络上的事件路由到这几个事件循环中。这个过程在 Server 的 `start` 方法中：

```rust
/// Start the server.
pub fn start(&mut self) {
    unsafe {
        grpc_sys::grpc_server_start(self.core.server);
        for cq in self.env.completion_queues() {
            let registry = self
                .handlers
                .iter()
                .map(|(k, v)| (k.to_owned(), v.box_clone()))
                .collect();
            let rc = RequestCallContext {
                server: self.core.clone(),
                registry: Arc::new(UnsafeCell::new(registry)),
            };
            for _ in 0..self.core.slots_per_cq {
                request_call(rc.clone(), cq);
            }
        }
    }
}
```

首先调用 `grpc_server_start` 来启动这个 Server，然后对每一个完成队列，复制一份 handler 字典。这个字典的 key 是一个字符串，而 value 是一个函数指针，指向对这个类型的请求的处理函数——其实就是前面所述的服务的具体实现逻辑。key 的构造方式其实就是 `/<ServiceName>/<RpcName>`，实际上就是 HTTP/2 中头部字段中的 path 的值。我们知道 gRPC 是基于 HTTP/2 的，关于 gRPC 的请求、响应是如何装进 HTTP/2 的帧中的，更多的细节可以参考 [官方文档](https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md)，这里就不赘述了。

接着我们创建一个 `RequestCallContext`，然后对每个完成队列调用几次 `request_call`。这个函数会往完成队列中注册若干个 Call，相当于用 `epoll_ctl` 往一个 `epoll fd` 中注册一些事件的关注。Call 是 gRPC 在进行远程过程调用时的基本单元，每一个 RPC 在建立的时候都会从完成队列里取出一个 Call 对象，后者会在这个 RPC 结束时被回收。因此，在 `start` 函数中每一个完成队列上注册的 Call 个数决定了这个完成队列上可以并发地处理多少个 RPC，在 grpc-rs 中默认的值是 1024 个。

## 小结

以上代码基本都在 grpc-rs 仓库中的 `src/server.rs` 文件中。在 `start` 函数返回之后，服务端的初始化及启动过程便结束了。现在，可以快速地用几句话回顾一下：首先创建一个 Environment，内部会为每一个完成队列启动一个线程；接着创建 Server 对象，绑定端口，并将一个或多个服务注册到这个 Server 上；最后调用 Server 的 `start` 方法，将服务的具体实现关联到若干个 Call 上，并塞进所有的完成队列中。在这之后，网络上新来的 RPC 请求便可以在后台的事件循环中被取出，并根据具体实现的字典分别执行了。最后，不要忘记 `start` 是一个非阻塞的方法，调用它的主线程在之后可以继续执行别的逻辑或者挂起。

本篇源码解析就到这里，下篇关于 grpc-rs 的文章我们会进一步介绍一个 Call 或者 RPC 的生命周期，以及每一阶段在 Server 端的完成队列中对应哪一种事件、会被如何处理，这一部分是 grpc-rs 的核心代码，敬请期待！
