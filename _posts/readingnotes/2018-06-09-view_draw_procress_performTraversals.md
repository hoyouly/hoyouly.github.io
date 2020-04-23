---
layout: post
title: View 的绘制 - 概览
category: 读书笔记
tags: View  Android开发艺术探索
---
<!-- * content -->
<!-- {:toc} -->
## ViewRootImpl
View 的绘制，离不开ViewRootImpl.他是链接 WindowManager 和 Decorview 的纽带， View 的三大流程都是通过 ViewRootImpl 完成，在 ActivityThread 中，当 Activity 对象被创建成功后，会将 DecorView 添加到 Window 中，同时创建 ViewRootImpl 对象，并通过 ViewRootImp 的 setView() 将 ViewRootImpl 和 DecorView 关联起来。
先说几个结论，这个以后会详细讲的。
* 每一个 Activity 对应一个 Window ，
* 每一个 Window 会有一个 DecorView ，
* 每一个 View 在被添加的时候会创建一个 ViewRootImpl , ViewRootImpl 是 View 的操作者。因为事件分发机制， View 的视图绘制都是从这个 ViewRootImpl 开始的
* 因为 DecorView 也是 View ，所以也会有一个 ViewRootImpl 。 Window 所在的 ViewRootImpl 属于这个 DecorView 的。

