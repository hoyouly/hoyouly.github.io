---
layout: post
title: Window 创建过程
category: 读书笔记
tags:  Android开发艺术探索 Window
description: Window 创建过程
---

* content
{:toc}

# Window 的一些结论
* View 是 Android 的视图呈现方式，
* View 不能单独存在，必须依附于 Window 这个抽象概念上面
* 有视图的地方就有 Window
* Android 提供视图的地方有 Activity ， Dialog ， Toast ，以及依托 Window 实现的视图，比如 PopUpWindow ，菜单
* Activity， Dialog ， Toast 等视图都对应一个 Window

# Activity 的 Window 创建过程
1. 分析 Activity 的 Window 的创建，必须了解 Activity 的启动过程，详细请看 [ Android 四大组件之 Activity ](../../../../2019/03/15/Android-Activity-Core/)。   
可以先记住结论：<font color="#ff000" > Activity 启动过程最终由 ActivityThread 中的 performLaunchActivity() 来完成启动, performLaunchActivity() 内部会通过类加载器创建 Activity 的对象实例，并调用 attach() 为其关联运行过程中所依赖的一系列上下文环境变量</font>   
```Java
// ActivityThread # performLaunchActivity()
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    。。。
    //通过类加载器创建 Activity 实例
    java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
    activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
    。。。
    //创建关联的上下文环境变量
    Application app = r.packageInfo.makeApplication(false, mInstrumentation);
    Context appContext = createBaseContextForActivity(r, activity);
    CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
    。。。
    //调用 attach 方法，为关联运行过程所依赖的一系列上下文变量
    activity.attach(appContext, this , getInstrumentation() , r.token,
                  r.ident, app , r.intent, r.activityInfo, title , r.parent,
                  r.embeddedID, r.lastNonConfigurationInstances, config ,
                  r.referrer, r.voiceInteractor, window);
                  。。。
    return activity;
}
```

2.  在 attach() 里面，创建 Activity 所属的 Window 对象    
```Java
//创建 Activity 所属的 Window 对象
mWindow = PolicyManager.makeNewWindow(this);        
mWindow.setWindowControllerCallback(this);
//并设置回调接口，因为 Activity 实现的 Window 的 Callback 接口，因此当 Window 接收到外界的状态改变的时候就会回调 Activity 的方法，
mWindow.setCallback(this);
mWindow.setOnWindowDismissedCallback(this);
mWindow.getLayoutInflater().setPrivateFactory(this);
```

Callback 接口中我们熟悉
* onAttachedToWindow()
* onDetachedFromWindow()
* dispatchTouchEvent()

Activity 的 Window 通过 PolicyManager 的一个工厂方法来创建，这是一个策略类。 PolicyManager 中实现的几个工厂方法全部在策略接口 IPolicy 中声明，
```java
public interface IPolicy {
    Window makeNewWindow(Context var1);

    LayoutInflater makeNewLayoutInflater(Context var1);

    WindowManagerPolicy makeNewWindowManager();

    FallbackEventHandler makeNewFallbackEventHandler(Context var1);
}
```
PolicyManager 的真正实现是 Policy 类， Policy 类中的 makeNewWindow() 方法如下，
```java
Policy
public Window makeNewWindow(Context context) {
    return new PhoneWindow(context);
}
```
由此可见， Window 的具体实现是 PhoneWindow

注意，在 attach() 中还有这样一段代码
```java
//设置WindowManager
mWindow.setWindowManager((WindowManager) context.getSystemService(Context.WINDOW_SERVICE),//
 mToken , mComponent.flattenToString(),//
    (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
```
这里说一下 mToken 吧
## Token
1. 在源码中 token 一般代表的是 Binder 对象，作用于 IPC 进程间数据通讯,并且它也包含着此次通讯所需要的信息。
2. 在 ViewRootImpl 里， token 用来表示 mWindow(W类，即 IWindow)，并且在 WMS 中只有符合要求的 token 才能让 Window 正常显示。
3. 在应用窗口中 token 表示的是 activity 的 mToken(ActivityRecord)
4. 在子窗口中 token 表示的是父窗口的 W 对象，也就是 mWindow(IWindow)

来一份图吧。
![添加图片](../../../../images/attach.png)
到这里 Window 已经创建完成,在 Activity 启动的时候，通过 attach() 方法，使用 PolicyManager.makeNewWindow(this) 创建一个 PhoneWindow 。
## Activity 显示到 Window

详情请看 [ setContentView() 探究 ](../../../../2019/01/27/setcontentview/)

