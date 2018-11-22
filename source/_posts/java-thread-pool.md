---
title: Java线程池分析
date: 2018-10-11 09:03:18
tags:
    - java
---

### 基于ThreadPoolExecutor构造线程池

我们来看一下ThreadPoolExecutor类的构造函数，一共需要传入7个参数，下面的注释中有详细的解释：

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    //先判断参数是否合法
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
        null :
    	AccessController.getContext();
    //核心线程数量
    this.corePoolSize = corePoolSize;
    //最大线程数量
    this.maximumPoolSize = maximumPoolSize;
    //工作队列
    this.workQueue = workQueue;
    //保活时间，unit为时间单位，转换为秒
    this.keepAliveTime = unit.toNanos(keepAliveTime);、
    //线程工厂
    this.threadFactory = threadFactory;
    //拒绝策略
    this.handler = handler;
}
```

工作队列使用的是一种阻塞队列，**关于阻塞队列的实现我将会在另一片文章中详细讲解**。

<!-- more -->

#### 提交任务

提交任务的过程主要分为三步：

```java
/*
* Proceed in 3 steps:
*
* 1. If fewer than corePoolSize threads are running, try to
* start a new thread with the given command as its first
* task.  The call to addWorker atomically checks runState and
* workerCount, and so prevents false alarms that would add
* threads when it shouldn't, by returning false.
*
* 2. If a task can be successfully queued, then we still need
* to double-check whether we should have added a thread
* (because existing ones died since last checking) or that
* the pool shut down since entry into this method. So we
* recheck state and if necessary roll back the enqueuing if
* stopped, or start a new thread if there are none.
*
* 3. If we cannot queue task, then we try to add a new
* thread.  If it fails, we know we are shut down or saturated
* and so reject the task.
*/
int c = ctl.get();
if (workerCountOf(c) < corePoolSize) {
    if (addWorker(command, true))
        return;
    c = ctl.get();
}
if (isRunning(c) && workQueue.offer(command)) {
    int recheck = ctl.get();
    if (! isRunning(recheck) && remove(command))
        reject(command);
    else if (workerCountOf(recheck) == 0)
        addWorker(null, false);
}
else if (!addWorker(command, false))
    reject(command);
```

* 1.如果运行的线程数小于`corePoolSize`，则会调用`addWorker()`，检查`runState`和`workerCount`，并且**创建一个新的线程**，把任务给它当做第一个任务来执行，然后就返回。
* 2.如果任务成功的**加入队列**，然后我们仍然需要去再次检查是否我们应该添加一个thread（因为在之前一次检查之后可能会有thread终止），或者线程池停止了自从进入这个函数。因此我们需要重复检查状态，并且在线程池停止的情况下回滚入队操作，或者开启一个新的线程如果这没有线程。
* 3.如果任务不能加入队列（队列已满），则尝试**创建一个新的线程**（此时线程数量应该小于maximumPoolSize）。如果创建失败，我们知道可能是线程池停止或者线程数量达到了`maximumPoolSize`，因此会拒绝这个任务。

#### 五种运行状态

```java
// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS; //接收新的任务并且处理队列中的任务
private static final int SHUTDOWN   =  0 << COUNT_BITS; //不接受新的任务，但是会继续处理队列中的任务
private static final int STOP       =  1 << COUNT_BITS; //不接受新的任务，也不处理队列中的任务，并且打断中正在处理的任务
private static final int TIDYING    =  2 << COUNT_BITS; //所有任务都终止，workerCount为零，线程状态过渡到TIDYING，将会执行terminated()方法
private static final int TERMINATED =  3 << COUNT_BITS; //terminated()函数执行完成
```
状态转换：

|      原始状态       |  目标状态  |                           转变原因                           |
| :-----------------: | :--------: | :----------------------------------------------------------: |
|       RUNNING       |  SHUTDOWN  | On invocation of shutdown(), perhaps implicitly in finalize() |
| RUNNING or SHUTDOWN |    STOP    |                On invocation of shutdownNow()                |
|      SHUTDOWN       |  TIDYING   |              When both queue and pool are empty              |
|        STOP         |  TIDYING   |                      When pool is empty                      |
|       TIDYING       | TERMINATED |       When the terminated() hook method has completed        |

### 饱和策略分析

当有界队列填满后，饱和策略开始发挥作用，所有饱和策略实现类都实现了`RejectExecutionHandler`接口，我们来看一下JDK提供了哪些饱和策略实现类：

![](https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/RejectedExecutionHandler.png?raw=true)

从类的关系图中可以看出，JDK实现了`DiscardPolicy、CallerRunsPolicy、DiscardOldestPolicy和AbortPolicy`四种饱和策略，下面我们将会一一进行分析。

#### AbortPolicy

```java
    /**
     * A handler for rejected tasks that throws a
     * {@code RejectedExecutionException}.
     */
    public static class AbortPolicy implements RejectedExecutionHandler {
        /**
         * Creates an {@code AbortPolicy}.
         */
        public AbortPolicy() { }

        /**
         * Always throws RejectedExecutionException.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         * @throws RejectedExecutionException always
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }
```



中止策略是默认的饱和策略，该则略将会抛出未检查的RejectedExecutionException，调用者可以捕获这个异常，然后根据需求编写自己的处理代码。

#### CallerRunsPolicy

```java
    /**
     * A handler for rejected tasks that runs the rejected task
     * directly in the calling thread of the {@code execute} method,
     * unless the executor has been shut down, in which case the task
     * is discarded.
     */
    public static class CallerRunsPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code CallerRunsPolicy}.
         */
        public CallerRunsPolicy() { }

        /**
         * Executes task r in the caller's thread, unless the executor
         * has been shut down, in which case the task is discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }
```



调用者运行策略实现了一种调节机制，该策略不抛弃任何任务，也不会抛出异常，而是将某些任务会退到调用者执行，从而降低新任务的流量。他不会在线程池中的某个线程中执行任务，而是在调用execute()方法的线程中执行该任务。

#### DiscardPolicy

```java
    /**
     * A handler for rejected tasks that silently discards the
     * rejected task.
     */
    public static class DiscardPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardPolicy}.
         */
        public DiscardPolicy() { }

        /**
         * Does nothing, which has the effect of discarding task r.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }
```



抛弃策略，直接抛弃该提交的任务。

#### DiscardOldestPolicy

```java
    /**
     * A handler for rejected tasks that discards the oldest unhandled
     * request and then retries {@code execute}, unless the executor
     * is shut down, in which case the task is discarded.
     */
    public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardOldestPolicy} for the given executor.
         */
        public DiscardOldestPolicy() { }

        /**
         * Obtains and ignores the next task that the executor
         * would otherwise execute, if one is immediately available,
         * and then retries execution of task r, unless the executor
         * is shut down, in which case task r is instead discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
```



抛弃最旧的策略，则会抛弃下一个将会执行的任务，然后尝试重新提交该任务。如果工作队列是一个优先级队列，那么该策略会导致抛弃优先级最高的任务，因此最好不要将优先级队列和DiscardOldestPolicy一起使用。

### 自定义线程工厂

在ThreadPoolExecutor构造函数中可以看到，在构造线程池时，可以传入自定义的线程工厂，也可以使用默认的线程工厂。我们先来看一下默认的线程工厂的实现，它是Executors的静态内部类：

```java
/**
     * The default thread factory
     */
