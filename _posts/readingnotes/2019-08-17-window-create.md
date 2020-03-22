---
layout: post
title: Window 创建过程
category: 读书笔记
tags:  Android开发艺术探索 Window
description: Window 创建过程
---

* content
{:toc}

## Window的创建过程
* View是Android的视图呈现方式，
* View 不能单独存在，必须依附于Window这个抽象概念上面
* 有视图的地方就有Window
* Android提供视图的地方有Activity，Dialog，Toast，以及依托Window实现的视图，比如PopUpWindow，菜单
* Activity，Dialog，Toast等视图都对应一个Window

### Activity的Window创建过程
1. 分析Activity的Window的创建，必须了解Activity的启动过程，详细流程后续。可以先记住结论：<font color="#ff000" > Activity启动过程最终由ActivityThread中的performLaunchActivity()来完成启动,performLaunchActivity()内部会通过类加载器创建Activity的对象实例，并调用attach()为其关联运行过程中所依赖的一系列上下文环境变量</font>
```Java
// ActivityThread # performLaunchActivity()
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    。。。
    //通过类加载器创建Activity实例
    java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
    activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
    。。。
    //创建关联的上下文环境变量
    Application app = r.packageInfo.makeApplication(false, mInstrumentation);
    Context appContext = createBaseContextForActivity(r, activity);
    CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
    。。。
    //调用attach方法，为关联运行过程所依赖的一系列上下文变量
    activity.attach(appContext, this, getInstrumentation(), r.token,
                  r.ident, app, r.intent, r.activityInfo, title, r.parent,
                  r.embeddedID, r.lastNonConfigurationInstances, config,
                  r.referrer, r.voiceInteractor, window);
                  。。。
    return activity;
}
```
2.  在attach()里面，创建 Activity 所属的Window对象

```Java
//创建Activity所属的Window对象
mWindow = PolicyManager.makeNewWindow(this);        
mWindow.setWindowControllerCallback(this);
//并设置回调接口，因为Activity实现的Window的Callback接口，因此当Window接收到外界的状态改变的时候就会回调Activity的方法，
mWindow.setCallback(this);
mWindow.setOnWindowDismissedCallback(this);
mWindow.getLayoutInflater().setPrivateFactory(this);

```
Callback接口中我们熟悉的，
* onAttachedToWindow()
* onDetachedFromWindow()
* dispatchTouchEvent()

Activity的Window通过PolicyManager的一个工厂方法来创建，这是一个策略类，PolicyManager中实现的几个工厂方法全部在策略接口IPolicy中声明，
```java
public interface IPolicy {
    Window makeNewWindow(Context var1);

    LayoutInflater makeNewLayoutInflater(Context var1);

    WindowManagerPolicy makeNewWindowManager();

    FallbackEventHandler makeNewFallbackEventHandler(Context var1);
}
```
PolicyManager的真正实现是Policy类，Policy类中中的makeNewWindow方法如下，
```java
Policy
public Window makeNewWindow(Context context) {
    return new PhoneWindow(context);
}
```
由此可见，Window的具体实现是PhoneWindow

注意，在attach()中还有这样一段代码
```java
//设置WindowManager
mWindow.setWindowManager((WindowManager) context.getSystemService(Context.WINDOW_SERVICE),//
		mToken, mComponent.flattenToString(),//
		(info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
```
这里说一下mToken 吧
#### Token
1. 在源码中token一般代表的是Binder对象，作用于IPC进程间数据通讯,并且它也包含着此次通讯所需要的信息。
2. 在ViewRootImpl里，token用来表示mWindow(W类，即IWindow)，并且在WMS中只有符合要求的token才能让Window正常显示。
3. 应用窗口 : token表示的是activity的mToken(ActivityRecord)
4. 子窗口 : token表示的是父窗口的W对象，也就是mWindow(IWindow)

![添加图片](../../../../images/attach.png)
到这里Window已经创建完成,在Activity启动的时候，通过attach()方法，使用PolicyManager.makeNewWindow(this) 创建一个PhoneWindow。
#### Activity 显示到Window
Window创建成功，Activity怎么显示到Window上呢?那就要说到我们经常用到的setContentView()了，
```java
public void setContentView(View view) {
  getWindow().setContentView(view);
  initWindowDecorActionBar();
}
```
Activity具体实现交给了Window处理，而Window的实现是PhoneWindow，
```java
//PhoneWindow.java
@Override
public void setContentView(int layoutResID) {
   if (mContentParent == null) {
       //创建DecorView对象和mContentParent对象
       installDecor();
   } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        //FEATURE_CONTENT_TRANSITIONS表示是否使用转场动画。
       //如果内容已经加载过，并且不需要动画，则会调用removeAllViews()移除内容以便重新填充Layout。
       mContentParent.removeAllViews();
   }
   //填充Layout
   if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
     //如果设置了FEATURE_CONTENT_TRANSITIONS，就会创建Scene完成转场动画。
       final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID, getContext());
       transitionTo(newScene);
   } else {
       //将Activity设置的布局文件，加载到mContentParent中
       mLayoutInflater.inflate(layoutResID, mContentParent);
       //到此，Activity的布局文件已经添加到DecorView里面
   }

   //通知Activity布局改变
   final Callback cb = getCallback();
   if (cb != null && !isDestroyed()) {
       cb.onContentChanged();
   }
}
```
上面注释很清楚了，接下来先说 installDecor()
* 创建DecorView

