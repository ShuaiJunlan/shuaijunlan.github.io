---
title: 深入研究Java阻塞队列实现
date: 2018-10-15 15:24:55
tags:
    - java
---

![](https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/BlockingQueue.png?raw=true)

<!-- more -->

阻塞队列是一个非常常见的数据结构，比如在线程池中用阻塞队列存放任务，下面我们将围绕阻塞队列的实现对其源码进行深度剖析，主要讲解`ArrayBlockingQueue、PriorityBlockingQueue、SynchronousQueue和LinkedBlockingQueue`四大阻塞队列的实现。



#### LinkedBlockingQueue

* 双锁队列算法的变体
* 

#### 优化

* [Why copy final member field into local final variable?](https://stackoverflow.com/questions/2785964/in-arrayblockingqueue-why-copy-final-member-field-into-local-final-variable)
* 双锁队列比单锁队列的好处？
* LinkedBlockingQueue#dequeue()的优化

```java
private E dequeue() {
    // assert takeLock.isHeldByCurrentThread();
    // assert head.item == null;
    Node<E> h = head;
    Node<E> first = h.next;
    h.next = h; // help GC
    head = first;
    E x = first.item;
    first.item = null;
    return x;
}
```

#### 基本接口

* offer()方法和put()方法的区别？



* 线程池如何关闭空闲线程的？