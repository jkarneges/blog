---
layout: post
title: Context reactor hook
date: 2022-12-28 21:27:00
author: justin
categories: programming
---

Async Rust interoperability is an open issue. This article recommends an approach to enable applications to run arbitrary futures without having to be explicitly designed for them.

## The Future contract

A key feature of [`Future`][future-trait] is its [`Waker`][waker] abstraction, which helps decouple the execution of the future from any evented I/O machinery. The most intuitive interpretation of the API contract is that the `Waker` will be triggered out-of-band, either from another thread or perhaps from a signal handler. Such an abstraction would be sufficient for applications to run arbitrary futures concurrently, regardless of where the futures come from.

For example, an application could mix futures from different third-party crates and drive them from the same thread:

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::Arc;
use std::task::{Context, Wake};
use std::thread::{self, Thread};

struct ThreadWaker(Thread);

impl Wake for ThreadWaker {
    fn wake(self: Arc<Self>) {
        self.0.unpark();
    }
}

fn main() {
    let mut futs: Vec<Pin<Box<dyn Future<Output = ()>>>> = Vec::new();

    // initialize 3 arbitrary futures
    futs.push(Box::pin(crate1::make_future()));
    futs.push(Box::pin(crate2::make_future()));
    futs.push(Box::pin(crate3::make_future()));

    let t = thread::current();
    let waker = Arc::new(ThreadWaker(t)).into();
    let mut cx = Context::from_waker(&waker);

    // execute the futures concurrently with code that
    // is not specific to any of the futures
    while !futs.is_empty() {
        let mut i = 0;

        while i < futs.len() {
            if futs[i].as_mut().poll(&mut cx).is_ready() {
                futs.remove(i);
                continue;
            }

            i += 1;
        }

        if !futs.is_empty() {
            thread::park();
        }
    }
}
```

However, in practice, futures tend to require custom assistance from the threads that drive them. Most futures do not say, "I'll autonomously trigger my waker in the background when I can make progress", as the above code assumes, but rather they say "I expect you to know to do something for me in order for me to determine if I can make progress".

For example, futures produced by [`tokio::net::TcpStream`][tokio-tcpstream] (e.g. by calling `read`) will only work if they are executed by [`tokio::runtime::Runtime`][tokio-runtime]. This is because `Runtime` interleaves calls to `Future::poll` with calls to an I/O driver to wait for events. Polling the futures directly would not be sufficent, as their wakers would never get triggered.

Unfortunately, this reliance on special behavior in the execution thread somewhat defeats the waker abstraction.

## Reactors and exclusivity

The core of the problem is futures typically require the presence of a shared component to handle I/O registrations, and they typically require some logic of this component to be invoked in the execution thread. Let's call this component a "reactor".

Sharing I/O registrations in order to wait for events from one thread makes sense. Some kinds of futures could be entirely self-sufficient and not rely on a shared reactor, for example a future that spawns a thread to perform a CPU-intensive task, but most futures typically need to wait for I/O operations or timers and should not be spawning their own threads. Indeed the whole point of `Future` is to avoid spawning a thread for every task.

Just because multiple futures should share a reactor doesn't mean reactor logic needs to be co-mingled in the execution thread, though. A reactor is arguably an implementation detail of futures, as a way to optimize resource usage. For example, if a bunch of different futures all require a reactor based on Linux's `epoll`, the first future could spawn a thread to run the reactor, and then all subsequent futures could share that reactor. The execution thread wouldn't need to include any custom logic specific to the futures, neither during initialization nor interleaved between calls to `Future::poll`.

Multiple, different shared reactors could even coexist in the same application. For example, there could be exactly one `epoll` reactor, exactly one `io_uring` reactor, exactly one `SIGALRM` reactor, etc, each with their own threads as applicable, created on demand, and used by whichever futures need them. Notably, though, multiple reactors would not be able to share the same thread, or at least they wouldn't be able to block to wait for I/O events from the same thread[^1]. In this sense, reactors that need to block for events require thread exclusivity.

That said, reactor logic typically *does* need to be co-mingled in the execution thread. This is especially true for single-threaded or thread-per-core architectures, which require polling and waking to occur in the same thread, but it can also be true for multi-threaded architectures that want to reuse sleeping threads to wait for events. Since reactors require thread exclusivity, a reactor that requires special behavior in the execution thread will require exclusivity in that thread.

## Extending Future

If we want applications to be able to drive arbitrary futures, we need to extend the `Future` contract to include a way for futures to tell the execution thread to do special things.

Specifically, futures need to be able to say, "I need you to call a blocking function in order to potentially trigger my waker". Notably, a future would not ask the executor to call a non-blocking function, as that would spin the CPU and defeat the point of async. It would also not ask the executor to call a function that would never trigger the waker, as the task would then never make progress, again defeating the point of async. Thus, we don't need an overly generalized hook here. We need a **blocking** function that triggers **wakers**.

We could add a hook to [`std::task::Context`][context] allowing a polled future to optionally provide two functions to the caller:

* A function that will block and potentially trigger the future's waker, to be used when all futures managed by the caller have returned `Poll::Pending`.

* A function that will unblock a call to the above, so that any futures not dependent on the above function can make progress.

It should be possible to efficiently check if these functions are already set to expected values, in order to avoid doing unnecessary initialization work if they are already set (similar to [`Waker::will_wake`][waker-will_wake]). Futures will want to be able to error out if the functions are set to unexpected values. It should also be made easy to propagate the values when `Context` is nested.

The `Context` thus becomes bidirectional. The caller uses it to provide a `Waker` to the future, and the future uses it to provide some functions back to the caller.

## Only one configured reactor allowed

Futures would only be able to configure one set of functions per execution context. If an application instantiates multiple futures that each require different reactors, and each reactor requires special behavior in the execution thread, then only one reactor would "win". Futures requiring any of the other reactors would be expected to fail, for example by resolving to an error or panicking.

Thus, while the approach described in this article could help developers avoid writing execution code that is specific to any futures, they must still take care to not mix futures that rely on conflicting reactors.

To avoid conflicts among reactors that require exclusivity in the execution thread, the Rust community would need to agree on a common reactor and only use that one. This is an orthogonal problem.

## Relationship to async main

There is interest in letting developers declare an `async fn main` in order to quickly write async Rust code with minimal boilerplate. The challenge is deciding how to implement such a thing when, as discussed above, most futures require custom logic in the exection thread. The common sentiment is it should be possible to declare a desired implementation via global directives or cargo settings. However, adding a hook to `Context` may be enough to solve the problem without involving globals. There could be a single built-in implementation of `async fn main` able to execute any futures.

## Footnotes

[^1]: This isn't to say that a single blocking call couldn't be used to monitor multiple I/O subsystems, e.g. with OS-specific tricks like chaining file descriptors, but it may be impractical to do this in an abstract way, and such an implementation would probably be better presented as a single, comprehensive reactor rather than as multiple reactors somehow composed.

[rust-lang]: https://www.rust-lang.org/
[future-trait]: https://doc.rust-lang.org/std/future/trait.Future.html
[waker]: https://doc.rust-lang.org/std/task/struct.Waker.html
[waker-will_wake]: https://doc.rust-lang.org/std/task/struct.Waker.html#method.will_wake
[tokio-tcpstream]: https://docs.rs/tokio/latest/tokio/net/struct.TcpStream.html
[tokio-runtime]: https://docs.rs/tokio/latest/tokio/runtime/struct.Runtime.html
[context]: https://doc.rust-lang.org/std/task/struct.Context.html