# Dialog 的 Window 创建过程
和 Activity 的类似。不同的在于将 DecorView 添加到 Window 中， Dialog 是在 show 方法中，
```java
// Dialog # show
try {
    this.mWindowManager.addView(this.mDecor, l);
    this.mShowing = true;
    this.sendShowMessage();
} finally {
                ;
}
```
当 Dialog 关闭的时，它会通过 WindowManager 来移除 DecorView 。` this.mWindowManager.removeView(this.mDecor);`


# Toast 的 Window 创建过程
Toast 和 Dialog 不同，虽然 Toast 也是基于 Window 来实现的，但是由于 Toast 具有定时取消功能，所以系统采用了 Handler ， Toast 内部的两类 IPC 过程：
1. Toast 访问 NotificationManagerService(NMS)
2. NotificationManagerService(NMS) 回调 Toast 里面的 TN 接口

Toast 属于系统 Window ，视图由两种方式指定，一种是系统默认的样式，
```java
public static Toast makeText(Context context, CharSequence text , @Duration int duration) {
    Toast result = new Toast(context);

    LayoutInflater inflate = (LayoutInflater)
            context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    View v = inflate.inflate(com.android.internal.R.layout.transient_notification, null);
    TextView tv = (TextView)v.findViewById(com.android.internal.R.id.message);
    tv.setText(text);

    result.mNextView = v;
    result.mDuration = duration;

    return result;
}
```
另一种是通过 setView() 指定一个自定义 View ，
```java
public void setView(View view) {
    mNextView = view;
}
```
不管哪种，都对应 Toast 的一个 View 类型的内部成员 mNextView , Toast 提供 show() 和 cancel() 分别用于显示和隐藏 Toast ，内部都是 IPC 过程
```java
public void show() {
    if (mNextView == null) {
        throw new RuntimeException("setView must have been called");
    }

    INotificationManager service = getService();
    String pkg = mContext.getOpPackageName();
    TN tn = mTN;
    tn.mNextView = mNextView;

    try {
        service.enqueueToast(pkg, tn , mDuration);
    } catch (RemoteException e) {
        // Empty
    }
}

public void cancel() {
    mTN.hide();
    try {         
      getService().cancelToast(mContext.getPackageName(), mTN);
    } catch (RemoteException e) {
        // Empty
    }
}
```
1. 显示和隐藏 Toast 都需要通过 NMS 来实现
2. NMS 运行在系统进程中，只能远程调用显示和隐藏 Toast
3. TN 是一个 Binder 类，在 Toast 和 NMS 进行 IPC 的过程中，当 NMS 处理 Toast 的显示或者隐藏请求的时候会跨进程回调 TN 中的方法，这个时候由于 TN 运行在 Binder 线程池中，所以需要通过 Handler 将其切换到当前线程，即发送 Toast 请求所在的线程。<span style="border-bottom:1px solid red;">由于使用的 Handler 。所以 Toast 无法在没有 Looper 的线程中弹出。</span>
4. Toast 在显示过程中，调用了 NMS 的 enqueueToast() 方法， enqueueToast() 第一个参数是当前应用的包名，第二个参数 tn 表示远程回调，第三个参数表示 Toast 时长。

```Java
//NotificationManagerService.java    # INotificationManager.Stub()
public void enqueueToast(String pkg, ITransientNotification callback , int duration){
  ...
  if (!isSystemToast) {//非系统应用来说，
      int count = 0;
      //mToastQueue 是一个 ArrayList ，
      final int N = mToastQueue.size();
      for (int i=0; i<N; i++) {
           final ToastRecord r = mToastQueue.get(i);
           if (r.pkg.equals(pkg)) {
               count++;
               //mToastQueue最多能同时存在 50 个 ToastRecord ，这样做是为了防止DOS（Denial of Service ）
               if (count >= MAX_PACKAGE_NOTIFICATIONS) {//MAX_PACKAGE_NOTIFICATIONS 值是50
                   return;
               }
           }
      }
  }
  //将 Toast 请求封装为 ToastRecord 对象并将其添加到一个名为 mToastQueue 队列中
  record = new ToastRecord(callingPid, pkg , callback , duration);
  mToastQueue.add(record);
  index = mToastQueue.size() - 1;
  keepProcessAliveLocked(callingPid);

  if (index == 0) {
    //显示当前的 Toast ，
    showNextToastLocked();
  }
  ...
}
```
1. mToastQueue 是一个 ArrayList ，对于非系统应用来说， mToastQueue 最多能同时存在 50 个 ToastRecord ，这样做是为了防止DOS（Denial of Service ）。正常情况下，一个应用是不可能达到上限的。
2. 将 Toast 请求封装为 ToastRecord 对象并添加到一个名为 mToastQueue 队列中。
3. 通过 showNextToastLocked() 来显示当前的Toast

