---
layout: post
title: Android 内存泄漏总结
category: 读书笔记
tags: Android 内存泄漏
description: Android 性能优化
---

* content
{:toc}

# 内存泄露优化
分为两个方面
1. 开发过程中避免写出有内存泄露的代码
2. 通过分析工具比如 MAT 来找出潜在的内存泄露继而解决

# 常见的内存泄漏
## 线程造成的内存泄漏
```java
public class ThreadActivity extends Activity {  
    public void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_main);  
        new MyThread().start();  
    }  

    private class MyThread extends Thread {  
        @Override  
        public void run() {  
            super.run();  
            dosomthing();  
        }  
    }  
    private void dosomthing(){  

    }  
}  
```
### 分析
在 java 中，非静态匿名类会隐式持有外部类的引用，当 run() 方法没有结束之前，线程是不会被销毁的，那么 MyThread 对象所引用的外部类 TheadActivity 实例也不会被销毁，这就造成的内存泄漏。
### 解决
1. 线程的内部类 写成 静态内部类,这样就不会再持有外部类的引用了
2. 如果在线程内要执行外类的方法，例如 dosomthing() 方法，那么采用弱引用的方式保存引用。
```java
private static class MyThread extends Thread {  
    WeakReference<ThreadAvoidActivity> mThreadActivityRef;  

    public MyThread(ThreadAvoidActivity activity) {  
        mThreadActivityRef = new WeakReference<ThreadAvoidActivity>(activity);  
    }  

    @Override  
    public void run() {  
        super.run();  
        if (mThreadActivityRef == null)  
            return;  
        if (mThreadActivityRef.get() != null)  
            mThreadActivityRef.get().dosomthing();  
          // dosomthing  
    }  
}  
```
3. 在 Activity 销毁的时候如果可以，中断线程或者取消线程

## Handler造成的内存泄漏
```java
public class DemoActivity extends AppCompatActivity{
    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            //...更新 UI 操作
        }
    };
    @Override
    protected void onCreate(Bundle savedInstanceState){
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_demo);
        initDatas();
    }

    private void initDatas() {
        //...子线程获取数据，在主线程中更新UI
        Message message = Message.obtain();
        mHandler.sendMessage(message);
    }
}
```
### 分析
因为 mHandler 是 Handler 的非静态匿名内部类，持有外部类 Activity 的引用，如果当前 Activity 退出时有未处理的消息或者正在处理的消息，而消息队列中的 Message 持有 Handler 实例，而 Handler 实例又持有 Activity 实例，导致 Activity 内存无法回收，造成内存泄漏
### 解决
1. 改为静态内部类
2. 对 Handler 持有的引用改为弱引用，这样在回收的时候可以回收 Handler 的持有对象
3. 当 Activity 退出的时候，在 Destory 方法中移除消息队列中的所有消息和所有 Runnable 对象
```java
public class DemoActivity extends AppCompatActivity{
    private MyHandler mHandler = new MyHandler(this);
    private static class MyHandler extends Handler {
        private WeakReference<Context> reference;
        public MyHandler(Context context) {
            reference = new WeakReference<>(context);
        }
        @Override
        public void handleMessage(Message msg) {
            MainActivity activity = (MainActivity) reference.get();
            if(activity != null) {
                //...更新 UI 操作
            }
        }
    }
    @Override
    protected void onCreate(Bundle savedInstanceState){
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_demo);
        initDatas();
    }

    private void initDatas() {
        //...子线程获取数据，在主线程中更新UI
        Message message = Message.obtain();
        mHandler.sendMessage(message);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        //移除消息队列中所有消息和所有的Runnable
        mHandler.removeCallbacksAndMessages(null);
        mHandler = null;
    }
}
```

