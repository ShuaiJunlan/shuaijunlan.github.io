---
title: Java中Memory Mapped File原理分析
date: 2018-12-08 12:56:55
tags:
    - java
    - mmap
---

> 在传统的文件读写方式中，会有两次数据拷贝，一次是从硬盘拷贝到操作系统内核，另一次是从操作系统内核拷贝到用户态的应用程序。而在内存映射文件中，一般情况下，只有一次拷贝，且内存分配在操作系统内核，应用程序访问的就是操作系统的内核内存空间，这显然要比普通的读写效率更高。
>
> 内存映射文件的另一个重要特点是，它可以被多个不同的应用程序共享，多个程序可以映射同一个文件，映射到同一块内存区域，一个程序对内存的修改，可以让其他程序也看到，这使得它特别适合用于不同应用程序之间的通信。**比普通的基于loopback接口的Socket要快10倍。**那么在Java语言中是如何实现Memory Mapped File的呢？

<!-- more -->

在Java nio包中引入了`MappedByteBuffer`来实现Memory Mapped File，从继承结构上来看MappedByteBuffer继承自ByteBuffer，内部维护了一个逻辑地址address。

下面写个简单的示例来演示如何使用FileChannel和MappedByteBuffer：

```java
/**
 * @author Shuai Junlan[shuaijunlan@gmail.com].
 * @since Created in 3:03 PM 12/8/18.
 */
public class MmapTest {
    public static void main(String[] args) throws IOException {
        File file = new File("test.txt");
        assert file.exists() || file.createNewFile();
        FileChannel channel = new RandomAccessFile(file, "rw").getChannel();
        MappedByteBuffer mappedByteBuffer = channel.map(FileChannel.MapMode.READ_WRITE, 0, 1000);
        for (int i = 0; i < 1000; i++){
            mappedByteBuffer.put((byte)i);
        }
        mappedByteBuffer.position(0);
        for (int i = 0; i < 1000; i++){
            System.out.println(mappedByteBuffer.get());
        }
    }
}
```

在上面的代码中可以看到，FileChannel通过调用map方法把文件映射到了虚拟内存，在Java中规定的最大映射大小为**Integer.MAX_VALUE**，如果文件太大可以进行分段映射，我们来分析一下map方法中各个参数的含义：

**mode**:内存映射文件访问的方式，包括以下三种：

* 1.`MapMode.READ_ONLY`：只读，试图修改得到的缓冲区将导致抛出异常。

* 2.`MapMode.READ_WRITE`：读/写，对得到的缓冲区的更改最终将写入文件；但该更改对映射到同一文件的其他程序不一定是可见的。

* 3.`MapMode.PRIVATE`：私用，可读可写,但是修改的内容不会写入文件，只是buffer自身的改变，这种能力称之为**copy on write**。

**position**:被映射文件的其实位置；

**size**:映射区域的大小，单位为byte，最大映射大小为**Integer.MAX_VALUE**；

#### 进一步分析map过程的内部实现原理：

##### 第一步，通过RandomAccessFile获取FileChannel：

```java
public final FileChannel getChannel() {
    synchronized (this) {
        if (channel == null) {
            channel = FileChannelImpl.open(fd, path, true, rw, this);
        }
        return channel;
    }
}
```

该方法中使用了同步关键字，保证了多线程情况下只能初始化一个FileChannel实例。

##### 第二步，使用FileChannel的map方法，把文件映射到用户虚拟内存空间，并返回逻辑地址

