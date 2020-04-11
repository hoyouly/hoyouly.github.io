---
layout: post
title: setContentView() 探究
category: 读书笔记
tags:   setContentView
---

* content
{:toc}

Activity 怎么显示到 Window 上呢?
## setContentView()
那就要说到我们经常用到的 setContentView() 了，
```java
public void setContentView(View view) {
  getWindow().setContentView(view);
  initWindowDecorActionBar();
}
```
Activity 具体实现交给了 Window 处理，而 Window 的实现是 PhoneWindow ，
```java
//PhoneWindow.java
@Override
public void setContentView(int layoutResID) {
   if (mContentParent == null) {
       //创建 DecorView 对象和 mContentParent 对象
       installDecor();
   } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        //FEATURE_CONTENT_TRANSITIONS表示是否使用转场动画。
       //如果内容已经加载过，并且不需要动画，则会调用 removeAllViews() 移除内容以便重新填充 Layout 。
       mContentParent.removeAllViews();
   }
   //填充Layout
   if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
     //如果设置了 FEATURE_CONTENT_TRANSITIONS ，就会创建 Scene 完成转场动画。
       final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID , getContext());
       transitionTo(newScene);
   } else {
       //将 Activity 设置的布局文件，加载到 mContentParent 中
       mLayoutInflater.inflate(layoutResID, mContentParent);
       //到此， Activity 的布局文件已经添加到 DecorView 里面
   }

   //通知 Activity 布局改变
   final Callback cb = getCallback();
   if (cb != null && !isDestroyed()) {
       cb.onContentChanged();
   }
}
```
上面注释很清楚了，在介绍 installDecor() 之前，先说一下 DecorView 是个啥东西

###  DecorView
关于 DecorView 的几点总结
* DecorView 是 Activity 的顶级 View 。一般来说包含标题栏和内容栏， id 是 android.R.id.content 固定不变。   
* DecorView 的创建由 installDecor() 完成，内部通过 generateDecor() 直接创建 DecorView ，但是这个时候 DecorView 还是一个空白的 FrameLayout
* DecorView 也是 PhoneWindow 的一个内部类，继承 Framelayout , 是对 FrameLayout 的一个扩展，更确切的说是一个修饰，比如添加 titleBar ,以及 titleBar 上的滚动条等。是所有应用窗口的根 View
* DecorView 呈现在 PhoneWindow 上。

 Window ， PhoneWindow 以及 DecorView 三者的关系

