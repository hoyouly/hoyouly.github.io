---
layout: post
title: View 的绘制流程
category: 扫盲系列
tags: view
---
* content
{:toc}

我们先从ViewRootImpl的setView 方法开始

# ViewRootImpl
ViewRoot对应与ViewRootImpl,他是链接WindowManager和Decorview的纽带，View的三大流程都是通过ViewRoot完成，在ActivityThread中，当Activity对象被创建成功后，会将DecorView添加到Window中，同时创建ViewRootImpl对象，并将ViewRootImpl和DecorView关联起来。
```java
// WindowManagerGlobal.java
//创建ViewRootImpl，并且将view与之绑定
root = new ViewRootImpl(view.getContext(), display);
root.setView(view, wparams, panelParentView);
```

## setView()
```java
public void setView(View view, WindowManager.LayoutParams attrs, View  panelParentView) {
			...
    // Schedule the first layout -before- adding to the window manager,
    //to make sure we do the relayout before receiving any other events from the system.
    requestLayout();
}
```
在setView 中，我们看到了这样一段注释
`Schedule the first layout -before- adding to the window manager, to make sure we do the relayout before receiving any other events from the system.` 大概意思是在添加到WindowManager之前，执行第一个布局，确保在接受 到来自系统的任何其他事件之前进行重新布局，所以我们就相信，requestLayout() 方法是重新布局的

## requestLayout()
```java
public void requestLayout() {
		if (!mHandlingLayoutInLayoutRequest) {
			checkThread();  //// 检查发起布局请求的线程是否为主线程
			mLayoutRequested = true; 、、//mLayoutRequested 是否measure和layout布局。
			scheduleTraversals();
		}
	}
```
mHandlingLayoutInLayoutRequest默认是false， 会在 performLayout 进行修改，

## scheduleTraversals()
Traversal  遍历的意思  ，方法的意思就是执行遍历操作呗
```java
void scheduleTraversals() {
  if (!mTraversalScheduled) {
      mTraversalScheduled = true;
      // handler消息传递绘制请求
      mTraversalBarrier = mHandler.getLooper().postSyncBarrier();
      //post一个runnable处理-->mTraversalRunnable
      //ViewRootImpl中W类是Binder的Native端，用来接收WMS处理操作，因为W类的接收方法是在线程池中的，所以我们可以通过Handler将事件处理切换到主线程中
      mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
      if (!mUnbufferedInputDispatch) {
          scheduleConsumeBatchedInput();
      }
      notifyRendererOfFramePending();
  }
}
```
## ViewRootImpl # TraversalRunnable
```java
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

final class TraversalRunnable implements Runnable {
		@Override
		public void run() {
			doTraversal();
		}
	}
```

## doTraversal()

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
在这里稍微总结一下： **ViewRootImpl在其创建过程中，通过requestLayout向主线程发送了一条触发遍历操作的消息，遍历操作是perforTraversals()方法，这是一个包罗万象的方法，ViewRootImpl中接收到的各种变化，如来自WMS的窗口属性变化，来自控件树的尺寸变化以及重绘都引发performTraversals()的调用，并在其中处理完成，View类及其子类中的onMeasure()、onLayout()、onDraw()等回调也都是在performTraversals()的执行过程中直接或间接的引发。也正是如此，一次次的performTraversals()调用驱动着控件树有条不紊的工作，一旦此方法无法正常执行，整个控件树都将处于僵死状态。<span style="border-bottom:1px solid red;">因此 performTraversals() 函数可以说是ViewRootImpl的心脏。</span>**


## performTraversals()

performTraversals（）首次绘制的大致流程，会依次调用performMeasure，performLayout，performDraw三个方法，这三个方法分别完成顶级VIew的measure，layout，draw这三大流程。
### performMeasure()
调用measure方法，在measure方法中又会调用onMeasure方法，在onMeasure方法中则会对所有的子元素进行measure过程，这个时候measure流程就从父容器传到子元素中了，这样就完成了一次measure过程。接着子View又会重复父容器的操作，如此往复，就完成了View的遍历。measure 决定了View的宽和高，Measure完成以后，可以通过getMeasuredWidth和getMeasureHeight方法来获取到View测量后的宽高。
### performLayout()
和performMeasure同理。Layout过程决定了View的四个顶点的坐标和实际View的宽高，完成以后，可以通过getTop/Bottom/Left/Right拿到View的四个顶点位置，并可以通过getWidth和getHeight方法来拿到View的最终宽高。
### performDraw()
和performMeasure同理，唯一不同的是，performDraw的传递过程是在draw方法中通过dispatchDraw来实现的。Draw过程则决定了View的显示，只有draw方法完成以后View的内容才能呈现在屏幕上。


