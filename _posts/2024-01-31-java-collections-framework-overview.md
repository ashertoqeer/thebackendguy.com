---
title: Java Collections Framework Overview
date: 2024-01-31 11:59:00 +0500
categories: [Java, Collections Framework]
tags: [java, collections-framework]     # TAG names should always be lowercase
---

In Java, **a group of objects** is called **a collection**. A collection is an object that holds other objects and provides methods to manipulate them. For example `ArrayList`, `HashSet`, `HashMap` etc.

Java collections framework provides mechanisms to hold and manipulate Java collections. It consists of three main parts:

1. Collection interfaces.
2. Collection implementations.
3. Collection Algorithms.

# 1. Collection interfaces: 
Collection interfaces represent different types of collections. For example `List`, `Set`, `Queue` and `Map`.

`List`, `Set` and `Queue` extend the `Collection` interface, while `Map`, even though it is part of the collections framework, doesn't extend from the `Collection` interface.

Java 21 introduced the concept of sequenced collections (collections with a defined encounter order), the following diagram shows different collection interfaces and their relations. `List`, `Set`, `Map` and `Queue` are the most commonly used collection interfaces.

![Java collections interface hierarchy](/assets/img/post/java-collection-interface-hierarchy.png)
_Java collections interface hierarchy in `java.util` package as of Java 21_

The `Collection` interface itself extends from the `Iterable` interface. It means that you can iterate over any collection using an enhanced `for` loop (It uses an iterator under the hood). For example:

```java
Collection<Integer> integers = new ArrayList<>(Arrays.asList(1, 2, 3));
    for (Integer integer : integers) {
      System.out.println(integer);
}
```

# 2. Collection implementations
Collection implementations can be subdivided into multiple categories:

## General-purpose implementations:
General-purpose implementations are primary or standard implementations of the collection interfaces. For example, `ArrayList` and `LinkedList` are standard implementations of the `List` interface.

You can find multiple general-purpose implementations for the same collection interface; like `List` has two. You should choose an implementation depending on the task at hand. For example, if you have a read-heavy task, use `ArrayList` since it provides constant time retrieval. If you frequently need to add elements or remove elements, `LinkedList` may perform better.

> General-purpose implementations are not designed for concurrent environments. You need to use a `synchronized` block to avoid unexpected behaviour in concurrent environments.
{: .prompt-danger }

Iterator implementation in general-purpose collections is based on the fail-fast principle. Every iterator maintains a state of collection it iterates and checks for modification before every retrieval. If it deducts any structural change (e.g. size change) then it throws `ConcurrentModificationException`. It doesn't matter if this modification was made by the same thread or some other thread. 

```java
Collection<Integer> integers = new ArrayList<>(Arrays.asList(1, 2, 3));
    for (Integer integer : integers) {
      integers.remove(integer); // Trying to remove elements while iterating over it, will throw exception
}
```

To avoid `ConcurrentModificationException`, you can directly work with an instance of an `iterator` and call `remove`. In this way `iterator` can update its state safely and won't throw `ConcurrentModificationException`:

```java
    Collection<Integer> integers = new ArrayList<>(Arrays.asList(1, 2, 3));

    // Remove all elements from the list using iterator
    for (var it = integers.iterator(); it.hasNext(); ) {
      it.next();
      it.remove();
    }
```


The following table has common interfaces and their general-purposes implementations:

|List           |Set            |Map            |Queue
|`ArrayList`    |`HashSet`      |`HashMap`      |`PriorityQueue`
|`LinkedList`   |`TreeSet`      |`TreeMap`      |
|               |`LinkedHashSet`|`LinkedHashMap`|

## Special-purpose implementations:
Special-purpose implementations are designed for nonstandard special use cases. For example, `EnumMap` is a specialized `Map` implementation for use with `enum` type keys. All of the keys in an enum map must come from a single enum type that is specified, explicitly or implicitly, when the map is created. 

Enum maps are represented internally as arrays which makes them extremely compact and efficient.

The following table has common interfaces and their special-purpose implementations:

|List                  |Set                  |Map              |Queue
|`CopyOnWriteArrayList`|`EnumSet`            |`EnumMap`        |
|                      |`CopyOnWriteArraySet`|`WeakHashMap`    |
|                      |                     |`IdentityHashMap`|

## Wrapper implementations:
These implementations wrap existing implementations and add functionality on top of them. These implementations are anonymous and you can use them by static factory methods found in `Collections` class. There are three types of wrappers:

### 1. Synchronization Wrappers:
The synchronization wrappers add synchronization (thread safety) to an arbitrary collection. For example:

```java
var threadSafeList = Collections.synchronizedList(Arrays.asList(1, 2, 3));
```

Method `Collections.synchronizedList(...)` will return a wrapper collection that enforces synchronization on the underlying list. This wrapper collection is also called a *view* since it is merely a wrapper over the original collection and delegates all tasks to the original collection.

