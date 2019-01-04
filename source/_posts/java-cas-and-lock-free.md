---
title: CAS and Lock-Free in Java
date: 2018-12-27 20:06:41
tags:
    - java
---

在Java中的原子类中频繁的使用CAS策略来保证数据更新的安全性，它是一种Lock-Free机制，这篇文章将会讲解CAS原理，以及它在Java中的应用。

<!-- more -->

### Lock-Free

> *如果一个方法是 Lock-Free 的, 它保证线程无限次调用这个方法都能够在有限步内完成.*

相比于传统的基于 Mutex 锁机制, Lock-Free 有下面的优势:

- Lock-Free 的速度更快
- 线程之间不会相互影响, 没有死锁
- 不受异步信号影响, 可重入
- 不会发生线程优先级反转

在通常使用 Mutex 互斥锁的场景, 有的线程抢占了锁, 其他线程则会被阻塞, 当获得锁的进程挂掉之后, 整个程序就 block 住了. 但在 Lock-Free 的程序中, 单个线程挂掉, 也不会影响其他线程, 因为线程之间不会相互影响.

但是, Lock-Free 也有不少缺陷:

- 只能利用有限的原子操作, 例如 CAS (操作的位数有限), 编码实现复杂
- 竞争会加剧, 优先级不好控制
- 测试时需要考虑各种软硬件环境, 很难做到尽善尽美

再引入一个 Wait-Free 概念:

> *假如一个方法是 Wait-Free 的, 那么它保证了每一次调用都可以在有限的步骤内结束.*

一般来说: **阻塞 > Lock-Free > Wait-Free**

### CAS原语

CAS (compare and swap) 是 CPU 硬件同步原语(primitive), CAS(V, A, B) 操作可以用下面的代码来示意:

```cpp
template <class T>
bool CAS(T* addr, T expect_val, T val) {
    if (*addr == expect_val) {
        *addr = val;
        return true;
    }
    return false;
}
```

从 80486 开始, 所有的 Intel 处理器上, 通过一条汇编指令 CMPXCHG 即可实现 CAS 操作. CAS 的价值在于它是一个**原子操作**, 不会被 CPU中断或者其他方式打断, 因为在硬件层实现, 所以**开销极小**.

CAS 并不是一项新技术, 它的使用可以追溯到 70 年代, 早在 80 年代就有很多经典书籍中提到使用 CAS 来实现并行编程, 如 USC 大牛 [Kai HWang](http://gridsec.usc.edu/Hwang.html) 的 "Computer Architecture and Parallel Processing".

GCC 4.1+ 开始支持 CAS 的[原子操作](https://gcc.gnu.org/onlinedocs/gcc-4.1.1/gcc/Atomic-Builtins.html):

```C
bool __sync_bool_compare_and_swap (type *ptr, type oldval, type newval)
type __sync_val_compare_and_swap (type *ptr, type oldval, type newval)
```

通常将 CAS 用于同步的方式是从地址 V 读取值 A, 执行多步计算来获得新值 B, 然后使用 CAS 将 V 的值从 A 改为 B, 如果 V 处的值尚未同时更改, 则 CAS 操作成功.

### CAS中的ABA问题

ABA 问题描述:

- 切换到线程 T1, 获取内存 V 的值 A
- 切换到线程 T2, 获取内存 V 的值 A, 修改成 B, 然后再修改成 A
- 切换到线程 T1, 获取内存 V 的值还是 A, 继续执行

[coolshell 上有篇文章](http://coolshell.cn/articles/8239.html)给出了一个生动的例子(From 维基百科):

> *你拿着一个装满钱的手提箱在飞机场，此时过来了一个火辣性感的美女，然后她很暖昧地挑逗着你，并趁你不注意的时候，把用一个一模一样的手提箱和你那装满钱的箱子调了个包，然后就离开了，你看到你的手提箱还在那，于是就提着手提箱去赶飞机去了.*

更具有参考意义的是[Hazard Pointer Wiki](http://en.wikipedia.org/wiki/Hazard_pointer)上提到的一个 Lock-Free 堆栈的例子:

- 当前栈元素 [A, B, C], 栈顶 head 指向 A
- 线程 T1 执行 pop() 准备 CAS(&head, B, A)
- 线程 T2 抢占, pop A, pop B, 然后 push A
- 线程 T1 恢复, CAS(&head, B, A) 成功, 则此时 head 指向一个被 pop 的元素 B

CAS机制还存在其他问题：

* **循环时间长开销大:**

  > 自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。如果JVM能支持处理器提供的pause指令那么效率会有一定的提升，pause指令有两个作用，第一它可以延迟流水线执行指令（de-pipeline）,使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零。第二它可以避免在退出循环的时候因内存顺序冲突（memory order violation）而引起CPU流水线被清空（CPU pipeline flush），从而提高CPU的执行效率。

* **只能保证一个共享变量的原子操作**:

  > 当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁，或者有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。比如有两个共享变量i＝2,j=a，合并一下ij=2a，然后用CAS来操作ij。从Java1.5开始JDK提供了**AtomicReference类来保证引用对象之间的原子性，你可以把多个变量放在一个对象里来进行CAS操作。**

### CAS在Java中的应用

* 在Java中所有原子类都采用了CAS机制，并且在jdk1.5之后提供了AtomicStampedReference来解决上述提到的ABA问题；
* AQS，非阻塞数据结构和原子变量类（java.util.concurrent.atomic包中的类），这些concurrent包中的基础类都是使用CAS机制来实现的，而concurrent包中的高层类又是依赖于这些基础类来实现的。从整体来看，concurrent包的实现示意图如下：

![](https://github.com/shuaijunlan/shuaijunlan.github.io/blob/master/images/Screenshot from 2018-12-27 20-39-35.png?raw=true)