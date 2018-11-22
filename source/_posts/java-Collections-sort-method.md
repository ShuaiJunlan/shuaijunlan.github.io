---
title: Collections.sort()源码分析（基于jdk1.8）
date: 2017-02-26 10:07:15
tags:
    - java
---

> Collections类中定义了一系列的静态方法，其中就包括sort方法(下面为该方法的源码),从这个方法的源码中可以看出，它调用的是list.sort()方法，在该方法中先将list转换成数组，然后调用Arrays.sort()方法。在Arrays.sort()方法中，有一个条件判断（LegacyMergeSort.userRequested），当此条件为true时，调用legacyMergeSort(a, c);若为false则调用TimSort.sort(a, 0, a.length, c, null, 0, 0);通过legacyMergeSort(a, c);源码就可以看出此方法实现的是归并排序，

<!-- more -->

* Collections.sort()方法源码
```java
@SuppressWarnings({"unchecked", "rawtypes"})
public static <T> void sort(List<T> list, Comparator<? super T> c) {
    list.sort(c);
}
```
* list.sort()方法源码
```java
@SuppressWarnings({"unchecked", "rawtypes"})
default void sort(Comparator<? super E> c) {
    Object[] a = this.toArray();
    Arrays.sort(a, (Comparator) c);
    ListIterator<E> i = this.listIterator();
    for (Object e : a) {
        i.next();
        i.set((E) e);
    }
}
```
* Arrays.sort()方法源码
```java
public static <T> void sort(T[] a, Comparator<? super T> c) {
    if (c == null) {
        sort(a);
    } else {
        if (LegacyMergeSort.userRequested)
            legacyMergeSort(a, c);
        else
            TimSort.sort(a, 0, a.length, c, null, 0, 0);
    }
}
```
> mergeSort()方法源码，legacyMergeSort()方法将会在未来的版本中被移除。mergeSort()方法中，当待排序数组长度小于7时，使用的是插入排序。

```java
/** To be removed in a future release. */
private static <T> void legacyMergeSort(T[] a, Comparator<? super T> c) {
    T[] aux = a.clone();
    if (c==null)
        mergeSort(aux, a, 0, a.length, 0);
    else
        mergeSort(aux, a, 0, a.length, 0, c);
}
/**
 * Tuning parameter: list size at or below which insertion sort will be
 * used in preference to mergesort.
 * To be removed in a future release.
 */
private static final int INSERTIONSORT_THRESHOLD = 7;

/**
 * Src is the source array that starts at index 0
 * Dest is the (possibly larger) array destination with a possible offset
 * low is the index in dest to start sorting
 * high is the end index in dest to end sorting
 * off is the offset to generate corresponding low, high in src
 * To be removed in a future release.
 */
@SuppressWarnings({"unchecked", "rawtypes"})
private static void mergeSort(Object[] src,
                              Object[] dest,
                              int low,
                              int high,
                              int off) {
    int length = high - low;

    // Insertion sort on smallest arrays
    if (length < INSERTIONSORT_THRESHOLD) {
        for (int i=low; i<high; i++)
            for (int j=i; j>low &&
                     ((Comparable) dest[j-1]).compareTo(dest[j])>0; j--)
                swap(dest, j, j-1);
        return;
    }

    // Recursively sort halves of dest into src
    int destLow  = low;
    int destHigh = high;
    low  += off;
    high += off;
    int mid = (low + high) >>> 1;
    mergeSort(dest, src, low, mid, -off);
    mergeSort(dest, src, mid, high, -off);

    // If list is already sorted, just copy from src to dest.  This is an
    // optimization that results in faster sorts for nearly ordered lists.
    if (((Comparable)src[mid-1]).compareTo(src[mid]) <= 0) {
        System.arraycopy(src, low, dest, destLow, length);
        return;
    }

    // Merge sorted halves (now in src) into dest
    for(int i = destLow, p = low, q = mid; i < destHigh; i++) {
        if (q >= high || p < mid && ((Comparable)src[p]).compareTo(src[q])<=0)
            dest[i] = src[p++];
        else
            dest[i] = src[q++];
    }
}

/**
 * Swaps x[a] with x[b].
 */
private static void swap(Object[] x, int a, int b) {
    Object t = x[a];
    x[a] = x[b];
    x[b] = t;
}
```