![performTraversals()流程图](http://p5sfwb51p.bkt.clouddn.com/performTraversals.png)

```java
private void performTraversals() {
        final View host = mView;
        int desiredWindowWidth;//decorView宽度
        int desiredWindowHeight;//decorView高度
        if (mFirst) {//为true的情况就是第一个添加view的时候，也就是创建ViewRootImpl对象的时候
            if (shouldUseDisplaySize(lp)) {
                //窗口的类型中有状态栏和，所以高度需要减去状态栏
                //获取屏幕的分辨率
                Point size = new Point();
                mDisplay.getRealSize(size);
                desiredWindowWidth = size.x;
                desiredWindowHeight = size.y;
            } else {
                //窗口的宽高即整个屏幕的宽高
                Configuration config = mContext.getResources().getConfiguration();
                desiredWindowWidth = dipToPx(config.screenWidthDp);
                desiredWindowHeight = dipToPx(config.screenHeightDp);
            }
            //在onCreate中view.post(runnable)和此方法有关
            host.dispatchAttachedToWindow(mAttachInfo, 0);
        }
        boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
        if (layoutRequested) {
           ...
            //创建了DecorView的MeasureSpec，并调用performMeasure
             measureHierarchy(host, lp, res,desiredWindowWidth, desiredWindowHeight);
        }
```
在measureHierarchy()方法中，创建DecorView的MeasureSpec   
这里涉及到关于MeasureSpce的问题   
## MeasureSpec
很大程度上决定了View的尺寸规格，之所以很大程度上，是因为这个过程还受到父容器的影响，因为父容器影响了View的MeasureSpec的创建过程，在测量过程中，系统会将View的LayoutParams,根据父容器的所施加的规则转换成对应的MeasureSpce，然后再根据这个MeasureSpce测量出View的宽高。
MeasureSpce代表一个32位的int值，高2位代表SpecMode,即测量模式，低30位代表SpecSize，即某种测量模式下的规格大小。
MeasureSpec通过打包把SpecMode和SpecSize组成一个int值从而避免过多的对象内存分配，同时提供打包和解包的方法，从而得到原始的SpecMode和SpecSize

### SpecMode
* UNSPECIFIED   父容器不对子View做任何限制，要多大给多大，一般用于系统内部，标示一种测量状态
* EXACTLY  父容器已经检测出子view所需要的大小，这个时候View的最终大小就是specSize所指的值，对应于LayoutParams中的match_parent和具体的数值这两种模式
* AT_MOST  父容器指定了一个可用大小即SpecSize,View 大小不能大于这个值，具体是什么值要看不同View的具体实现，它对应了Layoutparams中的warp_content

## MeasureSpce 与LayoutParams 关系
MeasureSpce 不是唯一由LayoutParams决定的，LayoutParams 需要和父容器一起才能决定View的MeasureSpce,从而进一步决定View的宽高。对应顶级View(DecorView)和普通的View，MeasureSpce的转换略有不同，
* DecorView  其MeasureSpce由窗口的尺寸和自身的LayoutParams共同决定
* 普通View   其MeasureSpce由父容器的MeasureSpce和自身的LayoutParams共同决定。   

MeasureSpce一旦确定，onMeasure()中就可以得到View的测量宽和高


## measureHierarchy()
Hierarchy  层级，阶层，等级制度

```java
private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp, final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
		int childWidthMeasureSpec;
		int childHeightMeasureSpec;
		boolean windowSizeMayChange = false;
		boolean goodMeasure = false;
		if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT) {
			//在大屏幕上，我们不希望允许对话框拉伸以填充整个屏幕宽度以显示一行文本。 首先尝试以较小的尺寸进行布局，看看是否适合。
			final DisplayMetrics packageMetrics = res.getDisplayMetrics();
			res.getValue(com.android.internal.R.dimen.config_prefDialogWidth, mTmpValue, true);
			int baseSize = 0;
			if (mTmpValue.type == TypedValue.TYPE_DIMENSION) {
				baseSize = (int) mTmpValue.getDimension(packageMetrics);
			}
			if (baseSize != 0 && desiredWindowWidth > baseSize) {
				childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
				childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
				performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
				if ((host.getMeasuredWidthAndState() & View.MEASURED_STATE_TOO_SMALL) == 0) {
					goodMeasure = true;
				} else {
					// Didn't fit in that size... try expanding a bit.
					baseSize = (baseSize + desiredWindowWidth) / 2;
					childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
					performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
					if ((host.getMeasuredWidthAndState() & View.MEASURED_STATE_TOO_SMALL) == 0) {
						goodMeasure = true;
					}
				}
			}
		}
		//goodMeasure 这个变量为true的时候，`(host.getMeasuredWidthAndState() & View.MEASURED_STATE_TOO_SMALL) == 0` 而刚开始的时候，也就是计算DecorView的时候，是不等于0的，所以第一次执行 measureHierarchy方法的时候，goodMeasure为false，也就是执行到了下面的if语句
		if (!goodMeasure) {//DecorView,宽度基本都为match_parent
			//创建measureSpec
			childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
			childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
			performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
			if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
				windowSizeMayChange = true;
			}
		}
		return windowSizeMayChange;
	}
```

measureHierarchy()用于测量整个控件树，传入的参数desireWindowWidth和desiredWindowHeight在前面方法中根据当前窗口的不同情况挑选而出，（desired 期望，愿望），不过**MeasureHierarchy()有自己的测量方法，让窗口更加优雅（主要是针对wrap_content的Dialog），所以设置了wrap_content的Dialog，有可能执行多次测量**

### DecorView 的MeasureSpce 创建工程

我们在measureHierarchy()看到这样一段代码，这是展示了DecorView 的MeasureSpce的创建过程。其中desireWindowWidth和desiredWindowHeight就是屏幕尺寸大小。

```java
childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
```
接下来看getRootMeasureSpec的代码

## getRootMeasureSpec()

```java
	private static int getRootMeasureSpec(int windowSize, int rootDimension) {
		int measureSpec;
		switch (rootDimension) {
			case ViewGroup.LayoutParams.MATCH_PARENT:
				// Window can't resize. Force root view to be windowSize.
				measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
				break;
			case ViewGroup.LayoutParams.WRAP_CONTENT:
				// Window can resize. Set max size for root view.
				measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
				break;
			default:
				// Window wants to be an exact size. Force root view to be that size.
				measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
				break;
		}
		return measureSpec;
	}
```
通过上述代码，DecorView的MeasureSpec的产生过程就很明确了，具体来说就是遵守如下规则。根据layouParams的宽高的参数来划分，
* LayoutParams.MATCH_PARENT   :  EXACTLY  精确模式，大小就是窗口的大小
* LayoutParams.WARP_CONTENT  : AT_MOST (最大模式)  大小不确定，但是不能 超过窗口的大小
* 固定大小（写死的值），EXACTLY（精确模式），大小就是当前写死的数值

计算出来MeasureSpce之后，就执行performMeasure()方法，

## performMeasure()

```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
		mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
	}
```
改方法很简单，就是直接调用mView.measure()方法，而这个mView就是执行setView方法中，传递过来的View，

# View
measure过程分为两种，一种是只有一个原始的View，那么通过measure()方法就可以完成了，还有一种就是ViewGroup,除了完成自己是测量过程，还要遍历调用子元素measure过程。各个子元素在递归调用这个流程，针对这两种情况分别讨论
## View的measure过程
又其的measure()方法完成。代码如下

```java
		public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
		boolean optical = isLayoutModeOptical(this);
		if (optical != isLayoutModeOptical(mParent)) {
			Insets insets = getOpticalInsets();
			int oWidth = insets.left + insets.right;
			int oHeight = insets.top + insets.bottom;
			widthMeasureSpec = MeasureSpec.adjust(widthMeasureSpec, optical ? -oWidth : oWidth);
			heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
		}
		// Suppress sign extension for the low bytes
		long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
		if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);

		final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
		final boolean isExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY && MeasureSpec.getMode(mOldWidthMeasureSpec) == MeasureSpec.EXACTLY && MeasureSpec.getMode(mOldHeightMeasureSpec) == MeasureSpec.EXACTLY;
		final boolean matchingSize = isExactly && getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec) && getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec);
		//仅当给与的MeasureSpec发生变化时，或要求强制重新布局时，才会进行测量。
		if (forceLayout || !matchingSize && (widthMeasureSpec != mOldWidthMeasureSpec || heightMeasureSpec != mOldHeightMeasureSpec)) {

			// first clears the measured dimension flag
			mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

			resolveRtlPropertiesIfNeeded();

			int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
			if (cacheIndex < 0 || sIgnoreMeasureCache) {
				// measure ourselves, this should set the measured dimension flag back
				/// M: Monitor onMeasue time if longer than 3s print log.
				long logTime = System.currentTimeMillis();
				onMeasure(widthMeasureSpec, heightMeasureSpec);
				long nowTime = System.currentTimeMillis();
				if (nowTime - logTime > DBG_TIMEOUT_VALUE) {
					Xlog.d(VIEW_LOG_TAG, "[ANR Warning]onMeasure time too long, this =" + this + "time =" + (nowTime - logTime) + " ms");
				}
				mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
			} else {
				long value = mMeasureCache.valueAt(cacheIndex);
				// Casting a long to int drops the high 32 bits, no mask needed
				setMeasuredDimensionRaw((int) (value >> 32), (int) value);
				mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
			}

			// flag not set, setMeasuredDimension() was not invoked, we raise
			// an exception to warn the developer
			if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
				throw new IllegalStateException("onMeasure() did not set the" + " measured dimension by calling" + " setMeasuredDimension()");
			}

			mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
		} else {
		}

		mOldWidthMeasureSpec = widthMeasureSpec;
		mOldHeightMeasureSpec = heightMeasureSpec;

		mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 | (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
	}
```
由代码可知：  
* 改方法定义的是final类型的，子类不能重写该方法
* 仅当给与的MeasureSpec发生变化时，或要求强制重新布局时，才会进行测量。
强制重新布局解决途径，当子控件内容发生变化时，从子控件到父控件回溯到ViewRootImpl，并依次调用父控件的requestLayout()方法，这个方法会在mPrivateFlage中加入标记PFLAG_FORCE_LAYOUT,从而是这些父控件的measure()方法得到顺利执行，进而这个子控件有机会进行重新布局与测量，这便是强制重新布局的意义所在。
* view.measure()方法其实没有实现任何测量的算法，它的作用在于判断是否需要引发onMeasure()的调用，并对onMeasure()行为的正确性进行检查。

接下来看onMeasure()方法，这个方法我们就比较熟悉了，起码经常会说道，自定义控件的时候经常会被复写的三个方法之一。
## onMeasure()

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
		setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec), getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
	}
