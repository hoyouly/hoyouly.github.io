---
layout: post
title: Android性能优化
category: 读书笔记
tags: Android  Android开发艺术探索 性能优化
description: Android性能优化
---

* content
{:toc}

## 布局优化
思想很简单，就是尽量减少布局文件的层级，层级越少，Android绘制的工作量越少，那么程序的性能自然越高
布局优化的方法
1. 删除布局中无用的空间和层级
2. 选择使用性能高较低的ViewGroup，如果可以使用LinearLayout和RelativeLayout都可以使用，那么就采用LinearLayout，因为RelativeLayout的功能比较复杂，他的布局过程需要花费更多的CPU时间，但是如果通过一个LinearLayout不能实现，需要嵌套方式完成，建议使用RelativeLayout，因为ViewGroup的嵌套相当于增加了布局的层级，
3. 采用 include  标签 ，merge标签和ViewStub
include 主要用于布局的重用，merge一般和include配合使用，可以降低布局层级，ViewStub提供按需加载的功能，

### include 标签，
可以将一个指定的布局文件加载到当前布局文件中，注意：*只支持 android：layout_开头的属性，比如android:layout_width， android:layout_height，但是android:background 这样的属性就不支持，但是android:id是个特例 ，如果指定了android：layout_开头的属性，那么android:layout_width， android:layout_height 这两个属性必须存在*

### merge标签
一般和include标签一起使用，如果当前布局是一个竖直方向的LinearLayout，这个时候如果包含的布局文件也是采用竖直方向的LinearLayout，那么显然被包含的布局文件中的LinearLayout是多余的，通过merge标签就可以去掉多有的，

### ViewStub
基础View，是一个非常轻量级并且宽高都是0，因此本身不参与任何布局和绘制过程，存在的意义就是按需加载所需要的布局文件。在实际开发过程中，很多布局文件在正常情况下是不会显示的，这个时候就没有必要再整个界面初始化的时候加载进来，通过VIewStub就可以做到在使用的时候进行加载，提高程序的初始化的性能

