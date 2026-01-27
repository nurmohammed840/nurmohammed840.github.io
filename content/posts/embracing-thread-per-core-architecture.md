+++
title = "Embracing Thread-Per-Core Architecture"
date = 2026-01-27
[taxonomies]
categories = ["rust"]
tags = ["async", "runtime", "nio"]
[extra]
lang = "en"
toc = true
+++


# What is *"thread-per-core"* ?

The term *"thread-per-core"* is confusing. It implies that there is one thread for each CPU core.

Aren't most modern multi-threaded async runtimes already following thread-per-core? Since most modern async runtimes use the [M:N threading](https://en.wikipedia.org/wiki/Thread_%28computing%29#M:N_.28hybrid_threading.29) model. It spawns multiple `N` OS threads to handle `M` tasks concurrently.

However, the community refers to the term *"thread-per-core"* as a single-threaded runtime. Tasks are pinned to a specific thread.

Projects such as [monoio](https://github.com/bytedance/monoio), [compio](https://github.com/compio-rs/compio), and [glommio](https://github.com/DataDog/glommio) are advertised as thread-per-core runtimes.


# Why "thread-per-core" ?

Some algorithms only function correctly or work best in single-thread context. The task may use thread-local storage.

In this model, the tasks are pinned to a thread, which is a very powerful trait.
The data is allowed to be mutated using a shared reference.  Also, the CPU can execute more efficiently because data stays hot in the cache. 

What I really like about this model is that the runtime doesn’t need any scheduler or synchronization.


In a traditional work-stealing runtime, such as [Tokio](https://github.com/tokio-rs/tokio) require the task to be safe to `Send` across different threads in order to balance the workload.

Work stealing occurs only when a thread is idle. In practice, there are typically far more tasks than CPU cores, so threads almost always have some work to execute.

# Announcing Nio v0.1.0 🎉


I’m pretty excited to announce the first [stable release](https://docs.rs/nio/0.1.1/nio) of [Nio](https://github.com/nurmohammed840/nio). It was completely rewritten from scratch.
  
```toml
[dependencies]
nio = { version = "0.1", features = ["tokio-io"] }
```

By default, Nio implements async traits from [futures-io](https://docs.rs/futures-io/latest/futures_io/). But the optional `"tokio-io"` feature implements async traits from [`tokio::io`](https://docs.rs/tokio/latest/tokio/io).

Nio now support this folloing API.

## `nio::spawn_local` 

Spawn a `!Send` task that is pinned on the current thread.

```rust
let counter = Rc::new(Cell::new(0));
nio::spawn_local(async move {
  counter.update(|n| n + 1);
});
```

## `nio::spawn_pinned` 

Spawn a `!Send` task and pin it to a thread that has the least amount of work.
  
```rust
nio::spawn_pinned(|| async { ... });
```

Since we are spawning on a different thread. It accepts an async closure; captured variables have to be `Send`.
However, the actual task itself **does not** need to be `Send`. 

## `nio::spawn_pinned_at` 


Same as `nio::spawn_pinned`, but spawns the task pinned to a specific worker thread by its index. 

```rust
nio::spawn_pinned_at(1, || async { ... });
```

## `nio::spawn` 

Spawn a new task. The task is not pinned to a worker thread and has to be `Send`. The runtime may move the task to a different thread at every `.await` point.

This is the same as [`tokio::spawn`](https://docs.rs/tokio/latest/tokio/task/fn.spawn.html). Here is [the benchmark](https://github.com/nurmohammed840/nio/tree/main/benchmarks). If you are curious how it compares to `tokio::spawn`. 

---

The amazing part is that Nio supports all of these APIs without any config. There is no runtime flavor configuration either. Nio welcomes both [“thread-per-core” folks](https://maciej.codes/2022-06-09-local-async.html) and the rest of us.
 

# Architecture

Under the hood, [Nio](https://github.com/nurmohammed840/nio) uses a concurrent architecture called the [Actor model](https://en.wikipedia.org/wiki/Actor_model).

Nio uses multiple worker threads to execute tasks. Each worker has a shared and local queue.
The shared queue is used for:

* Spawn a task on a different thread (`nio::spawn_pinned`,  `nio::spawn`).
* In cross-thread wakeup, the shared queue is also used to pin a `!Send` task to a specific worker thread.
* If the task is spawned via [`nio::spawn`](https://docs.rs/nio/0.1.1/nio/fn.spawn.html), the runtime use shared queue to move the task to a different worker at every `.await` point.


The worker thread can access its local queue without any synchronization. However, the async API also needs to utilize the current thread runtime to avoid cross-thread wakeup. Consider this example: 

```rust
let listener = TcpListener::bind("127.0.0.1:0").await?;
loop {
  // accept connection on thread `A` 
  let mut socket = listener.accept().await?;
  nio::spawn_pinned(|| async move {
    // using tcp connection on thread `B`
    let _ = socket.read(..).await?; 
  });
}
```

We accept tcp connection on worker `A`. Therefore, every TCP readiness happened on thread `A`, but the socket is waiting on thread `B`. 

In other words, thread `A` wakes up a task that is waiting on thread `B`. This is not good, as it requires the use of shared queue synchronization. It destroys the benefit of using the so-called *“thread-per-core”* runtime in the first place.

```rust
let listener = TcpListener::bind("127.0.0.1:0").await?;
loop { 
  let mut conn = listener.accept().await?;
  nio::spawn_pinned(|| async move {
    // accept connection on thread `B` 
    let socket = conn.connect().await?;
    let _ = socket.read(..).await?;
  });
}
```

In this example, by using the `connect()` method, we bind the TCP connection on thread `B`, ensuring that all TCP readiness events occur on thread `B` instead of thread `A`.

To avoid cross-thread wakeup, Nio does not allow I/O resources (such as TCP connections or timers) to be moved between threads. In other words, I/O resources do not implement `Send`. This enables Nio to provide a highly efficient async API, as I/O resources can be implemented without atomic operations. This design really benefits the Timer implementation. Timer algorithms are inherently single-threaded. Nio timer doesn’t require any mutex lock.

A worker can be seen as an independent async runtime. In a sense, Workers do not share any state, reactor, and timer with each other.


The Nio project is broken into separate, smaller crates. So the community can benefit from it:

* [nio-task](https://crates.io/crates/nio-task): Low-level task abstraction.
* [nio-threadpool](https://crates.io/crates/nio-threadpool): Thread pool implementation for nio runtime.