```
代码很简单，通过setMeasureDimension()设置View的宽和高，那么我们就首先看看宽和高是怎么得到的，这就需要查看getDefaultSize()方法了

## getDefaultSize()
```java
	public static int getDefaultSize(int size, int measureSpec) {
		int result = size;
		int specMode = MeasureSpec.getMode(measureSpec);
		int specSize = MeasureSpec.getSize(measureSpec);
		switch (specMode) {
			case MeasureSpec.UNSPECIFIED:
				result = size;
				break;
			case MeasureSpec.AT_MOST:
			case MeasureSpec.EXACTLY:
				result = specSize;
				break;
		}
		return result;
	}
```
从getDefaultSize()方法的实现来看，分两种情况：
* 对于AT_MOST和EXACTLY这两种情况，View的高度是由specSize决定的，简单来说，就是getDefautlSize()返回的就是MeasureSpce对应的specSize,也就是说**如果我们直接继承View的自定义控件，需要重写onMeasure方法并设置wrap_content时自身大小，否则布局中使用wrap_content就相当于使用match_parent** ，
* 至于 UNSPECIFIED情况，一般用于系统内部测量，这种情况，View的大小就是getDefaultSize()的第一个参数，即getSuggestedMinimumWidth()/getSuggestedMinimumHeight()

## getSuggestedMinimumWidth()
```java
protected int getSuggestedMinimumWidth() {
		//如果没有设置背景那么View宽度就是mMinWidth，
		//mBackground.getMinimumWidth() 得到的就是Drawble的原始宽度
		return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
	}
