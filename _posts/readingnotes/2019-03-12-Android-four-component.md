---
layout: post
title: Android 四大组件 概况
category: 读书笔记
tags: Android开发艺术探索 Activity Service ContentProvider Broadcast IntentService
---

<!-- * content -->
<!-- {:toc} -->

### 共性
1. 注册方式上： 除了 BroadcastReceiver ，其他三种组件必须在Android-Manifest中注册， BroadcastReceiver ，即可以在Android-Manifest中注册，也可以通过代码注册
2. 调用方式上： Activity， Service 和 BroadCastReceiver 需要借助 Intent ，而 ContentProvider 无需借助Intent
3. 用户感知上： 除了 Activity ，其他三个组件对用户来说都不是可感知的

### Activity 展示型组件
主要作用是展示一个界面并和用户交互，扮演一种前台界面的角色。
用于向用户直接展示一个界面，并且可以接受用户的输入信息从而进行交互。

启动由 Intent 发出，分为两种
* 显示 Intent 明确指向一个 Activity 组件
* 隐式 Intent 指向一个或者多个目标 Activity 组件

一个 Activity 组件具有特定的启动模式，详情见  [Activity 的生命周期和启动模式](../../../../2018/03/17/Activity-lifecycle-task)

通过 Activity 的 finish() 方法结束一个 Activity 组件运行。


### Service 计算性组件
用于后台执行一系列计算任务，<font color="#ff000" > 尽管在后台执行，但是本身是运行在主线程中 的，因此耗时的后台计算任然需要在单独的线程中完成</font>。用户无法直接感知到它的存在.

有两种状态。`都可以进行后台计算，并且可以共存`

#### startService() 启动状态
* 不能和外界有直接交互
* 停止使用 stopService() ，如果没有调用 stopService ， Service 会一直在后台运行
* 多次调用 startService() ，该 Service 只能创建一次， onCreat 只会被调用一次
* 每次调用 startService() ， onStartCommand() 方法都会被调用
* 生命周期：**startService() -> onCreat()->  onStartCommand()->服务启动->stopService() ->onDestory()**

#### bindService() 绑定状态
 * 外界可以很方便的和 Service 组件进行通信，
 * 停止使用 unBindService() ，或者调用中 Context 不存在了（例如 Activity 的 finish 了）,**绑定即两者共存亡**， Service 就会调用onUnbind()->onDestory()，
 * 第一次执行 bindService() 时， onCreate() 和 onBind() 方法会被调用，
 * 多次执行 bindService() 时， onCreate() 和 onBind() 方法并不会被多次调用
 * 生命周期： **bindService() -> onCreat()-> onBind()-> 服务启动 -> onUnbind() -> onDestory()**

#### 即bindService() 又 startService()
如果即使用了 bindService() 又使用 startService() ，该 Service 会一直在后台运行
1. onCreat()始终只会调用一次
2. 停止服务需要 unbindService() 和 stopService() 同时调用才行，不论先后



### BroadcastReceiver  消息型组件
用于在不同组件乃至不同应用直接传递消息，广播注册有两种方式，
* 静态注册  在Android-Manifest中注册广播，应用安装的时候被系统解析，不需要应用启动就可以收到相应的广播。
* 动态注册  通过代码注册，Context.registerRecevier()来实现，并且在不需要的时候通过Context.unregisterReceiver()来解除广播，应用必须启动才能注册接受广播，

发送和接收过程的匹配是通过广播接收者的`<intent-filter>`来描述

可以用来实现低耦合的观察者模式，不适合执行耗时操作

一般不需要停止。

#### 本地广播
一：  基本概念

由于之前的广播都是全局的，所有应用程序都可以收到，这样很容易引起安全问题，比如说我们发生一些携带关键数据的广播有可能被其他应用拦截，或者其他广播不停的向我们的广播接收器中发送各种垃圾广播等。

为了解决这个问题， Android 引入的本地广播，使用这个机制发送的广播，只能在应用程序内部进行传递。并且广播接收器也只接收来自程序内部发生的广播。
二：  核心用法

使用 LocalBroadcastManager 来管理广播：
* 调用LocalBroadcastManager.getInstance()来获得实例
* 调用xx.registerReceiver()来注册广播
* 调用xx.sendBroadcast()发送广播
* 调用xx.unregisterReceiver()取消注册

三：  注意事项

* 本地广播无法通过静态注册来接收，相比起系统全局广播更加高效
* 在广播中启动 activity 的话，需要为 intent 加入 FLAG_ACTIVITY_NEW_TASK 的标记，不然会报错，因为需要一个栈来存放新打开的 activity 。
* 广播中弹出 AlertDialog 的话，需要设置对话框的类型为:TYPE_SYSTEM_ALERT不然是无法弹出的。

### ContentProvider  数据共享性组件
* 用于向其他组件乃至其他应用共享数据。
* 用户无法直接感知。需要实现 CRUD 四种操作
* 在它内部维持着一份数据集合，这个数据集合即可以通过数据库来实现，也可以采用其他类型来实现，比如 List 和 Map ，
<!-- * content -->Provider 内部的 CURD 需要处理好线程同步，因为这几个方法在 Binder 线程迟被调用，不需要手动停止


---
搬运地址：    

Android 开发艺术探索
