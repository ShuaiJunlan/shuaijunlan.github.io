---
title: Java NIO and BIO
date: 2018-04-19 18:43:24
tags:
    - java
    - NIO
---

![](https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/1526020447.png?raw=true)

我们都知NIO是非阻塞IO，BIO是阻塞IO，那到底什么是阻塞，什么是非阻塞呢，它们与同步/异步又有什么区别呢？先来了解一下阻塞/非阻塞，同步/异步的概念。

<!-- more -->

#### 阻塞/非阻塞/同步/异步

* 阻塞：

  当某个事件或者任务在执行过程中，它发出一个请求操作，但是由于该请求操作需要的条件不满足，那么就会一直在那等待，直至条件满足；


* 非阻塞：

  当某个事件或者任务在执行过程中，它发出一个请求操作，如果该请求操作需要的条件不满足，会立即返回一个标志信息告知条件不满足，不会一直在那等待。

* 同步：

  可以理解为在执行一个函数或方法，只有接收到返回的值或消息后才会继续往下执行其他的命令。

  ![](https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/1525844998.png?raw=true)

  callee执行完成才返回
  返回值即结果


* 异步：

  可以理解为在执行一个函数或方法，不用等待其返回，继续往下执行其他的命令。

  ![](https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/1525845072.png?raw=true)

  callee不需要执行完成就可返回
  caller要获取结果，需要通过轮询、回调等机制


**Update**

> 同步与异步的区别：函数调用发生时，消息(参数)从caller传递到callee，控制权(指令执行)从caller转移到callee。调用返回时，控制权从callee转移到caller。两者的区别在于，callee是否需要等待执行完成才将控制权转移给caller。

#### 阻塞IO和非阻塞IO

通常来说，IO操作包括：对硬盘的读写、对socket的读写以及外设的读写。

当用户线程发起一个IO请求操作（本文以读请求操作为例），内核会去查看要读取的数据是否就绪，对于阻塞IO来说，如果数据没有就绪，则会一直在那等待，直到数据就绪；对于非阻塞IO来说，如果数据没有就绪，则会返回一个标志信息告知用户线程当前要读的数据没有就绪。当数据就绪之后，便将数据拷贝到用户线程，这样才完成了一个完整的IO读请求操作，也就是说一个完整的IO读请求操作包括两个阶段：

* 查看数据是否就绪；
* 进行数据拷贝（内核将数据拷贝到用户线程）。

那么阻塞（BIO）和非阻塞（NIO）的区别就在于第一个阶段，如果数据没有就绪，在查看数据是否就绪的过程中是一直等待，还是直接返回一个标志信息。

Java中传统的IO都是阻塞IO，比如通过socket来读数据，调用read()方法之后，如果数据没有就绪，当前线程就会一直阻塞在read方法调用那里，直到有数据才返回；而如果是非阻塞IO的话，当数据没有就绪，read()方法应该返回一个标志信息，告知当前线程数据没有就绪，而不是一直在那里等待，从jdk1.4开始引入NIO。	

#### 基于Java API实现NioServer

```java
/**
 * @author Junlan Shuai[shuaijunlan@gmail.com].
 * @date Created on 14:19 2018/4/15.
 *
 * 基于Java API实现NIO server
 */
public class PlainNioServer {
    public void server(int port) throws IOException {
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false);
        ServerSocket serverSocket =serverSocketChannel.socket();
        //将服务器绑定到选定的端口
        InetSocketAddress address = new InetSocketAddress(port);
        serverSocket.bind(address);
        //打开Selector来处理Channel
        Selector selector = Selector.open();
        //将ServerSocket注册到Selector以接受连接
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        final ByteBuffer msg = ByteBuffer.wrap("Hi! \r\n".getBytes());

        for (;;){
            try {
                //等待需要处理的新事件；阻塞将一直持续到下一个传入事件
                selector.select();
            } catch (IOException ex){
                ex.printStackTrace();
                break;
            }
            //获取所有接收事件的SelectorKey实例
            Set<SelectionKey> readyKeys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = readyKeys.iterator();
            while (iterator.hasNext()){
                SelectionKey key = iterator.next();
                iterator.remove();
                //检查事件是否是一个新的已经就绪可以被接受的连接
                if (key.isAcceptable()){
                    ServerSocketChannel server = (ServerSocketChannel) key.channel();
                    SocketChannel client = server.accept();
                    client.configureBlocking(false);
                    //接受客户端，并将它注册到选择器
                    client.register(selector, SelectionKey.OP_WRITE | SelectionKey.OP_READ, msg.duplicate());
                    System.out.println("Accepted connection from " + client);

                }
                //检查套接字是否已经准备好写数据
                if (key.isWritable()){
                    SocketChannel client = (SocketChannel)key.channel();
                    ByteBuffer byteBuffer = (ByteBuffer)key.attachment();
                    while (byteBuffer.hasRemaining()){
                        //将数据写到已连接的客户端
                        if (client.write(byteBuffer) == 0){
                            break;
                        }
                    }
                    //关闭连接
                    client.close();

                }
                key.cancel();
                key.channel().close();
            }
        }

    }
}
```







