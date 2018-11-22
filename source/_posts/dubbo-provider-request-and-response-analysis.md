---
title: Dubbo服务端接收请求及响应请求原理分析
date: 2018-09-24 10:57:36
tags:
    - dubbo
---

之前一篇文章[《Dubbo服务提供者发布及注册过程源码分析》](https://shuaijunlan.github.io/2018/09/09/dubbo-provider-calling-process-source-code-analysis/)已经介绍了Dubbo服务端的服务注册及发布过程，这篇文章将会介绍Dubbo服务端是如何接受请求以及响应请求的。

本文还是以[**Consumer-Provider的Demo**](https://github.com/shuaijunlan/Spring-Learning/tree/master/dubbo)为例，分析接收请求及响应请求的具体流程，在Dubbo服务端发布服务之后，它将会监听一个端口等待接收客户端的请求，当接收到请求后，会经过入站处理器进行处理，我们知道在发布服务的时候设置了`NettyServerHandler`入站处理器，接收到请求之后，会经过`NettyServerHandler#channelRead()`方法来获取请求的消息，我们来看一下它的实现：

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    //以ctx.channel()为key，以NettyChannel为value，存储在ConcurrentHashMap中
    NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
    try {
        handler.received(channel, msg);
    } finally {
        NettyChannel.removeChannelIfDisconnected(ctx.channel());
    }
}
```

<!-- more -->

下面先来看一张整个接收请求和处理请求的流程图，下面将会围绕这张图进行详细的分析：

![](https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/Screenshot from 2018-10-02 15-49-47.png?raw=true)![](https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/Screenshot from 2018-10-02 15-50-13.png?raw=true)

来看一下`HeaderExchangeHandler#received()`方法：

```java
@Override
public void received(Channel channel, Object message) throws RemotingException {
    channel.setAttribute(KEY_READ_TIMESTAMP, System.currentTimeMillis());
    final ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
    try {
        if (message instanceof Request) {//如果是你Request类型的消息
            // handle request.
            Request request = (Request) message;
            if (request.isEvent()) {//事件类型的消息
                handlerEvent(channel, request);
            } else {
                if (request.isTwoWay()) {//请求，需返回：第一种情况，使用handleRequest()处理
                    handleRequest(exchangeChannel, request);
                } else {//请求，不需返回：第二种情况，处理请求不需要响应
                    handler.received(exchangeChannel, request.getData());
                }
            }
        } else if (message instanceof Response) {//如果是Response类型的消息：第三种情况
            handleResponse(channel, (Response) message);
        } else if (message instanceof String) {//如果是String类型的消息
            if (isClientSide(channel)) {
                Exception e = new Exception("Dubbo client can not supported string message: " + message + " in channel: " + channel + ", url: " + channel.getUrl());
                logger.error(e.getMessage(), e);
            } else {//响应回显请求
                String echo = handler.telnet(channel, (String) message);
                if (echo != null && echo.length() > 0) {
                    channel.send(echo);
                }
            }
        } else {//其他情况
            handler.received(exchangeChannel, message);
        }
    } finally {
        HeaderExchangeChannel.removeChannelIfDisconnected(channel);
    }
}
```

下面我们来分析request-response的模式，也是就分析handleRequest()方法：

```java
void handleRequest(final ExchangeChannel channel, Request req) throws RemotingException {
    //创建一个Response对象
    Response res = new Response(req.getId(), req.getVersion());
    //isBroken？是什么意思？，待进一步探究
    if (req.isBroken()) {
        Object data = req.getData();

        String msg;
        if (data == null) msg = null;
        else if (data instanceof Throwable) msg = StringUtils.toString((Throwable) data);
        else msg = data.toString();
        res.setErrorMessage("Fail to decode request due to: " + msg);
        res.setStatus(Response.BAD_REQUEST);

        channel.send(res);
        return;
    }
    // find handler by message class.
    Object msg = req.getData();
    try {
        // handle data.
        //异步处理请求，使用CompletableFuture接收结果
        //handler实例是在DubboProtocol类中通过匿名内部类实例化传递进来的，可以查看DubboProtocol的77行代码
        CompletableFuture<Object> future = handler.reply(channel, msg);
        //执行完成，获取结果
        if (future.isDone()) {
            res.setStatus(Response.OK);
            res.setResult(future.get());
            //返回结果
            channel.send(res);
            return;
        }
        //基于通知机制，获取结果
        future.whenComplete((result, t) -> {
            try {
                if (t == null) {
                    res.setStatus(Response.OK);
                    res.setResult(result);
                } else {
                    res.setStatus(Response.SERVICE_ERROR);
                    res.setErrorMessage(StringUtils.toString(t));
                }
                //返回结果
                channel.send(res);
            } catch (RemotingException e) {
                logger.warn("Send result to consumer failed, channel is " + channel + ", msg is " + e);
            } finally {
                // HeaderExchangeChannel.removeChannelIfDisconnected(channel);
            }
        });
    } catch (Throwable e) {
        //处理异常，返回结果
        res.setStatus(Response.SERVICE_ERROR);
        res.setErrorMessage(StringUtils.toString(e));
        channel.send(res);
    }
}
```

继续看DubboProtocol类中匿名内部类`ExchangeHandlerAdapter#reply()`方法：

```java
@Override
public CompletableFuture<Object> reply(ExchangeChannel channel, Object message) throws RemotingException {
    if (message instanceof Invocation) {
        Invocation inv = (Invocation) message;
        //获取调用者
        Invoker<?> invoker = getInvoker(channel, inv);
        // need to consider backward-compatibility if it's a callback
        if (Boolean.TRUE.toString().equals(inv.getAttachments().get(IS_CALLBACK_SERVICE_INVOKE))) {
            String methodsStr = invoker.getUrl().getParameters().get("methods");
            boolean hasMethod = false;
            if (methodsStr == null || !methodsStr.contains(",")) {
                hasMethod = inv.getMethodName().equals(methodsStr);
            } else {
                String[] methods = methodsStr.split(",");
                for (String method : methods) {
                    if (inv.getMethodName().equals(method)) {
                        hasMethod = true;
                        break;
                    }
                }
            }
            if (!hasMethod) {
                logger.warn(new IllegalStateException("The methodName " + inv.getMethodName()
                                                      + " not found in callback service interface ,invoke will be ignored."
                                                      + " please update the api interface. url is:"
                                                      + invoker.getUrl()) + " ,invocation is :" + inv);
                return null;
            }
        }
        //获取调用的上下文，底层使用ThreadLocal实现
        RpcContext rpcContext = RpcContext.getContext();
        boolean supportServerAsync = invoker.getUrl().getMethodParameter(inv.getMethodName(), Constants.ASYNC_KEY, false);
        if (supportServerAsync) {
            CompletableFuture<Object> future = new CompletableFuture<>();
            rpcContext.setAsyncContext(new AsyncContextImpl(future));
        }
        rpcContext.setRemoteAddress(channel.getRemoteAddress());
        //发起调用
        Result result = invoker.invoke(inv);

        if (result instanceof AsyncRpcResult) {
            return ((AsyncRpcResult) result).getResultFuture().thenApply(r -> (Object) r);
        } else {
            return CompletableFuture.completedFuture(result);
        }
    }
    throw new RemotingException(channel, "Unsupported request: "
                                + (message == null ? null : (message.getClass().getName() + ": " + message))
                                + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
}
```

最后获取结果执行`channel.send()`方法，向服务消费方返回数据

调用栈：

```
"DubboServerHandler-211.69.197.55:20881-thread-2@3215" daemon prio=5 tid=0x1a nid=NA runnable
  java.lang.Thread.State: RUNNABLE
	  at org.apache.dubbo.remoting.transport.netty4.NettyChannel.send(NettyChannel.java:101)
	  at org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeChannel.send(HeaderExchangeChannel.java:89)
	  at org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeChannel.send(HeaderExchangeChannel.java:78)
	  at org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeHandler.handleRequest(HeaderExchangeHandler.java:103)
	  at org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeHandler.received(HeaderExchangeHandler.java:196)
	  at org.apache.dubbo.remoting.transport.DecodeHandler.received(DecodeHandler.java:51)
	  at org.apache.dubbo.remoting.transport.dispatcher.ChannelEventRunnable.run(ChannelEventRunnable.java:57)
	  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	  at java.lang.Thread.run(Thread.java:748)
```

