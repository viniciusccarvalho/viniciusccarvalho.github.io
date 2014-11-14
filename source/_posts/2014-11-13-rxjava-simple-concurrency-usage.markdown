---
layout: post
title: "RXJava - Simple concurrency usage"
date: 2014-11-13 17:03:51 -0500
comments: true
categories: 
---

Simplifying concurrency
-----------------------

I've been playing with [RXJava](https://github.com/ReactiveX/RxJava/wiki) for a while now. Concurreny has always been a facinating topic to me. I read Brian Goetz's [Java Concurrency in practice](http://www.amazon.com/Java-Concurrency-Practice-Brian-Goetz/dp/0321349601) several times, and if you are a java developer and you haven't, shame on you :-(

This post is not meant to go on any discussion around concurrent vs parallel, read this: [fundamentals of concurrent programming](http://www.nada.kth.se/~snilsson/concurrency/) and watch Rob Pike's [Concurrency is not Parallelism](https://www.youtube.com/watch?v=cN_DpYBzKso)

Instead I just want to share some code I wrote to solve a simple problem that was taking me way more time than needed by using traditional java constructs to deal with parallism (Executors, BlockingQueues, Runnables and CountDownLatches)

### The problem

I'm building a demo to show how amazing [Spring Boot](http://projects.spring.io/spring-boot/), [Spring cloud](http://projects.spring.io/spring-cloud/) and [Cloud Foundry](http://www.pivotal.io/platform-as-a-service/pivotal-cf) are when combined together to build next generation applications.

The demo uses data from [The Movie Database project](https://www.themoviedb.org/). Collecting the data needs to respect a rate limit of 
3 requests per second. 

So the problem is: download a maximum number of data in parallel while respecting the limits of the API. After downloading the data, load it
on a mongo db collection, but transform it to store only parts of data.

Starting with the basics I created a class that transforms a mongodb cursor into an Observable:

{% gist bbada1e463baf61b2088 ObservableMongo.java %}

In order to control the limit of access per second I've relied on [Guava](https://code.google.com/p/guava-libraries/) Rate limiter, I'm 
assuming it uses some sort of [Token Bucket](http://en.wikipedia.org/wiki/Token_bucket) to control, but who cares :). Note that this is a 
form of Callstack blocking as discussed on [RXJava Backpressure documentation](https://github.com/ReactiveX/RxJava/wiki/Backpressure), it worked
fine for me for the moment:

{% gist c10bb46f02cdca9b5ee9 RateLimitedTemplate.java %}

And finally the small snippet that loads the data:

{% gist ba3a79089af37c2a51ed TMDBLoader.java %}

We start from a pre-populated DB with movies, and look for those which images have not been loaded yet. That returns our observable. From that
observable we use the parallel operator to execute the subscription on a separate thread. The rate limited client will take care of blocking calls
to respect the limit. Finally we transform the observable into a BlockingObservable and persist each response back on the database on a different
collection.

I'm not sure about you, but before using RXJava to do this I would need a bunch of different constructs to fullfil the exact same purpose.

### Further reading

* [RX Java threading examples](http://www.grahamlea.com/2014/07/rxjava-threading-examples/) : Great example on how threading works on RX Java
* [Java 8 stream tutorials](http://winterbe.com/posts/2014/07/31/java8-stream-tutorial-examples/) : If you need to get up to speed with lambdas and java 8.
