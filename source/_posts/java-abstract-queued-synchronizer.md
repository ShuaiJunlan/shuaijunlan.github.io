---
title: 队列同步器(AbstractQueuedSynchronizer)源码分析
date: 2018-11-22 19:11:40
tags:
    - java
    - AQS
---

> `AbstractQueuedSynchronizer(队列同步器)`在Java并发工具中经常被用到，比如说我们常用的`CountDownLatch`、`ReentrantLock`、`ReentrantReadWriteLock`和`Semaphore`等等并发工具类底层都是基于`队列同步器`的，只有掌握了`队列同步器`底层的工作原理才能更好的理解其他的并发工具的工作机制，这篇文章将会从源码的角度分析`队列同步器`的工作原理。

<!-- more -->

队列同步器设计是基于模板方法的，使用者只需要继承同步器并重写指定的方法，随后将同步器组合在自定义同步组件的实现中，并调用同步器提供的模板方法，而这些模板方法将会调用使用者重写的方法。

通过对队列同步器的API进行分析，主要通过三个方法来修改队列同步器的状态：

* getState()  //获取当前的状态
* setState()  //设置当前的同步状态
* compareAndSetState()   //使用CAS机制设置当前的状态

需要重写的方法：

**独占操作**

| 方法名称                              | 描述                                                         |
| ------------------------------------- | ------------------------------------------------------------ |
| protected boolean tryAcquire(int arg) | 独占式获取同步状态，实现该方法需要查询当前状态并判断同步状态是否符合预期，然后再进行CAS设置同步状态 |
| protected boolean tryRelease(int arg) | 独占式释放同步状态，等待获取同步状态的线程将有机会获取同步状态 |

**共享操作**

| 方法名称                                    | 描述                                                         |
| ------------------------------------------- | ------------------------------------------------------------ |
| protected int tryAcquireShared(int arg)     | 共享式获取同步状态，返回大于等于0，表示获取成功，反之获取失败 |
| protected boolean tryReleaseShared(int arg) | 共享式释放同步状态                                           |
| protected boolean isHeldExclusively()       | 当前同步器是否独占模式下被线程占用，一般该方法表示是否被当前线程所占用 |

#### 同步队列

队列同步器一来内部的同步队列（一个FIFO双向队列）来完成同步状态的管理，当前显示获取同步状态失败时，队列同步器将会以当前线程以及等待状态等信息来构建一个Node节点，并将其加入同步队列的尾部，同时会阻塞当前线程，当同步状态释放时，会把同步队列的首节点中的线程唤醒，使其再次尝试获取同步状态。

我们来看一下Node节点有哪些属性：

| 属性类型和名称  | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| int waitStatus  | 等待状态，包含如下状态： <br> 1.CANCELLED，值为1，由于在同步队列中等待的线程等待超时或者被中断，需要从同步队列中取消等待。节点进入该状态将不会发生变化<br>2.SIGNAL，值为-1，后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或者被取消，将会通知后继节点，使后继节点的线程得以运行<br>3.CONDITION，值为-2，表示当前节点正在条件队列（AQS下的ConditionObject里也维护了个队列）中，<br>在从conditionObject队列转移到同步队列前，它不会在同步队列（AQS下的队列）中被使用，当成功转移后，该节点的状态值将由CONDITION设置为0<br>4.PROPAGATE，值为-3，共享模式下的释放操作应该被传播到其他节点。该状态值在doReleaseShared方法中被设置的<br>5.INITIAL，值为0，初始状态 |
| Node prev       | 前驱节点，当节点加入同步队列时被设置                         |
| Node next       | 后继节点                                                     |
| Node nextWaiter | ConditionObject链表的后继节点或者代表共享模式的节点SHARED。<br>Condition条件队列：因为Condition队列只能在独占模式下被能被访问， 我们只需要简单的使用链表队列来链接正在等待条件的节点。<br>再然后它们会被转移到同步队列（AQS队列）再次重新获取。由于条件队列只能在独占模式下使用，所以我们要表示共享模式的节点的话只要使用特殊值SHARED来标明即可。 |
| Thread thread   | 节点所指向的线程                                             |



#### 独占式同步状态获取与释放

![](https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/Screenshot from 2018-11-23 20-17-54.png?raw=true)

#### 共享式同步状态获取与释放

#### 独占式超时获取同步状态

![](https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/Screenshot from 2018-11-23 20-22-05.png?raw=true)

#### 自定义同步组件



#### Reference

* 《Java并发编程的艺术》