## 静态变量导致的内存泄漏
在 Dalivk 虚拟机中，静态变量所指向的内存引用，如果不设置为 null ，那么 GC 就永远不会回收这个对象。而又由于静态变量持有当前对象，当前对象也就不会回收。
```java
public class SecondActivity extends Activity{  
    private Handler mHandler = new Handler(){  
        @Override  
        public void handleMessage(Message msg) {  
            super.handleMessage(msg);  
            SecondActivity.this.finish();  
            this.removeMessages(0);  
        }  
    };  

    private static Haha haha;  
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        haha = new Haha();  
        mHandler.sendEmptyMessageDelayed(0,2000);  
    }  

    class Haha{  

    }  
}
```
### 分析
非静态内部类 Haha 的静态引用 haha ,内部类和外部类直接是相互持有引用的， SecondActivity 实例持有 Haha 的引用，但是 haha 是 static 修饰的，因为 Dalivk 虚拟机不会回收 haha 对象，这就导致 SecondActivity 不会回收，造成内存泄漏。
### 解决
在 SecondActivity 的 onDestory() 中把静态变量设置为 null 就可以了。
```java
protected void onDestroy() {  
    super.onDestroy();  
    if(haha!=null){  
        haha = null;  
    }  
}  
```
## 单例模式导致的内存泄露
### 分析
单例也是用了 static 属性，很多情况下，需要用到一个 Context ，因为单例的生命周期和 Application 保持一直，如果传递的 Context 是一个 Activity 的话，那么这个单例就一直持有这个 Activity 实例，这样就造成了内存泄漏，
### 解决
传递 Context 的时候，传递 Application 的全局 Context ，而不是 Activity 。

## 资源未关闭造成的内存泄漏   
当 Activity 销毁的时候，关闭 cursor ，关闭 Stream ，回收 Bitmap ，注销内容观察者 ContentObserver ,注销动态广播BroadCastReciver
## 注册监听器的泄漏  
### 分析
系统服务可以通过Context.getService()获取，他们负责执行某些后台程序或者为硬件访问提供接口，如果 Context 想要在服务内部事件发生改变收到通知，就需要把自己注册到服务监听器中，然后这些服务就会持有 Activity 的对象，如果在 Activity 的 onDestory() 中没有释放就会引起内存泄漏，

### 解决
1. 使用 Application 的 Context 来代替Activity    

```java
mSensorManager = (SensorManager) this.getSystemService(Context.SENSOR_SERVICE);
===改成===
mSensorManager = (SensorManager) getApplicationContext().getSystemService(Context.SENSOR_SERVICE);
```
2. 在 Activity 的 onDestory 中取消注册监听


## 集合中对象没有清理导致的内存泄漏
### 分析
我们通常把一些对象的引用添加到一个集合上，当我们不需要这些对象的时候，并没有把他们的引用从集合中清理掉，这样集合就会越来越大，如果这个集合是一个 static ，那么更严重。   
### 解决
在 Activity 退出之前， clear 集合，然后集合设置为 null ，再退出集合。
## WebView造成的泄露
  不使用 WebView 的时候，应该调用 onDestory() ,并释放占用内存。
## 属性动画导致的内存泄露  
属性动画有一类无限循环的动画，如果在 Activity 中播放此类动画且没有在 onDestory 中停止动画，那么动画就会一直播放下去，并且这个时候 Activity 的 view 会被动画只有，而 View 有持有 Activity ，最终 Activity 就无法释放

## 使用的检测工具
* LeakCanary
* Memory Monitor 内存监视器.
* Dump java heap
* Android Device Monitor
* MAT


---   
搬运地址：    

Android 开发艺术探索        

[Android中常见的内存泄漏及解决方案](https://blog.csdn.net/u014005316/article/details/63258107)   

[Android 内存泄漏分析心得](https://zhuanlan.zhihu.com/p/25213586)  

[内存泄漏分析二-线程](https://blog.csdn.net/wd_cloud/article/details/52848902)   

[static关键字所导致的内存泄漏问题](https://blog.csdn.net/lovejavasman/article/details/52643089)