```
getSuggestedMinimumWidth的逻辑就是： 如果没有设置背景，那么View的宽度就是mMinWidth,而mMinWidth 对应的就是Android：minWidth,如果不指定这个属性，默认为0;如果设置了背景，那么得到的就是mMindth和背景宽度的最大哪一个。


### 解决自定义View设置match_parent和wrap_content效果一致的问题

View使用了wrap_content ,那么他的MeasureSpec就是AT_MOST,在这种模式下，他的宽高等于specSize,而上面图中我们可知，View的specSize就是parentSize,这种情况下，和match_parent效果一样，那么怎么解决呢，就是给View设定一个默认的的内部宽高（mWith，mheight）,并且在wrap_content时候设置进去就可以了

## ViwGroup 的measure过程

因为ViewGroup是一个抽象类，并没有重写onMeaure(),要具体实现去实现该方法，
但是它提供了一个叫measureChildren()的方法

### measureChildren()
```java
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
			 final int size = mChildrenCount;
			 final View[] children = mChildren;
			 for (int i = 0; i < size; ++i) {
					 final View child = children[i];
					 if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
							 measureChild(child, widthMeasureSpec, heightMeasureSpec);
					 }
			 }
	 }
```
从代码上看，ViewGroup在measure时，会对每一个子View进行measure,measureChild()这个方法也很好理解的

### measureChild()
```java
protected void measureChild(View child, int parentWidthMeasureSpec, int parentHeightMeasureSpec) {
			 //取出来子元素的LayoutParams
			 final LayoutParams lp = child.getLayoutParams();
			 // 通过getChildMeasureSpec（）得到创建子元素的MeasureSpec
			 final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec, mPaddingLeft + mPaddingRight, lp.width);
			 final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec, mPaddingTop + mPaddingBottom, lp.height);
			 //将MeasureSpec传递到子view中的measure（）进行测量
			 child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
	 }
