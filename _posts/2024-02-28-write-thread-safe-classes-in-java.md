---
title: How to write Thread-Safe classes in Java
description: How to create classes that remain functionally correct in a multi-threaded environment in Java.
date: 2024-02-28 11:59:00 +0500
categories: [Java, Concurrency]
tags: [java, concurrency, thread-safety]     # TAG names should always be lowercase
---

## Thread-Safe Classes
Thread-Safe classes remain functionally correct when accessed from multiple interleaving threads, without any additional synchronization or coordination required on the caller side.

Thread-Safety allows us to protect against race conditions. However, it is required only for classes having a **Shared Mutable State**. If the class is not shared, or the class is *stateless* or *immutable*, it is thread-safe by nature and no extra synchronization is required.

For mutable classes, we have several options:

## Atomic Variables
These are part of Java concurrent utilities. These utility classes allow you to have atomic operations on single variables.

To understand this, let's say we have an unsafe counter class:

```java
public class UnsafeCounter {

  private int counter = 0;

  public void increment() {
    counter++;
  }

  public int getCount() {
    return counter;
  }
}
```

This `UnsafeCounter` will provide an inconsistent count when accessed from multiple threads since `counter++` is not a single atomic operation, it is a convenient syntax for three distinct operations: read counter value, add one to counter value, and write counter value back. *Read-Modify-Write*.

In a multi-threaded environment, race conditions will occur and threads will have stale values and lost updates. For example, two threads `t1` and `t2` can read the same counter value. `t1` increments the counter and writes back before `t2`. `t2` now has a stale value and it increments it and writes back, effectively overwriting the previous value from `t1`, which is now lost.

To write a thread-safe version of this class, we can use `AtomicInteger`:

```java
public class SafeCounter {

  private final AtomicInteger counter = new AtomicInteger(0);

  public void increment() {
    counter.getAndIncrement(); // Atomic increment analogous to count++
  }

  public int getCount() {
    return counter.get();
  }
}
```

`AtomicInteger` is a wrapper class over `int` that provides different methods for atomic operations. Like we have used `counter.getAndIncrement()` which is comparable to `count++` in `UnsafeCounter`.

Atomic variable classes are part of [java.util.concurrent.atomic](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/package-summary.html){:target="_blank"} package.

## Synchronized Blocks
Synchronized blocks are used when a group of statements combined needs to be executed atomically. For example, let's say we have the following `UnsafeHttpRequestCounter`:

```java
public class UnsafeHttpRequestCounter {

  private int totalRequests = 0;

  private int getRequestCount = 0;
  private int getRequestPercentage = 0;

  private int postRequestCount = 0;
  private int postRequestPercentage = 0;

  public void incrementGetRequest() {
    totalRequests++;
    getRequestCount++;
    getRequestPercentage = (getRequestCount / totalRequests) * 100;
  }

  public void incrementPostRequest() {
    totalRequests++;
    postRequestCount++;
    postRequestPercentage++;
  }

  // Getters and rest of the class
}
```

To write a safe version of this class, we might consider changing `int` variables to `AtomicInteger`. Although this will make individual counters atomic, it won't make them atomic as a combined execution group, as in `incrementGetRequest()` and `incrementPostRequest()` methods. We need to somehow lock these statements to achieve atomic execution of the whole group combined.

A synchronized block is a locking mechanism to enforce atomicity. It has two parts: an object reference to serve as a lock, and a block of code to be guarded. 

```java
synchronized (lock) {
// Access or modify shared state guarded by lock
}
```

You can use any Java object as a lock and the same lock should be shared between all threads that will execute synchronized block. These locks are called *intrinsic locks* or *monitor locks*. These locks act as mutual exclusion locks (mutex), which means that only one thread can hold the lock. If another thread needs the lock, it must wait for the holding thread to release it.

The lock (if available) is automatically acquired by an executing thread before entering the synchronized block and automatically released when the thread exists the synchronized block.

We can add synchronized blocks in our `UnsafeHttpRequestCounter` like:

```java
...
public void incrementGetRequest() {
  synchronized (this) { // using current object reference as lock
    totalRequests++;
    getRequestCount++;
    getRequestPercentage = (getRequestCount / totalRequests) * 100;
  }
}
public void incrementPostRequest() {
  synchronized (this) { // using current object reference as lock
    totalRequests++;
    postRequestCount++;
    postRequestPercentage++;
  }
}
...
```

Since we are protecting the whole method body using the current instance reference as a lock, we can directly add `synchronized` in the method signature. 

```java
...
public synchronized void incrementGetRequest() {
  totalRequests++;
  getRequestCount++;
  getRequestPercentage = (getRequestCount / totalRequests) * 100;
}
public synchronized void incrementPostRequest() {
  totalRequests++;
  postRequestCount++;
  postRequestPercentage++;
}
...
```

