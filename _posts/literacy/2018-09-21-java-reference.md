---
layout: post
title: 扫盲系列 - Java 引用类型
category: 扫盲系列
tags:  Java
---

* content
{:toc}

在JDK1.2 之前,java中引用的定义：如果引用类型的数据类型中存储的数值代表的是一块内存的起始地址，就称这块内存代表一个引用，在JDK1.2 之后，java把引用类型分为四种级别，这四种级别由高到低分别是：强引用，软引用，弱引用和虚引用

## 强引用（StrongReference）
当内存空间不足，Java虚拟机宁愿抛出OutOfMemoryError错误，使程序异常终止，也不会靠随意回收具有强引用的对象来解决内存不足的问题。

因此<font color="#ff000" >强引用是造成Java内存泄露的主要原因之一。</font>
其实就是我们平时的
 `String[] arr = new String[]{"a", "b", "c"};`

## 软引用（SoftReference）
* 如果一个对象只具有软引用，则内存空间足够，垃圾回收器就不会回收它；
* 如果内存空间不足了，就会回收这些对象。
* 只要垃圾回收器没有回收它，该对象就可以被程序使用。
* <font color="#ff000" >软引用可以用来实现内存敏感的高速缓存</font>
* <span style="border-bottom:1px solid red;"> 软引用可以和一个引用队列（ReferenceQueue）联合使用</span>，如果软引用所引用的对象被垃圾回收器回收，Java虚拟机就会把这个软引用加入到与之关联的队列引用中。
```java
SoftReference<String[]> softBean = new SoftReference<String[]>(new String[]{"a", "b", "c"});
```

## 弱引用（WeakReference）
* <font color="#ff000" > 只具有弱引用的对象拥有短暂的生命周期。只能生存到下一次垃圾发生收集之前。</font>
* 在垃圾回收器线程扫描它所管辖的内存区域中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存，
* 不过由于垃圾回收器是一个优先级很低的线程，因此不一定会很快发现那些只具有弱引用的对象
* <span style="border-bottom:1px solid red;"> 弱引用可以和一个引用队列（ReferenceQueue）联合使用</span>，如果弱引用所引用的对象被垃圾回收，java虚拟机就会把这个弱引用加入到与之关联的引用队列中
```java
WeakReference<String[]> weakBean = new WeakReference<String[]>(new String[]{"a", "b", "c"});
```

## 虚引用（PhantomReference）
顾名思义：形同虚设，<font color="#ff000" >虚引用并不会决定对象的什么周期，如果一个对象仅只有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收器回收。程序也并不能通过虚引用访问被引用的对象。</font>
虚引用主要用来跟踪垃圾回收器回收的活动，虚引用与软引用和弱引用的一个区别在于：<span style="border-bottom:1px solid red;">虚引用必须和引用队列（ReferenceQueue）联合使用</span>，当垃圾准备回收一个对象时，如果发现它还有虚引用，就会在回收对象内存之前，把这个虚引用加入到与之关联的队列中
```java
ReferenceQueue<String[]> referenceQueue = new ReferenceQueue<String[]>();
PhantomReference<String[]> referent = new PhantomReference<String>(new String[]{"a", "b", "c"}, referenceQueue);
```

## 表格对比

<style>
table th:first-of-type {
	width: 80px;
}
table th:nth-of-type(2) {
  	width: 80px;
}
</style>

| 类型 | 强引用 | 软引用 | 弱引用 | 虚引用 |
|:----|:------|:------| :------|:------|
|引用级别|最高|较高|较低|无|
|回收级别|不回收|较难回收|较易回收|容易回收|
|能否被JVM回收|不能|垃圾回收器工作并且内存不足条件下能回收|只要垃圾回收器工作就会回收|只要垃圾回收器工作就会回收|
|取得目标对象方式|直接调用|通过get()方法|通过get()方法|无法取得|
|用途|常住内存最高|内存敏感时代替强引用，常用于实现内存敏感的高速缓存|适用于存储元数据，如WeakHashMap(map 中的键值都被封装成了弱引用，一旦强引用被删除，WeakHashMap就无法阻止该对象被垃圾回收器回收)|跟踪对象被垃圾回收的状态|
|是否可能内存泄露|可能|不可能|不可能|可能|
|比如 (面向对象的现实场景) |家里的必须生活用品，你不会扔掉|家里可有可无的生活用品，家里有空间的时候你会保留，家里没空间的时候你就会扔掉|家里没什么用的东西，比如生活垃圾，不管家里有没有空间，你都不想放到家里，随时都想把他扔掉|你想知道那些必要的东西是否还保留着，于是你借用了一个储物柜（按照顺序存放），可以看到那些东西你扔掉了，那些东西你没扔掉|


- - - -
搬运地址：    

[Java中的四种引用类型](https://www.jianshu.com/p/147793693edc)    

[Java内存回收机制--Java引用的种类（强引用、弱引用、软引用、虚引用）](http://blog.csdn.net/daijin888888/article/details/49949283)
