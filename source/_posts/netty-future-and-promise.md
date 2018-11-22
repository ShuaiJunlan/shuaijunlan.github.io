---
title: Netty中Future与Promise的实现分析
date: 2018-09-06 15:53:53
tags:
    - Netty
---

从Java 1.5开始，JDK就提供了Callable和Future，通过它们可以在任务异步执行完毕之后获取任务的执行结果。

Netty扩展了JDK中的Future机制，下面我们来看一张Netty中Future和Promise类关系图：

![](https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/DefaultPromise.png?raw=true)

<!-- more -->

最上面的Future是JDK提供的Future接口，我们看一下里面有哪些方法：

```java
package java.util.concurrent;

/**
 * A {@code Future} represents the result of an asynchronous
 * computation.  Methods are provided to check if the computation is
 * complete, to wait for its completion, and to retrieve the result of
 * the computation.  The result can only be retrieved using method
 * {@code get} when the computation has completed, blocking if
 * necessary until it is ready.  Cancellation is performed by the
 * {@code cancel} method.  Additional methods are provided to
 * determine if the task completed normally or was cancelled. Once a
 * computation has completed, the computation cannot be cancelled.
 * If you would like to use a {@code Future} for the sake
 * of cancellability but not provide a usable result, you can
 * declare types of the form {@code Future<?>} and
 * return {@code null} as a result of the underlying task.
 */
public interface Future<V> {
	//尝试取消执行的任务
    boolean cancel(boolean mayInterruptIfRunning);
	//判断任务是否取消
    boolean isCancelled();
	//判断任务是否执行完成
    boolean isDone();
	//等待直到任务执行完成，并且返回结果
    V get() throws InterruptedException, ExecutionException;
	//等待最大超时时间，如果执行完成则返回结果，否则抛出超时异常
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

**并发编程中，我们通常会用到一组非阻塞的模型：Promise，Future 和 Callback。其中的 Future 表示一个可能还没有实际完成的异步任务的结果，针对这个结果可以添加 Callback 以便在任务执行成功或失败后做出对应的操作，而Promise交由任务执行者，任务执行者通过 Promise 可以标记任务完成或者失败。 这一套模型是很多异步非阻塞架构的基础。** 因为netty中也有类似`调用者和执行部件以异步的方式交互通信结果`的需求（要知道eventloop本质上是一个ScheduledExecutorService，ExecutorService是一种“提交-执行”模型实现，也存在线程间异步方式通信和线程安全问题），所以netty自己实现了一套。

Netty中所有的IO操作都是异步的（比如write(Object)，因为Netty中一个EventLoop要服务于多个SocketChannel，所以通过`socketChannel.xx`只是提交一个任务，何时返回结果是不确定的），而不像传统的BIO那样阻塞等待操作完成，来获取执行结果。

事实上，Netty Future的建议操作模式就是赤裸裸的通知，执行部件改变状态时，会执行注册在future上的Listener（变相的观察者模式）。

scala在语言层面提供对Promise，Future和Callback模型的支持，`https://bitbucket.org/qiyi/commons-future.git`作者自定义实现了该模型，去除了Netty的Future模型对EventLoop的依赖。

#### Netty中的Future

下面的Netty Future继承jdk提供的Future接口，添加了一些自己的方法：

```java
/**
 * The result of an asynchronous operation.
 */
@SuppressWarnings("ClassNameSameAsAncestorName")
public interface Future<V> extends java.util.concurrent.Future<V> {

    boolean isSuccess();

    boolean isCancellable();

    Throwable cause();

    Future<V> addListener(GenericFutureListener<? extends Future<? super V>> listener);

    Future<V> addListeners(GenericFutureListener<? extends Future<? super V>>... listeners);

    Future<V> removeListener(GenericFutureListener<? extends Future<? super V>> listener);

    Future<V> removeListeners(GenericFutureListener<? extends Future<? super V>>... listeners);

    Future<V> sync() throws InterruptedException;

    Future<V> syncUninterruptibly();

    Future<V> await() throws InterruptedException;

    Future<V> awaitUninterruptibly();

    boolean await(long timeout, TimeUnit unit) throws InterruptedException;

    boolean await(long timeoutMillis) throws InterruptedException;

    boolean awaitUninterruptibly(long timeout, TimeUnit unit);

    boolean awaitUninterruptibly(long timeoutMillis);

    V getNow();

    @Override
    boolean cancel(boolean mayInterruptIfRunning);
}
```

![](http://misc.linkedkeeper.com/misc/img/blog/201612/linkedkeeper0_c93937f4-f0f5-419e-a8b1-e71dc2288003.jpg)

a future is a read-only placeholder view of a variable, while a promise is a writable。Promise是可写的Future，提供写操作相关的接口，用于设置IO操作的结果。Future，Promise，callback抽象出一套调用者与执行部件间的通信模型（不只是netty中），**Future像是给调用者用的（拿结果），Promise像是给执行部件用的（设置结果），它们简化了调用者和执行部件对其的调用（调用者get，执行部件set），但本身要封装很多事**。比如Future必须是一个线程安全的类（大部分时候，调用者和执行部件身处两个线程），比如执行callback（或者listener）。

#### Netty Promise执行Listener

Netty中的Promise通常由EventLoop创建，也就是Promise通常会绑定executor。为何呢？**因为Netty保证Listener的执行，一定是在channel对应的EventLoop中**(????)。