```
Window 相当于一幅画，这是一个抽象概念。是什么画，山水画？肖像画？谁画的，齐白石，徐悲鸿还是其他的人啊，我们都未知。   
PhoneWindow 可以理解为齐白石的山水画，我们知道了是谁画的，什么性质的画。   
DecorView 就是这副山水画中的具体内容（有山，有水，有树等）     
```
接下来看 installDecor() 的源码
## installDecor()
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
      //根据主题 theme 设置对应的 xml 布局文件以及Feature(包括 style , layout ,转场动画,属性等)到 DecorView 中。
      mContentParent = generateLayout(mDecor);
      ...
  }          
  ...
}
```
generateDecor() 就是初始化一个 DecorView ,这个就不说了，主要说说 generateLayout() 吧。   
得到初始化 DecorView 后， PhoneWindow 通过 generateLayout() 方法加载具体的布局文件到 DecorView 中


### generateLayout()
```java
protected ViewGroup generateLayout(DecorView decor) {
  //1、根据 requestFreature() 和 Activity 节点的 android:theme="" 设置好 features 值
  ...
  //2 根据设定好的 features 值，即特定风格属性，选择不同的窗口修饰布局文件
  int layoutResource;  //窗口修饰布局文件  
  int features = getLocalFeatures();
  if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
      if (mIsFloating) {
          layoutResource = com.android.internal.R.layout.dialog_title_icons;
      } else {
          layoutResource = com.android.internal.R.layout.screen_title_icons;
      }
  } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0) {
      layoutResource = com.android.internal.R.layout.screen_progress;
  }
  ...
  //3 选定了窗口修饰布局文件 ，添加至 DecorView 对象里，并且指定 mcontentParent 值
  decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
  mContentRoot = (ViewGroup) in;
  ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
  ...
  return contentParent;
}
```

关键点已经注释了：通过上面的 1 和 2 ，所以我们可知:    
<font color="#ff000" > Activity 中必须在 setConentView() 之前调用 requestFreature() ,或者 setTheme() ，否则无效。</font>

ID_ANDROID_CONTENT 定义如下，
```java
public static final int ID_ANDROID_CONTENT = com.android.internal.R.id.content;
```
这个 id 对应的 ViewGroup 就是 mContentParent ，即`mContentParent = generateLayout(mDecor)`

generateLayout() 执行完成，那么 mContentParent 就不为 null 了，这样就可以将 View 添加到 DecorView 的 mContentparent 中，
即 `mLayoutInflater.inflate(layoutResID, mContentParent)`
```java
public void setContentView(int layoutResID) {
   if (mContentParent == null) {
       //创建 DecorView 对象和 mContentParent 对象
       installDecor();
   } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        //FEATURE_CONTENT_TRANSITIONS表示是否使用转场动画。
       //如果内容已经加载过，并且不需要动画，则会调用 removeAllViews() 移除内容以便重新填充 Layout 。
       mContentParent.removeAllViews();
   }
   //填充Layout
   if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
     //如果设置了 FEATURE_CONTENT_TRANSITIONS ，就会创建 Scene 完成转场动画。
       final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID , getContext());
       transitionTo(newScene);
   } else {
       //将 Activity 设置的布局文件，加载到 mContentParent 中
       mLayoutInflater.inflate(layoutResID, mContentParent);
       //到此， Activity 的布局文件已经添加到 DecorView 里面
   }
   //通知 Activity 布局改变
   final Callback cb = getCallback();
   if (cb != null && !isDestroyed()) {
       cb.onContentChanged();
   }
}
```

由于 Activity 实现了 Window 的 Callback 接口， Activity 的布局文件已经被添加到 DecorView 的 mContentParent 中，需要通知 Activity ，使其做相应处理， Activity 中默认为空实现，
```java
final Callback cb = getCallback();
if (cb != null && !isDestroyed()) {
    cb.onContentChanged();
}
```

来份流程图吧。
![添加图片](../../../../images/setcontentview.png)

## DecorView 添加到 Window 中
虽然现在 DecorView 已经创建并且初始化， Activity 的布局文件也添加到 DecorView 的 mContentParent 中，但这个时候 DecorView 还没有被 WindowManager 正式添加到 Window 中。
<font color="#ff000" > 真正完成 DecorView 添加和显示的是在 ActivityThread 的 handleResumeActivity() 方法中。
handleResumeActivity() 会先执行 Activity 的 onResume() ，然后执行 Activity 的 makeVisible() 方法，正是在 makeVisible() 方法中， DecorView 才会被添加到 WindowManager 中。</font>
```java
//ActivityThread.java
public void handleResumeActivity(IBinder token, boolean finalStateRequest , boolean isForward ,String reason) {
  final ActivityClientRecord r = performResumeActivity(token, finalStateRequest , reason);
  ...
  if (r.activity.mVisibleFromClient) {
    r.activity.makeVisible();
  }
}              
```
在 performResumeActivity() 中会执行 Activity 的 onResume() ，详情查看 [ Android 四大组件之 Activity ](http://hoyouly.fun/2019/03/15/Android-Activity-Core/#activitythread--handleresumeactivity)    
接下来就看 makeVisible()

```java
//Activity.java
void makeVisible() {
    if(!this.mWindowAdded) {
        ViewManager wm = this.getWindowManager();
        wm.addView(this.mDecor, this.getWindow().getAttributes());
        this.mWindowAdded = true;
    }
    this.mDecor.setVisibility(0);
}
```
这样 DecorView 才会被添加到 WindowManager 中。

----
搬运地址：    

Android 开发艺术探索

[Android中将布局文件/View添加至窗口过程分析 ---- 从 setContentView() 谈起](https://blog.csdn.net/qinjuning/article/details/7226787)
