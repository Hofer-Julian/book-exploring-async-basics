# What is Node?

We have to start with a short explanation of what Node is, just so we're on the same page.

Node is a Javascript runtime allowing Javascript to run on your desktop (or server). Javascript was originally designed as a scripting language for the browser, which means that it relies on the browser to both interpret it and provide a runtime for it.

This also means that Javascript on the desktop needs to be both interpreted (or compiled) and provided with a runtime to be able to do anything meaningful. On the desktop, the [V8 javascript engine](https://en.wikipedia.org/wiki/V8_JavaScript_engine) compiles Javascript, and [Node](https://en.wikipedia.org/wiki/Node.js) provides the runtime.

Javascript has one advantage from a language design perspective: Everything is designed to be handled asynchronously. And as you know by now, this is pretty crucial if we want to make the most out of our hardware, especially if you have a lot of I/O operations to take care of.

One such scenario is a Web server. Web servers handle a lot of I/O tasks whether it's reading from the file system or communicating via the network card.

## Why Node In Particular?

- Javascript is unavoidable when doing web development for the browser. Using Javascript on the server allows programmers to use the same language for both front-end and back-end development.
- There is a potential for code reuse between the back-end and the front-end
- The design of Node allows it to make very performant web servers
- Working with Json and web APIs is very easy when you only deal with Javascript

## Helpful Facts

Let's start off by debunking some myths that might make it easier to follow along when we start to code.

### The Javascript Event Loop

Javascript is a scripting language and can't do much on its own. It doesn't have an event loop. Now in a web browser, the browser provides a runtime, which includes an event loop. And on the server, Node provides this functionality.

You might say that Javascript as a language would be difficult to run (due to its callback-based model) without some sort of event loop, but that's beside the point.

### Node is Multithreaded

Contrary to what I've seen claimed on several occasions, Node uses a thread pool so it's multithreaded. However, the part of Node that "progresses" your code, does indeed run on a single thread. When we say "don't block the event loop" we're referring to this thread since that will prevent Node from making progress on other tasks.

We'll see exactly why blocking this thread is a problem and how that's handled.

### The V8 Javascript Engine

Now, this is where we need to focus a bit. The V8 engine is a javascript JIT compiler. That means that when you write a `for` loop, the V8 engine translates this to instructions that run on your CPU. There are many javascript engines, but Node was originally implemented on top of the V8 engine.

The V8 engine itself isn't very useful for us; it just interprets our Javascript. It can't do I/O, set up a runtime or anything like that. Writing Javascript only with V8 will be a very limited experience.

> Since we write Rust (even though we made it look a bit "javascripty"), we'll not cover the part of interpreting javascript. We'll just focus on how Node works and how it handles concurrency since that's our main focus right now.

### Nodes Event Loop(s)

Node internally divides its real work into two categories:

#### I/O-bound tasks

Tasks that mainly wait for some external event to occur are handled by the cross platform epoll/kqueue/IOCP event queue implemented in `libuv` and in our case `minimio`.

#### CPU-bound tasks

Tasks that are predominately CPU intensive are handled by a thread pool. The default size of this thread pool is 4 threads, but that can be configured by the Node runtime.

I/O tasks which can't be handled by the cross platform event queue are also handled here, which is the case with file reads that we use in our example.

Most C++ extensions for Node uses this thread pool to perform their work, and that is one of many reasons they are used for calculation-intensive tasks.

## Further Information

If you do want to know more about the Node event loop, I have one short page of the `libuv` documentation I can
refer you to and two talks for you that I find great (and correct) on this subject:

[Libuv Design Overview](http://docs.libuv.org/en/v1.x/design.html#design-overview)

This first talk one is held by [@piscisaureus](https://github.com/piscisaureus) and is an excellent 15 minute overview. I especially recommend this one as its short and to the point.
<iframe width="560" height="315" src="https://www.youtube.com/embed/PNa9OMajw9w" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


The second one is slightly longer but is also an excellent talk held by [Bryan Hughes](https://github.com/nebrius)
<iframe width="560" height="315" src="https://www.youtube.com/embed/zphcsoSJMvM" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


Now, relax, get a cup of tea and sit back while we go through everything together.

