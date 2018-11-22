---
title: Java类与对象初始化过程
date: 2018-04-01 18:58:32
tags:
    - java
---

## 看看如下代码，输出结果是啥？

```java
/**
 * @author Junlan Shuai[shuaijunlan@gmail.com].
 * @date Created on 18:59 2018/4/1.
 */
public class Test {
    public static int k = 0;
    public static Test t1 = new Test("t1");
    public static Test t2 = new Test("t2");
    public static int i = print("i");
    public static int n = 99;
    public int j = print("j");
    static {
        print("静态块");
    }
    public Test(String string){
        System.out.println((++k) + ":" + string + " i=" + i + " n=" + n);
        ++i;
        ++n;
    }
    {
        print("构造块");
    }
    public static int print(String string){
        System.out.println((++k) + ":" + string + " i=" + i + " n=" + n);
        ++n;
        return ++i;
    }

    public static void main(String[] args) {
        Test test = new Test("init");
    }
}

```
<!-- more -->

**Output**

```
1:j i=0 n=0
2:构造块 i=1 n=1
3:t1 i=2 n=2
4:j i=3 n=3
5:构造块 i=4 n=4
6:t2 i=5 n=5
7:i i=6 n=6
8:静态块 i=7 n=99
9:j i=8 n=100
10:构造块 i=9 n=101
11:init i=10 n=102
```

**Reference**

* [Java类与对象初始化的过程](https://maimai.cn/article/detail?fid=327435262)