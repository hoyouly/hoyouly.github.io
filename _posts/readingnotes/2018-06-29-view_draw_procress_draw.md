---
layout: post
title: View 的绘制 --- Draw 流程，invalidate的流程 以及 requestLayout 流程
category: 读书笔记
tags: View  Android开发艺术探索
---
* content
{:toc}
# View的Draw 流程
View的draw过程遵循如下几步
* 绘制背景    drawBackground();
* 绘制自己    onDraw();
* 如果是ViewGroup则绘制子View，  dispatchDraw();
* 绘制装饰（滚动条）和前景，     onDrawForeground();

View绘制过程的传递是通过dispatchDraw()来实现的，dispatchDraw()会遍历调用所有子元素的的draw(),如此draw事件就一层层传递下来，
## View # draw()
```java
public void draw(Canvas canvas) {
      final int privateFlags = mPrivateFlags;

      //检查是否是"实心(不透明)"控件。（后面有补充）
      final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
              (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
      mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;
      // Step 1, draw the background, if needed
      int saveCount;
      //非"实心"控件，将会绘制背景
      if (!dirtyOpaque) {
          drawBackground(canvas);
      }

      // skip step 2 & 5 if possible (common case)
      final int viewFlags = mViewFlags;
      boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
      boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;

      //如果控件不需要绘制渐变边界，则可以进入简便绘制流程
      if (!verticalEdges && !horizontalEdges) {
          // Step 3, draw the content
          if (!dirtyOpaque) onDraw(canvas);//非"实心"，则绘制控件本身

          // Step 4, draw the children
          dispatchDraw(canvas);//如果当前不是ViewGroup，此方法则是空实现
          // Overlay is part of the content and draws beneath Foreground
          if (mOverlay != null && !mOverlay.isEmpty()) {
              mOverlay.getOverlayView().dispatchDraw(canvas);
          }
          // Step 6, draw decorations (foreground, scrollbars)
          onDrawForeground(canvas);//绘制装饰和前景
          // we're done...
          return;
      }
      ``````
}
```
流程图如下：   
![Alt text](../../../../images/draw.png)
里面注释很详细，流程图也有，就不多解释了。但是里面涉及两个新名词。脏区域和实心控件，这个需要解释一下

## View的脏区域和实心控件

为了保证绘制效率，控件树仅对需要重绘的区域进行绘制，这部分区域成为 **脏区域（Dirty Area）**

当一个控件的内容发生变化而需要重绘时，它会通过View.invalidate()方法将其需要重绘的区域沿着控件树自下而上的交给ViewRootImpl，并保存在ViewRootImpl的mDirty成员中，最后通过scheduleTraversals()引发一次遍历，进而进行重绘工作，这样就可以保证仅位于mDirty所描述的区域得到重绘，避免了不必要的开销。

View的isOpaque()方法返回值表示此控件是否为”实心”的，**所谓”实心”控件，是指在onDraw()方法中能够保证此控件的所有区域都会被其所绘制的内容完全覆盖**。对于”实心”控件来说，背景和子元素（如果有的话）是被其onDraw()的内容完全遮住的，因此便可跳过遮挡内容的绘制工作从而提升效率。

**简单来说通过此控件所属的区域无法看到此控件下的内容。也就是既没有半透明也没有空缺的部分** 因为自定义ViewGroup控件默认是”实心”控件，所以默认不会调用drawBackground()和onDraw()方法，因为一旦ViewGroup的onDraw()方法，那么就会覆盖住它的子元素。但是我们仍然可以通过调用setWillNotDraw(false)和setBackground()方法来开启ViewGroup的onDraw()功能。

# View的invalidate()的流程