```java
    public MappedByteBuffer map(MapMode mode, long position, long size)
        throws IOException
    {
		//省略参数检查
		
        long addr = -1;
        int ti = -1;
        try {
            begin();
            ti = threads.add();
            if (!isOpen())
                return null;

            long mapSize;
            int pagePosition;
            //加锁，保证线程安全
            synchronized (positionLock) {
                long filesize;
                do {
                    filesize = nd.size(fd);
                } while ((filesize == IOStatus.INTERRUPTED) && isOpen());
                if (!isOpen())
                    return null;
				//如果映射范围超出文件的大小且不可写，则抛出异常
                if (filesize < position + size) { // Extend file size
                    if (!writable) {
                        throw new IOException("Channel not open for writing " +
                            "- cannot extend file to required size");
                    }
                    int rv;
                    //填充文件
                    do {
                        rv = nd.allocate(fd, position + size);
                    } while ((rv == IOStatus.INTERRUPTED) && isOpen());
                    if (!isOpen())
                        return null;
                }

                pagePosition = (int)(position % allocationGranularity);
                long mapPosition = position - pagePosition;
                mapSize = size + pagePosition;
                try {
                    // If map0 did not throw an exception, the address is valid
                    addr = map0(imode, mapPosition, mapSize);
                } catch (OutOfMemoryError x) {
                    // An OutOfMemoryError may indicate that we've exhausted
                    // memory so force gc and re-attempt map
                    System.gc();
                    try {
                        Thread.sleep(100);
                    } catch (InterruptedException y) {
                        Thread.currentThread().interrupt();
                    }
                    try {
                        addr = map0(imode, mapPosition, mapSize);
                    } catch (OutOfMemoryError y) {
                        // After a second OOME, fail
                        throw new IOException("Map failed", y);
                    }
                }
            } // synchronized

            // On Windows, and potentially other platforms, we need an open
            // file descriptor for some mapping operations.
            FileDescriptor mfd;
            try {
                mfd = nd.duplicateForMapping(fd);
            } catch (IOException ioe) {
                unmap0(addr, mapSize);
                throw ioe;
            }

            assert (IOStatus.checkAll(addr));
            assert (addr % allocationGranularity == 0);
            int isize = (int)size;
            Unmapper um = new Unmapper(addr, mapSize, isize, mfd);
            if ((!writable) || (imode == MAP_RO)) {
                return Util.newMappedByteBufferR(isize,
                                                 addr + pagePosition,
                                                 mfd,
                                                 um);
            } else {
                return Util.newMappedByteBuffer(isize,
                                                addr + pagePosition,
                                                mfd,
                                                um);
            }
        } finally {
            threads.remove(ti);
            end(IOStatus.checkAll(addr));
        }
    }
```

map方法最终是通过调用native函数map0()完成文件映射：

1.如果第一次文件映射导致OOM，则手动处罚垃圾回收，休眠100ms后再尝试映射，如果失败则抛出异常；

2.通过`newMappedByteBuffer`（ReadWrite）或者`newMappedByteBufferR`（Read Only）方法初始化MappedByteBuffer实例，最终返回DirectByteBuffer的实例，该类是MappedByteBuffer的子类；

```java
static MappedByteBuffer newMappedByteBuffer(int size, long addr,
                                            FileDescriptor fd,
                                            Runnable unmapper)
{
    MappedByteBuffer dbb;
    if (directByteBufferConstructor == null)
        initDBBConstructor();
    try {
        dbb = (MappedByteBuffer)directByteBufferConstructor.newInstance(
            new Object[] { new Integer(size),
                          new Long(addr),
                          fd,
                          unmapper });
    } catch (InstantiationException |
             IllegalAccessException |
             InvocationTargetException e) {
        throw new InternalError(e);
    }
    return dbb;
}

private static void initDBBRConstructor() {
    AccessController.doPrivileged(new PrivilegedAction<Void>() {
        public Void run() {
            try {
                Class<?> cl = Class.forName("java.nio.DirectByteBufferR");
                Constructor<?> ctor = cl.getDeclaredConstructor(
                    new Class<?>[] { int.class,
                                    long.class,
                                    FileDescriptor.class,
                                    Runnable.class });
                ctor.setAccessible(true);
                directByteBufferRConstructor = ctor;
            } catch (ClassNotFoundException |
                     NoSuchMethodException |
                     IllegalArgumentException |
                     ClassCastException x) {
                throw new InternalError(x);
            }
            return null;
        }});
}
```

