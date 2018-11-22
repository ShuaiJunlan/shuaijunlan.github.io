---
layout: post
title: java中“...”的含义
date: 2016-06-26 10:07:15
tags:
    - java
---

1. 问题来源

    > 在阅读spring源码时发现问题：

    ```java
    /**
     * Create a new ClassPathXmlApplicationContext, loading the definitions
     * from the given XML files and automatically refreshing the context.
     * @param configLocations array of resource locations
     * @throws BeansException if context creation failed
     */
    public ClassPathXmlApplicationContext(String... configLocations) throws BeansException {
    	this(configLocations, true, null);
    }
    ```
<!-- more -->

2. 简介

    > 是jdk1.5新增加特性，Java语言对方法参数支持一种新写法，叫可变长度参数列表，其语法就是类型后跟...，表示此处接受的参数为0到多个Object类型的对象，或者是一个Object[]。

3. 测试
    > com.sh.test

    ```java
    package com.sh.test;
    /**
     * Created by Mr SJL on 2016/11/19.
     *
     * @author Junlan Shuai
     */
    public class Test1
    {
        public static void main(String[] args)
        {
            //  创建string对象数组，调用测试函数
            String[] colors = new String[]{"green", "blue", "red"};
            t1(colors);
            t2(colors);

            //  创建string实例，并调用测试函数
            String color = "black";
            t1(color);
    //        t2(color);    // 此处报错 java.lang.String[] can not be applied java.lang.String
        }

        /**
         * 测试函数1
         * @param colors    String...类型
         */
        public static void t1(String... colors)
        {
            for (String c : colors)
            {
                System.out.println(c);
            }
        }

        /**
         * 测试函数2
         * @param colors    String[]类型
         */
        public static void t2(String[] colors)
        {
            for (String c : colors)
            {
                System.out.println(c);
            }
        }
    }

    ```

4. 注意事项
    > Error:(38, 24) java: 无法在com.sh.test.Test1中同时声明t1(java.lang.String[])和t1(java.lang.String...)

    ```java
    /**
     * 测试函数1
     * @param colors    String...类型
     */
    public static void t1(String... colors)
    {
        for (String c : colors)
        {
            System.out.println(c);
        }
    }
    /**
     * 测试函数2
     * @param colors    String[]类型
     */
    public static void t1(String[] colors)
    {
        for (String c : colors)
        {
            System.out.println(c);
        }
    }
    ```