接下来就看看 怎么显示 toast ，并且会定时取消这个 toast 的吧
## Toast 显示

```java
//NotificationManagerService.java    # INotificationManager.Stub()
void showNextToastLocked() {
    //取出队头的消息
    ToastRecord record = mToastQueue.get(0);
    while (record != null) {
        try {
          //这个 callback 就是 TN 对象的远程 Binder ，
            record.callback.show();
            //发送延迟消息，
            scheduleTimeoutLocked(record);
            return;
        } catch (RemoteException e) {
            。。。
        }
    }
}
```
取出队头的 Toast ,真正显示 Toast 的地方 的是record.callback.show()，这个 callback 就是创建 ToastRecord 时候传递过来的 TN 对象，他是一个远程 Binder ，也就是说显示 Toast 最终执行到了TN.show()中

```java
//Toast. TN
public void show() {
  mHandler.post(mShow);
}
final Runnable mShow = new Runnable() {
  @Override
  public void run() {
    handleShow();
  }
};

public void handleShow() {
if (mView != mNextView) {
  // 移除就的Toast
  handleHide();
  mView = mNextView;
  // 优先取 Application 的 Context ，如果用 Acivity 的 Context ，容易导致 Activity 内存泄露有关
  Context context = mView.getContext().getApplicationContext();
  String packageName = mView.getContext().getOpPackageName();
  if (context == null) {
    context = mView.getContext();
  }
  //取 WindowManager ，这里取的 WindowManagerImpl ，
  mWM = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
  final Configuration config = mView.getContext().getResources().getConfiguration();
  final int gravity = Gravity.getAbsoluteGravity(mGravity, config.getLayoutDirection());
  mParams.gravity = gravity;
  if ((gravity & Gravity.HORIZONTAL_GRAVITY_MASK) == Gravity.FILL_HORIZONTAL) {
    mParams.horizontalWeight = 1.0f;
  }
  if ((gravity & Gravity.VERTICAL_GRAVITY_MASK) == Gravity.FILL_VERTICAL) {
    mParams.verticalWeight = 1.0f;
  }
  mParams.x = mX;
  mParams.y = mY;
  mParams.verticalMargin = mVerticalMargin;
  mParams.horizontalMargin = mHorizontalMargin;
  mParams.packageName = packageName;
  // 直接 addView 了
  mWM.addView(mView, mParams);
  trySendAccessibilityEvent();
}}
```
Toast. TN 先切到主线程中，然后执行 handleShow() 方法
在 handleShow() 中，做了如下操作
1. 如果有必要，移除之前的 Toast 。所以 Toast 不可能同时出现两个。
2. 获取 ApplicationContext ，如果用 Acivity 的 Context ，容易导致 Activity 内存泄露有关
3. 取 WindowManager ，这里取的 WindowManagerImpl ，
4. 设置 Toast 显示的位置。默认就是中间
5. 调用WindowManager.addView()方法，在这里会把 Toast 这个 View 显示出来。

这样 Toast 就显示出来了。
## Toast 自动取消
但是 Toast 怎么自动取消的呢？

这就要说到在调用 showNextToastLocked() 中执行了record.callback.show()之后，又执行了发送延迟消息的方法scheduleTimeoutLocked()

```java
//NotificationManagerService.java    # INotificationManager.Stub()
private void scheduleTimeoutLocked(ToastRecord r){
    mHandler.removeCallbacksAndMessages(r);
    Message m = Message.obtain(mHandler, MESSAGE_TIMEOUT , r);
    long delay = r.duration == Toast.LENGTH_LONG ? LONG_DELAY : SHORT_DELAY;
    mHandler.sendMessageDelayed(m, delay);
}
```
原来是通过 Handler 发送了一个延迟消息，时间一到，执行到了cancelToastLocked()

```Java
void cancelToastLocked(int index) {
    ToastRecord record = mToastQueue.get(index);
    try {
        record.callback.hide();
    } catch (RemoteException e) {
    }
    mToastQueue.remove(index);
    keepProcessAliveLocked(record.pid);
    if (mToastQueue.size() > 0) {
      //如果队列不为 null ，那么继续执行显示下一个 Toast 。
        showNextToastLocked();
    }
}
```
也是通过 TN 来 hide() 的。 hide() 里面也是切换到主线程，最后调用了 WindowManaget 的 removeView() 实现的。


---
搬运地址：    

Android 开发艺术探索