All methods of `threadSafeList` are synchronized, except for the iterator object since `threadSafeList.iterator()` will create a new iterator object and that will not be thread-safe. This makes sense since the iterator object is used to iterate a collection, but it is not a *part* of the collection. To ensure thread safety, the iteration should be synchronized explicitly:

```java
var threadSafeList = Collections.synchronizedList(Arrays.asList(1, 2, 3));
    
synchronized (threadSafeList) {
    for (var element : threadSafeList) {
        doSomething(element);
    }
}
```

You can find synchronization wrappers for other collection interfaces in the `Collections` class.

### 2. Unmodifiable Wrappers:
Unmodifiable Wrappers take away the ability to modify the collection by intercepting all the operations that would modify the collection and throwing an `UnsupportedOperationException`. These wrappers are mainly used to get an immutable list view.

```java
var unmodifiableCollection = Collections.unmodifiableCollection(Arrays.asList(1, 2, 3));
unmodifiableCollection.add(4); // Throws UnsupportedOperationException
```

You can find unmodifiable wrappers for other collection interfaces in the `Collections` class.

### 3. Checked Interface Wrappers
Checked Interface Wrappers are designed for use with generic collections. These implementations return a dynamically type-safe view of the specified collection, which throws a `ClassCastException` if a client attempts to add an element of the wrong type.

For example, consider the following list:

```java
List numbers = new ArrayList();
numbers.add(1);     // An integer since Integer extends Number
numbers.add(2.567); // A double since Double also extends Number
```

Let's say you want to allow only `Integer` in your list then, you can use:

```java
List numbers = Collections.checkedList(new ArrayList(), Integer.class);
numbers.add(1);     // An integer since Integer extends Number
numbers.add(2.567); // will throw ClassCastException
```

## Convenience implementations:
Convenience implementations are mini-implementations that can be more convenient and more efficient than general-purpose implementations when you don't need their full power.

For example, `Arrays.asList` method returns a List view of its array argument. Under the hood, it creates an array and delegates all list operations to that array. The size of the List view is equal to the size of the underlying array and it can not be changed. If you call `add` or `remove`, `UnsupportedOperationException` will be thrown.

```java
List<Integer> integerList = Arrays.asList(1, 2, 3); // Create a List backed by an array of size 3
integerList.set(0, 4); // Set value at index 0 to 4
integerList.add(4); // throw UnsupportedOperationException since backing array size can't be changed
```

You can find different convenience implementations in `Arrays` and `Collections` utility classes as static factory methods.

## Concurrent implementations:
Concurrent implementations are designed for highly concurrent use. They are useful when you have a multithreaded environment and you want to use collections without the need for a `synchronized` block. These collections go beyond the synchronization wrappers discussed previously to provide features that are frequently needed in concurrent programming. 

#### These concurrent-aware interfaces are available:
- BlockingQueue
- TransferQueue
- BlockingDeque
- ConcurrentMap
- ConcurrentNavigableMap

#### Concurrent implmentations:

| List                  | Set                   |  Map                  |  Queue
|`CopyOnWriteArrayList` |`CopyOnWriteArraySet`  |`ConcurrentHashMap`    | `LinkedBlockingQueue`
|                       |`ConcurrentSkipListSet`|`ConcurrentSkipListMap`| `ArrayBlockingQueue`
|                       |                       |                       | `PriorityBlockingQueue`
|                       |                       |                       | `DelayQueue`
|                       |                       |                       | `SynchronousQueue`
|                       |                       |                       | `LinkedBlockingDeque`
|                       |                       |                       | `LinkedTransferQueue`

## Abstract implementations:
Abstract implementations are partial implementations that are useful for custom implementations. 

Let's say you want to add some additional logic at the time of `add()` or `remove()`. Your solution might be to subclass existing `ArrayList` and `override` `add()` and `remove()` methods, however, this is not trivial. You will first need to understand the existing logic behind `add()` and `remove()`, then reimplement that logic in a way that you add your custom logic as well.

A better and recommended solution would be to extend from `AbstractList<E>` which provides a skeletal implementation of the List interface. This class is designed to be extended and facilitate custom implementations.

You can find other partial implementations as well like `AbstractSet`, `AbstractMap`, `AbstractQueue` etc. 

## Legacy implementations:
These implementations are from earlier versions of Java. For example `Vector` and `Hashtable`.

# 3. Collection Algorithms:
These are common data manipulation algorithms to facilitate common operations like sorting, searching, shuffling etc. You can find them in the `Collections` class as static methods. For example: `Collections.sort(list)`, `Collections.shuffle(list)`, `Collections.binarySearch(list, key)` etc.

---

### Resources
[Official documentation](https://docs.oracle.com/en/java/javase/21/core/java-collections-framework.html#GUID-242A255F-D49C-4352-B09B-C09421B724D4)