![Alt text](http://p5sfwb51p.bkt.clouddn.com/1465729201927.png)

stub_import 是ViewStub的id，panel_import 是layout_network_error这个布局的根元素的id，如何进行按需加载ViewStub的布局呢，可以有两种方式
``` java
((ViewStub)findViewById(R.id.stub_import)).setVisibility(View.VISIBLE);
===
View importPanel=((ViewStub)findViewById(R.id.stub_import)).inflate();
```
当ViewStub通过setVisibility或者inflate方法加载后，ViewStub就会被它内部的布局替换掉，这个时候ViewStub就不在是整个布局的结构的一部分了，目前不支持merge标签
## 绘制优化
避免在onDraw方法中执行大量操作，
1. 不要在onDraw方法中创建新的局部对象，因为onDraw可能会被频繁调用，这样就会一瞬间产生大量临时对象，这样不仅占用过多的内存而且还导致系统频繁GC，降低程序的执行效率
2. 不要在onDraw中执行耗时操作，也不能执行成千上万次的循环操作

## 内存泄露优化
分为两个方面
1. 开发过程中避免写出有内存泄露的代码
2. 通过分析工具比如MAT来找出潜在的内存泄露继而解决

### 内存泄露的例子
* Handler 造成的内存泄漏
  1. 创建一个静态内部类，然后对Handler持有对象使用弱引用，
  2. 当前Activity退出的时候，移除消息队列中所有的消息和Runnable
* 线程造成的内存泄漏
  1. 线程写成静态内部类
  2. 在Activity销毁的时候中断线程，或者取消线程
* 非静态内部类造成的泄漏  
  写成静态内部类，因为非静态内部类默认持有外部类的一个引用。
* 资源未关闭造成的内存泄漏   
    当Activity销毁的时候，关闭cursor，关闭Stream，回收Bitmap，注销内容观察者ContentObserver,注销动态广播BroadCastReciver
* 注册监听器的泄漏  
  在Activity的onDestory中取消注册监听
* 集合中对象没有清理导致的内存泄漏
  我们通常把一些对象的引用添加到一个集合上，当我们不需要这些对象的时候，并没有把他们的引用从集合中清理掉，这样集合就会越来越大，如果这个集合是一个static，那么更严重。   
  解决办法：在Activity退出之前，clear集合，然后集合设置为null，再退出集合。
* WebView造成的泄露
  不使用WebView的时候，应该调用onDestory(),并释放占用内存。
* 静态变量导致的内存泄露
* 单例模式导致的内存泄露
* 属性动画导致的内存泄露  
属性动画有一类无限循环的动画，如果在Activity中播放此类动画且没有在onDestory中停止动画，那么动画就会一直播放下去，并且这个时候Activity的view会被动画只有，而View有持有Activity，最终Activity就无法释放

#### 使用的检测工具
* LeakCanary
* Memory Monitor 内存监视器.
* Dump java heap
* Android Device Monitor
* MAT

### 响应速度优化和ANR日志分析
响应速度优化核心思想：避免在主线程中做耗时操作，而是将耗时操作放到子线程中执行
#### ANR
ANR 时间：
Activity： 5秒
BroadcastRecevier ：10秒
发生ANR后，系统会在data/anr目录下创建一traces.txt
## Listview优化和Bitmap优化
1. 采用ViewHolder并且避免getView执行耗时操作
2. 根据滑动状态来控制执行频率
3. 尝试开启硬件加速来使Listview的滑动更加流畅

## Bitmap

### 基础知识
一张图片Bitmap所占用的内存=图片的长度*图片宽度*一个像素点占用的字节数
而Bitmap.Config正是单位像素占用的字节数的重要依据
* ALPHA_8  表示8个alpha位图，即A=8;一个像素点占用一个字节，他没有颜色，只有透明度
* ARGB_4444 表示16位的ARGB位图，即A=4,R=4，G=4,B=4，一个像素点占用4+4+4+4=16个位，即2字节
* ARGB_8888  表示32位的ARGB位图，即A=8,R=8，G=8,B=，一个像素点占用8+8+8+8=32个位，即4字节
* RGB_565  表示16位的ARGB位图，即R=5，G=6,B=4，一个像素点占用5+6+4=16个位，即2字节

其中 A 表示透明度，R表示红色，G表示绿色，B表示蓝色
如果以一个100*100 像素的图片计算所占用内存大小

|Bitmap.Config|单位像素占用的字节数|分辨率100*100的图片所占用内存大小|
|:----|:------|:------|
|ALPHA_8|1|100x100x1=10000B~=9.77KB|
|ARGB_4444|2|100x100x2=20000B~=19.53KB|
|ARGB_8888|4|100x100x4=40000B~=39.06KB||
|RGB_565|2|100x100x2=20000B~=19.53KB|
### 优化策略
1. Bitmap.Config的配置  质量压缩
2. 使用inJustDecoeBounds与判断Bitmap的大小以及使用inSampleSize进行压缩，边界压缩
3. 对Density>240的设备进行Bitmap适配(缩放Densty)
4. Bitmap的回收
5. 使用libjepg.so库进行压缩
## 线程优化
采用线程池。

## 一些性能优化建议
* 避免创建过多的对象
* 不要过多的使用枚举，枚举占用的内存空间要比整型大
* 常量使用static final 来修饰
* 使用一些Android特有的数据结构。比如SparseArray和Pair等，他们都具有更好的性能
* 适当的使用软引用和弱引用
* 采用内存缓存和磁盘缓存
* 尽量采用静态内部类，这样可以避免潜在的由于内部类而导致的内存泄露

## 代码优化
任何java类，都将占用大约500字节的内存空间，创建一个类的实例会消耗大约15字节的内存，
* 对常量使用static修饰符
* 使用静态方法，静态方法会比普通方法提高15%左右的访问速度
* 减少不必要的成员变量。如果一个变量可以定义为局部变量，则建议你不要定义为成员变量
* 减少不必要的对象，使用基础类型会比使用对象更加节省资源，同时更应该避免频繁创建短作用域的变量
* 尽量不要使用枚举，少用迭代器
* 对于Cursor，Receiver，Sensor,File等对象，要非常主要注意对他们的创建，回收与注册，取消注册
* 避免使用IOC框架，因为大量使用反射会对性能的下降
* 使用RenderScript，OpenGL来进行非常复杂的绘图操作
* 使用SurfaceView来代替View进行大量，频繁的绘图操作
* 尽量使用视图缓存，而不是每次都执行inflate()方法解析试图

---   
搬运地址：    
Android 开发艺术探索      
[Android性能优化系列之Bitmap图片优化](https://blog.csdn.net/u012124438/article/details/66087785)   
[Android中常见的内存泄漏及解决方案](https://blog.csdn.net/u014005316/article/details/63258107)   
[Android 内存泄漏分析心得](https://zhuanlan.zhihu.com/p/25213586)
