---
title: Netty内置传输方式
date: 2018-07-22 16:19:51
tags:
    - Netty
---
Netty中内置的传输方式主要包括：NIO、Epoll、OIO、Local和Embedded等方式，总结如下：


| 名称     | 包名                             | 描述                                                         |
| -------- | -------------------------------- | ------------------------------------------------------------ |
| NIO      | io.netty.channel.socket.io       | 使用java.nio.channels包作为基础，基于选择器的方式            |
| Epoll    | io.netty.channel.epoll           | 由JNI驱动的epoll()和非阻塞IO，这种传输只有Linux才支持，比NIO传输速度更快，而且是完全非阻塞的 |
| OIO      | io.netty.channel.socket.oio      | 使用java.net包作为基础                                       |
| Local    | io.netty.channel.local           | 可以在VM内部通过管道进行通信的本地传输                       |
| Embedded | io.netty.channel.socket.embedded | Embedded 传输，允许使用 ChannelHandler 而又不需要一个真正的基于网络的传输，测试ChannelHandler实现的时候非常有用 |

<!-- more -->

### 基于NIO的传输

![](https://shuaijunlan.github.io/images/Screenshot from 2018-07-21 17-33-12.png)

### Epoll---基于Linux的本地非阻塞传输

在Linux内核版2.5.44之后就引入了epoll，提供了比旧的POSIX select和poll系统调用更好的性能。如何应用程序运行在Linux系统智商，则可以利用基于Epoll的方式传输，只需要在代码中将`NioEventLoopGroup`替换为`EpollEventLoopGroup`，并且将`NioServerSocketChannel.class`替换为`EpollServerSocketChannel.class`。或者用如下方式判断系统是否支持Epoll：

```java
EventLoopGroup workGroup = Epoll.isAvailable() ? new EpollEventLoopGroup(4) : new NioEventLoopGroup(4);
Bootstrap bootstrap = new Bootstrap();
bootstrap.group(workGroup)
        .channel(Epoll.isAvailable() ? EpollSocketChannel.class : NioSocketChannel.class)
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
```

### OIO---阻塞I/O

![](https://shuaijunlan.github.io/images/Screenshot from 2018-07-21 19-41-49.png)

