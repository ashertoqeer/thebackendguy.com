---
layout: post
title: "Java 8 Functional Programming Simplified"
permalink: "java-8-functional-programming-simplified"
last_modified_at: 2019-06-29T00:00:00
excerpt: "This article will explain Java 8 functional programming related concepts, i.e Lambda expressions, Functional interfaces etc."
---

This article will explain Java 8 functional programming related concepts, i.e Lambda expressions, Functional interfaces etc.

Let's start with function: A function takes an input, generates an output.

```java
int sum(int num1, int num2) {
    return num1 + num2;
}
```

Java supports functions since start. A pure function doesn't mutate the state and always produce same output for same input.

In functional programming, functions can be passed to other functions as arguments and can be returned from a function too. The functions that **can take other functions as parameters** or **can return a function** are known as **Higher Order Functions**.

Functions can be passed to other functions by wrapping in an object. For example:

```java
Collections.sort(someList, new Comparator<Object>() {
	@Override
    public int compare(Object integer, Object t1) {
        // Passing object comparing logic to sort function       
        return 0;
       }
	});
```

This  approach is not new, we have been doing this for years. This function and high order function support is base of functional programming.

**What are lambdas, functional interfaces etc ?**

**Well, they are just a way of doing old things in a new way.**

Let's talk about these concepts one by one:

### Lambdas 

Suppose we want to write a function that should take two int arguments and a function argument:

```java
int compute(int num1, int num2, ...){
    // Needs to compute result on bases of third function argument
    // But, we can't pass function directly
}
```

To achieve this, we can define a **FunctionWrapper** interface:

 ```java
public interface FunctionWrapper {
    int function(int num1, int num2);
}
 ```

Now we can complete **compute(...)** function:

```java
public int compute(int num1, int num2, FunctionWrapper functionWrapper){
 	return functionWrapper.function(num1, num2);
}
```

The result of our **compute(...)** function depends on passed function.

```java
int sum = compute(2, 3, new FunctionWrapper() {
	@Override
    public int function(int num1, int num2) {
      return num1 + num2; // Addition
  }
});

int mutiple = compute(2, 3, new FunctionWrapper() {
	@Override
    public int function(int num1, int num2) {
     return num1 * num2; // Multiplication
  }
});
```

Although we have achieved what we want, it is too verbose and ugly. This is where **Lambdas** can help us. We can replace whole anonymous object creation with a lambda expression and focus on our core logic. We don't even need to write function name since our interface has only one function so Java can understand automatically:

``` java
int sum = compute(2, 3, (num1, num2) -> {
   return num1 + num2;
});

int mutiple = compute(2, 3, (num1, num2) -> {
   return num1 * num2;
});
```

In fact, since our function has only one statement, we can remove even return and extra brackets.

```
int sum = compute(2, 3, (num1, num2) -> num1 + num2);

int mutiple = compute(2, 3, (num1, num2) -> num1 * num2);
```

So, by use of Lambdas, we can greatly simply our code.

### Functional Interfaces

Remember our **FunctionWrapper** interface defined above:

```java
public interface FunctionWrapper {
    int function(int num1, int num2);
}
```

It has only **ONE** abstract method because we have to pass function using **Lambda expression**. Due to this unique constraint of **having only one abstract method**, Java was able to correctly identify method signature and hence allowed us to use **Lambda expression**. So, **Lambda expression** won't work with out this **have only one abstract method** constraint.

This type of interfaces are called **Functional Interfaces** and Java actually provided an annotation **@FunctionalInterface** to prevent any accidental change to break that constraint.

``` java
@FunctionalInterface
public interface FunctionWrapper {
    int function(int num1, int num2);
}
```



#### Functional Interfaces provided by JAVA

Although you can write as many functional interfaces as you need, Java has also provided its own set of functional interfaces. Java as used them in its API's and you are free to use them in your projects too. In fact many libraries are using them too. Here are some examples: [Full List Here](https://docs.oracle.com/javase/8/docs/api/java/util/function/package-summary.html)

| Interface     | Description                                                  |
| ------------- | ------------------------------------------------------------ |
| Consumer<T>   | Represents an operation that accepts a single input argument and returns no result. |
| Supplier<T>   | Represents a supplier of results.                            |
| Function<T,R> | Represents a function that accepts one argument and produces a result. |



Functional Programming is a different programming paradigm which requires practice to master, everything else is just syntax!
