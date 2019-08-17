---
layout: post
title: "Netflix Zuul 2 API Gateway Sample"
permalink: "netflix-zuul-2-api-gateway-sample/"
last_modified_at: 2019-05-03T00:00:00
excerpt: "Netflix Zuul 2 is an open source API Gateway for micro-service. In this article we will look into some basic concepts of Zuul 2 with a basic sample."
---

Zuul 2 (successor of Zuul 1) is an [API gateway](https://microservices.io/patterns/apigateway.html) developed and open sourced by Netflix. 

#### Overview

Zuul 2 essentially is an API gateway with primary responsibility of routing requests to proper back-end services. It doesn't really matter how back-end services are developed. In this article, I'll explain how to use Zuul 2 as an API gateway for REST based services.

#### Architecture

Zuul 2 is based on [Netty](https://netty.io/) which is an excellent framework for writing network applications in Java. For sake of simplicity, Netty is out of scope of this article. 

Zuul 2 introduced concept of **Filters**. According to [official doc](https://github.com/Netflix/zuul/wiki/Filters):

> Filters are the core of Zuul's functionality. They are responsible for the business logic of the application and can perform a variety of tasks.

Filters are divided in 3 categories:

> #### Incoming
>
> Incoming filters get executed before the request gets proxied to the origin. This is generally where the majority of the business logic gets executed. For example: authentication, dynamic routing, rate limiting, DDoS protection, metrics.
>
> #### Endpoint
>
> Endpoint filters are responsible for handling the request based on the execution of the incoming filters. Zuul comes with a built-in filter (`ProxyEndpoint`) for proxying requests to backend servers, so the typical use for these filters is for static endpoints. For example: health check responses, static error responses, 404 responses.
>
> #### Outgoing
>
> Outgoing filters handle actions after receiving a response from the backend. Typically they're used more for shaping the response and adding metrics than any heavy lifting. For example: storing statistics, adding/stripping standard headers, sending events to real-time streams, gziping responses.

#### See things in action

Before going any further, I recommend you to clone and open [Zuul 2 Basic Guide From Github](https://github.com/ashertoqeer/zuul2-basic-guide) which is based on official sample provided by Netflix. You can run `Bootstrap` file under `java` folder to run the application.	

Lets start from [application properties](https://github.com/ashertoqeer/zuul2-basic-guide/blob/master/src/main/resources/application.properties) file:

```properties
# Setting Port	(1)
zuul.server.port.main=8887 

# Configure filters (2)
zuul.filters.root=src/main/groovy/info/novatec/zuul2/filters
zuul.filters.locations=${zuul.filters.root}/inbound,${zuul.filters.root}/outbound,${zuul.filters.root}/endpoint
zuul.filters.packages=com.netflix.zuul.filters.common

# Routing to proxied backend services (3)
some-service-1.ribbon.listOfServers=localhost:8081
some-service-1.ribbon.client.NIWSServerListClassName=com.netflix.loadbalancer.ConfigurationBasedServerList

some-service-2.ribbon.listOfServers=localhost:8082,localhost:8083
some-service-2.ribbon.client.NIWSServerListClassName=com.netflix.loadbalancer.ConfigurationBasedServerList


# Deactivate Eureka (4)
eureka.registration.enabled=false
eureka.shouldFetchRegistry=false
eureka.validateInstanceId=false
```

**(1) -** Setting the port on which API gateway will listen for requests.

**(2) -** Setting paths for filters. We need to do it because filters can be dynamically added or removed. Zuul 2 constantly pool given directories to see for any changes. Just paste new filter file in that directory and Zuul 2 will pick it without having to restart. Filter are written in Groovy, although Zuul 2 supports any JVM based language.

**(3) -** Configuring routes. Zuul 2 internally uses several other components too, for example [Netflix Ribbon](https://github.com/Netflix/ribbon) for load balancing, [Netflix Archaius](https://github.com/Netflix/archaius/wiki/Overview) for dynamic property management, [Google Guice](https://github.com/google/guice) for dependency management etc. Now, lets examine property:

```
some-service-1.ribbon.listOfServers=localhost:8081
```

here `some-service-1` is a **VIP** or virtual IP.

>##### What is a VIP in Zuul 2 context?
>
>When we develop a micro-services architecture, We have different services running separately. Now as per need, it is entirely possible for one service to run on multiple VPS or **Instances**. These instances could be dynamic in nature, could grow in number automatically when load increases and shrink when there is no need for too many instances. Hence it would be very difficult to *refer services in terms of instance IPs* . 
>
>This is where **VIP** or virtual comes in. Instead of talking in terms of instances or instance IP, we assign each service a **VIP**. And then we can refer to service from its **VIP** which can be translated to instance IP at run time.

`ribbon` is for load balancing, since we are using static IP addresses for this demo, we need to provide a list of addresses or static instance IPs here.

So, collectively it means that "Map **vIP** *some-service-1* to any of given IPs". Later you'll see we will refer to **vIP** from the code to proxy the request.

**(4) - ** We are disabling [Netflix Eureka](https://github.com/Netflix/eureka/) which is used for [Service Discovery](https://dzone.com/articles/service-discovery-in-a-microservices-architecture). We have disabled to keep things simple.

#### Lets Checkout Filters

Now, lets see our filters from the sample. You can find them under[ groovy folder](https://github.com/ashertoqeer/zuul2-basic-guide/tree/master/src/main/groovy/info/novatec/zuul2/filters). We have handled our Proxy logic in `Routes` file:

```java
class Routes extends HttpInboundSyncFilter { //(1)

    @Override
    HttpRequestMessage apply(HttpRequestMessage request) {
        SessionContext context = request.getContext()
        switch (request.getPath()) {	//(2)
            case "/some-service-1":
                context.setEndpoint(ZuulEndPointRunner.PROXY_ENDPOINT_FILTER_NAME)
                context.setRouteVIP("some-service-1")
                break

            case "/some-service-2":
                context.setEndpoint(ZuulEndPointRunner.PROXY_ENDPOINT_FILTER_NAME)
                context.setRouteVIP("some-service-2")
                break

            default:	//(3)
                context.setEndpoint(NotFoundEndpoint.class.getCanonicalName())
        }
        return request
    }

    @Override	//(4)
    int filterOrder() {
        return 0
    }

    @Override	//(5)
    boolean shouldFilter(HttpRequestMessage request) {
        return true
    }
}
```

>**(1) -** From the [official doc](https://github.com/Netflix/zuul/wiki/Filters):
>
>Filters can be executed either synchronously or asynchronously. If your filter isn't doing a lot of work and is not blocking or going off-box, you can safely use a sync filter by extending `HttpInboundSyncFilter` or `HttpOutboundSyncFilter`.
>
>However, in some cases you need to get some data from another service or a cache, or perhaps do some heavy computations. In these cases, you should not block the Netty threads and should use an async filter that returns an `Observable` wrapper for a response. For these filters you would extend the `HttpInboundFilter` or `HttpOutboundFilter`.

**(2) -** A simple switch statement to check incoming request path, this switch will proxy request to corresponding origin. To proxy a request, you just need to refer its **vIP** and Zuul 2 will automatically resolve it into some real IP and pass the request accordingly.

**(3) -** In we have received some unknown request, just return some error without bothering any back-end service.

**(4) - **In which order this filter will execute, 0 means it would be the first one to get executed.

**(5) - **Should this filter apply? Returning true mean it would apply on all sort of requests, but definitely you can decide if this filter is applicable to received `HttpRequestMessage request`. 

#### Lets touch `java` Folder

This folder actually contains Configuration for Netty and Dependency Injection to bootstrap the application and Run properly. These are advance and require some knowledge of Netty, Google guice, and a bit deep knowledge of Zuul 2 itself and how it works. For the sake of a simple guide, that information might be an overkill. You must read [official documentation](https://github.com/Netflix/zuul/wiki) for better understanding.

Feel free to ask anything if you need to.
