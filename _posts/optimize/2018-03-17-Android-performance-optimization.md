---
layout: post
title: Android 性能优化
category: 性能优化
tags: Android开发艺术探索 性能优化
description: Android 性能优化
---

<!-- * content -->
<!-- {:toc} -->

做到以下四点：`稳，快，小，省`
* 稳   稳定性  
  内存泄漏，内存溢出， ARN ，Crash
* 快   流畅，不卡顿   
  过度绘制，嵌套，耗时，卡顿
* 小 APK 体积要小   
  资源优化，代码优化
* 省   省电，省流量   
  代码质量和逻辑

主要包括一下优化措施。

## 布局优化
详情： [ Android 性能优化 -- 布局优化 ](../../../../2018/07/17/Android-performance-optimization-layout/)
## 绘制优化
详情： [ Android 性能优化 -- 绘制优化 ](../../../../2018/08/05/Android-performance-optimization-draw/)
## 内存优化
详情： [ Android 性能优化 -- 内存优化 ](../../../../2019/05/12/Android-performance-optimization-memory/)

## 响应速度优化
响应速度优化核心思想：避免在主线程中做耗时操作，而是将耗时操作放到子线程中执行

### ANR
ANR 时间：   
Activity： 5秒    
BroadcastRecevier ：10秒     
Service : 20秒    
发生 ANR 后，系统会在data/anr目录下创建一traces.txt

## Listview优化
详情： [ Android 性能优化 -- ListView 优化 ](../../../../2018/01/20/Android-performance-optimization-listview/)


## Bitmap

详情： [ Android 性能优化 -- Bitmap 优化 ](../../../../2018/07/27/Android-performance-optimization-bitmap/)

## 卡顿优化
详情： [ Android 性能优化 -- 卡顿优化 ](../../../../2020/02/06/Android-performance-optimization-block/)

## 线程优化
采用线程池。

## 启动优化
* 启动加载逻辑优化。可以采用分布加载、异步加载、延期加载策略来提高应用启动速度。
* 数据准备。数据初始化分析，加载数据可以考虑用线程初始化等策略。

## 刷新优化
* 减少刷新次数；
* 缩小刷新区域；

## 动画优化
在实现动画效果时，需要根据不同场景选择合适的动画框架来实现。有些情况下，可以用硬件加速方式来提供流畅度。

## 耗电优化
* 计算优化，避开浮点运算等。
* 避免 WaleLock 使用不当。
* 使用 JobScheduler 。

## 一些性能优化建议
* 不要过多的使用枚举，枚举占用的内存空间要比整型大
* 常量使用 static final 来修饰
* 使用一些 Android 特有的数据结构。比如 SparseArray 和 Pair 等，他们都具有更好的性能
* 适当的使用软引用和弱引用
* 采用内存缓存和磁盘缓存
* 尽量采用静态内部类，这样可以避免潜在的由于内部类而导致的内存泄露
* 合理使用 static 成员
* 使用增强 for 循环
* 使用 package 代替 private 以便私有内部类高效访问外部类成员
* 合理使用浮点类型  浮点型大概比整型数据处理速度慢两倍，所以如果整型可以解决的问题就不要用浮点型。
* 广播 BroadCast 动态注册时，记得要在调用者生命周期结束时 unregisterReceiver ,防止内存泄漏。
* 注意使用线程的同步机制（synchronized）
* 合理使用 StringBuffer , StringBuilder ,String
* 尽量使用局部变量 临时变量都保存在栈（Stack）中，速度较快。其他变量，如静态变量、实例变量等，都在堆（Heap）中创建，速度较慢。另外，依赖于具体的编译器/JVM，局部变量还可能得到进一步优化。
* 使用 IntentService 代替Service
IntentService 和 Service 都是一个服务，区别在于 IntentService 使用队列的方式将请求的 Intent 加入队列，然后开启一个worker thread(线程)来处理队列中的Intent（在 onHandleIntent 方法中），对于异步的 startService 请求， IntentService 会处理完成一个之后再处理第二个，每一个请求都会在一个单独的 worker thread 中处理，不会阻塞应用程序的主线程，如果有耗时的操作与其在 Service 里面开启新线程还不如使用 IntentService 来处理耗时操作。
* 集合中的对象要及时清理
* 尽量不要使用整张的大图作为资源文件，尽量使用 9path 图片
* 使用静态方法，静态方法会比普通方法提高15%左右的访问速度
* 减少不必要的对象，使用基础类型会比使用对象更加节省资源，同时更应该避免频繁创建短作用域的变量
* 对于 Cursor ， Receiver ， Sensor , File ,IO操作等，要非常主要注意对他们的创建，回收与注册，取消注册
* 避免使用 IOC 框架，因为大量使用反射会对性能的下降
* 使用 RenderScript ， OpenGL 来进行非常复杂的绘图操作
* 使用 SurfaceView 来代替 View 进行大量，频繁的绘图操作
* 尽量使用视图缓存，而不是每次都执行 inflate() 方法解析试图




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
