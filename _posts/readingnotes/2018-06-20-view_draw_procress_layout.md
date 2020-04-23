---
layout: post
title: View 的绘制 - Layout 流程
category: 读书笔记
tags: View  Android开发艺术探索
---
<!-- * content -->
<!-- {:toc} -->

Layout 作用就是 ViewGroup 用来确定子元素的位置。当 ViewGroup 的位置确定后，他会在 onLayout() 中遍历所有子元素并调用其 layout() 方法，在 layout() 方法中执行我们熟悉的 onLayout() 方法
子 View 具体的 layout 的位置都是相对于父容器而言的， view 的 layout 过程同 Measure 同理，也是从顶级 View 开始，递归的完成整个控件树的布局操作

经过前面的测量，控件树中的控件对于自己的尺寸显然已经了然于胸。而且父控件对于子控件的位置也有了眉目，**所以经过测量过程后，布局阶段会把测量结果转化为控件的实际位置与尺寸。控件的实际位置与尺寸由 View 的 mLeft ， mTop ， mRight ， mBottom 等 4 个成员变量存储的坐标值来表示。**

需要注意的是： **View的 mLeft ， mTop ， mRight ， mBottom 这些坐标值是以父控件左上角为坐标原点进行计算的。倘若需要获取控件在窗口坐标系中的位置可以使用View.GetLocationWindow()或者是View.getRawX()/Y()。**

先看 ViewRootImp 的 performLayout() 方法
## ViewRootImpl #performLayout()
```java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth ,int desiredWindowHeight) {
  ...
  final View host = mView;
  host.layout(0, 0 , host.getMeasuredWidth(), host.getMeasuredHeight());
  ...
}
```
host 可以是一个 View ，也可以是一个 ViewGroup ，尽管 ViewGroup 也重写了 layout 方法，但是本质上还是通过super.layout(),调用 View 的 layout 方法
## ViewGroup # layout()
```java
public final void layout(int l, int t , int r , int b) {
  if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {
    //如果无动画，或者动画未运行
    super.layout(l, t , r , b);
  } else {
    //等待动画完成时再调用requestLayout()
    mLayoutCalledWhileSuppressed = true;
  }
}
```
所以我们直接看 View 的 layout() 即可。
## View # layout()
```java
public void layout(int l, int t , int r , int b) {
      int oldL = mLeft;
      int oldT = mTop;
      int oldB = mBottom;
      int oldR = mRight;

      //如果布局有变化，通过 setFrame 重新布局
      boolean changed = isLayoutModeOptical(mParent) ?
              setOpticalFrame(l, t , r , b) : setFrame(l, t , r , b);

      if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
          //如果这是一个 ViewGroup ，还会遍历子 View 的 layout() 方法
          //如果是普通 View ，通知具体实现类布局变更通知
          onLayout(changed, l , t , r , b);
          //清除 PFLAG_LAYOUT_REQUIRED 标记
          mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
          ``````
          //布局监听通知
      }
      //清除 PFLAG_FORCE_LAYOUT 标记
      mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
  }
```
1. 通过 setFrame() 将 l , t , r , b 分别设置到 mLeft ， mTop ， mRight ,和 mButton ，这样就可以确定子 View 在父容器上的位置了，也就是说这四个位置是相对于父容器的
2. 调用 onLayout 方法，具体实现类接收到布局变更通知，如果此类是 ViewGoup ，还会遍历子 View 的 layout 方法，使其更新布局.

## onLayout()
对于普通 View ， onlayout 方法是一个空实现，主要是具体实现类重写该方法后能接受到布局坐标更新信息
```java
protected void onLayout(boolean changed, int l , int t , int r , int b) {}
```
对于 ViewGroup 来说，和 measure 一样，不同的类有它不同的布局特性，在 ViewGroup 中 onLayout 方法中是 abstract 的，具体类必须重写该方法，以便接收布局坐标更新信息后，处理自己的子 View 的坐标信息。
```java
protected abstract void onLayout(boolean changed, int l , int t , int r , int b);
```
还是以 LinearLayout 为例，看看 onLayout() 的实现

## LinearLayout # onLayout()
```java
@Override
protected void onLayout(boolean changed, int l , int t , int r , int b) {
  if (mOrientation == VERTICAL) {
    layoutVertical(l, t , r , b);
  } else {
    layoutHorizontal(l, t , r , b);
  }
}
```
这里面和 measure() 类似，这里选择 layoutVertical() 讲解
## LinearLayout # layoutVertical()

```java
void layoutVertical(int left, int top , int right , int bottom) {
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
        setChildFrame(child, childLeft , childTop + getLocationOffset(child), childWidth , childHeight);
        childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

        i += getChildrenSkipCount(child, i);
      }
    }
  }

private void setChildFrame(View child, int left , int top , int width , int height) {
  child.layout(left, top , left + width, top + height);
}
```
* 遍历所有子元素并调用 setChildFrame() 来为子元素指定位置， setChildFrame() 内部调用子元素的 layout() 方法，
* childTop位置会逐渐增大，这就意味着后面的元素会放在靠下的位置，符合竖直方向的 LinearLayout 的特点，
这样父元素在 layout 完成自己的定位后，然后通过 onLayout() 调用子元素的 layout 方法，子元素又会通过 layout 方法定位自己的位置。这样一层层传递下来就完成了整个 View 树的 layout 过程。

这样 layout 流程就结束了。补充一份流程图吧
![](../../../../images/perform_layout.png)

## getMeasuredWidth()和 getWidth() 的区别
getMeasuredWidth()： 得到的是测量宽度。 形成与measure（）过程中
getWidth()：  得到的是最终宽度。形成与 layout() 过程中
所以 getMeasuredWidth() 赋值时机更早一些。但是在 measure() 过程中， View 可能需要多次才能确定自己的测量高度。就导致第一次 getMeasureWidth() 的值可能和 getWidth() 不一直。但最终 measure() 之后，两个值还是相等的。

在 View 的默认实现中。getWidht() == getMeasuredWidth()


## 小结
* measure确定的是控件的尺寸，并在一定程度上确定了子控件的位置。而布局则是针对测量结果来实施，并最终确定子控件的位置。
* measure结果对布局过程没有约束力。虽说子控件在 onMeasure() 方法中计算出了自己应有的尺寸，但是由于 layout() 方法是由父控件调用，因此控件的位置尺寸的最终决定权掌握在父控件手中，测量结果仅仅只是一个参考。
* 因为 measure 过程是后根遍历(DecorView最后setMeasureDiemension())，所以子控件的测量结果影响父控件的测量结果。
* 而 Layout 过程是先根遍历(layout()一开始就调用 setFrame() 完成 DecorView 的布局)，所以父控件的布局结果会影响子控件的布局结果。
* 完成 performLayout() 后，空间树的所有控件都已经确定了其最终位置，就剩下 Draw 绘制了。



[View 的绘制 - 概览](../../../../2018/06/09/view_draw_procress_performTraversals/)   
[View 的绘制 - Measure 流程](../../../../2018/06/12/view_draw_procress_measure/)   
[View 的绘制 - Layout 流程](../../../../2018/06/20/view_draw_procress_layout/)   
[View 的绘制 - Draw 流程， invalidate 的流程 以及 requestLayout 流程](../../../../2018/06/29/view_draw_procress_draw/)

---
搬运地址：    

Android 开发艺术探索
