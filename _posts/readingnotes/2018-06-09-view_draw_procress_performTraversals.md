---
layout: post
title: View 的绘制 - 概览
category: 读书笔记
tags: View  Android开发艺术探索
---
* content
{:toc}
## ViewRootImpl
View的绘制，离不开ViewRootImpl.他是链接WindowManager和Decorview的纽带，View的三大流程都是通过ViewRootImpl完成，在ActivityThread中，当Activity对象被创建成功后，会将DecorView添加到Window中，同时创建ViewRootImpl对象，并通过ViewRootImp的setView()将ViewRootImpl和DecorView关联起来。
先说几个结论，这个以后会详细讲的。
* 每一个Activity对应一个Window，
* 每一个Window 会有一个DecorView，
* 每一个View在被添加的时候会创建一个ViewRootImpl,ViewRootImpl 是View的操作者。因为事件分发机制，View的视图绘制都是从这个ViewRootImpl开始的
* 因为DecorView 也是View，所以也会有一个ViewRootImpl。Window 所在的ViewRootImpl 属于这个DecorView的。

在Window 中addView()的过程，会创建一个ViewRootImpl,并把这个View和ViewRootImpl 关联起来
## WindowManagerGlobal # addView()
```java
//创建ViewRootImpl，并且将view与之绑定
root = new ViewRootImpl(view.getContext(), display);
root.setView(view, wparams, panelParentView);
```
## ViewRootImpl # setView()
```java
public void setView(View view, WindowManager.LayoutParams attrs, View  panelParentView) {
    ...
    // Schedule the first layout -before- adding to the window manager,
    //to make sure we do the relayout before receiving any other events from the system.
    requestLayout();
}
```
在setView() 中，我们看到了这样一段注释
`Schedule the first layout -before- adding to the window manager, to make sure we do the relayout before receiving any other events from the system.` 大概意思是在添加到 WindowManager之前，执行第一个布局，确保在接受 到来自系统的任何其他事件之前进行重新布局，所以我们就相信，requestLayout() 方法是重新布局。

## ViewRootImpl # requestLayout()
```java
public void requestLayout() {
  if (!mHandlingLayoutInLayoutRequest) {
    checkThread();  // 检查发起布局请求的线程是否为主线程
    mLayoutRequested = true; //mLayoutRequested 是否measure和layout布局。
    scheduleTraversals();
  }
}
```
代码很短，也有注释，相信很好理解，只需要记住一点即可：mHandlingLayoutInLayoutRequest默认是false， 但它会在 performLayout() 被修改，

## ViewRootImpl # scheduleTraversals()
Traversal  遍历的意思，那么这个方法的意思就是执行遍历操作呗
```java
void scheduleTraversals() {
  if (!mTraversalScheduled) {
      mTraversalScheduled = true;
      // handler消息传递绘制请求
      mTraversalBarrier = mHandler.getLooper().postSyncBarrier();
      //post一个runnable处理-->mTraversalRunnable
      //ViewRootImpl中W类是Binder的Native端，用来接收WMS处理操作，
      //因为W类的接收方法是在线程池中的，所以我们可以通过Handler将事件处理切换到主线程中
      mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
      if (!mUnbufferedInputDispatch) {
          scheduleConsumeBatchedInput();
      }
      notifyRendererOfFramePending();
  }
}
```
虽然通过这个流程下来，scheduleTraversals()一直在主线程中执行的，但是调用scheduleTraversals()的地方有很多，不能保证都是在主线程中执行，所以就进行了切换到主线程的操作。即mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);其实内部逻辑就是post 一个Runnable 处理。所以关键在TraversalRunnable 的run()方法中。
## ViewRootImpl#TraversalRunnable # run()
```java
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
      //切换到主线程中执行。
      doTraversal();
    }
  }
```
这样就保证 doTraversal()一定在主线程中执行。
## ViewRootImpl # doTraversal()

