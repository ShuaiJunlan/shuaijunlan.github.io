---
title: log4j初级配置教程
date: 2017-04-26 10:07:15
tags:
    - log4j
---

* 先来看个采用log4j输出日志的例子
    > 添加依赖包

    ```XML
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-log4j12</artifactId>
        <version>1.7.7</version>
    </dependency>
    ```
    <!-- more -->

    > 添加配置文件log4j.properties放在/resources目录下面

    ```properties
    ### 设置###
    log4j.rootLogger = DEBUG,error,debug,stdout
    #log4j.rootLogger = INFO,stdout

    ### 输出信息到控制台 ###
    log4j.appender.stdout = org.apache.log4j.ConsoleAppender
    #log4j.appender.stdout.Threshold = ERROR
    log4j.appender.stdout.Target = System.out
    log4j.appender.stdout.layout = org.apache.log4j.PatternLayout
    log4j.appender.stdout.layout.ConversionPattern = [%-5p] %d{yyyy-MM-dd HH:mm:ss,SSS} method:%l%n%m%n

    ### 输出DEBUG 级别以上的日志到=E://logs/debug.log ###
    log4j.appender.debug = org.apache.log4j.DailyRollingFileAppender
    log4j.appender.debug.File = E://logs/debug.log
    log4j.appender.debug.Append = true
    ##Threshold是个全局的过滤器，它将把低于所设置的level的信息过滤不显示出来。
    log4j.appender.debug.Threshold = DEBUG
    log4j.appender.debug.layout = org.apache.log4j.PatternLayout
    log4j.appender.debug.layout.ConversionPattern = %-d{yyyy-MM-dd HH:mm:ss}  [ %t:%r ] - [ %p ]  %m%n

    ### 输出ERROR 级别以上的日志到=E://logs/error.log ###
    log4j.appender.error = org.apache.log4j.DailyRollingFileAppender
    log4j.appender.error.File =E://logs/error.log
    log4j.appender.error.Append = true
    log4j.appender.error.Threshold = ERROR
    log4j.appender.error.layout = org.apache.log4j.PatternLayout
    log4j.appender.error.layout.ConversionPattern = %-d{yyyy-MM-dd HH:mm:ss}  [ %t:%r ] - [ %p ]  %m%n
    ```

    > java代码

    ```java
    package com.sh.test;

    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;

    /**
     * Created by Mr SJL on 2016/11/26.
     *
     * @Author Junlan Shuai
     */
    public class App
    {
        public static void main(String[] args)
        {
            // 记录debug级别的信息
            log.debug("This is debug message.");
            // 记录info级别的信息
            log.info("This is info message.");
            // 记录error级别的信息
            log.error("This is error message.");

        }
    }
    ```
    > 控制台输出结果
    ![image]( http://learningnotes-1251679769.costj.myqcloud.com/javabasis/1.png  "结果")

* log4j主要组件

    > Log4j有三个主要的组件：Loggers(记录器)，Appenders  (输出源)和Layouts(布局)。这里可简单理解为日志类别，日志要输出的地方和日志以何种形式输出。综合使用这三个组件可以轻松地记录信息的类型和级别，并可以在运行时控制日志输出的样式和位置。

    * Loggers

        > Loggers组件在此系统中被分为五个级别：DEBUG、INFO、WARN、ERROR和FATAL。这五个级别是有顺序的，DEBUG < INFO < WARN < ERROR < FATAL，分别用来指定这条日志信息的重要程度，明白这一点很重要，Log4j有一个规则：只输出级别不低于设定级别的日志信息，假设Loggers级别设定为INFO，则INFO、WARN、ERROR和FATAL级别的日志信息都会输出，而级别比INFO低的DEBUG则不会输出。

    * Appenders

        > 禁用和使用日志请求只是Log4j的基本功能，Log4j日志系统还提供许多强大的功能，比如允许把日志输出到不同的地方，如控制台（Console）、文件（Files）等，可以根据天数或者文件大小产生新的文件，可以以流的形式发送到其它地方等等。

        > 常使用的类如下：

        > org.apache.log4j.ConsoleAppender（控制台）</br>
        org.apache.log4j.FileAppender（文件）</br>
        org.apache.log4j.DailyRollingFileAppender（每天产生一个日志文件）</br>
        org.apache.log4j.RollingFileAppender（文件大小到达指定尺寸的时候产生一个新的文件）</br>
        org.apache.log4j.WriterAppender（将日志信息以流格式发送到任意指定的地方） </br>

    * Layouts

        > 有时用户希望根据自己的喜好格式化自己的日志输出，Log4j可以在Appenders的后面附加Layouts来完成这个功能。 Layouts提供四种日志输出样式，如根据HTML样式、自由指定样式、包含日志级别与信息的样式和包含日志时间、线程、类别等信息的样式。

        > 常使用的类如下：

        > org.apache.log4j.HTMLLayout（以HTML表格形式布局）</br>
        org.apache.log4j.PatternLayout（可以灵活地指定布局模式）</br>
        org.apache.log4j.SimpleLayout（包含日志信息的级别和信息字符串）</br>
        org.apache.log4j.TTCCLayout（包含日志产生的时间、线程、类别等信息）</br>

* log4j.properties配置文件详解

    > 在实际应用中，要使Log4j在系统中运行须事先设定配置文件。配置文件事实上也就是对Logger、Appender及Layout进行相应设定。Log4j支持两种配置文件格式，一种是XML格式的文件，一种是properties属性文件。下面以properties属性文件为例介绍 log4j.properties的配置。

    * 配置根Logger

        > log4j.rootLogger = [ level ] , appenderName1, appenderName2, …</br>
        log4j.additivity.org.apache=false：表示Logger不会在父Logger的appender里输出，默认为true。</br>
        level ：设定日志记录的最低级别，可设的值有OFF、FATAL、ERROR、WARN、INFO、DEBUG、ALL或者自定义的级别，Log4j建议只使用中间四个级别。通过在这里设定级别，您可以控制应用程序中相应级别的日志信息的开关，比如在这里设定了INFO级别，则应用程序中所有DEBUG级别的日志信息将不会被打印出来。</br>
        appenderName：就是指定日志信息要输出到哪里。可以同时指定多个输出目的地，用逗号隔开。</br>
        例如：log4j.rootLogger＝INFO,A1,B2,C3 </br>

    * 配置控制台输出

        ```properties
        ### 输出信息到控制台 ###
        log4j.appender.stdout = org.apache.log4j.ConsoleAppender
        ### 输出ERROR级别以上的日志到控制台 ###
        log4j.appender.stdout.Threshold = ERROR
        log4j.appender.stdout.Target = System.out
        log4j.appender.stdout.layout = org.apache.log4j.PatternLayout
        log4j.appender.stdout.layout.ConversionPattern = [%-5p] %d{yyyy-MM-dd HH:mm:ss,SSS} method:%l%n%m%n
        ```

    * 配置日志文件输出

        ```properties
        ### 输出DEBUG 级别以上的日志到=E://logs/debug.log ###
        log4j.appender.debug = org.apache.log4j.DailyRollingFileAppender
        log4j.appender.debug.File = E://logs/debug.log
        log4j.appender.debug.Append = true
        ##Threshold是个全局的过滤器，它将把低于所设置的level的信息过滤不显示出来。
        log4j.appender.debug.Threshold = DEBUG
        log4j.appender.debug.layout = org.apache.log4j.PatternLayout
        log4j.appender.debug.layout.ConversionPattern = %-d{yyyy-MM-dd HH:mm:ss}  [ %t:%r ] - [ %p ]  %m%n
        ```
* REFERENCES

    1. <a href="http://www.open-open.com/lib/view/open1393488356958.html" target="\_blank">Log4j.properties配置详解</a>
    2. <a href="http://www.codeceo.com/article/log4j-usage.html" target="\_blank">最详细的Log4j使用教程</a>
