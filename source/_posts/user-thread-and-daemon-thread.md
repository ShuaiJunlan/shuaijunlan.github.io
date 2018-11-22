---
title: User Thread and Daemon Thread in Java
date: 2018-07-04 09:44:27
tags:
    - java
---

> 在Java当中有两种线程，一种是`User Thread`，另一种是`Daemon Thread`。一般来说`User Thread`具有较高的优先级，它主要运行在前台，然而，`Daemon Thread`具有较低的优先级，主要运行在后台。

#### User Thread

User Thread通常由应用或者用户创建的，JVM Instance等待所有的User Thread执行完成Tasks，直到所有的User Thread执行完成JVM才会退出。

#### Daemon Thread

Daemon Thread通常由JVM创建， 这些线程在后台运行，运行一些后台任务（包括，垃圾回收、内务任务等）。JVM 不会等待所有的Daemon Thrad执行完Tasks，只要所有的User Thread执行完Tasks，JVM就会退出。

<!-- more -->

#### User Thread VS Daemon Thread In Java

| **User Threads**                                             | **Daemon Threads**                                           |
| ------------------------------------------------------------ | :----------------------------------------------------------- |
| JVM waits for user threads to finish their work. It will not exit until all user threads finish their work. | JVM will not wait for daemon threads to finish their work. It will exit as soon as all user threads finish their work. |
| User threads are foreground threads.                         | Daemon threads are background threads.                       |
| User threads are high priority threads.                      | Daemon threads are low priority threads.                     |
| User threads are created by the application.                 | Daemon threads, in most of time, are created by the JVM.     |
| User threads are mainly designed to do some specific task.   | Daemon threads are designed to support the user threads.     |
| JVM will not force the user threads to terminate. It will wait for user threads to terminate themselves. | JVM will force the daemon threads to terminate if all user threads have finished their work. |

![user threads vs daemon threads](https://javaconceptoftheday.com/wp-content/uploads/2015/06/UserVsDaemon.png?x70034)

#### Some Things-To-Remember about user threads and daemon threads In Java :

- You can convert user thread into daemon thread explicitly by calling setDaemon() method of the thread.

```java
public class ThreadsInJava {

    //Main Thread
    public static void main(String[] args) {
        UserThread userThread = new UserThread();   //Creating the UserThread
        userThread.setDaemon(true);        //Changing the thread as Daemon
        userThread.start();               //Starting the thread
    }
}

class UserThread extends Thread {
    @Override
    public void run() {
        for (int i = 0; i < 1000; i++) {
            System.out.println("This is an user thread....");
        }
    }
}
```

- You can’t set a daemon property after starting the thread. If you try to set the daemon property when the thread is active, It will throw a IllegalThreadStateException at run time.

```java
class UserThread extends Thread {
    @Override
    public void run() {
        for (int i = 0; i < 1000; i++) {
            System.out.println("This is an user thread....");
        }
    }
}

public class ThreadsInJava {
    public static void main(String[] args) {
        UserThread userThread = new UserThread();   //Creating the UserThread

        userThread.start();               //Starting the thread

        userThread.setDaemon(true);        //This statement will throw IllegalThreadStateException
    }
}
```

- You can check whether the thread is user thread or a daemon thread by using isDaemon() method of Thread class. This method returns “true” for a daemon thread and “false” for a user thread.

```java
class UserThread extends Thread {
    @Override
    public void run() {
        for (int i = 0; i < 1000; i++) {
            System.out.println("This is an user thread....");
        }
    }
}

public class ThreadsInJava {
    public static void main(String[] args) {
        UserThread userThread = new UserThread();   //Creating the UserThread

        System.out.println(userThread.isDaemon());    //Output : false

        userThread.setDaemon(true);         //changing the thread as Daemon

        userThread.start();                 //Starting the thread

        System.out.println(userThread.isDaemon());      //Output : true
    }
}
```

- Daemon property of a thread is inherited from it’s parent thread. i.e The thread created by user thread will be user thread and the thread created by daemon thread will be a daemon thread.

```java
class Thread1 extends Thread {
    @Override
    public void run() {
        Thread t = new Thread();      //Creating a child thread

        System.out.println(t.isDaemon());   //Checking the Daemon property of a child thread
    }
}

public class ThreadsInJava {
    public static void main(String[] args) {
        Thread1 t1 = new Thread1();   //Creating the Thread1

        t1.start();                 //Starting the thread

        Thread1 t2 = new Thread1();   //Creating the Thread1

        t2.setDaemon(true);         //changing the thread as Daemon

        t2.start();          //Starting the thread
    }
}
```

- **The main thread or primary thread created by JVM is an user thread.**

- **Demonstration of User thread and daemon thread  :**  In the below program, The task of daemon thread will not be completed. Program terminates as soon as user thread finishes it’s task. It will not wait for daemon thread to finish it’s task.

```java
/**
 * @author Junlan Shuai[shuaijunlan@gmail.com].
 * @date Created on 10:32 AM 2018/07/06.
 */
class UserThread extends Thread {
    @Override
    public void run() {
        System.out.println("This is a user thread.....");
    }
}

class DaemonThread extends Thread {
    public DaemonThread() {
        setDaemon(true);
    }

    @Override
    public void run() {
        for (int i = 0; i < 1000; i++) {
            System.out.println("This is daemon thread....." + i);
        }
    }
}

public class ThreadsInJava {
    public static void main(String[] args) {
        DaemonThread daemon = new DaemonThread();   //Creating the DaemonThread

        daemon.start();                 //Starting the daemon thread

        UserThread userThread = new UserThread();   //Creating the UserThread

        userThread.start();          //Starting the user thread
    }
}
```

#### Conclusion

1) User threads are created by the **application** (user) to perform some specific task. Where as daemon threads are mostly created by the **JVM** to perform some background tasks like garbage collection.

2) JVM will wait for user threads to finish their tasks. JVM will not exit until all user threads finish their tasks. On the other side, JVM will not wait for daemon threads to finish their tasks. It will exit as soon as all user threads finish their tasks.

3) User threads are **high priority** threads, They are designed mainly to execute some important task in an application. Where as daemon threads are **less priority** threads. They are designed to serve the user threads.

4) User threads are **foreground threads**. They always run in foreground and perform some specific task assigned to them. Where as daemon threads are **background threads**. They always run in background and act in a supporting role to user threads.

5) JVM will not force the user threads to terminate. It will wait for user threads to terminate themselves. On the other hand, JVM will force the daemon threads to terminate if all the user threads have finished their task.

6) User threads are chosen to do the core work of an application. The application is very much dependent on the user threads for it’s smooth execution. Where as daemon threads are chosen to do some supporting tasks. The application is less dependent on the daemon threads for it’s smooth running.

#### REFERENCES

* [Types Of Threads In Java](https://javaconceptoftheday.com/types-threads-java/)
* [Difference Between User Threads Vs Daemon Threads In Java](https://javaconceptoftheday.com/difference-between-used-threads-vs-daemon-threads-in-java/)

