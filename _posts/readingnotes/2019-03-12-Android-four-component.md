---
layout: post
title: Android 四大组件 概况
category: 读书笔记
tags: Android开发艺术探索 Activity Service ContentProvider Broadcast IntentService 
---

* content
{:toc}

## 四大组件的运行状态
### 共性
1. 注册方式上： 除了BroadcastReceiver，其他三种组件必须在Android-Manifest中注册，BroadcastReceiver，即可以在Android-Manifest中注册，也可以通过代码注册
2. 调用方式上： Activity，Service和BroadCastReceiver需要借助Intent，而ContentProvider无需借助Intent
3. 用户感知上： 除了Activity，其他三个组件对用户来说都不是可感知的

### Activity 展示型组件
主要作用是展示一个界面并和用户交互，扮演一种前台界面的角色。
用于向用户直接展示一个界面，并且可以接受用户的输入信息从而进行交互。

启动由Intent出发，分为两种
* 显示Intent 明确指向一个Activity组件
* 隐式Intent 指向一个或者多个目标Activity组件

一个Activity组件具有特定的启动模式，详情见  [Activity的生命周期和启动模式](http://hoyouly.fun/2018/03/17/Activity-lifecycle-task)

通过Activity的finish()方法结束一个Activity组件运行。


### Service 计算性组件
用于后台执行一系列计算任务，<font color="#ff000" > 尽管在后台执行，但是本身是运行在主线程中 的，因此耗时的后台计算任然需要在单独的线程中完成</font>。用户无法直接感知到它的存在.

有两种状态。`都可以进行后台计算，并且可以共存`
#### startService() 启动状态
* 不能和外界有直接交互
* 停止使用stopService()，如果没有调用 stopService，Service会一直在后台运行
* 多次调用startService()，该Service只能创建一次，onCreat只会被调用一次
* 每次调用startService()，onStartCommand()方法都会被调用
* 生命周期：**startService() -> onCreat()->  onStartCommand()->服务启动->stopService() ->onDestory()**
#### bindService() 绑定状态
 * 外界可以很方便的和Service组件进行通信，
 * 停止使用unBindService()，或者调用中Context不存在了（例如Activity的finish了）,**绑定即两者共存亡**，Service就会调用onUnbind()->onDestory()，
 * 第一次执行bindService()时，onCreate()和onBind()方法会被调用，
 * 多次执行bindService()时，onCreate()和onBind()方法并不会被多次调用
 * 生命周期： **bindService() -> onCreat()-> onBind()-> 服务启动 -> onUnbind() -> onDestory()**

#### 即使用bindService()又使用startService()
如果即使用了bindService()又使用startService()，该Service会一直在后台运行
1. onCreat()始终只会调用一次
2. 停止服务需要 unbindService()和stopService()同时调用才行，不论先后

### IntentService
是继承Service，在被创建的时候，就已经创建了一个HandlerThread和ServiceHandler，有了他，就可以利用IntentService执行一些优先级较高的task,因为它不会被轻易杀死，
使用方法
1. 创建一个IntentService的子类。复写其中的onHandleIntent()方法，在这里进行耗时的操作
2. startService(intent)
这样就行了，简单。可是为啥IntentService就能直接处理耗时操作呢，普通的Service还需要在onStartCommand()中再开线程才行。

IntentService NB到哪里了，这就说到另外一个类了，HanderThread

#### HandlerThread
继承 Thread，在其run()方法中为自己创建了Looper,
```java
public class HandlerThread extends Thread {
		@Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
}
```
是个Thread不假，但是在run()方法中，竟然Looper.prepare()，还Looper.loop()，新建一个Looper对象，并且已经启动loop开始循环。这就有点意思了。
 <span style="border-bottom:1px solid red;">所以使用上HandlerThread和普通的Thread不一样,无法执行后台常见的操作，只能处理新消息。因为Looper.loop()是一个死循环</span>

有了HanderThread 这个奇怪的类，就可以实现IntentService的功能了。因为 <font color="#ff000" >IntentService可以看做是 Service和HanderThread的结合体。</font>

还是看IntentService的源码吧

```java
public abstract class IntentService extends Service {

	private final class ServiceHandler extends Handler {
         public ServiceHandler(Looper looper) {
             super(looper);
         }

         @Override
         public void handleMessage(Message msg) {
             onHandleIntent((Intent)msg.obj);
             stopSelf(msg.arg1);
         }
     }
	@Override
  public void onCreate() {
     super.onCreate();
     HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
     thread.start();
     mServiceLooper = thread.getLooper();
     mServiceHandler = new ServiceHandler(mServiceLooper);
  }

	@Override
	public void onStart(Intent intent, int startId) {
		 Message msg = mServiceHandler.obtainMessage();
		 msg.arg1 = startId;
		 msg.obj = intent;
		 mServiceHandler.sendMessage(msg);
	}

	 @Override
	 public int onStartCommand(Intent intent, int flags, int startId) {
			 onStart(intent, startId);
			 return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
	 }
	 protected abstract void onHandleIntent(Intent intent);
}
```
1. 在onCreate()的时候，就创建一个HandlerThread对象，然后启动，拿到这个HandlerThread的Looper对象，这才是关键。<span style="border-bottom:1px solid red;">创建一个继承Handler的类 ServiceHandler，把得到的子线程的Looper对象传进去。</span>
2. 是Service，startService()后，肯定就会执行到了onStartCommand()方法，里面执行了onStart()的。这个时候关键来了，<span style="border-bottom:1px solid red;">把intent封装成Message然后通过ServiceHandler 送出去。</span> 之前我们讲过，<font color="#ff000" >创建Handler的时候需要一个Looper对象，这个Looper对象属于哪个线程，那么Handler就会把消息发送给那个线程。</font> ServiceHandler 创建的时候传递的是子线程的Looper对象，那么会把消息发送到这个子线程中去。所以handleMessage()就肯定是在HanderThread这个线程中执行的。
3. handleMessage()中调用了 onHandleIntent(),这个是需要我们实现的方法，可以进行执行耗时操作的方法。因为它就是执行在子线程的。
4. handleMessage() 最后会 stopSelf(msg.arg1)，把自己停掉。这么智能啊。

所以IntentService有以下特点：
* 继承Service，所以优先级笔记高
* 内创建了一个HandlerThread 和ServiceHandler，可以做一些耗时操作
* 当任务执行完成后，IntentService自动停止，不需要我们手动结束
* 如果多次启动IntentService，使用串行方式，依次执行onHandlerIntent(),执行完自动结束。那么每个耗时的操作都会消息的形式发生到消息队列中。

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

为了解决这个问题，Android引入的本地广播，使用这个机制发送的广播，只能在应用程序内部进行传递。并且广播接收器也只接收来自程序内部发生的广播。
二：  核心用法

使用LocalBroadcastManager来管理广播：
* 调用LocalBroadcastManager.getInstance()来获得实例
* 调用xx.registerReceiver()来注册广播
* 调用xx.sendBroadcast()发送广播
* 调用xx.unregisterReceiver()取消注册

三：  注意事项

* 本地广播无法通过静态注册来接收，相比起系统全局广播更加高效
* 在广播中启动activity的话，需要为intent加入FLAG_ACTIVITY_NEW_TASK的标记，不然会报错，因为需要一个栈来存放新打开的activity。
* 广播中弹出AlertDialog的话，需要设置对话框的类型为:TYPE_SYSTEM_ALERT不然是无法弹出的。

### ContentProvider  数据共享性组件
* 用于向其他组件乃至其他应用共享数据。
* 用户无法直接感知。需要实现CRUD四种操作
* 在它内部维持着一份数据集合，这个数据集合即可以通过数据库来实现，也可以采用其他类型来实现，比如List和Map，
* ContentProvider 内部的CURD需要处理好线程同步，因为这几个方法在Binder线程迟被调用，不需要手动停止


---
搬运地址：   
Android 开发艺术探索