```
代码里面都带有注释，很好理解，就不多说了，getChildMeasureSpec()方法参照上面的表格就可以了  
ViewGroup并没有定义测量的具体过程，其过程需要子类自己具体实现，因为不同的ViewGroup ,布局方式不同，测量细节也就不同的，比如LinearLayout，RelativeLayout
下面通过LinearLayout的onMeasure()具体分析。

## LinearLayout  #onMeasure()
```java
@Override
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
		if (mOrientation == VERTICAL) {
			measureVertical(widthMeasureSpec, heightMeasureSpec);
		} else {
			measureHorizontal(widthMeasureSpec, heightMeasureSpec);
		}
	}
```
代码很简单，比如选择竖直方向的LinearLayout的测量过程。即measureVertical()，源码比较长，分看来看。
```java
// See how tall everyone is. Also remember max width.
for (int i = 0; i < count; ++i) {
	final View child = getVirtualChildAt(i);
		...
		//遍历每个子元素，并对每个子元素执行measureChildBeforeLayout（），这个方法内部调用measure()
		measureChildBeforeLayout(child, i, widthMeasureSpec, 0, heightMeasureSpec, totalWeight == 0 ? mTotalLength : 0);

		if (oldHeight != Integer.MIN_VALUE) {
			lp.height = oldHeight;
		}

		final int childHeight = child.getMeasuredHeight();
		final int totalLength = mTotalLength;
		// 存储LinearLayout竖直方向的高度
		mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin + lp.bottomMargin + getNextLocationOffset(child));

		if (useLargestChild) {
			largestChildHeight = Math.max(childHeight, largestChildHeight);
		}
	}
```
从代码看，系统会遍历子元素，并对每个子元素执行measureChildBeforeLayout()，在该方法内部，会执行View的measure()方法，这样子view就依次开始执行measure过程。
并且系统会通过mTotalLength来存储系在竖直方向的高度。每测量一个，mTotalLength就会增加。增加的部分主要包括子View的高度以及子元素在竖直方向的margin，当子元素测量完后，LinearLayout开始测量自己的大小。
```java
// Add in our padding
		mTotalLength += mPaddingTop + mPaddingBottom;

		int heightSize = mTotalLength;

		// Check against our minimum height
		heightSize = Math.max(heightSize, getSuggestedMinimumHeight());

		// Reconcile our calculated size with the heightMeasureSpec
		int heightSizeAndState = resolveSizeAndState(heightSize, heightMeasureSpec, 0);
		heightSize = heightSizeAndState & MEASURED_SIZE_MASK;

		...

		setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState), heightSizeAndState);
		if (matchWidth) {
			forceUniformWidth(count, heightMeasureSpec);
		}

```

### Activity 一启动得到View的宽高方式
1. onWindowFocusChanged()  VIew 已经初始化完毕，会被调用多次，当Activity的窗口得到焦点和失去焦点的时候，都会执行一次，如果频繁的进行onResume()和onPause()那么onWindowFocusChanged()也会被循环调用
2. View.post(Runnable) 通过post把一个Runnable对象加到添加队列中，然后等待Looper调用次Runnable的时候，View已经初始化好了，
3. ViewTreeObserver  使用ViewTreeObserver的众多回调可以完成此功能l，比如使用OnGlobalFocusChangeListener 这个借口，当View的状态树的状态发生改变或者View树内部的可见性发送改变，onGlobalFocusChanged（）这个方法被回调，但是需要注意，伴随View状态树改变，onGlobalFocusChanged（）会被调用多次。
4. 手动对View进行measure的宽和高。这种情况比较复杂，根据View的LayoutParams来分


因为DecorView是一个Framelayout，所以我们从FrameLayout开始我们的主线任务measureChildBeforeLayout()

# Framelayout
## onMeasure()
```java
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
		int count = getChildCount();
		final boolean measureMatchParentChildren = MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY || MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
		mMatchParentChildren.clear();

		int maxHeight = 0;
		int maxWidth = 0;
		int childState = 0;
		//遍历子View，只要View不是GONE，便处理
		for (int i = 0; i < count; i++) {
			final View child = getChildAt(i);
			if (mMeasureAllChildren || child.getVisibility() != GONE) {
			//子View结合父View的MeasureSpec和自己的LayoutParams算出子View自己的MeasureSpec
				measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
				final LayoutParams lp = (LayoutParams) child.getLayoutParams();
				maxWidth = Math.max(maxWidth, child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
				maxHeight = Math.max(maxHeight, child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
				childState = combineMeasuredStates(childState, child.getMeasuredState());
				if (measureMatchParentChildren) {
					if (lp.width == LayoutParams.MATCH_PARENT || lp.height == LayoutParams.MATCH_PARENT) {
						mMatchParentChildren.add(child);
					}
				}
			}
		}
		.....
	}
```
FrameLayout 中的onMeaure()主要做两件事
1. 遍历子View，只要View不是GONE，便会处理。
2. 每个处理的子View会结合父View的MeasureSpec和自己的LayoutParams 计算出自己的MeasureSpce
3.

# ViewGroup
## measureChildWithMargins()

```java
	protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed, int parentHeightMeasureSpec, int heightUsed) {
		final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
		//子View结合父View的MeasureSpec和自己的LayoutParams算出子View自己的MeasureSpec
		final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec, mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin + widthUsed, lp.width);
		final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec, mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin + heightUsed, lp.height);

		//如果当前child也是ViewGroup，执行measure方法，也就在相应的onMeasure中也继续遍历它的子View,
		//如果当前child是View，便根据这个MeasureSpec测量自己
		child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
	}