接下来说一个重要的方法，invalidate(),我们经常使用invalidate() 用来刷新UI，可是里面的逻辑到底是怎么样呢，invalidat()真正的又是干嘛的呢，和postInvalidata()区别又是啥呢？？
## View # invalidate()
invalidate()方法必须在主线程中执行，而scheduleTraversals()引发的遍历也是在主线程中执行的,但是调用invilidat() 方法并不会使遍历立即开始，因为在调用invalidate() 的方法执行完毕之前（准确说是主线程的Looper处理完其他消息之前），主线程根本没机会处理scheduleTraversals()所发出的消息，这种机制带来的好处就是 **在一个方法里面可以连续调用多个控件的invalidate()，而不用担心会由于多次重绘而产生的效率问题**

另外多次调用invalidate()方法会使得ViewRootImpl多次接收到设置脏区域的请求，ViewRootImpl会将这些脏区域累加到mDirty中，进而在随后的遍历中，一次性的完成所有脏区域的重绘。  

窗口第一次绘制时候，ViewRootImpl的mFullRedrawNeeded成员将会被设置为true，也就是说mDirty所描述的区域将会扩大到整个窗口，进而实现完整重绘。

接下来看源码吧。
```java
public void invalidate() {
    invalidate(true);
}

void invalidate(boolean invalidateCache) {
    invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
}

```
##  View # invalidateInternal()
```java
void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
            boolean fullInvalidate) {

        //如果View不可见，或者在动画中
        if (skipInvalidate()) {
            return;
        }

        //根据mPrivateFlags来标记是否重绘
        if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
                || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
                || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
                || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
            if (fullInvalidate) {//上面传入为true，表示需要全部重绘
                mLastIsOpaque = isOpaque();//
                mPrivateFlags &= ~PFLAG_DRAWN;//去除绘制完毕标记。
            }

            //添加标记，表示View正在绘制。PFLAG_DRAWN为绘制完毕。
            mPrivateFlags |= PFLAG_DIRTY;

            //清除缓存，表示由当前View发起的重绘。
            if (invalidateCache) {
                mPrivateFlags |= PFLAG_INVALIDATED;
                mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
            }

            //把需要重绘的区域传递给父View
            final AttachInfo ai = mAttachInfo;
            final ViewParent p = mParent;
            if (p != null && ai != null && l < r && t < b) {
                final Rect damage = ai.mTmpInvalRect;
                //设置重绘区域(区域为当前View在父容器中的整个布局)
                damage.set(l, t, r, b);
                p.invalidateChild(this, damage);
            }
            ``````
        }
    }
```
上述代码中，会设置一系列的标记位到mPrivateFlags中，并且通过父容器的invalidateChild()，将需要重绘的脏区域传给父容器。（ViewGroup和ViewRootImpl都继承了ViewParent类，该类中定义了子元素与父容器间的调用规范。）

## ViewGroup # invalidateChild()
```java
 public final void invalidateChild(View child, final Rect dirty) {
        ViewParent parent = this;

        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
                RectF boundingRect = attachInfo.mTmpTransformRect;
                boundingRect.set(dirty);
                ``````  
               //父容器根据自身对子View的脏区域进行调整
                transformMatrix.mapRect(boundingRect);
                dirty.set((int) Math.floor(boundingRect.left),
                        (int) Math.floor(boundingRect.top),
                        (int) Math.ceil(boundingRect.right),
                        (int) Math.ceil(boundingRect.bottom));

            // 这里的do while方法，不断的去调用父类的invalidateChildInParent()来传递重绘请求
            //直到调用到ViewRootImpl的invalidateChildInParent()（责任链模式）
            do {
                View view = null;
                if (parent instanceof View) {
                    view = (View) parent;
                }

                if (drawAnimation) {
                    if (view != null) {
                        view.mPrivateFlags |= PFLAG_DRAW_ANIMATION;
                    } else if (parent instanceof ViewRootImpl) {
                        ((ViewRootImpl) parent).mIsAnimating = true;
                    }
                }

                //如果父类是"实心"的，那么设置它的mPrivateFlags标识
                if (view != null) {
                    if ((view.mViewFlags & FADING_EDGE_MASK) != 0 &&
                            view.getSolidColor() == 0) {
                        opaqueFlag = PFLAG_DIRTY;
                    }
                    if ((view.mPrivateFlags & PFLAG_DIRTY_MASK) != PFLAG_DIRTY) {
                        view.mPrivateFlags = (view.mPrivateFlags & ~PFLAG_DIRTY_MASK) | opaqueFlag;
                    }
                }

                //往上递归调用父类的invalidateChildInParent
                parent = parent.invalidateChildInParent(location, dirty);

                //设置父类的脏区域
                //父容器会把子View的脏区域转化为父容器中的坐标区域
                if (view != null) {
                    // Account for transform on current parent
                    Matrix m = view.getMatrix();
                    if (!m.isIdentity()) {
                        RectF boundingRect = attachInfo.mTmpTransformRect;
                        boundingRect.set(dirty);
                        m.mapRect(boundingRect);
                        dirty.set((int) Math.floor(boundingRect.left),
                                (int) Math.floor(boundingRect.top),
                                (int) Math.ceil(boundingRect.right),
                                (int) Math.ceil(boundingRect.bottom));
                    }
                }
            }
            while (parent != null);
        }

```
就会继续上传，parent.invalidateChildInParent(location, dirty)，最终会执行到ViewRootImpl 中的invalidataChileInParent()中，至于原因，就是最上层的ViewParents就是ViewRootImpl。在ViewRootImpl 的 setView()中，由于传入的View正是DecorView，所以最顶层的ViewParent即ViewRootImpl。

