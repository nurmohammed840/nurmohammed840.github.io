+++
title = "Announcing Nio"
date = 2024-11-24
[taxonomies]
categories = ["rust"]
tags = ["async", "io", "runtime", "scheduler"]
[extra]
lang = "en"
toc = true
+++

[Nio](https://github.com/nurmohammed840/nio) is an experimental async runtime for Rust.\
This project initially began as an experiment to explore alternative scheduling strategies.

[Tokio](https://tokio.rs/) use [work-stealing](https://tokio.rs/blog/2019-10-scheduler#the-next-generation-tokio-scheduler)
scheduler, It is very complex scheduler and require a ton of bookkeeping.
Sadly, replacing the Tokio scheduler isn't as simple.

[Nio](https://github.com/nurmohammed840/nio) is designed with a modular architecture, Enables seamless switching between different scheduling algorithms.

## Finding Alternative Scheduler

In most thread-based schedulers, Each worker thread is associated with its own task queue.

A straightforward approach to assigning tasks would be to distribute them evenly across the task queues.
However, this approach doesn't work efficiently with a multi-threaded scheduler.
Some worker threads might become overloaded while others remain underutilized, leading to imbalanced workloads.
Another drawback is that if a single worker can handle all the tasks efficiently,
distributing them across multiple threads add unnecessary overhead.


### Least-Loaded (LL) Scheduler

The Least-Loaded scheduling algorithm is a simple yet effective strategy that addresses the issue of starvation.
It achieves this by assigning new task to the worker that currently has the least workload.

```rust
impl Scheduler {
    fn least_loaded_worker(&self) -> &Worker {
        self.workers.iter().min_by_key(|queue| queue.len.get()).unwrap()
    }

    fn schedule(&self, task: Task) {
        let worker = self.least_loaded_worker();
        worker.tx.send(task).unwrap();
        worker.len.inc();
    }
}

```

When a new task is waken to be re-assigned,
the scheduler assigns it to the worker that has the fewest tasks in its queue.

```rust
impl TaskQueue {
    fn fetch(&mut self) -> Option<Task> {
        ...
        let task = self.rx.recv().ok();
        self.len.dec();
        task
    }
}
```

Current [implementation](https://github.com/nurmohammed840/nio/blob/main/src/scheduler/least_loaded.rs) use `mpsc` channel, containing just 150 lines of code!

This scheduling statergy is simple, fast, solve starvation and minimizes context switching
(when a single worker can handle all tasks efficiently)

## Benchmark

The LL scheduler shows promising performance improvements.

```sh
> cd benchmarks
> cargo bench -F tokio --bench rt_multi_threaded
spawn_many_local         time: 3.0308 ms
spawn_many_remote_idle   time: 2.8096 ms
spawn_many_remote_busy1  time: 2.2446 ms
ping_pong                time: 345.60 µs
yield_many               time: 6.0869 ms
chained_spawn            time: 248.89 µs

> cargo bench --bench rt_multi_threaded
spawn_many_local         time: 2.1717 ms
spawn_many_remote_idle   time: 1.5445 ms
spawn_many_remote_busy1  time: 778.73 µs
ping_pong                time: 304.20 µs
yield_many               time: 1.0042 ms
chained_spawn            time: 97.996 µs
```


These days, whenever someone introduces a new runtime, it’s like a tradition to run an HTTP benchmark.
So, we honor the tradition with a  hyper ["Hello World"](https://github.com/nurmohammed840/nio/blob/main/example/hyper-server/main.rs) HTTP benchmark ceremony. Using:

```sh
CPU: Ryzen 7 5700 (8 cores, 16 threads)
OS:  Ubuntu 24 (WSL)
> wrk -t <2..=20> -c <50|100|500|1000> -d10 <URL>
```


{% note(header="") %} 
This benchmark is both **Meaningless** and **Misleading**,
as no real-world server would ever just respond with a simple "Hello, World" message.
{% end %}

![img](/posts/announcing-nio/hyper_bench_50_connections.png)
![img](/posts/announcing-nio/hyper_bench_100_connections.png)

> What's going on !? 😕

In work-stealing schedulers, when a local worker queue is empty,
it fetches a batch of tasks from the shared global (injector) queue. However, under high load,
multiple workers simultaneously attempt to steal tasks from injector queue.
This leads to increased contention and workers spend more time waiting...

let's increase the connection to reduce contention.

![img](/posts/announcing-nio/hyper_bench_500_connections.png)
![img](/posts/announcing-nio/hyper_bench_1000_connections.png)


## Conclusion

None of these benchmarks should be considered definitive measures of runtime performance.
That said, I believe there is potential for performance improvement.
I encourage you to experiment with this new scheduler and share the benchmarks from your real-world use-case.