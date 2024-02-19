---
title: Performance Analysis using JMH
date: 2024-02-19 11:59:00 +0500
categories: [Java, Performance]
mermaid: true
tags: [java, jmh]     # TAG names should always be lowercase
---

## What is JMH
**Java Microbenchmark Harness** or **JMH** is a performance analysis tool designed to benchmark and analyze the performance of Java code. Unlike tools like JMeter, which allows you to run performance tests against the whole system, JMH allows you to run benchmarks against individual methods in isolation.

## Setting Up JMH
The recommended way to run a JMH benchmark is to use Maven to setup a standalone project that depends on the jar files of your application. This approach is preferred to ensure that the benchmarks are correctly initialized and produce reliable results. It is possible to run benchmarks from within an existing project, and even from within an IDE, however, setup is more complex and the results are less reliable.

Maven archetypes are the primary mechanism used to bootstrap the project that has the proper build configuration. Run the following command to create a new project.

```bash
$ mvn archetype:generate \
  -DinteractiveMode=false \
  -DarchetypeGroupId=org.openjdk.jmh \
  -DarchetypeArtifactId=jmh-java-benchmark-archetype \
  -DgroupId=org.sample \
  -DartifactId=test \
  -Dversion=1.0
```

It will generate a maven project called test. The project will contain a `pom` file and a Java file called `MyBenchmark`.

```java
public class MyBenchmark {

    @Benchmark
    public void testMethod() {
        // This is a demo/sample template for building your JMH benchmarks. Edit as needed.
        // Put your benchmark code here.
    }
}
```

To run this benchmark, you can either add a main method like this:

```java
public class MyBenchmark {

    @Benchmark
    public void testMethod() {
        // This is a demo/sample template for building your JMH benchmarks. Edit as needed.
        // Put your benchmark code here.
    }

    // Run this method to run benchmarks
    public static void main(String[] args) throws Exception {
        org.openjdk.jmh.Main.main(args);
    }
}
```

Or, you can directly build and run from the command line (recommended approach, no need for the main method):

1. Build using maven
    ```bash
    ~/Projects/test$ mvn clean verify
    ```
    After the build is done, you will get the self-contained executable JAR, which holds your benchmark, and all essential JMH infrastructure code.

2. Run benchmarks
    ```bash
    ~/Projects/test$ java -jar target/benchmarks.jar
    ```
    Run with -h to see the command line options available.

Run this empty method with nothing to benchmark, it provides JMH infrastructure cost for running performance tests which provides you a good baseline to compare your benchmarks against. 

## Forks and Iterations

Once the benchmark starts running, you will see forks and iterations like:

```console
...
# Run progress: 0.00% complete, ETA 00:08:20
# Fork: 1 of 5
# Warmup Iteration   1: 2555589355.541 ops/s
# Warmup Iteration   2: 2584807845.820 ops/s
# Warmup Iteration   3: 2586001314.576 ops/s
# Warmup Iteration   4: 2586287588.385 ops/s
# Warmup Iteration   5: 2586732843.115 ops/s
Iteration   1: 2584108452.075 ops/s
Iteration   2: 2588291236.817 ops/s
Iteration   3: 2583726937.328 ops/s
Iteration   4: 2583386478.526 ops/s
Iteration   5: 2587391426.036 ops/s
...
```

A Fork is a sub-process in which your benchmark will run. For each fork, you have two types of iterations: Warmup iterations and measurement iterations. Warmup iterations are used to allow JVM to perform optimization, their result will be discarded. Once the warmup is completed, the next iterations are measured and their result will be accommodated in the final report.

An iteration doesn't mean a single call, instead, it means continuous calls to benchmark functions for a specified amount of time.

By default, JMH runs 5 forks, each fork with 5 warmup iterations and 5 measurement iterations. The number of forks and iterations can be configured as we will later see.

## Understanding Report

```console
Benchmark                Mode  Cnt           Score         Error  Units
MyBenchmark.testMethod  thrpt   25  2584270185.907 ± 2275384.273  ops/s
```

- **Benchmark:** Benchmark name.
- **Mode:** Benchmark Mode. `thrpt` stands for Throughput. There are several benchmark modes available.
- **Cnt:** Number of iterations from which result is compiled. 5 forks, 5 measurement iterations for each mean 5 X 5 = 25. 
- **Score:** This is the benchmark score, since the mode is Throughput, this score represents the total number of operations performed.
- **Error:** The margin of error in the score.
- **Units:** The measurement unit, `ops/s` means operations per second.

## Writing Benchmarks
Let's say we have a method that returns a random odd number, here is how we can write a benchmark that measures average time per execution:

