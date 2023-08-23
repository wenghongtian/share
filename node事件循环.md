### 简介

```
   ┌───────────────────────────┐
┌─>│           timers          │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤  connections, │
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │
   └───────────────────────────┘
```

* *timers*: 执行`setTimeout()`和`setInterval()`的回调
* *pending callbacks*: 对某些系统操作（如 TCP 错误类型）执行回调。例如，如果 TCP 套接字在尝试连接时接收到 ECONNREFUSED，则某些 *nix 的系统希望等待报告错误。这将被排队以在 挂起的回调 阶段执行。
* *idle, prepare*: nodejs内部使用
* *poll*: 获取新的IO事件；执行IO事件（除了`setImmediate`和计时的事件）；适当条件下`nodejs`会在这里阻塞
* *check*: 执行`setImmediate()`的回调
* *close callbacks*: 执行一些关闭的回调事件

### nodejs事件循环架构图

![架构图](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/11/18/16e7d83cb93ba59e~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

### nodejs各阶段详解

* **timers**阶段：
  一个timer指定一个下限时间而不是准确时间，在达到这个下限时间后执行回调。在指定时间过后，timers会尽可能早地执行回调，但系统调度或者其它回调的执行可能会延迟它们。
  <br/>

* **pending callbacks**阶段:
  执行一些系统操作的回调函数，比如当TCP套接字在连接时收到ECONNREFUSED类型的错误时。例如，在一些*nix系统中，会延迟报告该错误。此时会将该错误加入待处理的回调队列中，在后续的回调阶段被执行。
  <br/>

* **poll**阶段:
  主要做两件事:

  1. 计算应该阻塞和轮询IO的时间
  2. 处理轮询队列中的事件

  当事件循环进入**poll**阶段且没有**timers**时，会发生下面两件事之一:

  1. 如果**poll**队列不为空，事件循环将遍历其回调队列并同步执行它们，直到队列耗尽或达到系统相关的硬限制
  2. 如果**poll**队列为空，会发生以下两种情况之一：

   * 如果由`setImmediate()` 设置了回调，则事件循环会结束轮询阶段并进入到**check**阶段执行check对应的队列。
   * 如果没有通过`setImmediate()`设置回调，则事件循环将阻塞在这里，等待回调被添加到队列中，然后立即执行它们。

  > 当event loop进入 poll 阶段，并且 有设定的timers，一旦 poll 队列为空（poll 阶段空闲状态）：event loop将检查timers,如果有1个或多个timers的下限时间已经到达，event loop将绕回 timers 阶段，并执行 timer 队列。
  > <br/>

* **check**阶段：
  允许在**poll**阶段结束后立即执行回调。如果**poll**阶段空闲，并且有被`setImmediate()`设定的回调，event loop会转到**check**阶段而不是继续等待。

  > * setImmediate() 实际上是一个特殊的timer，跑在event loop中一个独立的阶段。它使用libuv的API 来设定在 poll 阶段结束后立即执行回调。
  > * 通常上来讲，随着代码执行，event loop终将进入**poll**阶段，在这个阶段等待 incoming connection, request 等等。但是，只要有被setImmediate()设定了回调，一旦**poll**阶段空闲，那么程序将结束**poll**阶段并进入**check**阶段，而不是继续等待**poll**事件 （poll events）。
  >   <br/>

* **close callbacks**阶段：
  如果一个 socket 或 handle 被突然关掉（比如 socket.destroy()），close事件将在这个阶段被触发，否则将通过process.nextTick()触发

### 微任务

`process.nextTick`和`Promise.then()`两者都是微任务，`nextTick`的优先级更高



## 事件循环的核心libuv

### libuv是什么

`libuv`是c语言实现的**单线程非阻塞异步io**解决方案，本质上它是对常见操作系统底层异步io操作的封装，并对外暴露功能一致的api，目的是尽可能为**nodejs**在不同系统平台上提供统一的事件循环模型。

> Node 是 V8 的宿主，它会给 V8 提供事件循环和消息队列。在 Node 中，事件循环是由 libuv 提供的，libuv 工作在主线程中，它会从消息队列中取出事件，并在主线程上执行事件。
> 同样，对于一些主线程上不适合处理的事件，比如消耗时间过久的网络资源下载、文件读写、设备访问等，Node 会提供很多线程来处理这些事件，我们把这些线程称为线程池。

### 线程池

* 耗时工作在工作线程完成，而工作的`callback`在主线程执行。
* 每一个`node`进程中，`libuv`都维护了一个线程池。
* 因为同处于一个进程，所以线程池中的所有线程都共享进程中的上下文。
* 线程池默认只有4个工作线程，用`UV_THREADPOOL_SIZE`常量控制。
* 并不是所有的操作都会使用线程池进行处理，只有`文件读取`、`dns查询`与`用户制定使用额外线程`的会使用线程池。

**nodejs**的事件循环核心逻辑如下[uv_run](https://github.com/libuv/libuv/blob/v1.35.0/src/unix/core.c#L365-L400)

```c
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  int timeout;
  int r;
  int ran_pending;

  r = uv__loop_alive(loop);
  if (!r)
    uv__update_time(loop);

  while (r != 0 && loop->stop_flag == 0) {
    uv__update_time(loop);
    uv__run_timers(loop);
    ran_pending = uv__run_pending(loop);
    uv__run_idle(loop);
    uv__run_prepare(loop);

    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);

    uv__io_poll(loop, timeout);
    uv__run_check(loop);
    uv__run_closing_handles(loop);

    if (mode == UV_RUN_ONCE) {
      /* UV_RUN_ONCE implies forward progress: at least one callback must have
       * been invoked when it returns. uv__io_poll() can return without doing
       * I/O (meaning: no callbacks) when its timeout expires - which means we
       * have pending timers that satisfy the forward progress constraint.
       *
       * UV_RUN_NOWAIT makes no guarantees about progress so it's omitted from
       * the check.
       */
      uv__update_time(loop);
      uv__run_timers(loop);
    }

    r = uv__loop_alive(loop);
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }

  /* The if statement lets gcc compile it to a conditional store. Avoids
   * dirtying a cache line.
   */
  if (loop->stop_flag != 0)
    loop->stop_flag = 0;

  return r;
}
```

上面代码中可以显而易见地看到时间循环对应的函数名

1. `timer`阶段：[uv__run_timers](https://github.com/libuv/libuv/blob/v1.35.0/src/timer.c#L159)
2. `pending callbakcs`阶段：[uv__run_pending](https://github.com/libuv/libuv/blob/v1.35.0/src/unix/core.c#L784)
3. `idle prepare`阶段:uv__run_idle、uv\_\_run\_prepare
4. `poll`阶段:[uv__io_poll](https://github.com/libuv/libuv/blob/v1.35.0/src/unix/linux-core.c#L201-L470)
5. `check`阶段:[uv__run_check](https://github.com/libuv/libuv/blob/v1.35.0/src/unix/loop-watcher.c#L67)
6. `close callbacks`阶段:[uv__run_closing_handles](https://github.com/libuv/libuv/blob/v1.35.0/src/unix/core.c#L303-L315)
