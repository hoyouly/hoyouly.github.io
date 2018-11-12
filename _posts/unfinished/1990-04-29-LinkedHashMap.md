---
layout: post
title: 扫盲系列之---LinkedHashMap的实现原理
category: 待整理
tags: java LinkedHashMap
---
* content
{:toc}
LinkedHashMap 可以认为是HashMap和LinkedList,既有HashMap的操作数据结构，又有LinkedList维护插入元素的先后顺序。

## 两个属性
* private transient Entry<K,V> header; 双向链表的头结点
* private final boolean accessOrder;  false表示按照插入顺序，true表示按照访问顺序


## LinkedHashMapEntry
继承自HashMapEntry，属性包括以下几个
* final K key;
* V value;
* HashMapEntry<K, V> next;
* int hash;
* LinkedHashMapEntry<K, V> before
* LinkedHashMapEntry<K, V> after

key ,value,和hash 这三个属性就不多说了，主要是说说 next,befor,after 这三个属性
* next属性是用于维护hashmap指定table位置上连接Entry顺序的
* befor 和after是用于维护Entry 插入顺序的，LinkedHashMap能有顺序排列，全靠这两个属性帮忙  

关系如下图：  
![](https://github.com/hoyouly/BlogResource/raw/master/imges249993-20161215143120620-1544337380.png)

![](https://github.com/hoyouly/BlogResource/raw/master/imges249993-20161215143544401-1850524627.jpg)
图一是LinkedHashMap的整体结构图，图二是专门把循环双向链表抽取出来。使用的是循环双向链表，头部存放的是最久访问的节点或者最先插入的节点。末尾存放的是最近访问的节点或者最后插入的节点。迭代器的访问是从链表的头部开始到链表的尾部，在链表的尾部有一个空的header节点，该节点不存放key-value的内容，是LinkedHashMap的成员属性，循环双向链表的入口。

## init()
```java
@Override
void init() {
    header = new LinkedHashMapEntry<>(-1, null, null, null);
    header.before = header.after = header;
}
```
在init()中，初始化双向链表的头结点 header，并且把before和after都指向自己，这个有点像是我们左手和右手握手。握的都是我自己的手。   
注意：header的hash值是-1.key,value 和next都是null，也就是说这个不放到数组里的，因为数组里面key为null，的时候，是设置hash值为0的，只是用来指示开始元素和标志结束元素的。

## put()
LinkedHashMap 并没有重新HashMap的put方法，只是重写了put()方法内部调用的子方法，void recordAccess(HashMap m)  ，void addEntry(int hash, K key, V value, int bucketIndex) 和void createEntry(int hash, K key, V value, int bucketIndex)，提供了自己特有的双向链接列表的实现。


---
搬运地址：   
[Java集合之LinkedHashMap](https://www.cnblogs.com/xiaoxi/p/6170590.html)   
[Map 综述（二）：彻头彻尾理解 LinkedHashMap](https://blog.csdn.net/justloveyou_/article/details/71713781)