```java
package org.sample;

import java.util.Random;
import java.util.concurrent.TimeUnit;
import org.openjdk.jmh.annotations.Benchmark;
import org.openjdk.jmh.annotations.BenchmarkMode;
import org.openjdk.jmh.annotations.Fork;
import org.openjdk.jmh.annotations.Measurement;
import org.openjdk.jmh.annotations.Mode;
import org.openjdk.jmh.annotations.OutputTimeUnit;
import org.openjdk.jmh.annotations.Warmup;

@Fork(value = 2)
@Warmup(iterations = 2)
@Measurement(iterations = 1)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
public class MyBenchmark {

  @Benchmark
  public void baseline() {
    // baseline
  }

  /*
    Notice we are returning result here, it is very important since 
    if we don't do it, JVM will consider this as dead code (value is unused) 
    and eliminate it entirely during optimization phase.

    Keep in mind JVM optimizations while writing benchmarks
  */
  @Benchmark
  public int testRandomOddNumber() {
    return randomOddNumber(); // Important!! We are returning result
  }

  public int randomOddNumber() {
    int number = new Random().nextInt(100);
    number += number % 2 == 0 ? 1 : 0;
    return number;
  }

  public static void main(String[] args) throws Exception {
    org.openjdk.jmh.Main.main(args);
  }
}
```

- **@Fork(value = 2):** Run in 2 forks (default 5)
- **@Warmup(iterations = 2):** For each fork, perform only 2 warmup iterations (default 5)
- **@Measurement(iterations = 1):** For each fork, run only 1 measurement iteration (default 5)
- **@BenchmarkMode(Mode.AverageTime):** Benchmark mode is the average time since we are interested in the average time it takes to complete a single execution.
- **@OutputTimeUnit(TimeUnit.NANOSECONDS):** We want results in terms of nanoseconds.

These annotations are at the class level, so they will affect all benchmarks in this class. You can define these annotations on individual benchmarks as well.

Run the benchmark, it will generate a report something like:

```console
Benchmark                        Mode  Cnt   Score   Error  Units
MyBenchmark.baseline             avgt    2   0.387          ns/op
MyBenchmark.testRandomOddNumber  avgt    2  39.992          ns/op
```

## Benchmark Modes
The following benchmark modes are available:

- **Throughput:** Measures the raw throughput by continuously calling the benchmark method in a time-bound iteration, and counting how many times we executed the method.
- **AverageTime:** Measures the average execution time.
- **SampleTime:** Samples the execution time. With this mode, we are still running the method in a time-bound iteration, but instead of measuring the total time, we measure the time spent in *some* of the benchmark method calls. This allows us to infer the distributions, percentiles, etc.
- **SingleShotTime:** Measures the single method invocation time. It performs only a single benchmark method invocation. The iteration time is meaningless in this mode. This mode is useful to do cold startup tests; when you specifically do not want to call the benchmark method continuously.
- **All:** Run all benchmark modes.

## Concurrent execution
By default, the benchmark iterations are single-threaded. If you want to run the benchmark in a multithreaded environment you can use `@Threads` annotation either at the benchmark method level or class level.

```java
@Fork(value = 1)
@Warmup(iterations = 2)
@Measurement(iterations = 1)
@Threads(3) // 3 threads will run benchmarks concurrently
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
public class MyBenchmark {
  ...
}
```

## Groups
By default, tests are symmetric, which means the same code was executed in all the threads. But, sometimes you need asymmetric tests, i.e. distribute tests between threads.

JMH provides this with the notion of `@Group`, which can bind several methods together, and all the threads are distributed among the test methods. You can use `@Group` annotation on benchmark method to bind it to a group. The number of threads executing a particular benchmark defaults to a single thread; and can be overridden by `@GroupThreads` annotations.

```java
...
public class MyBenchmark {

  @Group("A")
  @Benchmark
  public void baseline() {
    // baseline
  }

  @Group("A")
  @Benchmark
  public int testRandomOddNumber() {
    return randomOddNumber(); // Important!! We are returning result
  }

  ...
}
```

## State
State objects are used to maintain some state while the benchmark is running. You can use `@State` annotation on any class that represents the state. State objects will be instantiated on demand, and reused during the entire benchmark trial. Here is a simple `MyState` that returns some values for x and y.

```java
@State(Scope.Benchmark)
public class MyState {

  private int x;
  private int y;

  // Parameterized constructor is not allowed

  @Setup  // optional annotation, use to set up a state object
  public void setup() {
      x = 4;
      y = 5;
  }

  @TearDown // optional annotation, use to clean up resources
  public void tearDown() {
    // clean up if required
  }

  public int getX() {
    return x;
  }

  public int getY() {
    return y;
  }
}
```

You can define 3 types of scopes:
1. **Benchmark:** The same state instance will be shared in all threads.
2. **Group:** Each benchmark group will get its instance, which is shared between all threads within the same group.
3. **Thread:** Each thread in any group gets a new instance.

For `@Setup` and `@TearDown`, you can also define the invocation level. Three invocation levels are available:
1. **Level.Trial:** to be executed before/after each run of the benchmark.
2. **Level.Iteration:** to be executed before/after each iteration of the benchmark.
3. **Level.Invocation:** to be executed for each benchmark method execution.

#### Using State Object

To use a state object, we can simply add it to the benchmark parameter.

