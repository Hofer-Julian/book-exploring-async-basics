# Epoll, Kqueue and IOCP

There are some well-known libraries which implement a cross platform event queue using Epoll, Kqueue and IOCP for Linux, Mac, and Windows, respectively.

Part of Node's runtime is based on [libuv](https://github.com/libuv/libuv), which is a cross platform
asynchronous I/O library. `libuv` is not only used in Node but also forms the foundation
of how [Julia](https://julialang.org/) and [Pyuv](https://github.com/saghul/pyuv) create a cross platform event queue; most
languages have bindings for it.

In Rust we have [mio - Metal IO](https://github.com/tokio-rs/mio). `Mio` powers the OS event queue used in [Tokio](https://github.com/tokio-rs/tokio), which is a runtime that provides I/O, networking, scheduling etc. `Mio` is to `Tokio` what `libuv` is to `Node`. 

`Tokio` powers many web frameworks, among those is [Actix Web](https://github.com/actix/actix-web), which is known to be very performant.

Since we want
to understand how everything works, I decided to create an extremely
simplified version of an event queue. I called it `minimio` since it's greatly inspired by `mio`.

> I will write a short book (much shorter than this one) about how this works in
> detail; for now you can visit the code at its [Github repository if you're
> curious](https://github.com/cfsamson/examples-minimio). This book will also briefly cover
> `wepoll`, which is used as an optimization instead of IOCP in both the `mio` and `libuv` frameworks. 

Nevertheless, we'll give each of them a brief introduction here so you know the basics.

## Why use an OS-backed event queue?

If you remember my previous chapters, you know that we need to cooperate closely
with the OS to make I/O operations as efficient as possible. Operating systems like
Linux, MacOS and Windows provide several ways of performing I/O, both blocking and
non-blocking.

So blocking operations are the least flexible to use for us as programmers since we yield control to the OS, which suspends our thread. The big advantage is that our thread gets woken up once the event we're waiting for is ready.

Non-blocking methods are more flexible but need to have a way to tell us if a task is ready or not. This is most often done by returning some kind of data that says if it's `Ready` or `NotReady`. One drawback is that we need to check this status regularly to be able to tell if the state has changed. 

Event queuing via Epoll/kqueue/IOCP is a way to combine the flexibility of a non-blocking method without its aforementioned drawback.

> We will not cover methods like `poll` and `select`, but I have an [article for you
> here](https://people.eecs.berkeley.edu/~sangjin/2012/12/21/epoll-vs-kqueue.html)
> if you want to learn a bit about these methods and how they differ from `epoll`.

## Readiness-based event queues

Epoll and Kqueue are known as readiness-based event queues; this is because they let you know when an action is ready to be performed, e.g., when a socket is ready to be read from.

**Basically this happens when we want to read data from a socket using epoll/kqueue:**

1. We create an event queue by calling the syscall `epoll_create` or `kqueue`.
2. We ask the OS for a file descriptor representing a network socket.
3. Through another syscall, we register an interest in `Read` events on this socket. It's important that we also inform the OS that we'll be expecting to receive a notification when the event is ready in the event queue we created in (1).
4. Next, we call `epoll_wait` or `kevent` to wait for an event. This will block (suspend) the thread it's called on.
5. When the event is ready, our thread is unblocked (resumed), and we return from our "wait" call with data about the event that occurred.
6. We call `read` on the socket we created in 2.

## Completion-based event queues

IOCP stands for I/O Completion Ports; in this type of queue you get a
notification when events are completed, e.g., when data is read to a buffer.

Below is a basic breakdown of what happens in this type of event queue:

1. We create an event queue by calling the syscall `CreateIoCompletionPort`.
2. We create a buffer and ask the OS to give us a handle to a socket.
3. We register an interest in `Read` events on this socket with another syscall,
   but this time we also pass in the buffer we created in (2) to which the data will
   be read.
4. Next, we call `GetQueuedCompletionStatusEx`, which will block until an event has
   completed.
5. Our thread is unblocked, and our buffer is now filled with the data we're interested in.


## Epoll 

`Epoll` is the Linux way of implementing an event queue. In terms of functionality, it has a lot in common with `Kqueue`. The advantage of using `epoll` over other similar methods on Linux like `select` or `poll` is that `epoll` was designed to work very efficiently with a large number of events.

### Kqueue

`Kqueue` is the MacOS way of implementing an event queue, which originated from BSD, in operating systems such as FreeBSD, OpenBSD, etc. In terms of high level functionality,
it's similar to `Epoll` in concept but different in actual use.

Some argue it's a bit more complex to use and a bit more abstract and "general".

### IOCP

`IOCP` or Input Output Completion Ports is the way Windows handles this type of event queue. 

A `Completion Port` will let you know when an event has `Completed`. Now this might
sound like a minor difference, but it's not. This is especially apparent when you want to write a library since abstracting over both means you'll either have to model `IOCP` as `readiness-based` or model `epoll/kqueue` as completion-based.

Lending out a buffer to the OS also provides some challenges since it's very
important that this buffer stays untouched while waiting for an operation to
return.

> My experience investigating this suggests that getting `readiness-based`
> models to behave like the `completion-based` models is easier than the other
> way around. This means you should get IOCP to work first and then fit `epoll` or `kqueue`
> into that design.
