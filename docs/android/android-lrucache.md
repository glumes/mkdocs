---
title: "Android LruCache实现分析"
date: 2017-12-22T15:35:23+08:00
categories: ["Android"]
tags: ["Android"]
toc: true
original: true
addwechat: true
 
slug: "android-lrucache"
---


LruCache 是安卓开发中常用到的缓存技术，LRU 的全名是 Least Recently Used，表示最近最少使用算法，也就是说当内存快到达阈值时，若某个对象最近很少使用的，那么它就会被回收掉以释放内存。

<!--more-->

## 使用
LruCache 的使用方法大致如下 ：
``` java
// LruCache 的声明初始化
        int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);
        // 指定缓存的内存大小，一般为当前进程可用内存的 1/8 即可
        int cacheSize = maxMemory / 8 ;
        LruCache<String,Bitmap> cache = new LruCache<String,Bitmap>(cacheSize){

            // 如何计算缓存的资源的内存大小
            @Override
            protected int sizeOf(String key, Bitmap value) {
                return value.getRowBytes() * value.getHeight() / 1024 ;
            }
            // 当缓存对象被回收时执行的回调方法，以便释放一些资源之类的，默认为空
            @Override
            protected void entryRemoved(boolean evicted, String key, Bitmap oldValue, Bitmap newValue) {
                super.entryRemoved(evicted, key, oldValue, newValue);
            }
        };
        // LruCache 添加缓存对象
        cache.put(key,bitmap) ;
        // LruCache 获取缓存对象
        cache.get(key);
```

LruCache 的声明和使用都比较简单，在它的内部就已经帮我们完成了缓存对象的移除和添加，而我们无需过多的考虑对象的释放问题。


## 原理分析

既然要缓存对象，必然要有个容器，LruCache 使用的是 `LinkedHashMap`，以强引用方式存储外界的缓存对象。

`LinkedHashMap`是一个双向的环形链表，链表中的每个节点都有指向前一个元素和后一个元素的引用。使用链表的好处就在于它的删除和添加操作更加的方便快捷。

而 LRU 的最近最少使用算法思想就体现在`LinkedHashMap`的元素顺序中了。

`LinkedHashMap`中的顺序当然是从头到尾的，而一个元素最近的使用情况当然也是从头到尾的一个渐变了。在 LruCache 中，默认是最近最少使用的元素位于链表的头部，而最近使用最多的元素则是位于链表的尾部。这样一来，从头到尾的一个渐变过程也就是一个元素使用最少到多的过程了。

当我们在进行每个`get`和`put`操作时，`LinkedHashMap`中的每个元素使用情况就要发生变化了，毕竟缓存容量是有限的，一些不幸的元素可能就因此而被垃圾回收了，以此来体现 LRU 的最近最少使用算法。

下面就分别介绍其`get`和`put`操作。


### LruCache 的 get 操作

从 LruCache 中取出一个元素，显然，这个结果是可能为 null 的。

LruCache 使用`get`操作很简单，有就取出来，没有就返回个 null 。
``` java
		V mapValue;
        synchronized (this) {
            mapValue = map.get(key);
            if (mapValue != null) {
                hitCount++;
                return mapValue;
            }
            missCount++;
        }
```
而重点就在于`LinkedHashMap`的`get`方法了。

``` java
	int hash = Collections.secondaryHash(key); // 计算 key 的 hash 值
        HashMapEntry<K, V>[] tab = table;
        for (HashMapEntry<K, V> e = tab[hash & (tab.length - 1)]; e != null; e = e.next) {
            K eKey = e.key;
            if (eKey == key || (e.hash == hash && key.equals(eKey))) {
                if (accessOrder)
                    makeTail((LinkedEntry<K, V>) e);
                return e.value;
            }
        }
```
`LinkedHashMap`的节点类是`LinkedEntry`，它继承自`HashMapEntry`，在其继承之上多了指向前后元素的引用。

所以，从上述代码可以看出，无非就是一个遍历，找到 `key`对应的元素就取出来，并且多了一个操作`makeTail`。

`makeTail`的操作又是干什么的呢？它可谓是实现最近最少算法的精髓了。

``` java
 private void makeTail(LinkedEntry<K, V> e) {
        // Unlink e 将节点 e 的前节点和后节点进行连接，而将它自己给释放出来
        e.prv.nxt = e.nxt;
        e.nxt.prv = e.prv;

        // Relink e as tail 将节点 e 连接到链表的尾部
        LinkedEntry<K, V> header = this.header;
        LinkedEntry<K, V> oldTail = header.prv;
        e.nxt = header;
        e.prv = oldTail;
        oldTail.nxt = header.prv = e;
        modCount++;
    }
```

`makeTail`操作，将一个`get`的节点，从原本它在链表中的位置移到了整个链表中的尾部，使它的前节点是上一任的尾节点，而后节点则是头结点。

而在上面提到过，`LinkedHashMap`链表中元素的顺序正是完美的体现了最少最近算法的思想。

`get`得到的元素必然是最近使用最多的了，每次 `get`之后都将它移动到链表的尾部，也正是最近最少算法的体现了。

### LruCache 的 put 操作

`put`操作则是向`LinkedHashMap`中放入一个元素，而缓存的容量又是有限的，这么一来，在有限的情况下，就会有元素被垃圾回收了。
 
``` java
	  V previous;
        synchronized (this) {
            putCount++;
            size += safeSizeOf(key, value);
            previous = map.put(key, value); // 若存在 key 相同的元素，则返回之前的，否则返回 null
            if (previous != null) {    // 存在 key 相同的元素
                size -= safeSizeOf(key, previous);
            }
        }
        // 移除最近最少使用的元素
		trimToSize(maxSize);
```

当进行`put`操作时，若放入一个 key 存在的元素，则返回已经存在的对应的值。

同时，在每次`put`操作之后，都要进行容量的清除工作。

``` java
public void trimToSize(int maxSize) {
        while (true) { // 死循环
            K key;
            V value;
            synchronized (this) {
                if (size <= maxSize) {
                    break;
                }
                Map.Entry<K, V> toEvict = map.eldest(); // 取出最老的元素
                if (toEvict == null) {
                    break;
                }
                key = toEvict.getKey();
                value = toEvict.getValue();
                map.remove(key);
                size -= safeSizeOf(key, value);
                evictionCount++;
            }
            entryRemoved(true, key, value, null);
        }
    }
```

`trimToSize`的操作是一个死循环的操作，当`size`小于`maxSize`时，直接返回，否则从`LinkedHashMap`中移除最近最少使用的元素，也就是头节点后的元素。

以上就大致实现了 LRU 的最少最近算法。


## 同步锁相关操作

在 LruCache 的`get`和`put`操作中，都有用到的`synchronized`锁操作。

这是保证线程安全，因为在`get`和`put`方法中进行的都是复合操作，在并发条件下使用缓存时很容易出现脏数据，加上`synchronized`标识，则保证了操作的原子性，使得成为线程安全的操作。


## 参考
1、https://github.com/LittleFriendsGroup/AndroidSdkSourceAnalysis/blob/master/article/LruCache%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md
