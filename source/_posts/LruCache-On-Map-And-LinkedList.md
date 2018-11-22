---
title: 基于HashMap和双向链表实现LRUCache
date: 2018-04-12 13:23:18
tags:
    - LRU
---

> 使用HashMap和双向链表实现LRUCache，HashMap用来定位节点是否已经存在，时间复杂度为O(1)，双向链表用来用来实现LRU规则，移动节点的时间复杂度也是O(1)，代码如下：

<!-- more -->

```java
package cn.shuaijunlan.cache.lru;

import java.util.HashMap;

/**
 * @author Junlan Shuai[shuaijunlan@gmail.com].
 * @date Created on 10:48 2018/4/12.
 */
public class LruCacheOnLinkedAndMap<K, V> {
    /**
     * 定义双向链表节点
     * @param <K>
     * @param <V>
     */
    private class Entry<K, V>{
        Entry pre;
        Entry next;
        K key;
        V value;
        public Entry(K key,V value){
            this.key = key;
            this.value = value;
        }
    }

    /**
     * 缓存大小
     */
    private final int size;
    /**
     * 存储Entry Node
     */
    private HashMap<K,Entry<K,V>> map;
    /**
     * 链表尾节点
     */
    private Entry last;
    /**
     * 链表头节点
     */
    private Entry first;

    public LruCacheOnLinkedAndMap(int size){
        if (size <= 0){
            throw new IllegalArgumentException("The size of Cache must more than zero!");
        }
        this.size = size;
        map = new HashMap();
    }

    public void put(K key, V value){
        Entry entry = map.get(key);
        if (entry == null){
            if (map.size() >= this.size){
                removeLast();
            }
            entry = new Entry(key, value);
        }
        entry.value = value;
        moveToFirst(entry);
        map.put(key, entry);
    }

    /**
     * 删除尾节点
     */
    public void removeLast(){
        if (last != null){
            map.remove(last.key);
            last = last.pre;
            if (last == null){
                first = null;
            }else {
                last.next = null;
            }
        }
    }
    public Entry get(K key){
        Entry entry = map.get(key);
        if (entry == null){
            return null;
        }
        moveToFirst(entry);
        return entry;
    }

    /**
     * 根据LRU规则，将最近使用的节点移至链表头部
     * @param entry
     */
    public void moveToFirst(Entry entry){
        if (entry == first){
            return;
        }
        if (entry.pre != null){
            entry.pre.next = entry.next;
        }
        if (entry.next != null){
            entry.next.pre = entry.pre;
        }
        if (entry == last){
            last = last.pre;

        }
        if (first == null || last == null){
            first = last = entry;
            return;
        }
        entry.next = first;
        first.pre = entry;
        first = entry;
        entry.pre = null;
    }

    public void print() {
        Entry temp = first;
        while (temp != null){
            System.out.println("Key:" + temp.key + "   Value:" + temp.value);
            temp = temp.next;
        }
        System.out.println(map.size());
    }

    /**
     * 测试函数
     * @param args
     */
    public static void main(String[] args) {
        LruCacheOnLinkedAndMap<String, Integer> lruCacheOnLinkedAndMap = new LruCacheOnLinkedAndMap<>(6);
        lruCacheOnLinkedAndMap.put("s", 1);
        lruCacheOnLinkedAndMap.put("h", 2);
        lruCacheOnLinkedAndMap.put("u", 3);
        lruCacheOnLinkedAndMap.put("a", 8);
        lruCacheOnLinkedAndMap.put("i", 4);
        lruCacheOnLinkedAndMap.put("j", 7);
        lruCacheOnLinkedAndMap.put("u", 1);
        lruCacheOnLinkedAndMap.put("s", 10);
        lruCacheOnLinkedAndMap.print();
    }

}
```

