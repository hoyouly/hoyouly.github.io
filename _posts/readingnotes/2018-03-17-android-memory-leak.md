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
2. 通过分析工具比如MAT来找出潜在的内存泄露继而解决

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
在java中，非静态匿名类会隐式持有外部类的引用，当run()方法没有结束之前，线程是不会被销毁的，那么MyThread 对象所引用的外部类TheadActivity实例也不会被销毁，这就造成的内存泄漏。
### 解决
1. 线程的内部类 写成 静态内部类,这样就不会再持有外部类的引用了
2. 如果在线程内要执行外类的方法，例如dosomthing()方法，那么采用弱引用的方式保存引用。
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
3. 在Activity销毁的时候如果可以，中断线程或者取消线程

## Handler造成的内存泄漏
```java
public class DemoActivity extends AppCompatActivity{
    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            //...更新UI操作
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
因为mHandler是Handler的非静态匿名内部类，持有外部类Activity的引用，如果当前Activity退出时有未处理的消息或者正在处理的消息，而消息队列中的Message持有Handler实例，而Handler实例又持有Activity实例，导致Activity内存无法回收，造成内存泄漏
### 解决
1. 改为静态内部类
2. 对Handler持有的引用改为弱引用，这样在回收的时候可以回收Handler的持有对象
3. 当Activity退出的时候，在Destory方法中移除消息队列中的所有消息和所有Runnable对象
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
                //...更新UI操作
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
在Dalivk虚拟机中，静态变量所指向的内存引用，如果不设置为null，那么GC就永远不会回收这个对象。而又由于静态变量持有当前对象，当前对象也就不会回收。
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
非静态内部类Haha 的静态引用 haha,内部类和外部类直接是相互持有引用的，SecondActivity 实例持有Haha的引用，但是haha是static修饰的，因为Dalivk虚拟机不会回收haha对象，这就导致SecondActivity不会回收，造成内存泄漏。
### 解决
在SecondActivity的onDestory()中把静态变量设置为null就可以了。
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
单例也是用了static属性，很多情况下，需要用到一个Context，因为单例的生命周期和Application保持一直，如果传递的Context是一个Activity的话，那么这个单例就一直持有这个Activity实例，这样就造成了内存泄漏，
### 解决
传递Context的时候，传递Application的全局 Context，而不是Activity。

## 资源未关闭造成的内存泄漏   
当Activity销毁的时候，关闭cursor，关闭Stream，回收Bitmap，注销内容观察者ContentObserver,注销动态广播BroadCastReciver
## 注册监听器的泄漏  
### 分析
系统服务可以通过Context.getService()获取，他们负责执行某些后台程序或者为硬件访问提供接口，如果Context想要在服务内部事件发生改变收到通知，就需要把自己注册到服务监听器中，然后这些服务就会持有Activity的对象，如果在Activity的onDestory()中没有释放就会引起内存泄漏，

### 解决
1. 使用Application的Context来代替Activity    

```java
mSensorManager = (SensorManager) this.getSystemService(Context.SENSOR_SERVICE);
===改成===
mSensorManager = (SensorManager) getApplicationContext().getSystemService(Context.SENSOR_SERVICE);
```
2. 在Activity的onDestory中取消注册监听


## 集合中对象没有清理导致的内存泄漏
### 分析
我们通常把一些对象的引用添加到一个集合上，当我们不需要这些对象的时候，并没有把他们的引用从集合中清理掉，这样集合就会越来越大，如果这个集合是一个static，那么更严重。   
### 解决 
在Activity退出之前，clear集合，然后集合设置为null，再退出集合。
## WebView造成的泄露
  不使用WebView的时候，应该调用onDestory(),并释放占用内存。
## 属性动画导致的内存泄露  
属性动画有一类无限循环的动画，如果在Activity中播放此类动画且没有在onDestory中停止动画，那么动画就会一直播放下去，并且这个时候Activity的view会被动画只有，而View有持有Activity，最终Activity就无法释放

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
