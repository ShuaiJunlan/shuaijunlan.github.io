---
title: Netty 入门
date: 2018-05-10 21:58:26
tags:
    - Netty
---

> Netty is a NIO client server framework which enables quick and easy development of network applications such as protocol servers and clients. It greatly simplifies and streamlines network programming such as TCP and UDP socket server. 

![](https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/1526016569.png?raw=true)



<!-- more -->

### 特征

* 不同的传输类型（blocking and non-blocking socket ）使用统一的API
* 拥有灵活的易扩展的事件模型
* 高低自定义的线程模型-single thread, one or more thread pools such as SEDA 
* 高吞吐量，低时延
* 更少的资源占用
* 最小化不必要的内存拷贝
* 完全支持SSL/TLS和StartTLS

### 入门示例

#### 添加依赖

```xml
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.23.Final</version>
    <scope>compile</scope>
</dependency>
```

#### Server端

```java
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.serialization.ClassResolvers;
import io.netty.handler.codec.serialization.ObjectDecoder;
import io.netty.handler.codec.serialization.ObjectEncoder;

/**
 * @author Junlan Shuai[shuaijunlan@gmail.com].
 * @date Created on 13:44 2018/5/11.
 */
public class NettyServer {
    public void start(Integer port) throws InterruptedException {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workGroup = new NioEventLoopGroup(4);

        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(bossGroup, workGroup).channel(NioServerSocketChannel.class)
                .localAddress(port)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline()
                                .addLast(new ObjectDecoder(1024*1024,
                                        ClassResolvers.weakCachingConcurrentResolver(this.getClass().getClassLoader())) )
                                .addLast(new ObjectEncoder())
                                .addLast(new SimpleChannelInboundHandler<Object>() {
                                    @Override
                                    protected void channelRead0(ChannelHandlerContext ctx, Object msg) throws Exception {
                                        System.out.println("Receive message:" + msg);
                                        ctx.writeAndFlush("Hello " + msg);
                                    }
                                });
                    }
                });
        ChannelFuture channelFuture = bootstrap.bind().sync();
        channelFuture.channel().closeFuture().sync();
        bossGroup.shutdownGracefully().sync();
        workGroup.shutdownGracefully().sync();
    }

    public static void main(String[] args) {
        try {
            NettyServer server = new NettyServer();
            server.start(20000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

#### Client端

```java
import io.netty.bootstrap.Bootstrap;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.serialization.ClassResolvers;
import io.netty.handler.codec.serialization.ObjectDecoder;
import io.netty.handler.codec.serialization.ObjectEncoder;

/**
 * @author Junlan Shuai[shuaijunlan@gmail.com].
 * @date Created on 13:53 2018/5/11.
 */
public class NettyClient {
    public Channel channel;
    public void start(String host, Integer port) throws InterruptedException {
        EventLoopGroup workGroup = new NioEventLoopGroup(4);
        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(workGroup)
                .channel(NioSocketChannel.class)
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
                .handler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        // Using Java Object serializable, you can also use other serializable frameworks like thrift, Protobuf and so on.
                        ch.pipeline()
                                .addLast(new ObjectDecoder(1024*1024,
                                        ClassResolvers.weakCachingConcurrentResolver(this.getClass().getClassLoader())) )
                                .addLast(new ObjectEncoder())
                                .addLast(new SimpleChannelInboundHandler<Object>() {
                                    @Override
                                    protected void channelRead0(ChannelHandlerContext ctx, Object msg) throws Exception {
                                        System.out.println("Receive msg: " + msg);
                                    }
                                });
                    }
                });
        // Connect to the server sync
        channel = bootstrap.connect(host, port).sync().channel();
    }

    public static void main(String[] args) {
        try {
            NettyClient nettyClient = new NettyClient();
            nettyClient.start("127.0.0.1", 20000);
            if (nettyClient.channel != null && nettyClient.channel.isActive()){
                System.out.println("Send message to server");
                nettyClient.channel.writeAndFlush("Junlan");
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