## ViewRootImpl # setView()
```java
// 这个view 就是DecorView
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
   ...
   view.assignParent(this);
   ...
}

```
assign 分配的意思，assignParent()意思就很明显了，指定父布局，this 就是ViewRootImpl。
## View # assignParent()
```java
void assignParent(ViewParent parent) {
    if (mParent == null) {
        mParent = parent;
    } else if (parent == null) {
        mParent = null;
    } else {
        throw new RuntimeException("view " + this + " being added, but"
                + " it already has a parent");
    }
}
```
另外ViewGroup在addView方法中，也会调用assignParent()方法，设定子元素的父容器为它本身。这样一层层向上调用，最上层的ViewParent是ViewRootImpl，所以直接查看 ViewRootImpl 的 invalidateChildInParent() 即可。

## ViewRootImpl # invalidateChildInParent()
```java
@Override
public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
    //检查线程，这也是为什么invalidate一定要在主线程的原因
    checkThread();

    if (dirty == null) {
        invalidate();//有可能需要绘制整个窗口
        return null;
    } else if (dirty.isEmpty() && !mIsAnimating) {
        return null;
    }

    ``````
    invalidateRectOnScreen(dirty);
    return null;
}
```

## ViewRootImpl # invalidateRectOnScreen()
```java
//设置mDirty并执行View的工作流程
private void invalidateRectOnScreen(Rect dirty) {
    final Rect localDirty = mDirty;
    if (!localDirty.isEmpty() && !localDirty.contains(dirty)) {
        mAttachInfo.mSetIgnoreDirtyState = true;
        mAttachInfo.mIgnoreDirtyState = true;
    }

    // Add the new dirty rect to the current one
    localDirty.union(dirty.left, dirty.top, dirty.right, dirty.bottom);    
    //在这里，mDirty的区域就变为方法中的dirty，即要重绘的脏区域

    ``````
    if (!mWillDrawSoon && (intersected || mIsAnimating)) {
        scheduleTraversals();//执行View的工作流程
    }
}
```

到这里，就发现了`执行invalidate()方法居然会引起scheduleTraversals()`，难道说invalidate()会导致perforMeasure()、performLayout()、perforDraw()的调用 ，其实不然，**invalidate()只执行performmDraw()方法**。因为在requestLayout()中有一个 mLayoutRequested 用来表示是否执行 measure和layout流程。

## ViewRootImpl # requestLayout()
```java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();//检查是否在主线程
        mLayoutRequested = true;//mLayoutRequested 是否measure和layout布局。
        scheduleTraversals();
    }
}
```
scheduleTraversals()经过一系列调用， 最终执行到了performTraversals()中
## ViewRootImpl # performTraversals()

