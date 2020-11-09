---
layout: post
title: Android 性能优化 -- 卡顿优化
category: 性能优化
tags: Android开发艺术探索 性能优化
description: Android 性能优化
---

主要分为 UI 绘制，应用启动，页面跳转，事件响应

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

[Android优化】最强 ListView 优化方案](http://blog.csdn.net/gs12software/article/details/51173392)

[Android APP性能优化(最新总结)](http://blog.csdn.net/csdn_aiyang/article/details/74989318)

[Android开发性能优化总结(一)](http://blog.csdn.net/gs12software/article/details/51173392)