DecorView之前也说过。是Activity的顶级View。
一般来说包含标题栏和内容栏，内容栏的id固定：android.R.id.content，

DecorView的创建由installDecor完成，内部通过generateDecor方法直接创建DecorView，但是这个时候DecorView还是一个空白的FrameLayout
```java
private void installDecor() {
  if (mDecor == null) {
      mDecor = generateDecor();
      mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
      mDecor.setIsRootNamespace(true);
      if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
          mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
      }
  }
  if (mContentParent == null) {
      //根据主题theme设置对应的xml布局文件以及Feature(包括style,layout,转场动画,属性等)到DecorView中。
      mContentParent = generateLayout(mDecor);
      ...
  }          
  。。。
}
```
得到初始化DecorView后，PhoneWindow通过generateLayout()方法加载具体的布局文件到DecorView中
```java
protected ViewGroup generateLayout(DecorView decor) {
  ...
  View in = mLayoutInflater.inflate(layoutResource, null);
  ...
  decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
  mContentRoot = (ViewGroup) in;
  ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
  ...
  return contentParent;
}
```
ID_ANDROID_CONTENT 定义如下，
```java
public static final int ID_ANDROID_CONTENT = com.android.internal.R.id.content;
```
这个id对应的ViewGroup就是mContentParent `mContentParent = generateLayout(mDecor);`
* 这样 mContentParent 就不为null，就可以将View添加到DecorView的mContentparent中，
在setContentView方法中`mLayoutInflater.inflate(layoutResID, mContentParent);`


*  回调Activity的onContentChanged方法通知Activity的视图已经发生改变
由于Activity实现了Window的Callback接口，走到这里表示Activity的布局文件已经被添加到DecorView的mContentParent中，需要通知Activity，使其做相应处理，默认为空实现，
```java
final Callback cb = getCallback();
if (cb != null && !isDestroyed()) {
    cb.onContentChanged();
}
```

来份流程图吧。
![添加图片](../../../../images/setcontentview.png)

虽然现在DecorView已经创建并且初始化，Activity的布局文件也添加到DecorView的mContentParent中，但这个时候DecorView还没有被WindowManager正式添加到Window中，而<span style="border-bottom:1px solid red;">真正完成DecorView添加和显示的是在ActivityThread的handleResumeActivity()方法中。
handleResumeActivity()会先执行Activity的onResume()，然后执行Activity的makeVisible()方法，正是在makeVisible()方法中，DecorView才会被添加到WindowManager中。</span>
```java
void makeVisible() {
    if(!this.mWindowAdded) {
        ViewManager wm = this.getWindowManager();
        wm.addView(this.mDecor, this.getWindow().getAttributes());
        this.mWindowAdded = true;
    }
    this.mDecor.setVisibility(0);
}
```

### Dialog的Window创建过程
和Activity的类似，
不同的在于将DecorView添加到Window中，Dialog是在show方法中，
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
当Dialog关闭的时，它会通过WindowManager来移除DecorView。` this.mWindowManager.removeView(this.mDecor);`


### Toast的Window创建过程
Toast和Dialog不同，虽然Toast也是基于Window来实现的，但是由于Toast具有定时取消功能，所以系统采用了Handler，Toast内部的两类IPC过程：
1. Toast访问NotificationManagerService(NMS)
2. NotificationManagerService(NMS)回调Toast里面的TN接口

