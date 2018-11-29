---
title: Java中Unsafe类详解
date: 2018-11-29 11:05:23
tags:
    - java
---

1. VM "intrinsification." ie CAS (Compare-And-Swap) used in Lock-Free Hash Tables eg:sun.misc.Unsafe.compareAndSwapInt it can make real JNI calls into native code that contains special instructions for CAS

   read more about CAS here <http://en.wikipedia.org/wiki/Compare-and-swap>

2. The sun.misc.Unsafe functionality of the host VM can be used to allocate uninitialized objects and then interpret the constructor invocation as any other method call.

3. One can track the data from the native address.It is possible to retrieve an object’s memory address using the java.lang.Unsafe class, and operate on its fields directly via unsafe get/put methods!

4. Compile time optimizations for JVM. HIgh performance VM using "magic", requiring low-level operations. eg: <http://en.wikipedia.org/wiki/Jikes_RVM>

5. Allocating memory, sun.misc.Unsafe.allocateMemory eg:- DirectByteBuffer constructor internally calls it when ByteBuffer.allocateDirect is invoked

6. Tracing the call stack and replaying with values instantiated by sun.misc.Unsafe, useful for instrumentation

7. sun.misc.Unsafe.arrayBaseOffset and arrayIndexScale can be used to develop arraylets,a technique for efficiently breaking up large arrays into smaller objects to limit the real-time cost of scan, update or move operations on large objects

8. <http://robaustin.wikidot.com/how-to-write-to-direct-memory-locations-in-java>

### Reference

* https://stackoverflow.com/questions/5574241/why-does-sun-misc-unsafe-exist-and-how-can-it-be-used-in-the-real-world
* http://bytescrolls.blogspot.com/2011/04/interesting-uses-of-sunmiscunsafe.html