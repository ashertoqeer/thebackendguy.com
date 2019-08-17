---
layout: post
title: "Netflix Zuul 2 Push Messaging Sample"
permalink: "netflix-zuul-2-push-messaging-sample"
last_modified_at: 2019-04-29T00:00:00
excerpt: "You can use Netflix Zuul 2 as a powerful, robust push messaging server, this article explains how to do that."
---

Zuul 2, besides an awesome API gateway, can also act as a Push Messaging Server. [offical page](https://github.com/Netflix/zuul/wiki/Push-Messaging). xIn this article we will setup a sample zuul 2 push server using web socket support.

### A brief overview of Push Messaging:

The basic idea behind push messaging is that clients have a persistent long lived connection with some push server. That push server sits in middle of clients and application server. Now whenever application server need to send some message to some client, it will ask push server to send a push message to that client, usually calling some REST API. The push server then send that message to specified client on its existing connection.

### Setup Zuul 2 as Push Server:

You can find our sample zuul 2 push server code on [github](https://github.com/ashertoqeer/zuul2-push-messaging/tree/master/zuul-2-push). I recommend you to clone and examine it while going through this article. That sample is basically a modified version of official Zuul 2 Sample.

Our sample push server uses `version 2.1.4` of Zuul 2. You can confirm it from `build.gradle`.

Zuul 2 is based on Netty which is a great framework for network programming. However, to keep things simple, Netty configurations is out of scope of this article. The default configurations would work just fine. 

All the code regarding push is in `com.netflix.zuul.push` package.

A push server should provide two things:

1. A way for clients to register themselves to Push server, have a persistent long lived connection.

2. A way for application servers to submit push message to be sent to some specific client.

The way Zuul 2 works is it allow clients to register themselves. If successful it then add that persistent connection to an in memory registry of connections with client id as key. When some application server submits a message to Zuul 2, it has to provide client id. Zuul 2 will check its registry for that client id, if it founds a connection, it will send the push. Otherwise it will return 404 status code.

##### 1. Client Registration And Connection:

Zuul 2 provides a web-socket end point for clients to register and have a persistent connection.

```
ws://[host]:8887/ws
```

A client must have to authenticate itself before having a persistent connection. Have a look at `SamplePushAuthHandler`

```java
public class SamplePushAuthHandler extends PushAuthHandler {

	// Set the domain, the client must send this domain in Header
    public SamplePushAuthHandler(String path) {
        super(path, "example.domain.com");
    }

    @Override
    protected boolean isDelayedAuth(FullHttpRequest req, ChannelHandlerContext ctx) {
        return false;
    }

	// Checking for customerId Header, client must sent this too
    @Override
    protected PushUserAuth doAuth(FullHttpRequest req) {
        String customerId = req.headers().get("X-CUSTOMER_ID");
        if (!Strings.isNullOrEmpty(customerId)) {
            return new SamplePushUserAuth(customerId);
        }
        return new SamplePushUserAuth(HttpResponseStatus.UNAUTHORIZED.code());
    }

}
```

In this simple implementation, we are just checking if request have `X-CUSTOMER_ID` header then it is a valid customer and we will create its session, else we are returning error response.

Here is `SamplePushUserAuth` Just for reference.

```java
public class SamplePushUserAuth implements PushUserAuth {

    private String customerId;
    private int statusCode;
	...
        
    // Successful auth
    public SamplePushUserAuth(String customerId) {
        this(customerId, 200);
    }

    // Failed auth
    public SamplePushUserAuth(int statusCode) {
        this("", statusCode);
    }
	...
}
```

Now a client can simply make a connection by sending following request to web-socket.

```
URI:
ws://[host]:8887/ws

HEADERS:
origin 				example.domain.com
X-CUSTOMER_ID 		[some id]
```



##### 2. Push Submission For Application Server:

Zuul 2 provides a HTTP endpoint for application servers to submit a push message.

``` 
http://[host]:9001/push
```

Application servers need to send client id too, they can do it by `X-CUSTOMER_ID` header. The push message will be in http body, it could be anything but for demo purposes, lets assume it is JSON. The request would look like:

```
URI:
POST http://[host]:9001/push

HEADER:
X-CUSTOMER_ID 		[some id]

BODY:
{
	"field":"a sample field"
}
```

As soon as you push a message with connected client Id, client will immediately receive it.



###  A Simple Web Socket Client:

Although our Zuul 2 push server is working, unfortunately there are no good tools available for testing web-sockets. To get around this, it would be better if we can have a simple Java based Web Socket test client to test our Zuul 2 push server. You can find code [here](https://github.com/ashertoqeer/zuul2-push-messaging/tree/master/websocket-client). 

``` java
// A snippet 
URI uri = new URI("ws://localhost:8887/ws");

HashMap<String, String> headers = new HashMap<String, String>();
headers.put("origin","example.domain.com");
headers.put("X-CUSTOMER_ID","1234");

WebSocketClient client = new EmptyClient(uri, headers);
client.connect();
```



Run Zuul 2 Push Server, then start web socket client. Once connection get established, Use some REST API clients like Postman to submit a push and see it reflected in client's console.



This article and sample is for basic understanding and demo. Please read [official sources](https://github.com/Netflix/zuul/wiki/Push-Messaging) for more advance documentation.
