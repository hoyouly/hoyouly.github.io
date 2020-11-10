---
layout: post
title: RemoteView
category: 读书笔记
tags: RemoteView  Android开发艺术探索
---
<!-- * content -->
<!-- {:toc} -->

RemoteView  一种远程View，可以在其他进程中显示，提供了一组基础的操作用于跨进程更新它的界面。
使用场景
1. 通知栏
  通过NotificationManager的notify()来实现，可以自定义布局
2. 桌面小部件
  通过AppWidgetProvider来实现，本质是一个广播，

这两个都是运行在SystemServer进程中，为了能款进程更新UI，RemoteView 提供了一系列set方法，


RemoteView支持的类型
* Layout
  FrameLayout，LinearLayout，RelativeLayout,GridLayout
* View
  AnalogClock,Button,Chronometer,ImageButton,ImageView,ProgressBar,TextView,ViewFlipper,ListView,GridView,StackView,AdapterViewFlipper,ViewStub

不支持Edittext，不支持自定义View



---
搬运地址：    

Android 开发艺术探索