```java
private void performTraversals() {
    boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
    if (layoutRequested) {
        measureHierarchy(```);//measure
    }
    if (layoutRequested) {//把mLayoutRequested 设置回去。
      mLayoutRequested = false;
    }
    final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
    if (didLayout) {
        performLayout(lp, mWidth, mHeight);//layout
    }
    boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;
    if (!cancelDraw && !newSurface) {
        performDraw();//draw
    }
}
```
而在invalidate()的时候，只执行了scheduleTraversals(),并没有设置layoutRequested 值，并且在performTraversals()中会把 layoutRequested 设置为false。所以 layoutRequested =false，所以measureHierarchy()不会执行，即performMeasure()不会执行。进而didLayout 变量为false，即performLayout()也不会执行。只执行了performDraw()方法，并且在draw()中会清除mDirty区域,并且只有设置了标识的View 才会调用draw()方法进而调用onDraw()

## View # draw()
```java
private void draw(boolean fullRedrawNeeded) {
...
	final Rect dirty = mDirty;
		if (mSurfaceHolder != null) {
      //清除mDirty区域
			dirty.setEmpty();
			if (animating) {
				if (mScroller != null) {
					mScroller.abortAnimation();
				}
				disposeResizeBuffer();
			}
			return;
		}
...
}
```

整个流程图如下
![Alt text](../../../../images/invalidate.png)


## invalidate()  和 postInvalidate 区别
postInvalidate()（可以在子线程）和invalidate()（在主线程）都用于请求view重绘的方法


# View # requestLayout 流程

## View # requestLayout()
```java
 if (mMeasureCache != null) mMeasureCache.clear();
     ``````
     // 增加PFLAG_FORCE_LAYOUT标记，在measure时会校验此属性
     mPrivateFlags |= PFLAG_FORCE_LAYOUT;
     mPrivateFlags |= PFLAG_INVALIDATED;

     // 父类不为空&&父类没有请求重新布局(是否有PFLAG_FORCE_LAYOUT标志)
        //这样同一个父容器的多个子View同时调用requestLayout()就不会增加开销
     if (mParent != null && !mParent.isLayoutRequested()) {
         mParent.requestLayout();
     }

 }
```
上面说过，最顶层的ViewParent是ViewRootImpl

## ViewRootImpl # requestLayout()
```java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```
同样，requestLayout() 会调用scheduleTraversals（）,因为设置了mLayoutRequested=true
，所以在performTraversals() 中调用performMeasure()，performLayout()，但是由于没有设置mDirty，所以不会走performDraw()流程。

`requestLayout()方法就一定不会导致onDraw()的调用吗？`
在View的layout()方法里，首先执行setFrame()方法

## View # setFrame()
```java
 protected boolean setFrame(int left, int top, int right, int bottom) {
        boolean changed = false;


        if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
            //布局坐标改变了
            changed = true;

            int oldWidth = mRight - mLeft;
            int oldHeight = mBottom - mTop;
            int newWidth = right - left;
            int newHeight = bottom - top;
            boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

            // Invalidate our old position
            invalidate(sizeChanged);//调用invalidate重新绘制视图
            if (sizeChanged) {
                sizeChange(newWidth, newHeight, oldWidth, oldHeight);
            }
            ``````
        }
        return changed;
    }
```
看完代码我们就知道了，如果layout布局有变化，那么也会调用invalidate（）重绘的。
整个流程图如下：
![Alt text](../../../../images/layout.png)


[View 的绘制 --- 概览](http://hoyouly.fun/2018/04/29/view_draw_procress_performTraversals/)   
[View 的绘制 --- Measure 流程](http://hoyouly.fun/2018/04/29/view_draw_procress_measure/)   
[View 的绘制 --- Layout 流程](http://hoyouly.fun/2018/04/29/view_draw_procress_layout/)   
[View 的绘制 --- Draw 流程，invalidate的流程 以及 requestLayout 流程](http://hoyouly.fun/2018/04/29/view_draw_procress_draw/)

---
搬运地址：   