static class DefaultThreadFactory implements ThreadFactory {
    //记录创建默认线程池的数量，类变量
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    //线程组
    private final ThreadGroup group;
    //记录创建线程的数量，实例变量
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    //线程名称前缀
    private final String namePrefix;

    DefaultThreadFactory() {、
        //这句话啥意思？
        SecurityManager s = System.getSecurityManager();
        //赋值线程组
        group = (s != null) ? s.getThreadGroup() :
        Thread.currentThread().getThreadGroup();
        namePrefix = "pool-" +
            poolNumber.getAndIncrement() +
            "-thread-";
    }

    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r,
                              namePrefix + threadNumber.getAndIncrement(),
                              0);
        if (t.isDaemon()) //??为什么这里判断是守护进程，还设置为false？？
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)//设置优先级
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
```

我们可以参考上面的默认线程工厂的实现方式，可以自定义任意的线程工厂。

### Executors类分析

在Executors类中主要提供了三种创建线程池的方法，分别是newCachedThreadPool()、newFixedThreadPool()和newSIngleThreadPool()，下面对这三种方式进行对比分析：

|         方法          | corePoolSize |  maximumPoolSize  | keepAliveTime&TimeUnit |    BlockingQueue    |
| :-------------------: | :----------: | :---------------: | :--------------------: | :-----------------: |
| newCachedThreadPool() |      0       | Integer.MAX_VALUE |          60s           |  SynchronousQueue   |
| newFixedThreadPool()  |   nThreads   |     nThreads      |          0ms           | LinkedBlockingQueue |
| newSIngleThreadPool() |      1       |         1         |          0ms           | LinkedBlockingQueue |
|  ScheduledThreadPool  |              |                   |                        |                     |
在这里提两个问题：

* 1.为什么FixedThreadExecutor的corePoolSize和maximumPoolSize要设计成一样的？
* 2.为什么CachedThreadExecutor的maximumPoolSize要设计成Integer.MAX_VALUE？

> 对于问题一，因为线程池是先判断corePoolSize,再判断workQueue,最后判断maximumPoolSize，然而LinkedBlockingQueue是无界队列（Integer.MAX_VALUE），所以他是达不到判断maximumPoolSize这一步的，所以maximumPoolSize设置成多少，并没有多大关系。

------

> 对于问题二，因为SynchronousQueue设计的原因，如果maximumPoolSize不设计的很大，那么就很容易导致线程占满，然后抛出异常。

### 关闭线程池

通过源码可以看到，停止线程池执行主要是两个方法，第一个是`shutdown()`，第二个是`shutdownNow()`方法，他们的主要区别是：

* 调用shutdown()方法，将会把线程池的状态标记为`SHUTDOWN`，通过前面的状态分析，我们知道这个状态下线程池将不会接收任何新的任务，并且会继续执行队列里的任务，直到所有任务执行完成，终                                                                                                                                                                                                         止线程池；
* 调用shutdownNow()方法，将会把线程池的状态标记为`STOP`，此时线程池也不会接收任何新的任务，并且立即停止正在执行任务的线程，将队列里的任务返回给调用者；

我们可以看到，使用shutdownNow()强行关闭的速度更快，但风险也更大，因为任务很可能执行一半就被停止了；而使用shutdown()正常关闭虽然速度慢，但却更安全，因为会一直等到队列中的任务全部执行完成后才关闭。

### 使用建议

在阿里巴巴Java开发手册中，【强制】建议使用者不要通过Executors去创建线程池，而是通过ThreadPoolExecutor的方式，这样的处理方式让写的同学更加明确线程池的运行规则，从而规避资源耗尽的风险。