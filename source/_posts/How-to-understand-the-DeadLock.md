---
title: How-to-understand-the-DeadLock
date: 2018-03-25 13:23:26
tags:
    - java
    - MutliThread
---

## 如何理解如下代码会造成DeadLock

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * @author Junlan Shuai[shuaijunlan@gmail.com].
 * @date Created on 10:36 2018/4/14.
 */
public class DeadLock {
    static class Friend{
        private final String name;
        public Friend(String name){
            this.name = name;
        }

        public String getName(){
            return this.name;
        }

        public synchronized void bow(Friend friend){
            System.out.format("%s:%s" + " has bowed to me!%n", this.name, friend.getName());
            friend.bowBack(this);
        }
        public synchronized void bowBack(Friend friend){
            System.out.format("%s:%s" + " has bowed back to me!%n", this.name, friend.getName());
        }
    }

    public static void main(String[] args) throws InterruptedException {
        final Friend friendA = new Friend("Shuai");
        final Friend friendB = new Friend("Junlan");
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(2,
                2, 1000L, TimeUnit.MILLISECONDS, new ArrayBlockingQueue<>(2));
        /// Why not using this way to create ThreadPool?
        // ExecutorService fixedThreadPool = Executors.newFixedThreadPool(2);
        threadPoolExecutor.execute(() -> friendA.bow(friendB));
        threadPoolExecutor.execute(() -> friendB.bow(friendA));
        threadPoolExecutor.shutdown();

    }
}
```

**output**

```
Shuai:Junlan has bowed to me!
Junlan:Shuai has bowed to me!
```

**Conclusion**

* 类的实例对类中所有的synchronized方法都持有锁；（表述不够官方）