---
title: Introducing Precipice - A Java Library for Isolated Task Execution
date: 2016-03-13 12:00:00
layout: post
author: Tim Brooks
---

Modern software systems are often composed of multiple independently operating components. It has become common to split monolith services into microservices that live in different processes, containers, machines, or even different datacenters. Failure of individual components from time to time is all but guaranteed. Consistent uptime for the system as a whole depends on an application design that is resilient to these failures. 

[Precipice](https://github.com/tbrooks8/Precipice) is a library designed to provide the building blocks for improving system resiliency. It offers composable metrics and back pressure mechanisms for isolating and handling failure of individual tasks of execution. Additionally, it offers tools for developing patterns of execution.

There are a number of other libraries in the Java ecosystem which are designed to provide system resiliency. Precipice tends to be lower level than some of these alternatives. Precipice does impose any execution model upon you. It is not strictly coupled to threadpools, actors, communicating sequential processes, or any other concurrency model. Instead, Precipice is designed to be able to be used in conjuction with--or as a part of--one of these higher level libraries.

The basic abstraction provided by Precipice is the [GuardRail](https://github.com/tbrooks8/Precipice/blob/master/precipice-core/src/main/java/net/uncontended/precipice/GuardRail.java). A GuardRail isolates the execution of tasks have have failure conditions.

A GuardRail is parameterized by two different enum types. One type defining the possible outcomes of execution. And another type defining reasons why a task might be rejected.

Additionally, it has five main attributes.
1. Name - used for identification purposes.
2. Result metrics - counts of the different task execution results.
3. Rejection Metrics- counts of the different task rejection reasons.
4. Optional Latency Metrics - latencies for the different task exeuction results.
5. Zero or more back pressure mechanisms informed by these metrics.

A GuardRail can be constructed using the builder.
{% gist 7629496020911d2d1d8a %}

## Using a GuardRail

A GuardRail has semantics similar to a [semaphore](https://en.wikipedia.org/wiki/Semaphore_(programming)). When you are interested in accessing the code path isolated by the GuardRail, you must request permits. At this point, the GuardRail will consult with the provided back pressure mechanisms to determine whether the permits can be acquired or if the access must be rejected. If the access is rejected, the rejection metrics will be updated.

When permits are successfully acquired, the system can safely proceed with execution. Upon completion, the permits must be released. This can be done manually, or there are a number of contexts which Precipice provides to release permits automatically. The act of releasing the permits updates the result metrics and latency metrics (if present). It also informs backpressure mechanisms that the permits have been released.

{% gist 9c29d906f0bf7a75b713 %}

As mentioned, there are a number of completable contexts that will make this process less tedious. When complete or completeExceptionally is called in the example below, the permits will be release automatically.

{% gist 1e8f72a9f259f4575e28 %}

In the example above, the completable can only be written to by a single thread. Precipice provides a threadsafe [Eventual](https://github.com/tbrooks8/Precipice/blob/master/precipice-core/src/main/java/net/uncontended/precipice/concurrent/Eventual.java) for usage across thread boundaries. The threadsafe version can be constructed using the [Asynchronous](https://github.com/tbrooks8/Precipice/blob/master/precipice-core/src/main/java/net/uncontended/precipice/factories/Asynchronous.java) factory.

## Integrating Specialized Execution Models

Often Precipice users may be interested in integrating a GuardRail with specialized execution logic opposed to using GuardRails in the adhoc method shown in the example above. For the former case, the namesake [Precipice](https://github.com/tbrooks8/Precipice/blob/master/precipice-core/src/main/java/net/uncontended/precipice/Precipice.java) interface can be implemented.

{% gist fb63a62141c8545b6136 %}

This interface merely indicates is that the implementing class has a GuardRail. The actual acquiring and releasing of permits and executing of tasks must be implemented.

There a couple of provided examples for how this can be done:

1. [CallService](https://github.com/tbrooks8/Precipice/blob/master/precipice-core/src/main/java/net/uncontended/precipice/CallService.java) - This class is in the core module and will isolate the call() method on Callables with a GuardRail.
2. [ThreadPoolService](https://github.com/tbrooks8/Precipice/blob/master/precipice-threadpool/src/main/java/net/uncontended/precipice/threadpool/ThreadPoolService.java) - This class is similar to the CallService. However, it executes Callables on a threadpool opposed to the calling thread.
3. [HttpAsyncService](https://github.com/tbrooks8/Precipice/blob/master/precipice-samples/src/main/java/net/uncontended/precipice/samples/http/HttpAsyncService.java) - This class takes a URL and a [AsyncHttpClient](link) upon construction. Calls to makeRequest take a [RequestBuilder](https://github.com/AsyncHttpClient/async-http-client/blob/master/client/src/main/java/org/asynchttpclient/RequestBuilder.java), apply the url to the RequestBuilder, and execute the http request using the provided async client.

In the third example, demonstrates two interesting things that the design of Precipice allows.

1. By isolating one specific endpoint behind the GuardRail, you can still share a single [Netty](http://netty.io/) http client between multiple HttpAsyncService for maximum efficiency. Under the hood a single event loop group can handle all of your IO. This is possible due to the fact that Precipice does not mandate an specific threading model.
2. Passing RequestBuilder opposed to a fully build [Request](https://github.com/AsyncHttpClient/async-http-client/blob/master/client/src/main/java/org/asynchttpclient/Request.java) allows you to utilize a Pattern to submit the request to different endpoints.

## Precipice Patterns

Highly available systems often mandate some degree of redundancy. Precipice provides a number of tools to build patterns to utilize these redundant services.

When you construct a [Pattern](https://github.com/tbrooks8/Precipice/blob/master/precipice-core/src/main/java/net/uncontended/precipice/pattern/Pattern.java) you provide a collection Precipice implementations and a [PatternStrategy](https://github.com/tbrooks8/Precipice/blob/master/precipice-core/src/main/java/net/uncontended/precipice/pattern/PatternStrategy.java). You can call the getPrecipices(int permits) to return a sequence of precipices for which permits could be acquired.

The PatternStrategy defines the logic for which precipices we attempt to acquire permits.

{% gist 84568e36ee832de0819a %}

The Pattern will call nextIndices() which returns the indices of Precipices for which to acquire permits. The Pattern will continue until it has acquired permits for the number of Precipices definied by acquireCount or until the indices have been exhausted. Then it returns a sequence of the Precipices with acquired permits.

There are two examples provided in the core module.
1. A [LoadBalancer](https://github.com/tbrooks8/Precipice/blob/master/precipice-core/src/main/java/net/uncontended/precipice/pattern/RoundRobinLoadBalancer.java) - this balances acquire calls to different Precipices. It only defines a acquireCount of one.
2. A [Shotgun](https://github.com/tbrooks8/Precipice/blob/master/precipice-core/src/main/java/net/uncontended/precipice/pattern/Shotgun.java) - this randomly distributes acquire calls to different Precipices. It allows a configurable acquireCount. An example of when this strategy might be useful would be duplicating an idempotent http request to multiple endpoints and taking the first response.

## Back Pressure

There are currently three provided mechanisms of backpressure. 

1. [Semaphore](https://github.com/tbrooks8/Precipice/tree/master/precipice-core/src/main/java/net/uncontended/precipice/semaphore) - limits the maximum number of permits that can be acquired at one point in time.
2. [RateLimiter](https://github.com/tbrooks8/Precipice/blob/master/precipice-core/src/main/java/net/uncontended/precipice/rate/RateLimiter.java) - limits the maximum number of permits that can be acquired over a period of time.
3. [CircuitBreaker](https://github.com/tbrooks8/Precipice/tree/master/precipice-core/src/main/java/net/uncontended/precipice/circuit) - starts rejecting permit acquisition attempts if failures are occuring.

Users can also implement their own mechanisms of back pressure using the [BackPressure](https://github.com/tbrooks8/Precipice/blob/master/precipice-core/src/main/java/net/uncontended/precipice/BackPressure.java) interface.

{% gist 6a7a2980be8da110a941 %}

## Other Features

### ThreadPoolService

As (briefly) mentioned above, there is a Precipice [implementation](https://github.com/tbrooks8/Precipice/blob/master/precipice-threadpool/src/main/java/net/uncontended/precipice/threadpool/ThreadPoolService.java) that isolates callable execution behind a GuardRail and on a threadpool. Out of the box, this should work for many different use cases.

{% gist b29c424b60d5cbc02964 %}

### Configurable Metrics

There are multiple provided metric options provided. Some just keep total counts for the entire application lifetime. Others are rolling, so you can query the metrics for specific time periods. I am also working on others that are only written to by a background thread. This would be a specialized case for users that demand low latency on permit acquisition.

### Simulation Tests

There is a [Simulation](https://github.com/tbrooks8/Precipice/blob/master/precipice-core/src/main/java/net/uncontended/precipice/util/Simulation.java) class that provides basic simulation tests for user Precipice implementations. An example of these tests in use can be found in the [threadpool module](https://github.com/tbrooks8/Precipice/blob/master/precipice-threadpool/src/test/java/net/uncontended/precipice/threadpool/ThreadPoolServiceTest.java#L179).

### TimeoutService

There are some [facilities](https://github.com/tbrooks8/Precipice/tree/master/precipice-core/src/main/java/net/uncontended/precipice/timeout) to schedule timeouts for your tasks. Currently this is based on a DelayQueue strategy. However, in the future there will be [timer wheel](http://netty.io/4.0/api/io/netty/util/HashedWheelTimer.html) based strategy.

### An Emphasis on Performance

Precipice was design with performance in mind. I am still in the process of optimizing it. However, right now it should easily meet most user's needs.

There is no locking (or synchronized) on permit acquisition or release. All of the provided metrics and circuit breakers are updated and read in a lock-free manner. 

Everywhere nano time is used there is a method arity to pass in the nano time. Essentially for task execution there should only be one call to System.nanoTime() on acquisition and one call on release.

I am also working on variants of the [different](https://github.com/tbrooks8/Precipice/blob/master/precipice-core/src/main/java/net/uncontended/precipice/metrics/experimental/SWCountMetrics.java) [components](https://github.com/tbrooks8/Precipice/blob/master/precipice-core/src/main/java/net/uncontended/precipice/circuit/SWCircuitBreaker.java) that can be written to by background threads. This would allow many of the volatile reads and writes to be removed from the acquisition and release calls.

## Stability

Precipice is currently used in production at [Staples SparX](http://www.staples-sparx.com/). It is nearing 1.0. However, the API may still change between now and then.