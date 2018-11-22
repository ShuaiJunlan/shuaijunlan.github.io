---
title: Netty组件介绍
date: 2018-07-20 19:48:01
tags:
    - Netty
---

下面将介绍Netty中所包含的各种组件，主要包括：Channel、EventLoop、ChannelFuture、ChannelHandler和ChannelPipeline等。

### Channel、EventLoop和ChannelFuture



![Channel-EventLoop-ChannelFuture](https://shuaijunlan.github.io/images/Screenshot from 2018-07-20 20-39-27.png)

<!-- more -->

* 一个EventLoopGroup可以包含一个或者多个EventLoop；
* 一个EventLoop在它的生命周期内只和一个Thread绑定；
* 所有由EventLoop处理的I/O事件都将在它专有的Thread上被处理；
* 一个Channel在它的生命周期内只能注册于一个EventLoop；
* 一个EventLoop可能被分配给多个Channel；
* 在Netty中所有的I/O操作都是异步的，因此可以通过ChannelFuture来获取响应结果；

### ChannelHandler和ChannelPipeline

![](https://shuaijunlan.github.io/images/Screenshot from 2018-07-20 21-01-11.png)

* ChannelPipeline提供了ChannelHandler链的容器，并定义了同于在该链上传播入站的出站的事件流的API；
* 一个ChannelIninializer的实现被注册到ServerBootstrap中或这Client的BootStrap中；
* 当 ChannelInitializer.initChannel() 方法被调用时，ChannelInitializer将在 ChannelPipeline 中安装一组自定义的 ChannelHandler ；
* ChannelInitializer将它自己从 ChannelPipeline 中移除；
* 从一个客户端应用程序角度来看，当事件的运动方向是从客户端到服务器，我们称之为出站，反之则称之为入站；
* 数据出站运动，数据讲从ChannelOutboundHandler链的尾端开始流动，直到它到达链的头部为止，此时数据到达了网络传输层，通常情况下会触发一个写操作。

### BootStrap和ServerBootstrap

|         类别         |      Bootstrap       |  ServerBootstrap   |
| :------------------: | :------------------: | :----------------: |
|   网络编程中的作用   | 连接到远程主机和端口 | 绑定到一个本地端口 |
| EventLoopGroup的数目 |          1           |   2（也可以1个）   |

![](https://shuaijunlan.github.io/images/Screenshot from 2018-07-20 21-52-02.png)

上图中，客户端创建了一个EventLoopGroup，服务器端使用了两个EventLoopGroup，其中一个EventLoopGroup用来接收客户端发来的请求，另一个EventLoop用来处理连接任务。

### 示例

**服务器端**

```java
EventLoopGroup bossGroup = new EpollEventLoopGroup(1);
EventLoopGroup workerGroup = new EpollEventLoopGroup(4);
try {
    ServerBootstrap b = new ServerBootstrap();
    b.group(bossGroup, workerGroup)
            .channel(EpollServerSocketChannel.class)
            //保持长连接状态
            .childOption(ChannelOption.SO_KEEPALIVE, true)
            .childOption(ChannelOption.TCP_NODELAY, true)
            .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
            .childHandler(new HttpSnoopServerInitializer());

    ChannelFuture ch = b.bind(PORT).sync();
    if (ch.isSuccess()){
        logger.info("Http server start on port :{}", PORT );
    }

    ch.channel().closeFuture().sync();
} finally {
    bossGroup.shutdownGracefully();
    workerGroup.shutdownGracefully();
}
```

**客户端**

```java
EventLoopGroup eventLoopGroup = new EpollEventLoopGroup(8);
Bootstrap bootstrap = new Bootstrap()
        .group(eventLoopGroup)
        .option(ChannelOption.SO_KEEPALIVE, true)
        .option(ChannelOption.TCP_NODELAY, true)
        .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
        .channel(EpollSocketChannel.class)
        .handler(new RpcClientInitializer());
Channel channel = bootstrap.connect("127.0.0.1", port).sync().channel();
```

