---
layout: post
title: 扫盲系列之---LinkedHashMap的实现原理
category: 扫盲系列
tags: java LinkedHashMap
---
* content
{:toc}
LinkedHashMap 可以认为是HashMap和LinkedList,既有HashMap的操作数据结构，又有LinkedList维护插入元素的先后顺序。


使用的是循环双向链表，头部存放的是最久访问的节点或者最先插入的节点。末尾存放的是最近访问的节点或者最后插入的节点。迭代器的访问是从链表的头部开始到链表的尾部，在链表的尾部有一个空的header节点，该节点不存放key-value的内容，是LinkedHashMap的成员属性，循环双向链表的入口。



---
搬运地址：   
[Java集合之LinkedHashMap](https://www.cnblogs.com/xiaoxi/p/6170590.html)  
