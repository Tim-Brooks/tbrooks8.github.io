---
title: Precise Lock-Free Rolling Metrics
date: 2015-06-21 12:00:00
layout: post
author: Tim Brooks
---

## Precise Lock-Free Rolling Metrics

### Introduction

I have recently been spending a significant amount of time working on [Precipice](https://github.com/tbrooks8/Precipice), a library to manage local or remote services. One of the purposes of this library is to maintain metrics about service actions and results for a configurable time period.

Essentially I need some type of sliding window or circular buffer. And because I am interested in counts falling off the end after a period, the decisions on when to switch slots and when the oldest slot expires must be based the current time.

There are a few requirements for this structure:

1. It must support configurable time periods. 
2. It must support multi-thread writing.
3. It should be optimized to support write-heavy workloads. 
4. Building on number 3, the writes should be as computationally cheap as possible.

### Dropwizard Metrics

One popular Java library that is used for metrics is [Dropwizard](https://dropwizard.github.io/metrics/3.1.0/). After a quick perusal through its offerings, I determined that there was nothing that met my needs. There is a [exponentially decaying reservoir](https://github.com/dropwizard/metrics/blob/master/metrics-core/src/main/java/com/codahale/metrics/ExponentiallyDecayingReservoir.java). And a [Meter](https://github.com/dropwizard/metrics/blob/master/metrics-core/src/main/java/com/codahale/metrics/Meter.java) that provides three different exponential moving averages for time periods. However, neither of those provide precise counts nor do they allow configurable times to track.

As far as I can tell, the closest acceptable class for my use case is [SlidingTimeWindowReservoir](https://github.com/dropwizard/metrics/blob/master/metrics-core/src/main/java/com/codahale/metrics/SlidingTimeWindowReservoir.java). However, this class creates a new Map entry for every update potentially causing major memory and GC issues for write-heavy loads. Additionally, it is backed by a ConcurrentSkipListMap which should be less performant than a circular array.

### Single-writer Variant

So I resolved that I was going to have to produce my own class. I started by creating a variant that works for a single-writer scenario. This class tracks three important pieces of state: the array that will contain counts, the current slot, and the time at which to advance to the next slot. All three pieces of state had to be atomic so that readers would know where to start reading and the counts for array slots.

{% gist 2df7a205b3717d2840f8 %}

When a call to increment a metric is made, the code checks if it is time to advance. If it is, the metrics in that slot are incremented. If it is not time, then the code advances to the correct slot and clears any slots that are passed along the way.

{% gist 2d63cae29dc0fae70802 %}

Unfortunately, this approach is difficult to extend to a multi-writer scenario. When it is time to advance a slot three actions must be completed:

1. Update current slot
2. Update next time to advance
3. And clear old state.

I spent some time trying to figure out a state machine that would extend to the multi-writer scenario. However, in the process of exploration, I found a simpler solution that seems to work better. It is quite possible that there is some elegant way to update these three variables in a thread-safe manner. However, it was not immediately apparent to me while investigating.

### Hystrix

Before moving on to the solution I did produce, I want to look at the circular buffer in the [Hystrix](https://github.com/Netflix/Hystrix) library. Hystrix heavily influenced the creation of Precipice. While there are a number of key differences that motivated my work (probably a topic for another blog post), both libraries share a need for performant rolling metrics.

To my surprise when I looked a few days ago, Hystrix uses a [strategy](https://github.com/Netflix/Hystrix/blob/master/hystrix-core/src/main/java/com/netflix/hystrix/util/HystrixRollingNumber.java) similar to my single-writer variant. There is some notion of a "currentBucket". And that bucket is written to until a certain time period has passed.

After that time period has passed, some bookkeeping must occur to create a new bucket.

{% gist 598e8aac8e59cceff8d9 %}

I have simplified the code above quite a bit. However, the idea should be clear. One thread wins a race to acquire a lock. While that thread is doing its work to create a new bucket, all the other threads use the most recent bucket.

This strategy clearly comes with a tradeoff. If a hundred failures occur around the time to switch buckets, all but one of those will be recorded in the metrics one second (the default bucket resolution) before they actually occurred. These metrics will also roll out of the window a second earlier than they should.

Finally, this situation may be exacerbated if there has not been a request in a while. If there have been no metrics within the last 10 seconds (the default window size), and a bunch come in, I believe it is possible that all but one data point will be erased by the resetting of buckets. (However, I am not familiar enough with the Hystrix code to declare this for certain.)

This tradeoff is clearly an edge case. And probably hardly ever matters (especially since it seems like Netflix has great success running Hystrix in production). But it still is not ideal.

### Multi-writer Solution for Precipice

As I was messing around trying to convert my single-writer code to support multi-writers, I realized that each additional piece of atomic state made the state machine more difficult to create.

Instead, I realize that because my metrics are based on time, I could use the current time to determine the index.

Using this approach, when I construct the object I record a start time. This makes it easy to determine the current absolute slot (the slot without considering array bounds).

{% gist 467900e84e4002d1a232 %}

Next it is easy to take the absoluteSlot modulo totalSlots to determine the relative slot. Only instead, as a performance optimization, I always use an array based on a power of two. This allows me to instead use a bitwise mask to determine the relative slot. I learned of this strategy watching a Martin Thompson talk about concurrent [queues](https://github.com/mjpt777/examples/tree/master/src/java/uk/co/real_logic/queues). The strategy is described [here](http://psy-lob-saw.blogspot.com/2013/03/single-producerconsumer-lock-free-queue.html).

{% gist 3bfabd2289d54a7edd3e %}

Each slot has an absoluteSlot int assigned to it at construction time. If that value matches the current absoluteSlot value, this is the slot to update. Otherwise, compareAndSet until I have the correct slot.

{% gist d988b47ffacd4ded1b73 %}

And that describes the strategy for updating metrics.

Reading the metrics is pretty simple. Using time, determine the current absolute slot. Subtract the number of slots back that you would like. Loop through those slots. Use the bitwise mask to convert to a relative slot. If that slot has the correct absoluteSlot value, it is still active. Accumulate values.

The complete code is [here](https://github.com/tbrooks8/sliding-window-benchmark/blob/master/src/main/java/net/uncontended/precipice/CircularBuffer.java). A version that is tightly coupled to the needs of Precipice is [here](https://github.com/tbrooks8/Precipice/blob/master/src/main/java/net/uncontended/precipice/metrics/DefaultActionMetrics.java).

### Performance

I spent some time benchmarking my solution and the metrics in Hystrix. The benchmarking code can be found [here](https://github.com/tbrooks8/sliding-window-benchmark).

A couple of points:

* I simplified my code and Hystrix's code to use System.currentTimeMillis() opposed to the mockable version they normally use.
* Both Hystrix and Precipice normally include a version of Doug Lea's LongAdder class to be compatible with pre-Java 8 projects. However, I ran these tests on JDK 8, so I just used the class in the standard library.
* The benchmark was performed on my MacBook Pro (Retina, 15-inch, Late 2013) using Java HotSpot(TM) 64-Bit Server VM (build 25.40-b25, mixed mode).

Hystrix 1 thread results:

> * HystrixBenchmark.testIncrement    avgt      20  57.702  1.206  ns/op
> * HystrixBenchmark.testIncrement  sample  331181  98.571  0.846  ns/op

Hystrix 6 thread results:

> * HystrixBenchmark.testIncrement    avgt       20   72.575  1.234  ns/op
> * HystrixBenchmark.testIncrement  sample  1365516  163.300  3.253  ns/op

Precipice 1 thread results:

> * PrecipiceBenchmark.testIncrement    avgt      20   63.415  1.113  ns/op
> * PrecipiceBenchmark.testIncrement  sample  290321  105.916  0.905  ns/op

Precipice 6 thread results:

> * PrecipiceBenchmark.testIncrement    avgt       20   76.813  1.341  ns/op
> * PrecipiceBenchmark.testIncrement  sample  1349301  166.700  6.650  ns/op

The complete JMH output is [here](https://github.com/tbrooks8/sliding-window-benchmark/tree/master/doc).

Hystrix is slightly better in the average case:

* Single-thread: 57.702 ns vs. 63.415
* Six-thread: 72.575 vs. 76.813).

And better in the max case:

* Single-thread: 18080.000 ns/op vs. 24992.000 ns/op
* Six-thread: 1331200.000 ns/op vs. 2293760.000 ns/op

### Final Thoughts

Ultimately Hystrix has slightly better performance on my machine. However, I am happy with my solution as I feel that it is easy to understand and provides the ideal characteristic that metrics always go into the correct bucket.

I hope to follow up with some metrics on a server class machine soon.

If you have any thoughts, note any errors, or see any clear performance wins that I did not catch, please reach out!