---
layout: post
title: View 的绘制流程 --- Measure 过程
category: 读书笔记
tags: view  Android开发艺术探索
---
* content
{:toc}

在说Measure 过程之前，需要解释几个概念。MeasureSpec 和 SpecMode

### MeasureSpec
很大程度上决定了View的尺寸规格，之所以很大程度上，是因为这个过程还受到父容器的影响，因为父容器影响了View的MeasureSpec的创建过程，在测量过程中，系统会将View的LayoutParams,根据父容器的所施加的规则转换成对应的MeasureSpce，然后再根据这个MeasureSpce测量出View的宽高。
MeasureSpce代表一个32位的int值，高2位代表SpecMode,即测量模式，低30位代表SpecSize，即某种测量模式下的规格大小。
MeasureSpec通过打包把SpecMode和SpecSize组成一个int值从而避免过多的对象内存分配，同时提供打包和解包的方法，从而得到原始的SpecMode和SpecSize

### SpecMode
* UNSPECIFIED   父容器不对子View做任何限制，要多大给多大，一般用于系统内部，表示一种测量状态
* EXACTLY  父容器已经检测出子view所需要的大小，这个时候View的最终大小就是specSize所指的值，对应于LayoutParams中的match_parent和具体的数值这两种模式
* AT_MOST  父容器指定了一个可用大小即SpecSize,View 大小不能大于这个值，具体是什么值要看不同View的具体实现，它对应了Layoutparams中的warp_content

### MeasureSpce 与LayoutParams 关系
MeasureSpce 不是唯一由LayoutParams决定的，LayoutParams 需要和父容器一起才能决定View的MeasureSpce,从而进一步决定View的宽高。对应顶级View(DecorView)和普通的View，MeasureSpce的转换略有不同，
* DecorView  其MeasureSpce由窗口的尺寸和自身的LayoutParams共同决定
* 普通View   其MeasureSpce由父容器的MeasureSpce和自身的LayoutParams共同决定。   

**MeasureSpce一旦确定，onMeasure()中就可以得到View的测量宽和高**

上一篇我们知道。在performTraversals()会执行 measureHierarchy()。我们就从此继续分析
## measureHierarchy()
Hierarchy  层级，阶层，等级制度

```java
private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp, final Resources res,
														final int desiredWindowWidth, final int desiredWindowHeight) {
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
		//goodMeasure 这个变量为true的时候，`(host.getMeasuredWidthAndState() & View.MEASURED_STATE_TOO_SMALL) == 0` 而刚开始的时候，
		//也就是计算DecorView的时候，是不等于0的，所以第一次执行 measureHierarchy方法的时候，goodMeasure为false，也就是执行到了下面的if语句
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

### DecorView 的 MeasureSpce 创建工程

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
该方法很简单，就是直接调用mView.measure()方法，而这个mView就是执行setView方法中，传递过来的View。

measure过程分为两种，一种是只有一个原始的View，那么通过measure()方法就可以完成了，还有一种就是ViewGroup,除了完成自己是测量过程，还要遍历调用子元素measure过程。各个子元素在递归调用这个流程，针对这两种情况分别讨论
## View的measure过程
measure()方法完成。代码如下，只留下关键代码。

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
	。。。
	final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
	final boolean isExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY
									&& MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY
									&& MeasureSpec.getMode(mOldWidthMeasureSpec) == MeasureSpec.EXACTLY
									&& MeasureSpec.getMode(mOldHeightMeasureSpec) == MeasureSpec.EXACTLY;
	final boolean matchingSize = isExactly && getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec)
																			&& getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec);
		//仅当给与的MeasureSpec发生变化时，或要求强制重新布局时，才会进行测量。
	if (forceLayout|| !matchingSize && (widthMeasureSpec !=mOldWidthMeasureSpec
										|| heightMeasureSpec != mOldHeightMeasureSpec)) {
		。。。
		if (cacheIndex < 0 || sIgnoreMeasureCache) {
			long logTime = System.currentTimeMillis();
			onMeasure(widthMeasureSpec, heightMeasureSpec);
			long nowTime = System.currentTimeMillis();
			mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
		} else {
			long value = mMeasureCache.valueAt(cacheIndex);
			setMeasuredDimensionRaw((int) (value >> 32), (int) value);
			mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
		}
		。。。
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
	setMeasuredDimension(
			getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
			getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
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
	return (mBackground == null) ? mMinWidth : max(mMinWidth,mBackground.getMinimumWidth());
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
setMeasuredDimension(resolveSizeAndState(maxWidthwidthMeasureSpec, childState), heightSizeAndState);
if (matchWidth) {
	forceUniformWidth(count, heightMeasureSpec);
}

```

### Activity 一启动得到View的宽高方式
1. onWindowFocusChanged()  View 已经初始化完毕，会被调用多次，当Activity的窗口得到焦点和失去焦点的时候，都会执行一次，如果频繁的进行onResume()和onPause()那么onWindowFocusChanged()也会被循环调用
2. View.post(Runnable) 通过post把一个Runnable对象加到添加队列中，然后等待Looper调用次Runnable的时候，View已经初始化好了，
3. ViewTreeObserver  使用ViewTreeObserver的众多回调可以完成此功能l，比如使用OnGlobalFocusChangeListener 这个借口，当View的状态树的状态发生改变或者View树内部的可见性发送改变，onGlobalFocusChanged（）这个方法被回调，但是需要注意，伴随View状态树改变，onGlobalFocusChanged（）会被调用多次。
4. 手动对View进行measure的宽和高。这种情况比较复杂，根据View的LayoutParams来分

接下来再看看DecorView的Measure过程。
## DecorView #onMeasure()
因为DecorView是一个Framelayout，所以我们从FrameLayout 开始我们的分析
```java
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
		int count = getChildCount();
		final boolean measureMatchParentChildren = MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY
					|| MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
		mMatchParentChildren.clear();
		。。。
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

## ViewGroup # measureChildWithMargins()

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
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
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
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
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
![](https://github.com/hoyouly/BlogResource/raw/master/imges/view_measurespce_parent_measurespce.png)
1. 如果子布局的LayoutParams是具体的值，那么子布局的MeasureSpce是EXACTLY，子布局的可用大小是自己的LayoutParams设置的的大小
2. 如果子布局设置的match_parent,那么子布局的MeasureSpce是和父布局的一样，大小就是父布局可用的大小，UNSPECIFIED除外，这种情况下为0
3. 如果子布局设置的是warp_conent,那么子布局的MeasureSpce就是AT_MOST,可用大小就是父布局最大的可用大小，UNSPECIFIED除外，这种情况下还是UNSPECIFIED，并且可用大小为0

整体的流程图如下
![Alt text](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCE751f50bcba2207aab617f8ca29d9083e/2444)

---
搬运地址：   
Android开发艺术探索
