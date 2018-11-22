---
title: HTTP详解
date: 2018-05-19 13:31:08
tags:
    - http
---

The Hypertext Transfer Protocol (HTTP) is an application-level protocol that uses TCP as an underlying transport and typically runs on port 80. HTTP is a stateless protocol i.e. server maintains no information about past client requests. 

<!-- more -->

### HTTP与TCP/IP的关系

HTTP的长连接和短连接本质上是TCP长连接和短连接。HTTP属于应用层协议，在传输层使用TCP协议，在网络层使用IP协议。IP协议主要解决网络路由和寻址问题，TCP协议主要解决如何在IP层之上可靠的传递数据包，使在网络上的另一端收到发端发出的所有包，并且顺序与发出顺序一致。TCP有可靠，面向连接的特点。 

### HTTP是无状态的

HTTP协议是无状态的，指的是协议对于事务处理没有记忆能力，服务器不知道客户端是什么状态。也就是说，打开一个服务器上的网页和你之前打开这个服务器上的网页之间没有任何联系。HTTP是一个无状态的面向连接的协议，无状态不代表HTTP不能保持TCP连接，更不能代表HTTP使用的是UDP协议（无连接）。 

### 长连接和短连接

**在HTTP/1.0中，默认使用的是短连接**。也就是说，浏览器和服务器每进行一次HTTP操作，就建立一次连接，但任务结束就中断连接。如果客户端浏览器访问的某个HTML或其他类型的 Web页中包含有其他的Web资源，如JavaScript文件、图像文件、CSS文件等；当浏览器每遇到这样一个Web资源，就会建立一个HTTP会话。

但从 **HTTP/1.1起，默认使用长连接**，用以保持连接特性。使用长连接的HTTP协议，会在响应头有加入这行代码：`Connection:keep-alive`

相对于持久化连接HTTP1.1还有另外比较重要的改动：

- HTTP 1.1增加host字段
- HTTP 1.1中引入了Chunked transfer-coding，范围请求，实现断点续传(实际上就是利用HTTP消息头使用分块传输编码，将实体主体分块传输)
- HTTP 1.1管线化(pipelining)理论，客户端可以同时发出多个HTTP请求，而不用一个个等待响应之后再请求
  - ​    注意：这个pipelining仅仅是限于理论场景下，大部分桌面浏览器仍然会选择默认关闭HTTP pipelining！
  - ​    所以现在使用HTTP1.1协议的应用，都是有可能会开多个TCP连接的！

#### 短连接

我们模拟一下TCP短连接的情况，client向server发起连接请求，server接到请求，然后双方建立连接。client向server 发送消息，server回应client，然后一次读写就完成了，这时候双方任何一个都可以发起close操作，不过一般都是client先发起 close操作。为什么呢，一般的server不会回复完client后立即关闭连接的，当然不排除有特殊的情况。从上面的描述看，短连接一般只会在 client/server间传递一次读写操作

* 优点：管理起来比较简单，存在的连接都是有用的连接，不需要额外的控制手段

#### 长连接

接下来我们再模拟一下长连接的情况，client向server发起连接，server接受client连接，双方建立连接。Client与server完成一次读写之后，它们之间的连接并不会主动关闭，后续的读写操作会继续使用这个连接。 

* 优点
  * Lower CPU and memory usage because there are less number of connections. 
  * Allows HTTP pipelining of requests and responses. 
  * Reduced network congestion (fewer TCP connections). 
  * Reduced latency in subsequent requests (no handshaking). 
  * Errors can be reported without the penalty of closing the TCP connection. 

* 缺点
  * Resources may be be kept occupied even when not needed and may not be available to others. 



### REFERENCE

http://developer.51cto.com/art/201808/580780.htm

[Persistent Connections](https://www.oreilly.com/library/view/http-the-definitive/1565925092/ch04s05.html)

[HTTP Non-Persistent & Persistent Connection | Set 1](https://www.geeksforgeeks.org/http-non-persistent-persistent-connection/)

[HTTP长连接和短连接](http://www.cnblogs.com/0201zcr/p/4694945.html)

