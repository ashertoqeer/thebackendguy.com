---
layout: post
title: "Netty Simple TCP Server"
permalink: "netty-simple-tcp-server/"
last_modified_at: 2020-04-12T00:00:00
excerpt: "Netty Simple TCP Server"
category: "netty"
---

I have been using Netty for quite sometime now. Although, once you understand it, it is pretty easy and straight forward, but starting out with Netty is somewhat confusing. The goal for this article is to share some basic concepts around Netty in most simple way and demo a simple TCP server.

### What is Netty?

In simple terms, Netty is a **Framework** for writing network applications in Java. Netty is neither a server like Tomcat, nor a client. Instead it provides a **Framework** to write either server or client. 

### When to use Netty?

If your application require network programming, for example you need to implement a custom protocol, what would you do? You need to manage sockets and perform I/O operations. Network programming in Java is not so trivial task. It is prone to errors, unseen issues, confusing exceptions and so on. 

Netty wraps [Java NIO](https://en.wikipedia.org/wiki/Non-blocking_I/O_(Java) ) and provides you an elegant, nice and simple API to use.

### Netty Basic concepts

Netty built around concept of **Channels**. Simply put **channels** represents *connections*. Every time a new connection is established, a new channel is created. A channel supports both read and write operations.

To perform read and write, Netty provides you **Channel Handlers**, inbound & outbound. When there is something to read, **Channel Inbound Handlers** are invoked and when there is something to write, **Channel Outbound Handlers** are invoked.

Business logic are written in handlers which can be chained together to support separation of concerns. Netty provides **Channel Pipeline** API to build chain of handlers.

### Setting Up Netty

Netty setup mainly involve three setups: 

1. **Bootstrap:** General connection configurations. 

2. **Channel Initializing:** Setup channel handlers pipeline at this point

3. **Channel Handlers:** Write handlers for handling I/O

These configurations mainly depends on weather you are writing server or client. Let's see how to write a simple TCP Server.

### Simple TCP Server

Let's write a simple, minimal TCP server that listens for connections. After connection has been made, client can send any string to server and server will respond with some message. 

You can find code here: [Simple TCP Server on Git](https://github.com/ashertoqeer/NettySimpleTCPServer)

##### Maven Dependency: 

[On maven central](https://mvnrepository.com/artifact/io.netty/netty-all/4.1.36.Final)

```
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.36.Final</version>
</dependency>
```

##### Bootstrap Netty Server:

Netty is based on [Java NIO](https://en.wikipedia.org/wiki/Non-blocking_I/O_(Java) ) which uses event based model as opposed to classic Java IO or old IO which uses thread based model. It means that if you are writing server using old IO, you need to have separate thread for each connection. But in NIO, multiple connections can share same thread which makes it a lot more efficient.

How exactly NIO event based model works is out of scope of this article. 

Let's create a server bootstrap:

```java
class SimpleNettyServerBootstrap {

    void start(int port) throws InterruptedException {
        Utils.log("Starting server at: " + port);
        EventLoopGroup bossGroup = new NioEventLoopGroup();	// (1)
        EventLoopGroup workerGroup = new NioEventLoopGroup(); // (2)
        try {
            ServerBootstrap b = new ServerBootstrap();	// (3)
            b.group(bossGroup, workerGroup)	// (4)
                    .channel(NioServerSocketChannel.class) // (5)
                    .childHandler(new SimpleTCPChannelInitializer()) // (6)
                    .childOption(ChannelOption.SO_KEEPALIVE, true); // (7)

            // Bind and start to accept incoming connections.
            ChannelFuture f = b.bind(port).sync();	// (8)
            if(f.isSuccess()) Utils.log("Server started successfully"); // (9)
            f.channel().closeFuture().sync(); // (10)
        } finally {
            Utils.log("Stopping server");
            workerGroup.shutdownGracefully(); // (11)
            bossGroup.shutdownGracefully(); // (12)
        }
    }
}
```

**(1):** Event loop group for accepting and establishing client connections

**(2):** Event loop group for handling established connections

**(3):** Create a server bootstrap

**(4):** Setting group in server bootstrap

**(5):** Configure Socket channel, since we are writing raw TCP server we will make TCP connection

**(6):** Setting our custom channel initializer, we will see its code in next section

**(7):**We need our connection to establish with Keep_Alive flag.

**(8):** Connect to given port

**(9):** Check if connection is successfully made

**(10):** Waiting for connection to close, this call is blocking

**(11):** When server is shutting down, shutdown worker event loop group

**(12):** When server is shutting down, shutdown boss event loop group

##### Initialize Netty Channel

In bootstrap point (6), you have notice `new SimpleTCPChannelInitializer()` is given. This class is responsible for initializing channel and setting channel handlers pipeline.

```java
public class SimpleTCPChannelInitializer extends ChannelInitializer<SocketChannel> {

    @Override
    protected void initChannel(SocketChannel socketChannel) throws Exception {
        socketChannel.pipeline().addLast(new StringEncoder()); // (1)
        socketChannel.pipeline().addLast(new StringDecoder());// (2)
        socketChannel.pipeline().addLast(new SimpleTCPChannelHandler()); // (3)
    }
}
```

In setup **(1) & (2)** we are setting encoders/decoders. Remember a channel is representation of a connection. In our case, a channel represents a TCP connection which only support data transfer in byte format. You can't directly send or receive objects over TCP channel. You have to convert from bytes to object while reading and from objects to bytes while writing.

For this purpose of conversion, Netty provides special kind of channel handlers called encoders/decoders. You can use any of the provided encoder/decoder or write your own for any custom needs.

Since for this simple TCP server, we only need strings to get transfered, we have use `StringEncoder` and `StringDecoder`.

At **(3)** we are adding our own handler to channel pipeline.

##### Adding Channel Handler

From Initialize step (3), we are setting our handler `new SimpleTCPChannelHandler()`. Lets create this handler

```java
public class SimpleTCPChannelHandler extends SimpleChannelInboundHandler<String> {

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        Utils.log(ctx.channel().remoteAddress(), "Channel Active");
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String s) throws Exception {
        Utils.log(ctx.channel().remoteAddress(), s);// (1)
        ctx.channel().writeAndFlush("Thanks\n"); // (2)
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        Utils.log(ctx.channel().remoteAddress(), "Channel Inactive");
    }
}
```

Here we have an **Inbound** handler as you can see we are extending from `SimpleChannelInboundHandler`. In handlers space, we have different options, for example you can directly implement from `ChannelInboundHandler` or extend from `ChannelInboundHandlerAdapter` which provides a default implementation of `ChannelInboundHandler`. Similar options are available for outbound handlers too.

`channelRead0(...)` would get call when there is a new message from client. In **(1)** we are just logging incoming message and In **(2)** we are send a response back, simply writing `Thanks` on channel.

The code is available on [Github.](https://github.com/ashertoqeer/NettySimpleTCPServer)

### Testing 

To test our server, we need to have a TCP client. You can use [netcat](http://netcat.sourceforge.net/) for that purpose. First run the server and then on terminal type following command (Assuming you have netcat installed). 

```
netcat localhost 8080
```

After connection, you can write messages to server and get a response back.