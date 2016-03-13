---
title: Introducing Precipice - A Low Level Java Library for Isolated Task Execution
date: 2016-03-14 12:00:00
layout: post
author: Tim Brooks
---

Modern software systems are often composed of multiple independently operating components. It has become common to split monolith services into microservices that live in different processes, containers, machines, or even different machines spread across datacenters. Failure of individual components from time to time is all but guaranteed. Consistent uptime for the system as a whole depends on an application design that is resilient to these failures. 

Precipice is a library designed to provide the building blocks for improving system resiliency. It provides composable metrics and back pressure mechanisms for isolating and handling failure of individual tasks of execution. Additionally, it provides the tools for developing patterns of execution.

There are a number of other libraries in the Java ecosystem that are designed to provide system resiliency. Precipice tends to be lower level than some of these alternatives. Precipice does impose any execution model upon you. It is not strictly coupled to threadpools, actors, communicating sequential processes, or any other concurrency model. Instead, Precipice is designed to be able to be used in conjuction with or as a part of one of these higher level libraries.

The basic abstraction provided by Precipice is the [GuardRail](<https://github.com/tbrooks8/Precipice/blob/master/precipice-core/src/main/java/net/uncontended/precipice/GuardRail.java>). A GuardRail is a class that isolates task execution.

A GuardRail is parameterized by two different enum types. A type defining the possible outcomes of task execution. And a type defining reasons why a task might be executed.

Additionally, it has five main attributes.
1. Name - used for identification purposes.
2. Result metrics - counts of the different task execution results.
3. Rejection Metrics- counts of the different task rejection reasons.
4. Optional Latency Metrics - latencies for the different task exeuction results.
5. Zero or more back pressure mechanisms informed by these metrics.

A GuardRail can be constructed using the builder.
```
String name = "Identity Service";
GuardRailBuilder builder = new GuardRailBuilder<>();
builder.name(name);
builder.resultMetrics(new RollingCountMetrics<>(SimpleResult.class));
builder.resultLatency(new IntervalLatencyMetrics(SimpleResult.class));
builder.rejectedMetrics(new RollingCountMetrics<>(RejectedReason.class));
builder.addBackPressure(new LongSemaphore<>(RejectedReason.MAX_CONCURRENCY, 10));

GuardRail guardRail = builder.build();
```

## Using a GuardRail

A GuardRail has semantics similar to a semaphore. When you are interested in accessing the code path isolated by the GuardRail, you must request permits. At this point, the GuardRail will consult the provided back pressure mechanisms to determine whether the permits can be acquired or if the access must be rejected. If the access is rejected, the rejection metrics will be updated.

When permits are successfully acquired, the system can safely proceed with execution. Upon completion, the permits must be released. This can be done manually or there are a number of contexts that Precipice provides to release permits automatically. Releasing the permits updates the result metrics and latency metrics (if present). It also calls release on the backpressure mechanisms.
```
RejectedReason rejected = guardRail.acquirePermits(1L, startNanoTime);
if (rejected != null) {
  throw new RejectedException(rejected);
}

String response = null;
try {
    URL url = new URL("http://www.google.com");
    URLConnection urlConnection = url.openConnection();
    response = readToString(urlConnection.getInputStream());
    guardRail.releasePermits(1L, Result.SUCCESS, startNanoTime);
} catch (Exception ex) {
    guardRail.releasePermits(1L, Result.EXCEPTION, startNanoTime);
}
```
As mentioned, there are a number of completable contexts that will make this process less tedious. When complete or completeExceptionally is called in the example below, the permits will be release automatically.

```
CompletionContext<SimpleResult, String> completable = Synchronous.acquireSinglePermitAndCompletable(guardRail);

try {
  URL url = new URL("http://www.google.com");
  URLConnection urlConnection = url.openConnection();
  completable.complete(SimpleResult.SUCCESS, readToString(urlConnection.getInputStream()));
} catch (Exception ex) {
  completable.completeExceptionally(SimpleResult.ERROR, ex);
}

// To read the result:
String value = completable.getValue();
// or:
Throwable error = completable.getError();
```

In the example above, the completable can only be written to by a single thread. Precipice provides a threadsafe version for usage accross thread boundaries.

## Integrating GuardRails with specialized execution models

Often Precipice users may be interested in integrating a GuardRail with specialized execution logic opposed to using GuardRails in the adhoc method shown in the example above. For the former case, the Precipice interface can be implemented.

```
public interface Precipice<Result extends Enum<Result> & Failable, Rejected extends Enum<Rejected>> {

    GuardRail<Result, Rejected> guardRail();
}
```

All this interface indicates is that the implementing class has a GuardRail. The actual acquiring and releasing of permits and executing of tasks must be implemented.

There a couple of provided examples for how this can be done:

1. [CallService](link) - This class is in the core module and will isolate the call() method on Callables with a GuardRail.
2. [ThreadPoolService](link) - This class is similar to the CallService. However, it executes Callables on a threadpool opposed to the calling thread.
3. [HttpAsyncService](link) - This class takes a URL and a [AsyncHttpClient](link) upon construction. Calls to makeRequest take a [RequestBuilder](link), apply the url to the RequestBuilder, and execute the http request using the provided async client.

In the third example, demonstrates two interesting things that the design of Precipice allows.

1. By isolating on the specific endpoint behind the GuardRail, you can still share a single [Netty](link) http client for maximum efficiency. Under the hood a single event loop group can handle all your IO. This is allow by the fact that Precipice does not mandate an specific threading model.
2. Passing RequestBuilder opposed to a fully build [Request](link) allows you to utilize a Pattern to submit the request to different endpoints.

## Precipice Patterns

Highly available systems often mandate some degree of redundency. Precipice provides a number of tools to build patterns to be utilize these redundent services.

When you construct a [Pattern](link) you provide a collection Precipice implementations and a [PatternStrategy](link). You can call the getPrecipices(int permits) to return a sequence of precipices for which permits could be acquired.

The PatternStrategy defines the logic for which precipices we attempt to acquire permits.
```
public interface PatternStrategy {

  Iterable<Integer> nextIndices();

  int acquireCount();
}
```

The Pattern will call nextIndices() which returns the indices of Precipices for which to acquire permits. The Pattern will continue until it has acquired permits for the number of Precipices definied by acquireCount or until the indices have been exhausted. Then it returns a sequence of the Precipices with acquired permits.

There are two examples provided in the core module.
1. A [LoadBalancer](link) - this balances acquire calls to different Precipices. It only defines a acquireCount of one.
2. A [Shotgun](link) - this randomly distributes acquire calls to different Precipices. It allows a configurable acquireCount. An example of when this strategy might be useful would be duplicating an idempotent http request to multiple endpoints and taking the first response.

## Back Pressure

There are currently three provided mechanisms of backpressure. 

1. [Semaphore]() - limits the maximum number of permits that can be acquired at one point in time.
2. [RateLimiter]() - limits the maximum number of permits that can be acquired over a period of time.
3. [CircuitBreaker]() - starts rejecting permit acquisition attempts if failures are occuring.

Users can also implement their own mechanisms of back pressure using the BackPressure interface.

```
public interface BackPressure<Rejected extends Enum<Rejected>> {

  Rejected acquirePermit(long number, long nanoTime);

  void releasePermit(long number, long nanoTime);
  
  void releasePermit(long number, Failable result, long nanoTime);
  
  <Result extends Enum<Result> & Failable> void registerGuardRail(GuardRail<Result, Rejected> guardRail);
}
```

## And More

Precipice is designed for maximum flexibility.

### Configurable Metrics

There are multiple provided metric options provided. Some just keep total counts for the entire application lifetime. Others are rolling, so you can query the metrics for specific time periods. I am also working on others that are only written to by a background thread. This would be a specialized case for users that demand low latency on permit acquisition.

### Simulation Tests

There is a [Simulation](link) class that provides basic simulation tests for user Precipice implementations. An example of these tests in use is in the [threadpool module](link).

### Timeout Service

There is are some facilities to schedule timeouts for you tasks. Currently this is based on a DelayQueue strategy. However, as some point in the future there will be [timer wheel](link) based strategy.

## Stability

Precipice is currently used in production at [Staples SparX](link). It is nearing 1.0. However, the API may still change between now and then.