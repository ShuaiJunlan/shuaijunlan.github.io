---
title: Autoboxing and Autounboxing
date: 2016-09-26 10:07:15
tags:
    - java
---
### 前言：

* 首先我们要知道Java中有哪些基本数据类型以及它们各自的封装类:package java.lang;

| 基本数据类型   | 封装类         |
|:-------------:|:-------------:|
| byte          | Byte          |
| boolean       | Boolean       |
| char          | Character     |
| short         | Short         |
| int           | Integer       |
| long          | Long          |
| float         | Float         |
| double        | Double        |

<!-- more -->

### 一、什么是Autoboxing

> java中Autoboxing是指将基本数据类型自动转换成封装类类型。比如说：

```java
public void test1()
{
    int a = 10;
    Integer b = a;
    System.out.println(b);

    Character d = 'c';
    System.out.println(d);
}
```

> * 函数参数为封装类类型时，调用时传递基本数据类型，会发生Autoboxing。
* 将基本数据类型变量赋值给封装类类型时，会发生Autoboxing。

### 二、什么是Autounboxing

> java中Autounboxing是指将封装类类型自动转换成基本数据类型。比如说：

```java
public void test2()
{
    Integer a = new Integer(10);
    int b = a;
    System.out.println(b);

    Character c = 'c';
    char d = c;
    System.out.println(d);
}
```
> * 函数参数为基本数据类型，调用时传递封装类类型，会发生Autounboxing。
* 将封装类类型变量赋值给基本数据类型，或者直接用封装类类型进行基本运算，会发生Autounboxing。


### 三、以int类型为例，讲解Autoboxing和Autounboxing实现原理
> 先来看一段代码反汇编的结果

* java代码
    ```java
    package com.sh.$16.$12.$22;
    /**
     * Created by Mr SJL on 2016/12/22.
     *
     * @Author Junlan Shuai
     */

    import java.util.ArrayList;
    import java.util.List;

    public class App1
    {
        public static void main(String[] args)
        {
            Integer integer = 10;
            int i = integer;
        }

    }
    ```
* 反汇编结果
    ```java
    public class com.sh.$16.$12.$22.App1 {
      public com.sh.$16.$12.$22.App1();
        Code:
           0: aload_0
           1: invokespecial #1                  // Method java/lang/Object."<init>":()V
           4: return

      public static void main(java.lang.String[]);
        Code:
           0: bipush        10
           2: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
           5: astore_1
           6: aload_1
           7: invokevirtual #3                  // Method java/lang/Integer.intValue:()I
          10: istore_2
          11: return
    }

    ```

> 从以上java代码可以看出，“Integer integer = 10;”此句发生了Autoboxing。</br>
从汇编结果可以看出，实际在编译的时候发生了，Integer a = Integer.valueOf(10);调用了Integer类的valueOf方法。

```java
/**
 * Returns an {@code Integer} instance representing the specified
 * {@code int} value.  If a new {@code Integer} instance is not
 * required, this method should generally be used in preference to
 * the constructor {@link #Integer(int)}, as this method is likely
 * to yield significantly better space and time performance by
 * caching frequently requested values.
 *
 * This method will always cache values in the range -128 to 127,
 * inclusive, and may cache other values outside of this range.
 *
 * @param  i an {@code int} value.
 * @return an {@code Integer} instance representing {@code i}.
 * @since  1.5
 */
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```
> 从以上java代码可以看出，“int i = integer;”此句发生了Autounboxing。</br>
从汇编结果可以看出，实际在编译的时候发生了，int i = integer.intValue();调用了Integer类的intValue方法。

```java
/**
 * Returns the value of this {@code Integer} as an
 * {@code int}.
 */
public int intValue() {
    return value;
}
```
### 四、总结

> 其他基本数据类型的Autoboxing and Autounboxing也满足此。

----
#### 思考：
```java
package com.sh.$16.$12.$24;

/**
 * Created by Mr SJL on 2016/12/24.
 *
 * @Author Junlan Shuai
 */
public class App2
{
    public static void main(String[] args)
    {
        int a = 10;
        int b = 10;
        Integer c = new Integer(10);
        Integer d = Integer.valueOf(10);
        Integer e = 2000;
        Integer f = 2000;

        System.out.println("a=b:" +(a==b));
        System.out.println("a=c:" +(a==c));
        System.out.println("a=d:" +(a==d));
        System.out.println("c=d:" +(c==d));
        System.out.println("e=f:" +(e==f));
    }
}

// 运行结果：
a=b:true
a=c:true
a=d:true
c=d:false
e=f:false
```