```

上诉方法会对子元素进行Measure，在调用子元素的Measure方法之前，会通过getChildMeasureSpec()方法来得到子元素的MeasureSpec，该方法主要是根据父容器MeasureSpec同时结合View本身的LayoutParams来确定子元素的MeasureSpec,显然子元素的MeasureSpec的创建与父容器的MeasureSpec和子元素本身的LayoutParams有关，此外还和View的margin和padding有关，
<font color="ff000">
子View的MeasureSpec=LayoutParams+margin+padding+父容器的MeasureSpec</font>

## getChildMeasureSpec()
```java
//spec为父容器的MeasureSpec
    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);//父容器的specMode
        int specSize = MeasureSpec.getSize(spec);//父容器的specSize
        int size = Math.max(0, specSize - padding);
        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {//根据父容器的specMode
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        //noinspection ResourceType
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```
代码换成表格形式就是如下。
![](http://p5sfwb51p.bkt.clouddn.com/view_measurespce_parent_measurespce.png)
1. 如果子布局的LayoutParams是具体的值，那么子布局的MeasureSpce是EXACTLY，子布局的可用大小是自己的LayoutParams设置的的大小
2. 如果子布局设置的match_parent,那么子布局的MeasureSpce是和父布局的一样，大小就是父布局可用的大小，UNSPECIFIED除外，这种情况下为0
3. 如果子布局设置的是warp_conent,那么子布局的MeasureSpce就是AT_MOST,可用大小就是父布局最大的可用大小，UNSPECIFIED除外，这种情况下还是UNSPECIFIED，并且可用大小为0


![Alt text](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCE751f50bcba2207aab617f8ca29d9083e/2444)


# View

# Layout 流程
Layout 作用就是ViewGroup用来确定子元素的位置。当ViewGroup的位置确定后，他会在onLayout中遍历所有子元素并调用其layout方法，在layout方法中onLayout()方法


子View具体的layout的位置都是相对于父容器而言的，view的layout过程同Measure同理，也是从顶级View开始，递归的完成整个控件树的布局操作

经过前面的测量，控件树中的控件对于自己的尺寸显然已经了然于胸。而且父控件对于子控件的位置也有了眉目，**所以经过测量过程后，布局阶段会把测量结果转化为控件的实际位置与尺寸。控件的实际位置与尺寸由View的mLeft，mTop，mRight，mBottom 等4个成员变量存储的坐标值来表示。**

并且需要注意的是： **View的mLeft，mTop，mRight，mBottom 这些坐标值是以父控件左上角为坐标原点进行计算的。倘若需要获取控件在窗口坐标系中的位置可以使用View.GetLocationWindow()或者是View.getRawX()/Y()。**

# ViewRootImpl

## performLayout
```java
  private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,int desiredWindowHeight) {
		  ...
        final View host = mView;
        host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
		...
    }
```

# ViewGroup
尽管ViewGroup也重写了layout方法，但是本质上还是通过super.layout,调用View的layout方法
## layout()
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

# View
## layout()
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
1. 通过setFrame()将l,t,r,b分别设置到mLeft，mTop，mRight,和mButton，这样就可以确定子View在父容器上的位置了，也就是说这四个位置是相当于父容器的
2. 调用onLayout方法，具体实现类接收到布局变更通知，如果此类是ViewGoup，还会遍历子View的layout方法，使其更新布局，如果是调用onLayout方法，这会导致子View无法调用setFrame(),从而无法更新控件坐标信息

## onLayout
```java
    protected void onLayout(boolean changed, int l, int t, int r, int b) {}
```
对于普通View，onlayout方法是一个空实现，主要是具体实现类重写该方法后能接受到布局坐标更新信息

# ViewGroup
## onLayout
```java
    protected abstract void onLayout(boolean changed,int l, int t, int r, int b);
