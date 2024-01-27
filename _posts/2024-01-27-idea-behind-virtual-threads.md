---
title: The Idea Behind Java Virtual Threads - It's not  about speed
date: 2024-01-27 11:59:00 +0500
categories: [Java, Concurrency]
tags: [java21, concurrency, virtual-threads]     # TAG names should always be lowercase
---

Virtual threads are super lightweight threads managed by JVM instead of OS. They are introduced in Java 19 as a preview feature and finalized in Java 21. It is a major addition to the Java concurrency toolkit and it took Java language developers years of effort and an overhaul of JDK to implement.

## But, why virtual threads are such a big deal? 

To understand this, we need to understand two things: 
1. How threads work in Java
2. One thread per request model

### How threads work in Java
Threads in Java are lightweight wrappers over OS threads. Creating a thread is an expensive operation. A thread requires a memory stack to keep function calls, local variables, return addresses, etc. This stack is allocated at the time of creation with a fixed amount of memory. It can not grow or shrink dynamically. If a thread's stack is full, it will throw `StackOverflowException`. Since we can not add more memory to an existing stack, a strategy to avoid `StackOverflowException` is to allocate a sufficient amount of memory (stack size) at the time of creation.

By default, the thread stack size depends on multiple factors, like underlying OS, available memory, JVM, etc, but typically it is around `1MB` for a standard JVM. You can also configure it manually.

> You can check thread stack size on your JMV by running: <br/> 
`$ java -XX:+PrintFlagsFinal -version | grep ThreadStackSize` <br/>
{: .prompt-tip }

It means that in a standard application, if we want to work with 1000 live threads, we would need an additional around `1GB` of memory just to keep them, and this is just one aspect. A thread also needs to be registered at the OS level and JVM has to manage all of them.

Threads are expensive.

### One Thread per Request Model
In a typical Java server, a servlet container like Tomcat or Jetty is responsible for handling client requests and call mapped functions in the Java application (servlet). The servlet container binds each client request with a server thread and that thread is responsible for completing the entire request.

![java-one-thread-per-request-image](/assets/img/post/java-one-thread-per-request.png){: width="700" height="400" }
_High level view of Java one thread per request_

> Since threads are expensive resources, servlet containers use thread pools with limited threads (200 for Tomcat) to handle user requests. When a request hits the servlet container, a thread is pulled from the pool to handle the request and returned to the pool once the request is completed.
{: .prompt-info }

In a typical server-side application, a request-handling logic looks something like this:

```java
  public Order placeOrder(String cartId) {

    // Fetch cart from DB, it is a blocking DB call
    var cart = cartRepo.fetchCartById(cartId);

    var order = Order.newOrder(cart);

    // some business logic here, more DB calls, maybe some network calls

    // Save Order to DB, again a blocking call
    orderRepo.save(order);

    return order;
  }
```

Here on line 4, we are fetching a cart from the database. It is a blocking call and the handler thread is doing nothing while waiting for this call to complete.

Once we have the cart, we are creating an order. After that, we executed more blocking calls, like DB calls, some network calls etc.

After that, we saved the state of the order in DB and returned the order.

### Problem: Inefficient Resource Management
Let's say, all of these blocking calls, combined, took `100ms` to complete (which is a long time in CPU terms). And stack memory **used** to complete **one request** is around `512KB` (out of `1MB` allocated). Then for 200 concurrent users, we are wasting around `100MB` of memory for `20,000ms` or `20 seconds`.

We are holding `100MB` for `20 seconds` and doing nothing. These wasted resources are sufficient to serve another 200 concurrent users.

## Solution: Virtual Threads
Virtual threads are a new addition to the Java concurrency toolkit. They are managed by JVM and don't require `MBs` of memory beforehand since they offer a dynamic thread stack that can grow or shrink even after a thread is created.

Virtual threads run on top of platform threads.
 
> As of Java 21, native threads are now called platform threads.
{: .prompt-info }

When we start a virtual thread, JVM binds our virtual thread to the platform thread and starts execution. If the virtual thread encounters a blocking call, JVM unbinds the platform thread and uses it somewhere else while our virtual thread waits for the blocking call to return.

Once the blocking call is completed, JVM again binds our virtual thread to a platform thread (not necessarily the same one) for further execution. In this way, JVM ensures maximum resource utilization. One platform thread can support thousands of virtual threads and we can scale up to millions of virtual threads in a typical JVM.

![java-virtual-threads](/assets/img/post/java-virtual-threads-01.png)
_Java virtual threads_

### How to create Virtual Threads

Virtual threads are fully compatible with the existing `Thread` API. You can use helper methods added to `Thread` class to create virtual or platform threads:

```java
    // creates a virtual thread
    Thread virtualThread = Thread.ofVirtual().unstarted(() -> System.out.println("Hi from virtual thread"));
    virtualThread.start();

    // creates a platform thread
    Thread platformThread = Thread.ofPlatform().unstarted(() -> System.out.println("Hi from platform thread"));
    platformThread.start();

    // short hand to create and start virtual thread
    var virtualThread2 = Thread.startVirtualThread(() -> System.out.println("Hi from virtual thread"));
```
> Virtual threads are very lightweight and designed to execute a task on the fly and throw them away. Since they are inexpensive to create, pooling virtual threads is not recommended.
{: .prompt-tip }

You can also use `newVirtualThreadPerTaskExecutor` executor service. This executor service will create a **new** virtual thread for each task:

```java
    // Executes each task in a new virtual thread
    try (var executorService = Executors.newVirtualThreadPerTaskExecutor()) {
      executorService.submit(() -> System.out.println("Hi from virtual thread executor service"));
    }
```

## Conclusion

Virtual threads are designed around a very specific use case, resource efficiency in case of blocking calls. If your tasks are CPU-heavy computations with minimum to no blocking calls then virtual threads provide no additional benefit. They might perform worse since there is now an additional layer between your task and the executing platform thread. **For CPU-heavy tasks, use platform threads.**

Another thing to keep in mind is that virtual threads can not make your application run faster. Virtual threads can never run faster than platform threads. **They are about scalability, not speed.**
