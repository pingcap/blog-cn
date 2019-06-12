---
title: TiKV 源码解析系列文章（八）grpc-rs 的封装与实现
author: ['李建俊']
date: 2019-06-12
summary: 本篇将带大家深入到 grpc-rs 这个库里，查看 RPC 请求是如何被封装和派发的，以及它是怎么和 Rust Future 进行结合的。
tags: ['TiKV 源码解析','社区']
---

上一篇《[gRPC Server 的初始化和启动流程](https://pingcap.com/blog-cn/tikv-source-code-reading-7/)》为大家介绍了 gRPC Server 的初始化和启动流程，本篇将带大家深入到 [grpc-rs](https://github.com/pingcap/grpc-rs) 这个库里，查看 RPC 请求是如何被封装和派发的，以及它是怎么和 Rust Future 进行结合的。

## gRPC C Core

gRPC 包括了一系列复杂的协议和流控机制，如果要为每个语言都实现一遍这些机制和协议，将会是一个很繁重的工作。因此 gRPC 提供了一个统一的库来提供基本的实现，其他语言再基于这个实现进行封装和适配，提供更符合相应语言习惯或生态的接口。这个库就是 gRPC C Core，grpc-rs 就是基于 gRPC C Core 进行封装的。

要说明 grpc-rs 的实现，需要先介绍 gRPC C Core 的运行方式。gRPC C Core 有三个很关键的概念 `grpc_channel`、`grpc_completion_queue`、`grpc_call`。`grpc_channel` 在 RPC 里就是底层的连接，`grpc_completion_queue` 就是一个处理完成事件的队列。`grpc_call` 代表的是一个 RPC。要进行一次 RPC，首先从 `grpc_channel` 创建一个 grpc_call，然后再给这个 `grpc_call` 发送请求，收取响应。而这个过程都是异步，所以需要调用 `grpc_completion_queue` 的接口去驱动消息处理。整个过程可以通过以下代码来解释（为了让代码更可读一些，以下代码和实际可编译运行的代码有一些出入）。


```rust
grpc_completion_queue* queue = grpc_completion_queue_create_for_next(NULL);
grpc_channel* ch = grpc_insecure_channel_create("example.com", NULL);
grpc_call* call = grpc_channel_create_call(ch, NULL, 0, queue, "say_hello");
grpc_op ops[6];
memset(ops, 0, sizeof(ops));
char* buffer = (char*) malloc(100);
ops[0].op = GRPC_OP_SEND_INITIAL_METADATA;
ops[1].op = GRPC_OP_SEND_MESSAGE;
ops[1].data.send_message.send_message = "gRPC";
ops[2].op = GRPC_OP_SEND_CLOSE_FROM_CLIENT;
ops[3].op = GRPC_OP_RECV_INITIAL_METADATA;
ops[4].op = GRPC_OP_RECV_MESSAGE;
ops[4].data.recv_message.recv_message = buffer;
ops[5].op = GRPC_OP_RECV_STATUS_ON_CLIENT;
void* tag = malloc(1);
grpc_call_start_batch(call, ops, 6, tag);
grpc_event ev = grpc_completion_queue_next(queue);
ASSERT_EQ(ev.tag, tag);
ASSERT(strcmp(buffer, "Hello gRPC"));
```

可以看到，对 `grpc_call` 的操作是通过一次 `grpc_call_start_batch` 来指定的。这个 start batch 会将指定的操作放在内存 buffer 当中，然后通过 `grpc_completion_queue_next` 来实际执行相关操作，如收发消息。这里需要注意的是 `tag` 这个变量。当这些操作都完成以后，`grpc_completion_queue_next` 会返回一个包含 tag 的消息来通知这个操作完成了。所以在代码的末尾就可以在先前指定的 `buffer` 读出预期的字符串。

由于篇幅有限，对于 gRPC C Core 的解析就不再深入了，对这部分很感兴趣的朋友也可以在 [github.com/grpc/grpc](https://github.com/grpc/grpc) 阅读相关文档和源码。

## 封装与实现细节

通过上文的分析可以明显看到，gRPC C Core 的通知机制其实和 Rust Future 的通知机制非常类似。Rust Future 提供一个 poll 方法来检验当前 Future 是否已经 ready。如果尚未 ready，poll 方法会注册一个通知钩子 `task`。等到 ready 时，`task` 会被调用，从而触发对这个 Future 的再次 poll，获取结果。`task` 其实和上文中的 `tag` 正好对应起来了，而在 grpc-rs 中，`tag` 就是一个储存了 `task` 的 enum。

```rust
pub enum CallTag {
   Batch(BatchPromise),
   Request(RequestCallback),
   UnaryRequest(UnaryRequestCallback),
   Abort(Abort),
   Shutdown(ShutdownPromise),
   Spawn(SpawnNotify),
}
```

`tag` 之所以是一个 enum 是因为不同的 call 会对应不同的行为，如对于服务器端接受请求的处理和客户端发起请求的处理就不太一样。

grpc-rs 在初始化时会创建多个线程来不断调用 `grpc_completion_queue_next` 来获取已经完成的 `tag`，然后根据 `tag` 的类型，将数据存放在结构体中并通知 `task` 来获取。下面是这个流程的代码。

```rust
// event loop
fn poll_queue(cq: Arc<CompletionQueueHandle>) {
   let id = thread::current().id();
   let cq = CompletionQueue::new(cq, id);
   loop {
       let e = cq.next();
       match e.event_type {
           EventType::QueueShutdown => break,
           // timeout should not happen in theory.
           EventType::QueueTimeout => continue,
           EventType::OpComplete => {}
       }

       let tag: Box<CallTag> = unsafe { Box::from_raw(e.tag as _) };

       tag.resolve(&cq, e.success != 0);
   }
}
```

可以看到，`tag` 会被强转成为一个 `CallTag`，然后调用 `resolve` 方法来处理结果。不同的 enum 类型会有不同的 `resolve` 方式，这里挑选其中 `CallTag::Batch` 和 `CallTag::Request` 来进行解释，其他的 `CallTag` 流程类似。

`BatchPromise` 是用来处理上文提到的 `grpc_call_start_batch` 返回结果的 `tag`。`RequestCallback` 则用来接受新的 RPC 请求。下面是 `BatchPromise` 的定义及其 `resolve` 方法。

```rust
/// A promise used to resolve batch jobs.
pub struct BatchPromise {
   ty: BatchType,
   ctx: BatchContext,
   inner: Arc<Inner<Option<MessageReader>>>,
}

impl BatchPromise {
   fn handle_unary_response(&mut self) {
       let task = {
           let mut guard = self.inner.lock();
           let status = self.ctx.rpc_status();
           if status.status == RpcStatusCode::Ok {
               guard.set_result(Ok(self.ctx.recv_message()))
           } else {
               guard.set_result(Err(Error::RpcFailure(status)))
           }
       };
       task.map(|t| t.notify());
   }

   pub fn resolve(mut self, success: bool) {
       match self.ty {
           BatchType::CheckRead => {
               assert!(success);
               self.handle_unary_response();
           }
           BatchType::Finish => {
               self.finish_response(success);
           }
           BatchType::Read => {
               self.read_one_msg(success);
           }
       }
   }
}
```

上面代码中的 `ctx` 是用来储存响应的字段，包括响应头、数据之类的。当 `next` 返回时，gRPC C Core 会将对应内容填充到这个结构体里。`inner` 储存的是 `task` 和收到的消息。当 `resolve` 被调用时，先判断这个 `tag` 要执行的是什么任务。`BatchType::CheckRead` 表示是一问一答式的读取任务，`Batch::Finish` 表示的是没有返回数据的任务，`BatchType::Read` 表示的是流式响应里读取单个消息的任务。拿 `CheckRead` 举例，它会将拉取到的数据存放在 `inner` 里，并通知 `task`。而 `task` 对应的 Future 再被 poll 时就可以拿到对应的数据了。这个 Future 的定义如下：

```rust
/// A future object for task that is scheduled to `CompletionQueue`.
pub struct CqFuture<T> {
    inner: Arc<Inner<T>>,
}

impl<T> Future for CqFuture<T> {
    type Item = T;
    type Error = Error;

    fn poll(&mut self) -> Poll<T, Error> {
        let mut guard = self.inner.lock();
        if guard.stale {
            panic!("Resolved future is not supposed to be polled again.");
        }

        if let Some(res) = guard.result.take() {
            guard.stale = true;
            return Ok(Async::Ready(res?));
        }

        // So the task has not been finished yet, add notification hook.
        if guard.task.is_none() || !guard.task.as_ref().unwrap().will_notify_current() {
            guard.task = Some(task::current());
        }

        Ok(Async::NotReady)
    }
}
```

`Inner` 是一个 `SpinLock`。如果在 poll 时还没拿到结果时，会将 `task` 存放在锁里，在有结果的时候，存放结果并通过 `task` 通知再次 poll。如果有结果则直接返回结果。

下面是 `RequestCallback` 的定义和 `resolve` 方法。

```rust
pub struct RequestCallback {
   ctx: RequestContext,
}

impl RequestCallback {
   pub fn resolve(mut self, cq: &CompletionQueue, success: bool) {
       let mut rc = self.ctx.take_request_call_context().unwrap();
       if !success {
           server::request_call(rc, cq);
           return;
       }

       match self.ctx.handle_stream_req(cq, &mut rc) {
           Ok(_) => server::request_call(rc, cq),
           Err(ctx) => ctx.handle_unary_req(rc, cq),
       }
   }
}
```

上面代码中的 `ctx` 是用来储存请求的字段，主要包括请求头。和 `BatchPromise` 类似，`ctx` 的内容也是在调用 `next` 方法时被填充。在 `resolve` 时，如果失败，则再次调用 `request_call` 来接受下一个 RPC，否则会调用对应的 RPC 方法。

`handle_stream_req` 的定义如下：

```rust
pub fn handle_stream_req(
   self,
   cq: &CompletionQueue,
   rc: &mut RequestCallContext,
) -> result::Result<(), Self> {
   let handler = unsafe { rc.get_handler(self.method()) };
   match handler {
       Some(handler) => match handler.method_type() {
           MethodType::Unary | MethodType::ServerStreaming => Err(self),
           _ => {
               execute(self, cq, None, handler);
               Ok(())
           }
       },
       None => {
           execute_unimplemented(self, cq.clone());
           Ok(())
       }
   }
}
```

从上面可以看到，整个过程先通过 `get_handler`，根据 RPC 想要执行的方法名字拿到方法并调用，如果方法不存在，则向客户端报错。可以看到这里对于 `Unary` 和 `ServerStreaming` 返回了错误。这是因为这两种请求都是客户端只发一次请求，所以返回错误让 `resolve` 继续拉取消息体然后再执行对应的方法。

为什么 `get_handler` 可以知道调用的是什么方法呢？这是因为 gRPC 编译器在生成代码里对这些方法进行了映射，具体的细节在生成的 `create_xxx_service` 里，本文就不再展开了。

## 小结

最后简要总结一下 grpc-rs 的封装和实现过程。当 grpc-rs 初始化时，会创建数个线程轮询消息队列（`grpc_completion_queue`）并 `resolve`。当 server 被创建时，RPC 会被注册起来，server 启动时，grpc-rs 会创建数个 `RequestCall` 来接受请求。当有 RPC 请求发到服务器端时，`CallTag::Request` 就会被返回并 `resolve`，并在 `resolve` 中调用对应的 RPC 方法。而 client 在调用 RPC 时，其实都是创建了一个 Call，并产生相应的 `BatchPromise` 来异步通知 RPC 方法是否已经完成。

还有很多 grpc-rs 的源码在我们的文章中暂未涉及，其中还有不少有趣的技巧，比如，如何减少唤醒线程的次数而减少切换、如何无锁地注册调用各个 service 钩子等。欢迎有好奇心的小伙伴自行阅读源码，也欢迎大家提 issue 或 PR 一起来完善这个项目。