```
对于ViewGroup来说，和measure一样，不同的类有它不同的布局特性，在ViewGroup中onLayout方法中是abstract的，具体类必须重写该方法，以便接收布局坐标更新信息后，处理自己的子View的坐标信息，可以看一下 LinearLayout的onLayout()

# LinearLayout
## onLayout
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
这里面和measure（）类似，这里选择layoutVertical()讲解
## layoutVertical()

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
```
* 遍历所有子元素并调用setChildFrame()来为子元素指定位置，setChildFrame()内部调用子元素的layout方法，
* childTop位置会逐渐增大，这就意味着后面的元素会放在靠下的位置，符合竖直方向的LinearLayout的特点，
这样父元素在layout完成自己的定位后，然后通过onLayout()调用子元素的layout方法，子元素又会通过layout方法定位自己的位置。这样一层层传递下来就完成了整个View树的layout过程。

## 小结
* measure确定的是控件的尺寸，并在一定程度上确定了子控件的位置。而布局则是针对测量结果来实施，并最终确定子控件的位置。
* measure结果对布局过程没有约束力。虽说子控件在onMeasure()方法中计算出了自己应有的尺寸，但是由于layout()方法是由父控件调用，因此控件的位置尺寸的最终决定权掌握在父控件手中，测量结果仅仅只是一个参考。
* 因为measure过程是后根遍历(DecorView最后setMeasureDiemension())，所以子控件的测量结果影响父控件的测量结果。
* 而Layout过程是先根遍历(layout()一开始就调用setFrame()完成DecorView的布局)，所以父控件的布局结果会影响子控件的布局结果。
* 完成performLayout()后，空间树的所有控件都已经确定了其最终位置，就剩下绘制了。

# draw的流程
# View
View的draw过程遵循如下几步

* 绘制背景    drawBackground();
* 绘制自己    onDraw();
* 如果是ViewGroup则绘制子View，  dispatchDraw();
* 绘制装饰（滚动条）和前景，     onDrawForeground();

View绘制过程的传递是通过dispatchDraw()来实现的，dispatchDraw()会遍历调用所有子元素的的draw(),如此draw事件就一层层传递下来，

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

![Alt text](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCE1fcf8b5683a7ecc0a4fb71a4348d2411/2440)


# invalidate   postInvalidate
postInvalidate ()（可以在子线程）和invalidate()（在主线程）都用于请求view重绘的方法
他们是如何实现的呢

invalidate()方法必须在主线程中执行，而scheduleTraversals（）引发的遍历也是在主线程中自画像的，所以调用invilidate（）方法并不会使得遍历立即开始，因为在调用invalidate（）的方法执行完毕之前（准确说是主线程的Looper处理完其他消息之前），主线程根本没机会处理scheduleTraversals所发出的消息，这种机制带来的好处就是 **在一个方法里面可以连续调用多个控件的invalidate方法，而不用担心会由于多次重绘而产生的效率问题**
另外多次调用invalidate()方法会使得ViewRootImpl多次接收到设置脏区域的请求，ViewRootImpl会将这些脏区域累加到mDirty中，进而在随后的遍历中，一次性的完成所有脏区域的重绘。


窗口第一次绘制时候，ViewRootImpl的mFullRedrawNeeded成员将会被设置为true，也就是说mDirty所描述的区域将会扩大到整个窗口，进而实现完整重绘。

## View的脏区域和实心控件

为了保证绘制效率，控件树仅对需要重绘的区域进行绘制，这部分区域成为**脏区域（Dirty Area）**
当一个控件的内容发生变化而需要重绘时，它会通过View.invalidate()方法将其需要重绘的区域沿着控件树自下而上的交给ViewRootImpl，并保存在ViewRootImpl的mDirty成员中，最后通过scheduleTraversals()引发一次遍历，进而进行重绘工作，这样就可以保证仅位于mDirty所描述的区域得到重绘，避免了不必要的开销。

View的isOpaque()方法返回值表示此控件是否为”实心”的，**所谓”实心”控件，是指在onDraw()方法中能够保证此控件的所有区域都会被其所绘制的内容完全覆盖**。对于”实心”控件来说，背景和子元素（如果有的话）是被其onDraw()的内容完全遮住的，因此便可跳过遮挡内容的绘制工作从而提升效率。

**简单来说透过此控件所属的区域无法看到此控件下的内容**，**也就是既没有半透明也没有空缺的部分** 因为自定义ViewGroup控件默认是”实心”控件，所以默认不会调用drawBackground()和onDraw()方法，因为一旦ViewGroup的onDraw()方法，那么就会覆盖住它的子元素。但是我们仍然可以通过调用setWillNotDraw(false)和setBackground()方法来开启ViewGroup的onDraw()功能。

