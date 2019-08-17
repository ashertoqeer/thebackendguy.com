---
layout: post
title: "Netty Simple TCP Client"
permalink: "netty-simple-tcp-client"
last_modified_at: 2019-06-23T00:00:00
excerpt: "Netty is Java NIO based framework for writing network applications. This article explains how to write a simple TCP client using Netty."
category: "Network Programming"
---

Netty is Java NIO based framework for writing network applications. Let's write a simple TCP client.

```
<!-- https://mvnrepository.com/artifact/io.netty/netty-all -->
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
	<version>4.1.36.Final</version>
</dependency>
```

Netty represent connections as **Channels** and provide **Channel Handlers** (Inbound/Outbound) for read/write operations. A special class **Bootstrap** put things together and help establish connection.

```java
public class NettyTCPClient {

    private Logger logger = Logger.getLogger(getClass().getSimpleName());

    void start(String host, int port){
        // Event loop group to Handle I/O operations for channel
        EventLoopGroup eventLoopGroup = new NioEventLoopGroup();

        // Help boot strapping a channel
        Bootstrap clientBootstrap = new Bootstrap();
        clientBootstrap
          .group(eventLoopGroup) // associate event loop to channel
          .channel(NioSocketChannel.class) // create a NIO socket channel
          .handler(new TCPClientChannelInitializer()); // Add channel initializer

        try {
            // Connect to listening server
            ChannelFuture channelFuture = clientBootstrap.connect(host, port).sync();
            // Check if channel is connected
            if(channelFuture.isSuccess()) logger.log(Level.INFO, "connected") ;
            // Block till channel is connected
            channelFuture.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            logger.log(Level.INFO, "connection failed: {}", e);
        }finally {
            // Connection is closed, clean up
            logger.log(Level.INFO, "closing") ;
            eventLoopGroup.shutdownGracefully();
        }
    }
}
```

Lets see implementation of `TCPClientChannelInitializer()` which basically configure channel handler pipeline. Handlers will be called one by one. [pipeline official docs](https://netty.io/4.0/api/io/netty/channel/ChannelPipeline.html)

```java
class TCPClientChannelInitializer extends ChannelInitializer<SocketChannel>{

    @Override
    protected void initChannel(SocketChannel socketChannel) {
        // Configure encoders/decoder or codecs
        socketChannel.pipeline().addLast(new StringDecoder());
        socketChannel.pipeline().addLast(new StringEncoder());
        // Add Custom Inbound handler to handle incoming traffic
        socketChannel.pipeline().addLast(new TCPClientInboundHandler());
    }
}
```

**Codecs** are special handlers for byte/object conversion. **TCPClientInboundHandler()** is our custom inbound handler to handle incoming traffic, lets see its implementation:

```java
class TCPClientInboundHandler extends SimpleChannelInboundHandler<String>{

    private Logger logger = Logger.getLogger(getClass().getSimpleName());

    @Override
    protected void channelRead0(ChannelHandlerContext channelHandlerContext, String s) {
        logger.log(Level.INFO, "Received: " + s); // Display received data
        channelHandlerContext.writeAndFlush("Acknowledged"); // Return a sample response
    }
}
```

**channelRead0(...)** will be called whenever server sent something. **channelHandlerContext.writeAndFlush(...)** is used to send something to server.

To run the application:

```java
public class Main {
 
    public static void main(String[] args){
        NettyTCPClient nettyTCPClient = new NettyTCPClient();
        nettyTCPClient.start("localhost", 8080);
	}   
}
```



### Use Netcat to verify:

Run in terminal:

```
nc -l 8080
```

Netcat will start listening on 8080. You can then test Netty Simple TCP Client.
