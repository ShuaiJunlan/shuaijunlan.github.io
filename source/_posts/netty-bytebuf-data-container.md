---
title: Netty数据容器---ByteBuf
date: 2018-07-24 15:38:09
tags:
    - Netty
---

### ByteBuf工作原理

ByteBuf维护了两个不同的索引：一个用于读取，一个用于写入。当从ByteBuf读取数据时，它的readerINdex将会递增已被读取的字节数。同样的，当你写入ByteBuf的时候，它的writerIndex也会被递增。

![](https://shuaijunlan.github.io/images/Screenshot from 2018-07-24 16-02-04.png)

上图中表示的是一个读索引和写索引都设置为0的16字节ByteBuf，若果试图访问超出writerIndex范围的数据将会触发一个IndexOutOfBoundsException异常。

<!-- more -->

### ByteBuf的使用模式

#### 堆缓冲区

最常用的ByteBuf模式是将数据存储在JVM的堆空间中，这种模式被称为`支撑数组（backing array）`，它能在没有使用池化的情况下提供快速的分配和释放。

#### 直接缓冲区

直接缓冲区是通过本地方法调用来分配堆外内存，这样可以避免在每次调用本地I/O操作之前（或者之后）将缓冲区的内容复制到一个中间缓冲区（或者从中间缓冲区把内容复制到缓冲区）。

直接缓冲区的主要缺点是：相对于基于堆的缓冲区，它们的分配和释放都较为昂贵。因为数据不在堆上，所以在操作数据之前不得不进行一次数据复制。

#### 复合缓冲区

复合缓冲区主要是为多个ByteBuf提供一个聚合视图，这是一个JDK的ByteBuffer实现完全缺失的特性。

### 操作ByteBuf字节