## invalidate()
```java
 public void invalidate() {
        invalidate(true);
    }

    void invalidate(boolean invalidateCache) {
        invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
    }

```
## invalidateInternal()
```java
void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
            boolean fullInvalidate) {

        //如果VIew不可见，或者在动画中
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
上述代码中，会设置一系列的标记位到mPrivateFlags中，并且通过父容器的invalidateChild方法，将需要重绘的脏区域传给父容器。（ViewGroup和ViewRootImpl都继承了ViewParent类，该类中定义了子元素与父容器间的调用规范。）

# ViewGroup

## invalidateChild()
```java
 public final void invalidateChild(View child, final Rect dirty) {
        ViewParent parent = this;

        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null) {
                RectF boundingRect = attachInfo.mTmpTransformRect;
                boundingRect.set(dirty);
                ``````  
               //父容器根据自身对子View的脏区域进行调整
               #
                transformMatrix.mapRect(boundingRect);
                dirty.set((int) Math.floor(boundingRect.left),
                        (int) Math.floor(boundingRect.top),
                        (int) Math.ceil(boundingRect.right),
                        (int) Math.ceil(boundingRect.bottom));

            // 这里的do while方法，不断的去调用父类的invalidateChildInParent方法来传递重绘请求
            //直到调用到ViewRootImpl的invalidateChildInParent（责任链模式）
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
                // If the parent is dirty opaque or not dirty, mark it dirty with the opaque
                // flag coming from the child that initiated the invalidate
                if (view != null) {
                    if ((view.mViewFlags & FADING_EDGE_MASK) != 0 &&
                            view.getSolidColor() == 0) {
                        opaqueFlag = PFLAG_DIRTY;
                    }
                    if ((view.mPrivateFlags & PFLAG_DIRTY_MASK) != PFLAG_DIRTY) {
                        view.mPrivateFlags = (view.mPrivateFlags & ~PFLAG_DIRTY_MASK) | opaqueFlag;
                    }
                }

                //***往上递归调用父类的invalidateChildInParent***
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
# ViewRootImpl
验证最上层ViewParents为啥是ViewRootImpl

## setView()
```java

 public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
               view.assignParent(this);
    }

```
# view
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
在ViewRootImpl的setView方法中，由于传入的View正是DecorView，所以最顶层的ViewParent即ViewRootImpl。另外ViewGroup在addView方法中，也会调用assignParent()方法，设定子元素的父容器为它本身。
由于最上层的ViewParent是ViewRootImpl，所以我们可以查看ViewRootImpl的invalidateChildInParent方法即可。

# ViewRootImpl
## invalidateChildInParent（）

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

## invalidateRectOnScreen()
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

到这里，就发现了`执行invalidate()方法居然会引起scheduleTraversals()`，也就是说也就是说invalidate()会导致perforMeasure()、performLayout()、perforDraw()的调用 ，其实不是，**invalidate只执行performmDraw()方法**
因在在requestLayout()中有一个 mLayoutRequested用来表示是否measure和layout。

## requestLayout()
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
## performTraversals()

```java
    private void performTraversals() {
        boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
        if (layoutRequested) {
            measureHierarchy(```);//measure
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
由于在invalidate的时候，并没有设置layoutRequested 值，所以layoutRequested =false，所以measureHierarchy（）不会执行，即performMeasure()不会执行。进而didLayout 变量为false，即performLayout（）也不会执行。只执行了performDraw（）方法，并且在draw（）中会清除mDirty区域,并且只有设置了标识的View才会调用draw方法进而调用onDraw()

## draw()
```java
private void draw(boolean fullRedrawNeeded) {
...
	final Rect dirty = mDirty;
		if (mSurfaceHolder != null) {
			// The app owns the surface, we won't draw.
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
![Alt text](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCEcd6b171ef0b34c2358c8fcc46e9dfcb6/2455)


# requestLayout 流程
# View

## requestLayout()
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
上面说过，最顶层的ViewParent是ViewRootImpl，
# ViewRootImpl

## requestLayout（）
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
同样，requestlayout（）会调用scheduleTraversals（）,因为设置了mLayoutRequested=true
，所以在performTraversals（）中调用performMeasure()，performLayout()，但是由于没有设置mDirty，所以不会走performDraw()流程。

`requestLayout()方法就一定不会导致onDraw()的调用吗？`
在View的layout()方法里，首先执行setFrame()方法
# View

## setFrame()
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
看完代码我们就知道了，如果layout布局有变化，那么也会调用invalidate（）重绘的
![Alt text](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCE7cb8f85cb3b309e5996880a2fc0661e5/2442）

---
搬运地址：   
