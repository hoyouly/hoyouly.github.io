---
layout: post
title: 街题系列之---HashMap实现原理。
category: 扫盲系列
tags: Java HashMap
---
* content
{:toc}
# HashMap
特点：
1. 基于Map接口实现
2. 运行null键/值
3. 非同步
4. 不保证有序，比如插入的顺序
5. 不保证顺序不随时间变化

## HashMap 的数据结构
Java中常见的两种结构是数组和模拟指针(引用)，几乎所有的数据结构都可以用这两种结构组合实现，HashMap也不例外，实际上HashMap是一个“链表散列”，结构如下
![](http://p5sfwb51p.bkt.clouddn.com/20170209185858523.jpeg)

从图中可以看出，HashMap底层还是数组，只是数组中每一项成为一个桶，即bucket，每一个桶只存一个元素，也就是一个Node对象,由于Node对象可以包含一个引用变量next用于指向另外一个Node,源码如下，因此可能出现，尽管桶里面只有一个Node对象，但是这个对象又指向另外一个Node对象，这样就形成了一条链，即Node链，其中Capacity就是该数组的长度，这个长度是可以改变的，

### Node
是HashMap的一个内部类，实现了Map.Entry接口，它也维护这一个key-value映射关系，除了key-value,还有一个next引用，该引用指向当前table位置的链表，hash值，用来确定每一个Node链表在table的位置。

```java
static class Node<K, V> implements Map.Entry<K, V> {
        final int hash;
        final K key;
        V value;
        Node<K, V> next;

        ...
}
```


## 两个重要参数
1. Capacity  容量 hash 表中桶的数量，默认从初始容量是16
2. Load foctor 负载因子 hash表在自动增加之前可以达到多满的一种程度。负载因子越大，表示散列表的装填程度越高。反之越小。   
负载因子存在的意思： 对于使用链表法的散列表来说，查找一个元素的平均时间是O(1+a),因此，负载因子越大，对空间的利用率越高，然而查找效率就越低，反之，负载因子越小，那么散列表数据过于稀疏，空间浪费严重，默认负载因子是0.75
当哈希表中条目数据超出了负载因子与容量的乘积的时候，则要对该hash表进行rehash操作，即重建内部数据，从而该哈希表具有两倍的桶数(buckets)




---
搬运地址：  
[Java容器（四）：HashMap（Java 7）的实现原理](https://blog.csdn.net/jeffleo/article/details/54946424)   
[Java HashMap工作原理及实现](https://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)   
[HashMap 里的“bucket”、“负载因子” 介绍
](https://blog.csdn.net/wenyiqingnianiii/article/details/52204136)
