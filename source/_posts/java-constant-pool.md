---
title: Java中常量池详解
date: 2016-07-26 10:07:15
tags:
    - java
---

>阅读这篇文章之前先来理解几个基本的概念


* 什么是常量
* equals()方法和==的区别
* 引用和对象的区别

### 1. String常量池

#### 1.1 创建String对象的两种方式

* 通过new来创建String创对象，例如：String a = new String("a");
* 直接将字符串常量赋值给一个对象引用，例如：String b = "b";

    > 这两种不同的创建方法是有差别的，第一种方式是直接在Java heap内存空间创建一个新的对象，并且引用变量a指向这个对象，第二种方式是引用变量b指向常量池中的字符串。

<!-- more -->

#### 1.2 先来看一个Demo

```java
package com.sh.oc.test;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.junit.runners.BlockJUnit4ClassRunner;

/**
 * Created by Mr SJL on 2016/12/16.
 *
 * @Author Junlan Shuai
 */
@RunWith(BlockJUnit4ClassRunner.class)
public class Test1
{
    @Test
    public void test1()
    {
        String a = "helloWorld";
        String b = "helloWorld";
        String c = new String("helloWorld");
        String d = "hello";
        String e = new String("hello");
        String temp = "World";
        String f = "hello" + temp;
        String g = "hello" + "World";


        System.out.println("(1) a=b:" + (a == b));
        System.out.println("(2) b=c:" + (b == c));
        System.out.println("(3) a=d:" + (a == (d + "World")));
        System.out.println("(4) a=e:" + (a == (e + "World")));
        System.out.println("(5) c=d:" + (c == (d + "World")));
        System.out.println("(6) c=e:" + (c == (e + "World")));
        System.out.println("(7) a=g:" + (a == g));
        System.out.println("(8) a=f:" + (a == f));
    }
}
```

> 结果输出：

```java
(1) a=b:true
(2) b=c:false
(3) a=d:false
(4) a=e:false
(5) c=d:false
(6) c=e:false
(7) a=g:true
(8) a=f:false

Process finished with exit code 0
```

#### 1.3 总结：
* 对于第(1)个结果，直接将相同的字符串常量赋值给字符串引用变量（后称‘变量’）a，b，在编译的时候该字符串直接保存在常量池中，同时变量a，b同时指向这个字符串常量，所以a==b返回true；
* 第(2)个结果，对于变量c，是通过new一个对象（该对象保存在java heap中），然后变量c指向这个对象。故变量b和c指向的是不同的内存空间，b==c返回false；

* 第(3)个结果，对于加法运算d + "World"在执行的时候，首先是通过创建一个StringBuilder对象，然后调用该对象的append()方法，最后调用该对象的toString()方法。也就是相当于执行了new StringBuilder().append(d).append("Word").toString()。由StringBuilder源码可知，最后结果返回一个String对象(存放在java heap中)。故变量a和d指向不同的内存地址空间，虽然value是一样的，最终返回false。对于第(4)个结果，原理和(3)一样。

    ```java
    @Override
    public String toString() {
        // Create a copy, don't share the array
        return new String(value, 0, count);
    }
    ```
* 对于第(5)和(6)个结果，因为变量c指向的是在java heap中的一个String对象，并且加法运算(d + "World")，返回的也是一个String对象，但是它们指向的地址空间不同，故返回false。
* 对于第(7)和(8)的结果，原理和前面相同。

----
持续更新中。。。。。。
