---
layout: post
title: Android 性能优化
category: 性能优化
tags: Android开发艺术探索 性能优化
description: Android 性能优化
---

* content
{:toc}

做到以下四点
* 稳   稳定性  
  内存泄漏，内存溢出， ARN ，Crash
* 快   流畅，不卡顿
  过度绘制，嵌套，耗时，卡顿
* 小 APK 体积要小
  资源优化，代码优化
* 省   省电，省流量
  代码质量和逻辑


## 布局优化
详情： [ Android 性能优化 -- 布局优化 ](../../../../2018/07/17/Android-performance-optimization-layout/)

## 绘制优化
详情： [ Android 性能优化 -- 绘制优化 ](../../../../2018/08/05/Android-performance-optimization-draw/)


## 内存优化
### 名词解释
* 寄存器： 速度最快的存储场所。因为寄存器处于 CPU 内部，在程序中无法控制。
* 栈（Stack）:存放基本类型的数据和对象引用。但对象本身不存放在栈中，而是堆中
* 堆（Heap）: 存放由 new 创建的对象和数组。在堆中分配的内存，由 Java 虚拟机的 GC 来管理
* 静态存储区域（Static Field）: 在固定的位置存放应用程序运行试一直存在的数据， Java 在内存中专门划分了一个静态存储区域来管理一些特殊的数据变量，如静态的数据变量
* 常量池（Constant Pool）: JVM必须为每一个被装载的类型维护一个常量池。常量池就是该类型所有用到的常量的有序集合，包括直接常量（基本类型，String）和对其他类型，字段和方法的符号引用。

调用Syste.gc(),也只是建议系统进行 GG ，但是系统是否会采纳你的建议，就不一定了。

### 内存泄露优化
分为两个方面
1. 开发过程中避免写出有内存泄露的代码
2. 通过分析工具比如 MAT 来找出潜在的内存泄露继而解决

### 内存泄露的例子
详情： [Android 内存泄漏总结](../../../../2018/03/17/android-memory-leak/)


## 响应速度优化和 ANR 日志分析
响应速度优化核心思想：避免在主线程中做耗时操作，而是将耗时操作放到子线程中执行

### ANR
ANR 时间：
Activity： 5秒
BroadcastRecevier ：10秒
发生 ANR 后，系统会在data/anr目录下创建一traces.txt

## Listview优化
1. 采用 ViewHolder 并且避免 getView 执行耗时操作
2. 根据滑动状态来控制执行频率
3. 尝试开启硬件加速来使 Listview 的滑动更加流畅

## Bitmap

详情： [ Android 性能优化 -- Bitmap 优化 ](../../../../2018/07/27/Android-performance-optimization-bitmap/)

## 线程优化
采用线程池。

## 一些性能优化建议
* 避免创建过多的对象
* 不要过多的使用枚举，枚举占用的内存空间要比整型大
* 常量使用 static final 来修饰
* 使用一些 Android 特有的数据结构。比如 SparseArray 和 Pair 等，他们都具有更好的性能
* 适当的使用软引用和弱引用
* 采用内存缓存和磁盘缓存
* 尽量采用静态内部类，这样可以避免潜在的由于内部类而导致的内存泄露
* 自定义 View 的优化，使用canvas.clipRect()来帮助系统识别那些可见区域。只有在这个区域才可以被绘制到。

## 代码优化
任何 java 类，都将占用大约 500 字节的内存空间，创建一个类的实例会消耗大约 15 字节的内存，
* 对常量使用 static 修饰符
* 使用静态方法，静态方法会比普通方法提高15%左右的访问速度
* 减少不必要的成员变量。如果一个变量可以定义为局部变量，则建议你不要定义为成员变量
* 减少不必要的对象，使用基础类型会比使用对象更加节省资源，同时更应该避免频繁创建短作用域的变量
* 尽量不要使用枚举，少用迭代器
* 对于 Cursor ， Receiver ， Sensor , File 等对象，要非常主要注意对他们的创建，回收与注册，取消注册
* 避免使用 IOC 框架，因为大量使用反射会对性能的下降
* 使用 RenderScript ， OpenGL 来进行非常复杂的绘图操作
* 使用 SurfaceView 来代替 View 进行大量，频繁的绘图操作
* 尽量使用视图缓存，而不是每次都执行 inflate() 方法解析试图



## 卡顿优化
分为 UI 绘制，应用启动，页面跳转，事件响应

Android 显示过程：Android应用把经过测量，布局，绘制后的 Surface 缓存数据，通过 SurfaceFlinger 把数据渲染到显示屏幕上。通过 Android 的刷新机制来刷新数据
也就是说，应用层负责绘制，系统层负责渲染，应用层通过进程通信机制把应用层需要的数据传递到系统服务，系统层服务通过刷新机制把数据更新到屏幕上

SurfaceFlinger 服务的主要工作
* 相应客户端事件，创建 Layer 与客户端建立连接
* 接受客户端数据及属性，修改 Layer 属性，如尺寸，颜色，透明度等。
* 将创建的 Layer 内容刷新到屏幕上
* 维持 Layer 层序列，并对 Layer 最终输出做出裁剪计算。

Android 每隔 16ms 发送一个 VSYNC 信号，触发对 UI 的渲染，如果每次渲染都成功，这样就流畅，如果有一次一个操作花费时间超过 16ms ,那么就会出现丢帧现象，卡顿就出现了

卡顿的根本原因
* 绘制任务太重。绘制一帧耗时太长
* 主线程太忙，系统传递过来的 VSYNC 信号还没准备好，数据就丢失了。

### TraceView

* 代码生成 TraceView 日志
  1. Debug.startMethodTracing(); 开启监听
  2. Debug.stopMethodTracing();  结束监听。
  3. 文件路径 /sdcard/dmtrace.trace
* 通过 Android Device Monitor 生成。

#### 分析 TraceView 日志

* Incl CUP Time - 某方法占用 CPU 的时间
* Excl CUP Time - 某方法本身（不包括子方法）占用 CPU 的时间
* Incl Real Time - 某方法真正执行的时间
* Excl Real Time - 某方法本身（不包括子方法）真正执行时间
* Calls+RecurCalls -  调用次数+递归回调的次数

每个时间都包含两列，一个是实际的时间，一个是百分比。

分析的时候，<span style="border-bottom:1px solid red;"> 通常 从 Incl CPU  Time 和 Calls+RecurCalls 开始分析，对占用时间长的方法进行重点分析。如果占用时间长且Calls+RecurCalls次数少，那么就很可疑了。</span>

### 使用 Dumpsys 命令 分析系统状态
使用 Dumpsys 命令可以列出 Android 系统相关的信息和服务状态。
使用时，只需要输入 adb shell dumpsys + 参数  

常用的 Dumpsys 参数

|参数|内容|
|:----|:------:|
|activity| 显示所有的 Activity 栈的信息|
|meminfo|内存信息|
|battery|电池信息|
|package|包信息|
|wifi|显示 WIFI 信心|
|alarm|显示 alarm 信息|
|procstas|显示内存状态|

---   
搬运地址：    

Android 开发艺术探索      

Android 群英传     

[Android性能优化系列之 Bitmap 图片优化](https://blog.csdn.net/u012124438/article/details/66087785)   

[Android中常见的内存泄漏及解决方案](https://blog.csdn.net/u014005316/article/details/63258107)   

[Android 内存泄漏分析心得](https://zhuanlan.zhihu.com/p/25213586)     

[Android 布局优化之 include 与merge](https://blog.csdn.net/a740169405/article/details/50473909)
