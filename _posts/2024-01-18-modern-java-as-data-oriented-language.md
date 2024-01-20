---
title: Modern Java as Data-Oriented Language
date: 2024-01-18 00:00:00 +0500
categories: [Java, Data Oriented Programming]
tags: [java,record-pattern,dop]     # TAG names should always be lowercase
---

# Modern Java as Data-Oriented Language

Java, at its core, is an Object Oriented Language. But, Object Oriented Programming alone can't solve all of the problems efficiently. So, with time Java added support for helpful concepts from other programming paradigms. Like, Java 8 introduced first-class support for Functional Programming and now, Java is introducing first-class support for Data-Oriented Programming or DOP.

In this article, we will see what is Data Oriented Programming, why it is useful and how to benefit as a Java developer.

## Data-Oriented Programming
Data-Oriented Programming or DOP (not to be confused with Data-Oriented Design) is a programming paradigm that revolves around **data as a first-class citizen**. The programs are written in a way that **functions merely perform transformations over data** as they accept one form of data, perform computation and return another form of data. The **data is immutable**, so it can never be changed during a computation and the result is represented as another form of immutable data.

It is fundamentally different from Object Oriented Programming style where we try to model the real world in the form of **"Objects"** that have **mutable "state"** in them that can be modified from an instance method.

For example:

```java
/**
 *  OOP way
 */
cart.addItem(item); // Cart state is mutated as new item is added

/**
 * DOP way
 */
var newCart = addItem(cart, item); // A new cart is created from old cart with new item
```

### Benefits of Data-Oriented Programming
The idea of immutable data and transformer functions is very helpful for certain tasks, especially in the micro-services architecture. Let's say you have a service that consumes system-level messages, categorizes them based on action types, transforms them to run analytics queries or train some ML model and perist them to different data stores based on category and intended use. For this type of task, you are merely operating on data so the Data-Oriented Programming style makes more sense than the traditional OOP style.

## Java and Data-Oriented Programming

[Yehonathan Sharvit](https://blog.klipse.tech/) explained DOP very well in this excellent write-up: [Principles of Data-Oriented Programming](https://blog.klipse.tech/dop/2022/06/22/principles-of-dop.html). He provided four principles of DOP:

1. Separate code (behavior) from data.
2. Represent data with generic data structures (like maps, lists, sets etc. classes are not allowed).
3. Treat data as immutable.
4. Separate data schema from data representation.

### Can follow these principles in Java?

We **can follow principles 1 and 3** by having immutable data types with no instance methods and writing functions separately, however, we **can not follow principle 2**. Technically, we can avoid classes and represent data in generic data structures like maps or lists, but, it would mean that we will **lose all of type safety**. For example, here is how we can represent a user as a map of key-value pairs:

```java
/**
 * A user object represented as a map of values, we have lost type safty
 * 
 * */
Map<String, Object> user = new HashMap<>();
user.put("name", "John Doe");
user.put("email", "john@example.com");
user.put("age", 24);
user.put("dateOfBirth", "2000-01-01");

// we need to type cast to get values
String name = (String) user.get("name");
int age = (int) user.get("age");
```

**In Java we don't compromise on type safety**, hence we prefer classes over a generic data structure. Since we are following structure anyway, there is no need for principle 4 as well.

So, In Java, we primarily follow two principles of DOP:

- Separate code (behavior) from data.
- Treat data as immutable.

Now you might say this is not (according to 4 principles) *pure* DOP, and you are right, it is not *pure*. The focus of Java is not being *pure*. It is on being as effective as possible as a general-purpose programming language.

## Data-Oriented Programming as of Java 21

Let's say we are working on some e-commerce system and we have different order-related events like the order created, shipped, delivered, canceled etc. We need to execute some business logic based on event type. Let's implement it in DOP style.

### Record type and Sealed classes

The data (events) needs to be immutable so we will use Java Record type. It will provide immutability out of the box. We are also going to need an `Event` base class for which we want to restrict inheritance to order events only. We will use `sealed` keyword for that. Our events would look like this:

```java

public sealed interface Event {

  String orderId();

  record OrderCreated(String orderId) implements Event { }
  record OrderShipped(String orderId, ShippingDetails shippingDetails) implements Event { }
  record OrderDelivered(String orderId) implements Event { }
  record OrderCancelled(String orderId) implements Event { }
}
```
### Pattern Matching in instanceof

We can use enhanced pattern-matching capabilities in `instanceof` operator to write conditionals. For example:

#### Pattern variable

```java

if (event instanceof OrderCreated orderCreated) {
  var orderId = orderCreated.orderId();
  out.printf("Order created with orderId %s", orderId);
}
```
This will first check if (`event instanceof OrderCreated`) is `true`. If yes, then it will automatically perform `(OrderCreated) event` and store value in `orderCreated` that you can directly use.

#### Record Pattern

Record patterns allow you to deconstruct a record directly in conditional:

```java
if (event instanceof OrderShipped(var orderId, var shippingDetails)) {
  var state = shippingDetails.state();
  var city = shippingDetails.city();
  var address = shippingDetails.address();
  out.printf("OrderShipped to %s %s %s", state, city, address);
}
```
Here we are getting `orderId` and `shippingDetails` directly instead of going through some `orderShipped` variable.

These record patterns can also be nested, allowing you to get values from a deeper hierarchy of records:

```java
if (event instanceof OrderShipped(var orderId, ShippingDetails(var state, var city, var address))) {
  out.printf("OrderShipped to %s %s %s", state, city, address);
}
```
The above pattern will only be matched if `ShippingDetails` is not null and then deconstruct the record and provide access to its members.

### Pattern matching in `switch` statement

We can also use this power of pattern matching in switch statements, allowing a concise way of implementing business logic:

```java
switch (event) {
  case OrderCreated orderCreated ->  out.printf("Order created with id: %s", orderCreated.orderId());
  case OrderShipped(var orderId, ShippingDetails(var state, var city, var address)) ->  out.printf("OrderShipped to %s %s %s", state, city, address);
  case OrderDelivered(var orderId) -> out.printf("Order delivered %s", orderId);
  case OrderCancelled(var orderId) -> out.printf("Order cancelled %s", orderId);
}
```

### The future
Java will add more functionality related to DOP in the future. For example [JEP 443: Unnamed Patterns and Variables](https://openjdk.org/jeps/443) is already in preview. Once finalized, it will allow developers to write even more concise deconstruction patterns where unused variables can be marked by unscored `_`:

```java
if (event instanceof OrderShipped(_, ShippingDetails(var state, _, _)) && state == "NY") {
  // gets executed only if state code is NY
}
```

There are also more exciting features in progress. Check out [Project Amber](https://openjdk.org/projects/amber/) for more details.

## Resources & References
- [Data Oriented Programming in Java](https://www.infoq.com/articles/data-oriented-programming-java/) by **Brian Goetz**
- [Data-Oriented programming in Java](https://blog.klipse.tech/java/2021/03/05/data-oriented-programming-in-java.html) by **Yehonathan Sharvit**
- [Data-Oriented Programming in Java](https://www.youtube.com/watch?v=UQAw3pvZPCY) by **Gavin Bierman**
- [Principles of Data-Oriented Programming](https://blog.klipse.tech/dop/2022/06/22/principles-of-dop.html) by **Yehonathan Sharvit**
- [Project Amber](https://openjdk.org/projects/amber/)
- [Is Data-Oriented programming applicable in Java?](https://coderanch.com/t/740716/engineering/Data-Oriented-programming-applicable-Java)