Toast属于系统Window，视图由两种方式指定，一种是系统默认的样式，
```java
public static Toast makeText(Context context, CharSequence text, @Duration int duration) {
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
另一种是通过setView方法指定一个自定义View，
```java
public void setView(View view) {
    mNextView = view;
}
```
不管哪种，都对应Toast的一个View类型的内部成员mNextView,Toast提供show和cancel分别用于显示和隐藏Toast，内部都是IPC过程
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
        service.enqueueToast(pkg, tn, mDuration);
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
1. 显示和隐藏Toast都需要通过NMS来实现
2. NMS运行在系统进程中，只能远程调用显示和隐藏Toast
3. TN是一个Binder类，在Toast和NMS进行IPC的过程中，当 NMS 处理Toast的显示或者隐藏请求的时候会跨进程回调TN中的方法，这个时候由于TN运行在Binder线程池中，所以需要通过Handler将其切换到当前线程，即发送Toast请求所在的线程。<span style="border-bottom:1px solid red;">由于使用的Handler。所以Toast无法在没有Looper的线程中弹出。</span>
4. Toast在显示过程中，调用了NMS的enqueueToast()方法，enqueueToast()第一个参数是当前应用的包名，第二个参数tn表示远程回调，第三个参数表示Toast时长。

```Java
//NotificationManagerService.java    # INotificationManager.Stub()
public void enqueueToast(String pkg, ITransientNotification callback, int duration){
  ...
  if (!isSystemToast) {//非系统应用来说，
      int count = 0;
      //mToastQueue 是一个ArrayList，
      final int N = mToastQueue.size();
      for (int i=0; i<N; i++) {
           final ToastRecord r = mToastQueue.get(i);
           if (r.pkg.equals(pkg)) {
               count++;
               //mToastQueue最多能同时存在50个ToastRecord，这样做是为了防止DOS（Denial of Service ）
               if (count >= MAX_PACKAGE_NOTIFICATIONS) {//MAX_PACKAGE_NOTIFICATIONS 值是50
                   return;
               }
           }
      }
  }
  //将Toast请求封装为ToastRecord对象并将其添加到一个名为mToastQueue队列中
  record = new ToastRecord(callingPid, pkg, callback, duration);
  mToastQueue.add(record);
  index = mToastQueue.size() - 1;
  keepProcessAliveLocked(callingPid);

  if (index == 0) {
    //显示当前的Toast，
    showNextToastLocked();
  }
  ...
}
```
1. mToastQueue 是一个ArrayList，对于非系统应用来说，mToastQueue最多能同时存在50个ToastRecord，这样做是为了防止DOS（Denial of Service ）。正常情况下，一个应用是不可能达到上限的。
2. 将Toast请求封装为ToastRecord对象并添加到一个名为mToastQueue队列中。
3. 通过showNextToastLocked()来显示当前的Toast

接下来就看看 怎么显示toast，并且会定时取消这个toast的吧
#### Toast 显示

```java
//NotificationManagerService.java    # INotificationManager.Stub()
void showNextToastLocked() {
    //取出队头的消息
    ToastRecord record = mToastQueue.get(0);
    while (record != null) {
        try {
          //这个callback就是TN对象的远程Binder，
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
取出队头的Toast,真正显示Toast的地方 的是record.callback.show()，这个callback就是创建ToastRecord 时候传递过来的TN对象，他是一个远程Binder，也就是说显示Toast最终执行到了TN.show()中

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
	// 优先取Application的Context，如果用Acivity的Context，容易导致Activity内存泄露有关
	Context context = mView.getContext().getApplicationContext();
	String packageName = mView.getContext().getOpPackageName();
	if (context == null) {
		context = mView.getContext();
	}
	//取WindowManager，这里取的WindowManagerImpl，
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
	// 直接addView了
	mWM.addView(mView, mParams);
	trySendAccessibilityEvent();
}}
```
Toast. TN 先切到主线程中，然后执行handleShow()方法
在handleShow()中，做了如下操作
1. 如果有必要，移除之前的Toast。所以Toast 不可能同时出现两个。
2. 获取ApplicationContext，如果用Acivity的Context，容易导致Activity内存泄露有关
3. 取WindowManager，这里取的WindowManagerImpl，
4. 设置Toast显示的位置。默认就是中间
5. 调用WindowManager.addView()方法，在这里会把Toast这个View显示出来。

这样Toast就显示出来了。
#### Toast 自动取消
但是Toast 怎么自动取消的呢？

这就要说到在调用showNextToastLocked()中执行了record.callback.show()之后，又执行了发送延迟消息的方法scheduleTimeoutLocked()

```java
//NotificationManagerService.java    # INotificationManager.Stub()
private void scheduleTimeoutLocked(ToastRecord r){
    mHandler.removeCallbacksAndMessages(r);
    Message m = Message.obtain(mHandler, MESSAGE_TIMEOUT, r);
    long delay = r.duration == Toast.LENGTH_LONG ? LONG_DELAY : SHORT_DELAY;
    mHandler.sendMessageDelayed(m, delay);
}
```
原来是通过Handler发送了一个延迟消息，时间一到，执行到了cancelToastLocked()

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
      //如果队列不为null，那么继续执行显示下一个Toast。
        showNextToastLocked();
    }
}
```
也是通过TN来hide()的。hide()里面也是切换到主线程，最后调用了WindowManaget的removeView()实现的。


---
搬运地址：    

Android 开发艺术探索
