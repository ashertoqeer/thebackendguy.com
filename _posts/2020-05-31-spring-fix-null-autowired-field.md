---
layout: post
title: "Spring Fix Null @Autowired Field"
permalink: "spring-fix-null-autowired-field/"
last_modified_at: 2020-05-31T00:00:00
excerpt: "Why is your @Autowired field is null and how to fix it"
category: "java-spring-framework"
---

Null `@Autowired` field is a common problem in Spring and the main reason for this is, the instance, in which `@Autowired` is used, is not a Spring-managed instance. 

Spring is a dependency injection framework and dependency injection in Spring only works with Spring-managed instances, called Beans. Hence, the `@Autowired` field will only get populated when Spring manages both instances, the field itself and the object in which it is getting injected too.

#### Example:

Let us create a simple application that should return 'HelloWorld'. Let's start with the service layer.

```java
@Service
public class DemoService {

    @Autowired
    private DemoMessageUtil messageUtil;

    public String getMessage() {
        return messageUtil.beautify("HelloWorld");
    }
}
```
Our service layer depends on `DemoMessageUtil` utility class, let's create it too:

```java
@Component
public class DemoMessageUtil {

    public String beautify(String text) {
        return "--[" + text + "]--";
    }
}
```
Perfect, now we need a controller to expose a REST API for `GET /message` endpoint
```java
@RestController
public class DemoController {

    private DemoService demoService;

    public DemoController() {
        this.demoService = new DemoService(); // Create an instance of DemoService
    }

    @GetMapping("/message")
    public String getMessage() {
        return demoService.getMessage();
    }
}
```
Run this application and call `GET http://localhost:8080/message`. You will get an Internal Server Error and you will find following statements in logs:

```
java.lang.NullPointerException: null
	at com.example.demo.service.DemoService.getMessage(DemoService.java:14) ~[classes/:na]
	at com.example.demo.controller.DemoController.getMessage(DemoController.java:18) ~[classes/:na]
	...
```

We got a `NullPointerException` in `DemoService` at line: ```return messageUtil.beautify("HelloWorld");```. The `messageUtil` was `NULL` there.

But, why did this happen? We have used proper annotations like `@Component`, `@Service`, `Controller`.

Problem is in the constructor of `DemoController`:
```java
...
public DemoController() {
        this.demoService = new DemoService(); // Create an instance of DemoService
    }
...    
```
Here we are creating a new instance of `DemoService()`, which is NOT a spring-managed instance because we created it, not the Spring context, so the Spring can't manage this instance. That's why `@Autowired` has no effect in that instance and we got a Null `@Autowired` field.

To solve this issue, Just remove `DemoService` creation. Since it is annotated by `@Service`, Spring will automatically create its instance and we can `@Autowired` the service in the controller. The updated controller would look like:

```java
@RestController
public class DemoController {

    @Autowired
    private DemoService demoService;

    @GetMapping("/message")
    public String getMessage() {
        return demoService.getMessage();
    }
}
```
Run the application, and the `NullPointerException` will be gone.

#### Conclusion:
Spring dependency injection only works with Spring-managed objects or Beans. If the object in which a Bean is getting injected is not a spring managed object, you will get null `@Autowired` fields. To fix this, make sure only the framework create and manage the related dependencies.
