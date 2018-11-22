---
title: analyse the source code of Timer
date: 2017-07-26 10:58:57
tags:
    - java
categories: 
    - java
---

## Timer Class Introduction

> 在JDK库中Timer类主要负责计划任务的功能，也就是在指定的时间开始执行某任务。

<!-- more -->

## A Simple Example

```java
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Timer;
import java.util.TimerTask;

/**
 * @author Junlan Shuai[shuaijunlan@gmail.com].
 * @date Created on 9:53 2017/7/26.
 */
public class TimerTest {
    //  define a timer
    private static Timer timer = new Timer();
    
    // define MyTask class
    static public class MyTask extends TimerTask{
        private String str;
        public MyTask(String str){
            this.str = str;
        }

        @Override
        public void run() {
            System.out.println(this.str + "running:" + new Date());
        }
    }
    //  test main
    public static void main(String[] args) throws ParseException {
        MyTask myTask1 = new MyTask("task1");
        MyTask myTask2 = new MyTask("task2");
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");


        String dateString1 = "2017-07-26 10:06:01";
        Date date1 = sdf.parse(dateString1);

        String dateString2 = "2017-07-26 10:05:01";
        Date date2 = sdf.parse(dateString2);

        timer.schedule(myTask1, date1);
        timer.schedule(myTask2, date2);
    }
}
```
> 在这个例子中，定义了两个TimerTask，并且设定执行时间，调用Timer的schedule()方法，传入任务和时间，执行正确。

## Timer源码分析
> 首先我们以timer.schedule()方法为入口，进一步剖析整个执行流程。看看Timer的schedule()的源码：</br>

```java
public void schedule(TimerTask task, Date time) {
    sched(task, time.getTime(), 0);
}
```

> 再进一步看sched()方法的源码</br>

```java
private void sched(TimerTask task, long time, long period) {
    if (time < 0)
        throw new IllegalArgumentException("Illegal execution time.");

    // Constrain value of period sufficiently to prevent numeric
    // overflow while still being effectively infinitely large.
    if (Math.abs(period) > (Long.MAX_VALUE >> 1))
        period >>= 1;

    synchronized(queue) {
        if (!thread.newTasksMayBeScheduled)
            throw new IllegalStateException("Timer already cancelled.");

        synchronized(task.lock) {
            if (task.state != TimerTask.VIRGIN)
                throw new IllegalStateException(
                    "Task already scheduled or cancelled");
            task.nextExecutionTime = time;
            task.period = period;
            task.state = TimerTask.SCHEDULED;
        }

        queue.add(task);
        if (queue.getMin() == task)
            queue.notify();
    }
}
```

* 在这个函数里面涉及到了两个重要的变量（queue和task）；
* 首先是获取queue同步锁，设置task的基本属性，包括nextExecutionTime、perid、state；
* 将task添加到queue中，等待task被执行；

## TaskQueue源码分析
* 底层是定义了一个**private TimerTask[] queue = new TimerTask[128];**数组，用来存储TimerTask，默认值为128；
* TaskQueue是直接定义在Timer.java的类，是一个优先级队列，是根据nextExecutionTime排序的；最小的nextExecutionTime
* 如果queue不是空的，最小的TimeTask.nextExecutionTime就是queue[1]；
* 主要理解两个函数：fixUp()和fixDown();

```java
private void fixUp(int k) {
    while (k > 1) {
        int j = k >> 1;
        if (queue[j].nextExecutionTime <= queue[k].nextExecutionTime)
            break;
        TimerTask tmp = queue[j];  queue[j] = queue[k]; queue[k] = tmp;
        k = j;
    }
}

private void fixDown(int k) {
    int j;
    while ((j = k << 1) <= size && j > 0) {
        if (j < size &&
            queue[j].nextExecutionTime > queue[j+1].nextExecutionTime)
            j++; // j indexes smallest kid
        if (queue[k].nextExecutionTime <= queue[j].nextExecutionTime)
            break;
        TimerTask tmp = queue[j];  queue[j] = queue[k]; queue[k] = tmp;
        k = j;
    }
}
```

* 每次调用add()方法，先将任务添加到quene的最后一个，然后调用fixUp()方法，调整整个queue，将拥有最小nextExecutionTime的TimerTask调整到queue[1]位置，如果位置不够，则需要扩容，按照原来容量的两倍扩容；

