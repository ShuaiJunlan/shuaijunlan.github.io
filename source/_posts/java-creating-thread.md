---
title: Java创建线程的三种方式(Thread/Runnable/Callable)
date: 2017-05-26 10:07:15
tags:
    - java
    - MutliThread
---

### 1.继承Thread类
> 此方式只需要重写Thread类中的run()方法即可，示例如下：

```java
/**
 * @author Junlan Shuai[shuaijunlan@gmail.com].
 * @date Created on 19:41 2017/4/10.
 */
public class ExtendThread extends Thread
{
    String name;
    public ExtendThread(String name)
    {
        this.name = name;
    }
    @Override
    public void run()
    {
        System.out.println(name);
    }
}
```

<!-- more -->
### 2.实现Runnable接口
> 此方式只需要实现Runnable接口中的run()方法，示例如下：

```java
/**
 * @author Junlan Shuai[shuaijunlan@gmail.com].
 * @date Created on 19:40 2017/4/10.
 */
public class ImplRunnable implements Runnable
{
    public String name;
    public ImplRunnable(String name)
    {
        this.name = name;
    }
    public void run()
    {
        System.out.println(name);
    }
}
```
### 3.实现Callable接口
> 实现Callable接口中的calla()方法，示例如下：

```java
import java.util.concurrent.Callable;

/**
 * @author Junlan Shuai[shuaijunlan@gmail.com].
 * @date Created on 19:43 2017/4/10.
 */
public class ImplCallable implements Callable<String>
{
    public String name;
    public ImplCallable(String name)
    {
        this.name = name;
    }
    public String call() throws Exception
    {
        return name;
    }
}
```
### 4.Runnable和Callable的区别
```java
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
public interface Runnable {
    /**
     * When an object implementing interface <code>Runnable</code> is used
     * to create a thread, starting the thread causes the object's
     * <code>run</code> method to be called in that separately executing
     * thread.
     * <p>
     * The general contract of the method <code>run</code> is that it may
     * take any action whatsoever.
     *
     * @see     java.lang.Thread#run()
     */
    public abstract void run();
}

```
* Callable中申明的方法是call(),Runnable中申明的方法是run();
* Callable中的call()方法有返回值，而run()方法没有返回值;
* call()方法可抛出异常，而run()方法则没有;
### 5.Future详解

```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
* cancel()方法：当参数为true时，直接终止当前执行的任务，当参数为false是允许当前的任务执行完成；
* get()方法：等待任务执行完成，并可以获取任务执行完成的返回结果；

> ExecutorService中所有的submit()方法都将返回一个Future，从而将Callable或Runnable提交给Executor，并得到一个Future来获得任务的执行结果或取消任务；

### 附录：测试代码
```java
import org.junit.Test;
import org.junit.runner.RunWith;
import org.junit.runners.BlockJUnit4ClassRunner;
import java.util.concurrent.*;

/**
 * @author Junlan Shuai[shuaijunlan@gmail.com].
 * @date Created on 19:45 2017/4/10.
 */
@RunWith(BlockJUnit4ClassRunner.class)
public class Test1
{
    @Test
    public void test1()
    {
        Thread thread = new ExtendThread("Junlan Shuai");
        thread.start();
        try
        {
            Thread.sleep(1000);
        } catch (InterruptedException e)
        {
            e.printStackTrace();
        }
    }
    @Test
    public void test2()
    {
        Thread thread = new Thread(new ImplRunnable("Junlan Shuai"));
        thread.start();
        try
        {
            Thread.sleep(1000);
        } catch (InterruptedException e)
        {
            e.printStackTrace();
        }
    }
    @Test
    public void test3()
    {
        ImplCallable implCallable = new ImplCallable("Junlan Shuai");
        ExecutorService es = Executors.newFixedThreadPool(3);
        Future future = es.submit(implCallable);
        try
        {
            System.out.println(future.get());
            Thread.sleep(1000);
        } catch (InterruptedException e)
        {
            e.printStackTrace();
        } catch (ExecutionException e)
        {
            e.printStackTrace();
        }
    }
}
```