### Reentrancy
Intrinsic locks are reentrant. If a thread tries to acquire a lock held by another thread, it will block, but, if a thread tries to acquire an **already-holding** lock, it will succeed. Reentrancy means that locks are acquired on a per-thread basis, instead of per invocation bases. For example:

```java
public synchronized void multipleLocks() {
  System.out.println("Acquired lock");
  synchronized (this) {                                 // This call won't block
    System.out.println("Again acquired lock");
    synchronized (this) {                               // Neither will this
      System.out.println("Third time acquired lock");
    }
  }
}
```

## Volatile Keyword
Volatile keyword solves memory visibility problem between threads.

### Memory Visibility between threads
 In a single thread, if you write some value to a variable and later read it, it would be the same value. But, if the reads and writes are happening in different threads, it is not guaranteed to be the same (without synchronization).

Consider the  following example:

```java

public class Main {

  private static boolean ready = false;
  private static int value = 0;

  public static void main(String[] args) throws Exception {

    var producer = new Thread(() -> {
      try {
        Thread.sleep(Duration.ofMillis(100)); // Doing some work
      } catch (InterruptedException e) {
        throw new RuntimeException(e);
      }
      value = 49;
      ready = true;
      System.out.println("Producer done!");
    });

    var consumer = new Thread(() -> {
      while (!ready) { /* wait */ }
      System.out.println("Consumer done with value: " + value);
    });

    // Start and Join threads
  }
}

```

This is a classic producer-consumer setup. The producer sets a value in a shared variable and sets the `ready` flag to `true` so that the consumer can consume it, however, this program might not behave as expected. The problem is JVM and the underlying platform performs lots of optimizations. From the prospect of the consumer thread, the `ready` flag is `false` and the `value` is `0`. These values can be stored in processor cache or even reordered if JVM thinks it's better for performance.

Consumer may:
- Always get a `false` value for the `ready` flag and loop forever.
- Read the `value` as `0` before reading `ready` as `true` (reordering) and print the incorrect value `0`.
- Get lucky and print the correct `value` of `49`.

This is called a memory visibility problem.

### Using Volatile

`volatile` keyword ensures that updates to a variable are propagated predictably to other threads. Volatile variables are always written to and read from the memory and JVM won't reorder them with other operations. This enables volatile reads to always return the most recent write by any other thread.

```java
public class Main {

  private static volatile boolean ready = false; // Marked as volatile
  private static int value = 0;

  public static void main(String[] args) throws Exception {

    var producer = new Thread(() -> {
      try {
        Thread.sleep(Duration.ofMillis(100)); // Doing some work
      } catch (InterruptedException e) {
        throw new RuntimeException(e);
      }
      value = 49;
      ready = true;
      System.out.println("Producer done!");
    });

    var consumer = new Thread(() -> {
      while (!ready) { /* wait */ }
      System.out.println("Consumer done with value: " + value);
    });

    // Start and Join threads
  }
}
```

We have set the `ready` flag to volatile, now as soon as the producer updates the values of `ready`, the consumer can immediately see it in the next read.

### Happens Before Guarantee
Write to volatile variable comes with *Happens Before Guarantee*. It means that all shared variables that are updated before an update to a volatile variable will be visible to any other thread after it reads the value of a volatile variable. Or, in other words, all writes that *Happens-Before* a volatile variable write are visible to other threads after the volatile variable read.

From the above producer-consumer example:

```java

var producer = new Thread(() -> {
  try {
    Thread.sleep(Duration.ofMillis(100)); // Doing some work
  } catch (InterruptedException e) {
    throw new RuntimeException(e);
  }
  value = 49;                             // setting the value
  ready = true;                           // setting the flag
  System.out.println("Producer done!");
});

```

First, we set `value = 49`, then we set `read = true`. Since the `read` variable is volatile, the value of `value` also gets synced with memory. When the consumer fetches the value of `read`, it will also fetch the latest values of other variables as well (that are written before the `ready` flag).

```java
var consumer = new Thread(() -> {
  while (!ready) { /* wait */ }                               // read the value of ready flag
  System.out.println("Consumer done with value: " + value);   // value is not voaltile, but due to "happens-before guarantee" we have recent value
});
```

### When to use `volatile`
Volatile provides a weaker form of synchronization with shared variables without any need for mutual exclusion, however, it can't save you from race conditions. They can be used in scenarios such as:

- Writes to a variable doesn't depend on its current state.
- Only one threads write to a variable and all other threads read.
- Locking (mutual exclusion) is not required.

## Conclusion
Thread safety is an important aspect of Java multi-threaded programming. Stateless and Immutable classes are by nature threadsafe and do not require any special treatment. Classes with **Shared Mutable States** need proper synchronization and coordination in a multi-threaded environment. Java provides different tools to deal with synchronization like Atomic Variables, Synchronized Blocks and Volatile variables. These tools help us write correct thread-safe classes.