由于FileChannelImpl和DirectByteBuffer不在同一个包中，并切DirectByteBuffer类是默认的访问权限，因此无法直接在FileChannelImpl的map函数中直接实例化DirectByteBuffer，通过Util.java类的newMappedByteBuffer()方法去实例化，在上面的代码中，实例化的核心逻辑就是通过AccessController获取DirectByteBuffer类的构造函数进行实例化。

关于本地方法map0()，在JDK源码中找到了他的具体实现，如下：

```c
Java_sun_nio_ch_FileChannelImpl_map0(JNIEnv *env, jobject this,
                                     jint prot, jlong off, jlong len)
{
    void *mapAddress = 0;
    jobject fdo = (*env)->GetObjectField(env, this, chan_fd);
    jint fd = fdval(env, fdo);
    int protections = 0;
    int flags = 0;

    if (prot == sun_nio_ch_FileChannelImpl_MAP_RO) {
        protections = PROT_READ;
        flags = MAP_SHARED;
    } else if (prot == sun_nio_ch_FileChannelImpl_MAP_RW) {
        protections = PROT_WRITE | PROT_READ;
        flags = MAP_SHARED;
    } else if (prot == sun_nio_ch_FileChannelImpl_MAP_PV) {
        protections =  PROT_WRITE | PROT_READ;
        flags = MAP_PRIVATE;
    }

    mapAddress = mmap64(
        0,                    /* Let OS decide location */
        len,                  /* Number of bytes to map */
        protections,          /* File permissions */
        flags,                /* Changes are shared */
        fd,                   /* File descriptor of mapped file */
        off);                 /* Offset into file */

    if (mapAddress == MAP_FAILED) {
        if (errno == ENOMEM) {
            JNU_ThrowOutOfMemoryError(env, "Map failed");
            return IOS_THROWN;
        }
        return handle(env, -1, "Map failed");
    }

    return ((jlong) (unsigned long) mapAddress);
}
```

#### get和put方法

调用get和put方法对数据进行读写，最终其实是调用DirectByteBuffer类的get和put方法，

```java
public byte get() {
    return ((unsafe.getByte(ix(nextGetIndex()))));
}
public ByteBuffer put(byte x) {
    unsafe.putByte(ix(nextPutIndex()), ((x)));
    return this;
}
```

通过上面的代码可以看出，底层都是通过调用Unsafe类的getByte和putByte方法来操作数据的。

* 第一次访问address所指向的内存区域，导致缺页中断，中断响应函数会在交换区中查找相对应的页面，如果找不到（也就是该文件从来没有被读入内存的情况），则从硬盘上将文件指定页读取到物理内存中（非jvm堆内存）。

* 如果在拷贝数据时，发现物理内存不够用，则会通过虚拟内存机制（swap）将暂时不用的物理页面交换到硬盘的虚拟内存中。

#### 性能分析

从代码层面上看，从硬盘上将文件读入内存，都要经过文件系统进行数据拷贝，并且数据拷贝操作是由文件系统和硬件驱动实现的，理论上来说，拷贝数据的效率是一样的。
 但是通过内存映射的方法访问硬盘上的文件，效率要比read和write系统调用高，这是为什么？

- read()是系统调用，首先将文件从硬盘拷贝到内核空间的一个缓冲区，再将这些数据拷贝到用户空间，实际上进行了两次数据拷贝；
- map()也是系统调用，但没有进行数据拷贝，当缺页中断发生时，直接将文件从硬盘拷贝到用户空间，只进行了一次数据拷贝。

所以，采用内存映射的读写效率要比传统的read/write性能高。

#### 总结

* MappedByteBuffer使用虚拟内存，因此分配(map)的内存大小不受JVM的-Xmx参数限制，但是也是有大小限制的。

* 如果当文件超出Integer.MAX_VALUE字节限制时，可以通过position参数重新map文件后面的内容。

* MappedByteBuffer在处理大文件时的确性能很高，但也存在一些问题，如内存占用、文件关闭不确定，被其打开的文件只有在垃圾回收的才会被关闭，而且这个时间点是不确定的。
   javadoc中也提到：*A mapped byte buffer and the file mapping that it represents remain valid until the buffer itself is garbage-collected.*