```java
void doTraversal() {
    if (mTraversalScheduled) {
      mTraversalScheduled = false;
      mHandler.getLooper().removeSyncBarrier(mTraversalBarrier);
      Trace.traceBegin(Trace.TRACE_TAG_VIEW, "performTraversals");
      try {
         //View的绘制流程正式开始。
        performTraversals();
      } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
      }
    }
  }
```
解释一下：
在scheduleTraversals 方法中，通过mHandle发送一个runnable对象，在run方法中处理绘制流程，这一点和ActivityThread中的H类相似，因为我们知道ViewRootImpl中的W类是Binder的Native端，用来接收WMS处理的操作，W类的方法是发生线程池中的，所以我们需要 <font color="#ff0000">
通过Handler将事件处理切换到主线程中。也就是说doTraversal(),以及接下来的performTraversals()都是在主线程中进行的</font>
### 总结  
**ViewRootImpl在其创建过程中，通过requestLayout向主线程发送了一条触发遍历操作的消息，遍历操作是perforTraversals()方法，这是一个包罗万象的方法，ViewRootImpl中接收到的各种变化，如来自WMS的窗口属性变化，来自控件树的尺寸变化以及重绘都引发performTraversals()的调用，并在其中处理完成，View类及其子类中的onMeasure()、onLayout()、onDraw()等回调也都是在performTraversals()的执行过程中直接或间接的引发。也正是如此，一次次的performTraversals()调用驱动着控件树有条不紊的工作，一旦此方法无法正常执行，整个控件树都将处于僵死状态。<span style="border-bottom:1px solid red;">因此 performTraversals() 函数可以说是ViewRootImpl的心脏。</span>**   
流程图如下  
![添加图片](../../../../images/ViewRootImpl_setView.png)

## ViewRootImpl # performTraversals()

performTraversals()首次绘制的大致流程，会依次调用performMeasure()，performLayout()，performDraw()三个方法，这三个方法分别完成顶级View的measure，layout和draw这三大流程。如下图。
![performTraversals()流程图](../../../../images/performTraversals.png)
### performMeasure()
调用measure方法，在measure方法中又会调用onMeasure方法，在onMeasure方法中则会对所有的子元素进行measure过程，这个时候measure流程就从父容器传到子元素中了，这样就完成了一次measure过程。接着子View又会重复父容器的操作，如此往复，就完成了View的遍历。<span style="border-bottom:1px solid red;">Measure 决定了View的宽和高。 Measure完成以后，可以通过getMeasuredWidth和getMeasureHeight方法来获取到View测量后的宽高。</span>
### performLayout()
和performMeasure同理。<span style="border-bottom:1px solid red;">Layout过程决定了View的四个顶点的坐标和实际View的宽高。完成以后，可以通过getTop/Bottom/Left/Right拿到View的四个顶点位置，并可以通过getWidth()和getHeight()方法来拿到View的最终宽高。</span>
### performDraw()
和performMeasure同理，唯一不同的是，performDraw()的传递过程是在draw()方法中通过dispatchDraw()来实现的。<span style="border-bottom:1px solid red;">Draw过程则决定了View的显示，只有draw方法完成以后View的内容才能呈现在屏幕上。</span>

再贴一份更详细的流程图
![performTraversals()流程图](../../../../images/performTraversals_1.png)

下面是源码
```java
private void performTraversals() {
  ...
  if (layoutRequested) {
     ...
      //创建了DecorView的MeasureSpec，并调用performMeasure
       measureHierarchy(host, lp, res,desiredWindowWidth, desiredWindowHeight);
    ...
    final boolean didLayout = layoutRequested && !mStopped;
            boolean triggerGlobalLayoutListener = didLayout || mAttachInfo.mRecomputeGlobalAttributes;
         if (didLayout) {
                //控件树中的控件对于自己的尺寸显然已经了然于胸。而且父控件对于子控件的位置也有了眉目，所以经过测量过程后，布局阶段会把测量结果转化为控件的实际位置与尺寸。
                // 控件的实际位置与尺寸由View的mLeft，mTop，mRight，mBottom 等4个成员变量存储的坐标值来表示。
                performLayout(lp, desiredWindowWidth, desiredWindowHeight);
          }

    ...
    boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || viewVisibility != View.VISIBLE;
        if (!cancelDraw && !newSurface) {
            if (!skipDraw || mReportNextDraw) {
                if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                    for (int i = 0; i < mPendingTransitions.size(); ++i) {
                        mPendingTransitions.get(i).startChangingAnimations();
                    }
                    mPendingTransitions.clear();
                }
                performDraw();
            }
        } else {
            if (viewVisibility == View.VISIBLE) {
                // Try again
                scheduleTraversals();
            } else if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                for (int i = 0; i < mPendingTransitions.size(); ++i) {
                    mPendingTransitions.get(i).endChangingAnimations();
                }
                mPendingTransitions.clear();
            }
        }
}
```
后面会详细讲这三大流程。
perforMeasure() 是经由measureHierarchy() 调用的.这个会在下一篇讲

[View 的绘制 - 概览](../../../../2018/06/09/view_draw_procress_performTraversals/)   
[View 的绘制 - Measure 流程](../../../../2018/06/12/view_draw_procress_measure/)   
[View 的绘制 - Layout 流程](../../../../2018/06/20/view_draw_procress_layout/)   
[View 的绘制 - Draw 流程，invalidate 的流程 以及 requestLayout 流程](../../../../2018/06/29/view_draw_procress_draw/)
---
搬运地址：    

Android开发艺术探索
