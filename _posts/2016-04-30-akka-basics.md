---
layout: post
name: Akka-basics
title: 'Akka basics'
date: 2016-04-30 00:20:00 +00:00
author: Felipe Fernández
image:
    src: /assets/img/custom/blog/akka.jpg
    attribution:
      text: Akka_20120331 by Helena Whitlock, SGU
      href: https://www.flickr.com/photos/geologicalsurveyofsweden/8703286181
tags:
- akka
- OOP
- functional programming

---

As [the previous post](http://codurance.com/2016/04/28/async-systems-with-sync-clients) explained our system is based on Akka. Before going into more detail about our solution, I'd like to explain what is Akka about and why is so awesome.

Akka is a toolkit and not a framework, you can simply use the bits that you need for your service. In this series we'll focus on the core features and we won't go through Akka Stream, Akka-Http or Akka Cluster (since they're out of scope, not because they're not fantastic).

Akka provides a different abstraction to deal with concurrency, paralellism and fault-tolerance. That abstraction is called Actor Model. If you have experience with old Java ecosystem you will know how hard is writing safe and correct multithreaded code. Providing human-friendly abstractions like Actors it's a revolution similar to what Java did with memory management.

## Actors all the way

Threads are an expensive resource, hence we need to use them judiciously. Blocking threads while we're waiting for some IO operation to respond is really inefficient. Actors use threads in a different way, as a result of which they're pretty lightweight ([several million actors per GB of heap memory](http://doc.akka.io/docs/akka/2.4.4/intro/what-is-akka.html))

Actors mix Object Oriented and Functional Programming principles. OOP is basically about messages as Alan Kay [stated](http://lists.squeakfoundation.org/pipermail/squeak-dev/1998-October/017019.html)

> I'm sorry that I long ago coined the term "objects" for this topic because it gets many people to focus on the lesser idea. The big idea is "messaging".

An actor exchanges immutable messages and keeps their own encapsulated state: every interaction needs to be done through messages.

## Fault tolerance

There is not a clear and shared error model in Java, and you can realise that after working in a couple of projects. The origin of that is a confusing abstraction called Exception and its subtypes Checked and Unchecked Exceptions. The only real difference is that checked expections force us to deal with the issue in the direct caller, except for that they don't hint at the strategy that we should apply.

I strongly recomment [this](http://joeduffyblog.com/2016/02/07/the-error-model/) post about it. A powerful idea in the post is that bugs aren't recoverable errors. This quote is contained on that post, [original source](https://wiki.haskell.org/Error_vs._Exception):

> I was involved in the development of a library that was written in C++. One of the developers told me that the developers are divided into the ones who like exceptions and the other ones who prefer return codes. As it seem to me, the friends of return codes won. However, I got the impression that they debated the wrong point: Exceptions and return codes are equally expressive, they should however not be used to describe errors. Actually the return codes contained definitions like ARRAY_INDEX_OUT_OF_RANGE. But I wondered: How shall my function react, when it gets this return code from a subroutine? Shall it send a mail to its programmer? It could return this code to its caller in turn, but it will also not know how to cope with it. Even worse, since I cannot make assumptions about the implementation of a function, I have to expect an ARRAY_INDEX_OUT_OF_RANGE from every subroutine. My conclusion is that ARRAY_INDEX_OUT_OF_RANGE is a (programming) error. It cannot be handled or fixed at runtime, it can only be fixed by its developer. Thus there should be no according return code, but instead there should be asserts.

Akka provides a great way for dealing with failures/errors based on these tenets:

* Single responsibility principle. Failure management is delegated to supervisors, creating focused business actors. We agreed some time ago that lifecycle management, e.g. object creation, should be moved to factories, therefore an object is not responsible for creating itself. Resuming or restarting an object, after some failure happened, is part of that lifecycle management and Akka enforces you to move that responsibility to supervisors.

* Default recovery strategies. It's important to understand which are our options after the system is under some failure condition. Is DB temporarily down? Has some input, in combination with the existing data, created an undesired state? Is that a bug? Do we need to abandon that particular request or that part of the system is messed up until some patch is applied? Those questions will determine our response to that incident, and Akka provides some [built-in strategies](http://doc.akka.io/docs/akka/2.4.4/scala/fault-tolerance.html#fault-tolerance-scala).

```
override val supervisorStrategy =
  OneForOneStrategy(maxNrOfRetries = 10, withinTimeRange = 1 minute) {
    case _: ArithmeticException      => Resume
    case _: NullPointerException     => Restart
    case _: IllegalArgumentException => Stop
    case _: Exception                => Escalate
  }
```

* Failure as first citizen. Some libraries or ecosystems hide failures through obscure APIs. Akka is usually deployed in distributed environments, therefore using unreliable resources like networks. That forces us to bring failure to the front page. Even in 'safer' environments as monoliths, failures are all around, so using toolkits like Akka is critical to create reliable and robust software.

## Concurrency and parallelism

With Akka we're not going to deal directly with threads, they're hidden under an abstraction layer. The backbone of an Akka app is the [Actor System](http://doc.akka.io/api/akka/2.0/akka/actor/ActorSystem.html):

> An actor system is a hierarchical group of actors which share common configuration, e.g. dispatchers, deployments, remote capabilities and addresses. It is also the entry point for creating or looking up actors.

A Dispatcher is also an Execution Context, so in the end is where the thread pool is located. Let's assume that your app uses a single dispatcher with 4 threads allocated. As you can see threads are a scarce resource, if we do blocking IO or heavy CPU work in some of the actors we'll use exclusively one of the threads, and you can see how quickly your service will run out of threads.

The solution is wrapping those operation with constructs like Scala Futures and then provide a different execution context for those tasks.

## Summary

Akka is a fantastic toolkit full of well suited abstractions for modern problems. In next post we'll see with code how we can coordinate and supervise actors with the constraints of our example application.

[Part 1](http://codurance.com/2016/04/28/async-systems-with-sync-clients/) | [Part 2](http://codurance.com/2016/04/30/akka-basics/)