在 Window 中 addView() 的过程，会创建一个 ViewRootImpl ,并把这个 View 和 ViewRootImpl 关联起来
## WindowManagerGlobal # addView()
```java
//创建 ViewRootImpl ，并且将 view 与之绑定
root = new ViewRootImpl(view.getContext(), display);
root.setView(view, wparams , panelParentView);
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
在 setView() 中，我们看到了这样一段注释
`Schedule the first layout -before- adding to the window manager, to make sure we do the relayout before receiving any other events from the system.` 大概意思是在添加到 WindowManager 之前，执行第一个布局，确保在接受 到来自系统的任何其他事件之前进行重新布局，所以我们就相信， requestLayout() 方法是重新布局。

## ViewRootImpl # requestLayout()
```java
public void requestLayout() {
  if (!mHandlingLayoutInLayoutRequest) {
    checkThread();  // 检查发起布局请求的线程是否为主线程
    mLayoutRequested = true; //mLayoutRequested 是否 measure 和 layout 布局。
    scheduleTraversals();
  }
}
```
代码很短，也有注释，相信很好理解，只需要记住一点即可：mHandlingLayoutInLayoutRequest默认是 false ， 但它会在 performLayout() 被修改，

## ViewRootImpl # scheduleTraversals()
Traversal 遍历的意思，那么这个方法的意思就是执行遍历操作呗
```java
void scheduleTraversals() {
  if (!mTraversalScheduled) {
      mTraversalScheduled = true;
      // handler消息传递绘制请求
      mTraversalBarrier = mHandler.getLooper().postSyncBarrier();
      //post一个 runnable 处理-->mTraversalRunnable
      //ViewRootImpl中 W 类是 Binder 的 Native 端，用来接收 WMS 处理操作，
      //因为 W 类的接收方法是在线程池中的，所以我们可以通过 Handler 将事件处理切换到主线程中
      mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable , null);
      if (!mUnbufferedInputDispatch) {
          scheduleConsumeBatchedInput();
      }
      notifyRendererOfFramePending();
  }
}
```
虽然通过这个流程下来， scheduleTraversals() 一直在主线程中执行的，但是调用 scheduleTraversals() 的地方有很多，不能保证都是在主线程中执行，所以就进行了切换到主线程的操作。即mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable , null);其实内部逻辑就是 post 一个 Runnable 处理。所以关键在 TraversalRunnable 的 run() 方法中。
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
这样就保证 doTraversal() 一定在主线程中执行。
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
在 scheduleTraversals 方法中，通过 mHandle 发送一个 runnable 对象，在 run 方法中处理绘制流程，这一点和 ActivityThread 中的 H 类相似，因为我们知道 ViewRootImpl 中的 W 类是 Binder 的 Native 端，用来接收 WMS 处理的操作， W 类的方法是发生线程池中的，所以我们需要 <font color="#ff0000">
通过 Handler 将事件处理切换到主线程中。也就是说 doTraversal() ,以及接下来的 performTraversals() 都是在主线程中进行的</font>
### 总结  
**ViewRootImpl在其创建过程中，通过 requestLayout 向主线程发送了一条触发遍历操作的消息，遍历操作是 perforTraversals() 方法，这是一个包罗万象的方法， ViewRootImpl 中接收到的各种变化，如来自 WMS 的窗口属性变化，来自控件树的尺寸变化以及重绘都引发 performTraversals() 的调用，并在其中处理完成， View 类及其子类中的onMeasure()、onLayout()、onDraw()等回调也都是在 performTraversals() 的执行过程中直接或间接的引发。也正是如此，一次次的 performTraversals() 调用驱动着控件树有条不紊的工作，一旦此方法无法正常执行，整个控件树都将处于僵死状态。<span style="border-bottom:1px solid red;">因此 performTraversals() 函数可以说是 ViewRootImpl 的心脏。</span>**   
流程图如下  
![添加图片](../../../../images/ViewRootImpl_setView.png)

## ViewRootImpl # performTraversals()

performTraversals() 首次绘制的大致流程，会依次调用 performMeasure() ， performLayout() ， performDraw() 三个方法，这三个方法分别完成顶级 View 的 measure ， layout 和 draw 这三大流程。如下图。
![performTraversals()流程图](../../../../images/performTraversals.png)
### performMeasure()
调用 measure 方法，在 measure 方法中又会调用 onMeasure 方法，在 onMeasure 方法中则会对所有的子元素进行 measure 过程，这个时候 measure 流程就从父容器传到子元素中了，这样就完成了一次 measure 过程。接着子 View 又会重复父容器的操作，如此往复，就完成了 View 的遍历。<span style="border-bottom:1px solid red;">Measure 决定了 View 的宽和高。 Measure 完成以后，可以通过 getMeasuredWidth 和 getMeasureHeight 方法来获取到 View 测量后的宽高。</span>
### performLayout()
和 performMeasure 同理。<span style="border-bottom:1px solid red;">Layout过程决定了 View 的四个顶点的坐标和实际 View 的宽高。完成以后，可以通过getTop/Bottom/Left/Right拿到 View 的四个顶点位置，并可以通过 getWidth() 和 getHeight() 方法来拿到 View 的最终宽高。</span>
### performDraw()
和 performMeasure 同理，唯一不同的是， performDraw() 的传递过程是在 draw() 方法中通过 dispatchDraw() 来实现的。<span style="border-bottom:1px solid red;">Draw过程则决定了 View 的显示，只有 draw 方法完成以后 View 的内容才能呈现在屏幕上。</span>

再贴一份更详细的流程图
![performTraversals()流程图](../../../../images/performTraversals_1.png)

下面是源码
```java
private void performTraversals() {
  ...
  if (layoutRequested) {
     ...
      //创建了 DecorView 的 MeasureSpec ，并调用performMeasure
       measureHierarchy(host, lp , res , desiredWindowWidth , desiredWindowHeight);
    ...
    final boolean didLayout = layoutRequested && !mStopped;
            boolean triggerGlobalLayoutListener = didLayout || mAttachInfo.mRecomputeGlobalAttributes;
         if (didLayout) {
                //控件树中的控件对于自己的尺寸显然已经了然于胸。而且父控件对于子控件的位置也有了眉目，所以经过测量过程后，布局阶段会把测量结果转化为控件的实际位置与尺寸。
                // 控件的实际位置与尺寸由 View 的 mLeft ， mTop ， mRight ， mBottom 等 4 个成员变量存储的坐标值来表示。
                performLayout(lp, desiredWindowWidth , desiredWindowHeight);
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
perforMeasure() 是经由 measureHierarchy() 调用的.这个会在下一篇讲

[View 的绘制 - 概览](../../../../2018/06/09/view_draw_procress_performTraversals/)   
[View 的绘制 - Measure 流程](../../../../2018/06/12/view_draw_procress_measure/)   
[View 的绘制 - Layout 流程](../../../../2018/06/20/view_draw_procress_layout/)   
[View 的绘制 - Draw 流程， invalidate 的流程 以及 requestLayout 流程](../../../../2018/06/29/view_draw_procress_draw/)
---
搬运地址：    

Android 开发艺术探索
