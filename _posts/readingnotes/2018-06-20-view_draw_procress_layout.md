---
layout: post
title: View 的绘制 - Layout 流程
category: 读书笔记
tags: View  Android开发艺术探索
---
* content
{:toc}

Layout 作用就是ViewGroup用来确定子元素的位置。当ViewGroup的位置确定后，他会在onLayout()中遍历所有子元素并调用其layout()方法，在layout()方法中执行我们熟悉的onLayout()方法
子View具体的layout的位置都是相对于父容器而言的，view的layout过程同Measure同理，也是从顶级View开始，递归的完成整个控件树的布局操作

经过前面的测量，控件树中的控件对于自己的尺寸显然已经了然于胸。而且父控件对于子控件的位置也有了眉目，**所以经过测量过程后，布局阶段会把测量结果转化为控件的实际位置与尺寸。控件的实际位置与尺寸由View的mLeft，mTop，mRight，mBottom 等4个成员变量存储的坐标值来表示。**

需要注意的是： **View的mLeft，mTop，mRight，mBottom 这些坐标值是以父控件左上角为坐标原点进行计算的。倘若需要获取控件在窗口坐标系中的位置可以使用View.GetLocationWindow()或者是View.getRawX()/Y()。**

先看ViewRootImp的performLayout()方法
## ViewRootImpl #performLayout()
```java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,int desiredWindowHeight) {
  ...
  final View host = mView;
  host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
  ...
}
```
host 可以是一个View，也可以是一个ViewGroup，尽管ViewGroup也重写了layout方法，但是本质上还是通过super.layout(),调用View的layout方法
## ViewGroup # layout()
```java
public final void layout(int l, int t, int r, int b) {
  if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {
    //如果无动画，或者动画未运行
    super.layout(l, t, r, b);
  } else {
    //等待动画完成时再调用requestLayout()
    mLayoutCalledWhileSuppressed = true;
  }
}
```
所以我们直接看View的layout()即可。
## View # layout()
```java
public void layout(int l, int t, int r, int b) {
      int oldL = mLeft;
      int oldT = mTop;
      int oldB = mBottom;
      int oldR = mRight;

      //如果布局有变化，通过setFrame重新布局
      boolean changed = isLayoutModeOptical(mParent) ?
              setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

      if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
          //如果这是一个ViewGroup，还会遍历子View的layout()方法
          //如果是普通View，通知具体实现类布局变更通知
          onLayout(changed, l, t, r, b);
          //清除PFLAG_LAYOUT_REQUIRED标记
          mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
          ``````
          //布局监听通知
      }
      //清除PFLAG_FORCE_LAYOUT标记
      mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
  }
```
1. 通过setFrame()将l,t,r,b分别设置到mLeft，mTop，mRight,和mButton，这样就可以确定子View在父容器上的位置了，也就是说这四个位置是相对于父容器的
2. 调用onLayout方法，具体实现类接收到布局变更通知，如果此类是ViewGoup，还会遍历子View的layout方法，使其更新布局.

## onLayout()
对于普通View，onlayout方法是一个空实现，主要是具体实现类重写该方法后能接受到布局坐标更新信息
```java
protected void onLayout(boolean changed, int l, int t, int r, int b) {}
```
对于ViewGroup来说，和measure一样，不同的类有它不同的布局特性，在ViewGroup中onLayout方法中是abstract的，具体类必须重写该方法，以便接收布局坐标更新信息后，处理自己的子View的坐标信息。
```java
protected abstract void onLayout(boolean changed,int l, int t, int r, int b);
```
还是以 LinearLayout为例，看看onLayout()的实现

## LinearLayout # onLayout()
```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
  if (mOrientation == VERTICAL) {
    layoutVertical(l, t, r, b);
  } else {
    layoutHorizontal(l, t, r, b);
  }
}
```
这里面和measure()类似，这里选择layoutVertical()讲解
## LinearLayout # layoutVertical()

```java
void layoutVertical(int left, int top, int right, int bottom) {
    ...
    final int count = getVirtualChildCount();

    for (int i = 0; i < count; i++) {
      final View child = getVirtualChildAt(i);
      if (child == null) {
        childTop += measureNullChild(i);
      } else if (child.getVisibility() != GONE) {
        final int childWidth = child.getMeasuredWidth();
        final int childHeight = child.getMeasuredHeight();

        final LayoutParams lp = (LayoutParams) child.getLayoutParams();

        int gravity = lp.gravity;
        if (gravity < 0) {
          gravity = minorGravity;
        }
        final int layoutDirection = getLayoutDirection();
        final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
        ...

        childTop += lp.topMargin;
        //遍历所有子元素并执行setChildFrame（）来确定子元素的指定位置
        setChildFrame(child, childLeft, childTop + getLocationOffset(child), childWidth, childHeight);
        childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

        i += getChildrenSkipCount(child, i);
      }
    }
  }

private void setChildFrame(View child, int left, int top, int width, int height) {
  child.layout(left, top, left + width, top + height);
}
```
* 遍历所有子元素并调用setChildFrame()来为子元素指定位置，setChildFrame()内部调用子元素的layout()方法，
* childTop位置会逐渐增大，这就意味着后面的元素会放在靠下的位置，符合竖直方向的LinearLayout的特点，
这样父元素在layout完成自己的定位后，然后通过onLayout()调用子元素的layout方法，子元素又会通过layout方法定位自己的位置。这样一层层传递下来就完成了整个View树的layout过程。

这样layout 流程就结束了。补充一份流程图吧
![](../../../../images/perform_layout.png)

## getMeasuredWidth()和 getWidth()的区别
getMeasuredWidth()： 得到的是测量宽度。 形成与measure（）过程中
getWidth()：  得到的是最终宽度。形成与layout()过程中
所以 getMeasuredWidth()赋值时机更早一些。但是在measure()过程中，View 可能需要多次才能确定自己的测量高度。就导致第一次getMeasureWidth()的值可能和getWidth()不一直。但最终measure()之后，两个值还是相等的。

在View的默认实现中。getWidht() == getMeasuredWidth()


## 小结
* measure确定的是控件的尺寸，并在一定程度上确定了子控件的位置。而布局则是针对测量结果来实施，并最终确定子控件的位置。
* measure结果对布局过程没有约束力。虽说子控件在onMeasure()方法中计算出了自己应有的尺寸，但是由于layout()方法是由父控件调用，因此控件的位置尺寸的最终决定权掌握在父控件手中，测量结果仅仅只是一个参考。
* 因为measure过程是后根遍历(DecorView最后setMeasureDiemension())，所以子控件的测量结果影响父控件的测量结果。
* 而Layout过程是先根遍历(layout()一开始就调用setFrame()完成DecorView的布局)，所以父控件的布局结果会影响子控件的布局结果。
* 完成performLayout()后，空间树的所有控件都已经确定了其最终位置，就剩下Draw绘制了。



[View 的绘制 - 概览](../../../../2018/06/09/view_draw_procress_performTraversals/)   
[View 的绘制 - Measure 流程](../../../../2018/06/12/view_draw_procress_measure/)   
[View 的绘制 - Layout 流程](../../../../2018/06/20/view_draw_procress_layout/)   
[View 的绘制 - Draw 流程，invalidate 的流程 以及 requestLayout 流程](../../../../2018/06/29/view_draw_procress_draw/)

---
搬运地址：    

Android开发艺术探索
