HashMap是工作中使用频度非常高的一个K-V存储容器。在多线程环境下，使用HashMap是不安全的，可能产生各种非期望的结果。

关于HashMap线程安全问题，可参考笔者的另一篇文章：
[深入解读HashMap线程安全性问题](link:深入解读HashMap线程安全性问题)

针对HashMap在多线程环境下不安全这个问题，HashMap的作者认为这并不是bug，而是应该使用线程安全的HashMap。

目前有如下一些方式可以获得线程安全的HashMap：
* Collections.synchronizedMap
* HashTable
* ConcurrentHashMap

其中，前两种方式由于全局锁的问题，存在很严重的性能问题。所以，著名的并发编程大师Doug Lea在JDK1.5的java.util.concurrent包下面添加了一大堆并发工具。其中就包含ConcurrentHashMap这个线程安全的HashMap。

本文就来简单介绍一下ConcurrentHashMap的实现原理。

PS：基于JDK8

### 0 ConcurrentHashMap在JDK7中的回顾
ConcurrentHashMap在JDK7和JDK8中的实现方式上有较大的不同。首先我们先来大概回顾一下ConcurrentHashMap在JDK7中的原理是怎样的。

#### 0.1 分段锁技术
针对HashTable会锁整个hash表的问题，ConcurrentHashMap提出了分段锁的解决方案。

分段锁的思想就是：锁的时候不锁整个hash表，而是只锁一部分。

如何实现呢？这就用到了ConcurrentHashMap中最关键的Segment。

ConcurrentHashMap中维护着一个Segment数组，每个Segment可以看做是一个HashMap。

而Segment本身继承了ReentrantLock，它本身就是一个锁。

在Segment中通过HashEntry数组来维护其内部的hash表。

每个HashEntry就代表了map中的一个K-V，用HashEntry可以组成一个链表结构，通过next字段引用到其下一个元素。

上述内容在源码中的表示如下：

```java
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>
        implements ConcurrentMap<K, V>, Serializable {

    // ... 省略 ...
    /**
     * The segments, each of which is a specialized hash table.
     */
    final Segment<K,V>[] segments;

    // ... 省略 ...

    /**
     * Segment是ConcurrentHashMap的静态内部类
     * 
     * Segments are specialized versions of hash tables.  This
     * subclasses from ReentrantLock opportunistically, just to
     * simplify some locking and avoid separate construction.
     */
    static final class Segment<K,V> extends ReentrantLock implements Serializable {
    	// ... 省略 ...
    	/**
         * The per-segment table. Elements are accessed via
         * entryAt/setEntryAt providing volatile semantics.
         */
        transient volatile HashEntry<K,V>[] table;
        // ... 省略 ...
    }
    // ... 省略 ...

    /**
     * ConcurrentHashMap list entry. Note that this is never exported
     * out as a user-visible Map.Entry.
     */
    static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K,V> next;
        // ... 省略 ...
    }
}
```

所以，JDK7中，ConcurrentHashMap的整体结构可以描述为下图这样子。

![JDK7_ConcurrentHashMap结构](https://github.com/Lord-X/awesome-it-blog/blob/master/images/java/Java8%E4%B8%ADConcurrentHashMap%E6%98%AF%E5%A6%82%E4%BD%95%E4%BF%9D%E8%AF%81%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%E7%9A%84/JDK7_ConcurrentHashMap%E7%BB%93%E6%9E%84.png?raw=true)

由上图可见，只要我们的hash值足够分散，那么每次put的时候就会put到不同的segment中去。
而segment自己本身就是一个锁，put的时候，当前segment会将自己锁住，此时其他线程无法操作这个segment，
但不会影响到其他segment的操作。这个就是锁分段带来的好处。

#### 0.2 线程安全的put

#### 0.3 线程安全的get

### 1 初始化


### 2 Unsafe类的使用


### 3 内部数据结构


### 4 线程安全的hash表初始化


### 5 线程安全的put


### 6 线程安全的扩容


### 7 总结