```java
@Fork(value = 1)
@Warmup(iterations = 3)
@Measurement(iterations = 2)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
public class MyBenchmark {

  @Benchmark
  public void baseline() {
    // baseline
  }

  /*
    Using MyState
  */
  @Benchmark
  public int testSum(MyState myState) {
    return sum(myState.getX(), myState.getY()); // Important!! We are returning result
  }

  public int sum(int x, int y) {
    return x + y;
  }

  public static void main(String[] args) throws Exception {
    org.openjdk.jmh.Main.main(args);
  }
}
```

#### Default State

You can mark Benchmark class with `@State` as well, making it a default state. Members of default state are available to all benchmark methods:

```java
@Fork(value = 1)
@Warmup(iterations = 3)
@Measurement(iterations = 2)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)  // Threaded scoped default state
public class MyBenchmark {

  int x;
  int y;

  @Setup(Level.Iteration)
  public void setup() {
    x = 567;
    y = 987;
  }

  @TearDown(Level.Iteration)
  public void tearDown() {
    x = 0;
    y = 0;
  }

  @Benchmark
  public void baseline() {
    // baseline
  }

  @Benchmark
  public int testSum() {
    return sum(x, y); // Default state members can be accessed directly
  }

  public int sum(int x, int y) {
    return x + y;
  }

  public static void main(String[] args) throws Exception {
    org.openjdk.jmh.Main.main(args);
  }
}
```

#### Param

You can also provide a list of different parameters for testing in the state:

```java
@Fork(value = 1)
@Warmup(iterations = 3)
@Measurement(iterations = 2)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)
public class MyBenchmark {

  @Param({"1", "31", "65", "101", "103"})
  int x;

  @Param({"4", "123", "1", "13", "987"})
  int y;

  @Benchmark
  public void baseline() {
    // baseline
  }

  @Benchmark
  public int testSum() {
    return sum(x, y);
  }

  public int sum(int x, int y) {
    return x + y;
  }

  public static void main(String[] args) throws Exception {
    org.openjdk.jmh.Main.main(args);
  }
}
```

In the above example, the `testSum` benchmark will be executed with all `(x, y)` pairs. 

## Blackholes
Blackhole is an alternate way of tricking JVM into thinking that benchmark methods return values are being used. Till now, we are returning values in a benchmark, but if we have more than one value, we can use Blackhole to consume them.

```java
  @Benchmark
  public void testSum(MyState myState, Blackhole blackhole) {
    int result = sum(myState.getX(), myState.getY());
    blackhole.consume(result); // Important!! consume value so it is getting used
  }  
```

## Batch Size
Batch size allows you to specify the number of invocations (method calls) for each iteration. This is used where the cost of operation significantly varies from invocation to invocation. In this case, the only acceptable benchmark mode is a single shot.

For example, here we are comparing inserting 10,000 integers in the linked list and an array list:

```java
@Fork(value = 5)
@Warmup(iterations = 5)
@Measurement(iterations = 5)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Thread)
public class MyBenchmark {

  List<Integer> linkedList;
  List<Integer> arrayList;

  @Setup(Level.Iteration)
  public void setup() {
    linkedList = new ArrayList<>();
    arrayList = new ArrayList<>();
  }

  @TearDown(Level.Iteration)
  public void tearDown() {
    linkedList.clear();
    arrayList.clear();
  }

  @Benchmark
  @BenchmarkMode(Mode.SingleShotTime)
  @Measurement(batchSize = 10_000)
  public void baseline() {
    // baseline
  }

  @Benchmark
  @BenchmarkMode(Mode.SingleShotTime)
  @Measurement(batchSize = 10_000)
  public List<Integer> testLinkedListInsertions() {
    linkedList.add(linkedList.size() + 1);
    return linkedList;
  }

  @Benchmark
  @BenchmarkMode(Mode.SingleShotTime)
  @Measurement(batchSize = 10_000)
  public List<Integer> testArrayListInsertions() {
    arrayList.add(arrayList.size() + 1);
    return arrayList;
  }

  public static void main(String[] args) throws Exception {
    org.openjdk.jmh.Main.main(args);
  }
}
```

```console
Benchmark                             Mode  Cnt  Score   Error  Units
MyBenchmark.baseline                    ss   25  0.301 ± 0.076  ms/op
MyBenchmark.testArrayListInsertions     ss   25  1.080 ± 0.448  ms/op
MyBenchmark.testLinkedListInsertions    ss   25  0.995 ± 0.479  ms/op
```
---

## Benchmarking Guidelines
- Always have an empty base method to set a baseline.
- If a benchmark is (unexpectedly) very close to the baseline, then there might be some JVM optimizations going on making the benchmark unreliable.
- Don't use loops to control benchmark method invocation, use batch size.
- Avoid constants and locally scoped variables for which values are never changed to avoid JVM optimizations.
- Benchmarking reports are just data, you need to understand patterns instead of relying on concrete values since they are specific to the underlying hardware.

Microbenchmarking allows you to have a "feel" for how the specific method would perform under load. Benchmarks heavily depend on underlying hardware and JVM, don't take them as "absolute values" since values can change from machine to machine, or even from CPU temperature. Try to understand patterns, since they more or less, remain the same.
