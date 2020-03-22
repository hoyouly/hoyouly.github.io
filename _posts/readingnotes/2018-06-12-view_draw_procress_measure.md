---
layout: post
title: View 的绘制 - Measure 流程
category: 读书笔记
tags: View  Android开发艺术探索
---
* content
{:toc}

在说 Measure 过程之前，需要解释几个概念。MeasureSpec 和 SpecMode

## MeasureSpec
* MeasureSpec很大程度上决定了View的尺寸规格，之所以很大程度上，是因为这个过程还受到父容器的影响，在测量过程中，系统会将View的LayoutParams,根据父容器的所施加的规则转换成对应的MeasureSpce，然后再根据这个MeasureSpce测量出View的宽高。
* MeasureSpce代表一个32位的int值，高2位代表SpecMode,即测量模式，低30位代表SpecSize，即某种测量模式下的规格大小。
* MeasureSpec通过打包把SpecMode和SpecSize组成一个int值从而避免过多的对象内存分配，同时提供打包和解包的方法，从而得到原始的SpecMode和SpecSize

### SpecMode
分三种模式：
* UNSPECIFIED   父容器不对子View做任何限制，要多大给多大，一般用于系统内部，表示一种测量状态
* EXACTLY  父容器已经检测出子view所需要的大小，这个时候View的最终大小就是specSize所指的值，对应于LayoutParams中的match_parent和具体的数值这两种模式
* AT_MOST  父容器指定了一个可用大小即SpecSize,View 大小不能大于这个值，具体是什么值要看不同View的具体实现，它对应了Layoutparams中的warp_content

### SpecSize
某种模式的规格大小。是一个值。

## MeasureSpce 与LayoutParams 关系
MeasureSpce 不是唯一由LayoutParams决定的，LayoutParams 需要和父容器一起才能决定View的MeasureSpce,从而进一步决定View的宽高。对应顶级View(DecorView)和普通的View，MeasureSpce的转换略有不同，
* DecorView：  其MeasureSpce由窗口的尺寸和自身的LayoutParams共同决定
* 普通View：   其MeasureSpce由父容器的MeasureSpce和自身的LayoutParams共同决定。概况起来就是：   
	<font color="ff000">
子View的MeasureSpec = LayoutParams + margin + padding + 父容器的MeasureSpec</font>

### DecorView 的 MeasureSpce 创建工程
上一篇我们知道。在performTraversals()会执行 measureHierarchy()。在measureHierarchy()会创建到DecorView 的 MeasureSpce

#### ViewRootImpl # measureHierarchy()
```java
childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
```
接下来看getRootMeasureSpec()的代码

#### ViewRootImpl # getRootMeasureSpec()

```java
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
	int measureSpec;
	switch (rootDimension) {
		case ViewGroup.LayoutParams.MATCH_PARENT:
			measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
			break;
		case ViewGroup.LayoutParams.WRAP_CONTENT:
			measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
			break;
		default:
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

### 普通View的 MeasureSpace 过程
DecorView 是顶级的View，也是一个ViewGroup，普通的View在measure()的时候。会根据父容器MeasureSpec同时结合View本身的LayoutParams来确定该View 的MeasureSpec,该View的MeasureSpec的创建与父容器的MeasureSpec和本身的LayoutParams有关，此外还和View的margin和padding有关。这个主要是在ViewGroup # getChildMeasureSpec()中执行的。

代码如下。
#### ViewGroup # getChildMeasureSpec()
```java
//spec为父容器的MeasureSpec  padding 子View的padding，childDimension 子View想要的宽度/高度
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);//父容器的specMode
    int specSize = MeasureSpec.getSize(spec);//父容器的specSize
    int size = Math.max(0, specSize - padding);
    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {//根据父容器的specMode
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
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
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```
虽然代码很长，但是很好理解，
1. 如果子布局的LayoutParams是具体的值，那么子布局的MeasureSpce是EXACTLY，子布局的可用大小是自己的LayoutParams设置的的大小
2. 如果子布局设置的match_parent,那么子布局的MeasureSpce是和父布局的一样，大小就是父布局可用的大小，UNSPECIFIED除外，这种情况下为0
3. 如果子布局设置的是warp_conent,那么子布局的MeasureSpce就是AT_MOST,可用大小就是父布局最大的可用大小，UNSPECIFIED除外，这种情况下还是UNSPECIFIED，并且可用大小为0

换成表格形式就是如下。
![](../../../../images/view_measurespce_parent_measurespce.png)

**MeasureSpce一旦确定，onMeasure()中就可以得到View的测量宽和高**

我们就从此继续分析。Hierarchy  层级，阶层，等级制度。measureHierarchy()用于测量整个控件树，
## ViewRootImpl # measureHierarchy()

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
**MeasureHierarchy()有自己的测量方法，会先以较小的尺寸进行布局。让窗口更加优雅（主要是针对wrap_content的Dialog），所以设置了wrap_content的Dialog，有可能执行多次测量**
通过getRootMeasureSpec()计算出来MeasureSpce之后，就执行performMeasure()方法，

## ViewRootImpl # performMeasure()

```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
	mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```
该方法很简单，就是直接调用mView.measure()方法，而这个mView就是执行setView方法中，传递过来的View。

measure过程分为两种：
* 只有一个原始的View，那么通过measure()方法就可以完成了
* 是一个ViewGroup,除了完成自己是测量过程，还要遍历调用子View 的measure过程。各个子View在递归调用这个流程

针对这两种情况分别讨论。先说View的 measure()
## View 的 measure 过程

### View # measure()
关键代码如下。

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
* 该方法定义的是final类型的，子类不能重写该方法
* 仅当给与的MeasureSpec发生变化时，或要求强制重新布局时，才会进行测量。 **当子控件内容发生变化时，从子控件到父控件会回溯到ViewRootImpl，并依次调用父控件的requestLayout()方法，requestLayout()会在mPrivateFlage中加入标记PFLAG_FORCE_LAYOUT,从而使这些父控件的measure()方法得到顺利执行，进而这个子控件有机会进行重新布局与测量，这便是强制重新布局的意义所在。**
* view.measure()方法其实没有实现任何测量的算法，它的作用在于判断是否需要引发onMeasure()的调用，并对onMeasure()行为的正确性进行检查。

接下来看onMeasure()方法，这个方法我们就比较熟悉了，自定义控件的时候经常会被复写的三个方法之一。
### View # onMeasure()

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	setMeasuredDimension(
			getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
			getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```
代码很简单，通过setMeasureDimension()设置View的宽和高，那么我们就首先看看宽和高是怎么得到的，这就需要查看getDefaultSize()方法了

### View # getDefaultSize()
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
1. 对于AT_MOST和EXACTLY这两种情况，View的高度是由specSize决定的，简单来说，就是getDefautlSize()返回的就是MeasureSpce对应的specSize。  
注意：**如果我们直接继承View的自定义控件，需要重写onMeasure方法并设置wrap_content时自身大小，否则布局中使用wrap_content就相当于使用match_parent。**   
因为View使用了wrap_content ,那么他的MeasureSpec就是AT_MOST,在这种模式下，他的宽高等于specSize,由上面 ViewGroup # getChildMeasureSpec() 得到的图我们可知，View的specSize就是parentSize,这种情况下，和match_parent效果一样，那么怎么解决呢，就是给View设定一个默认的的内部宽高（mWith，mheight）,并且在wrap_content时候设置进去就可以了
2. 至于 UNSPECIFIED情况，一般用于系统内部测量，这种情况，View的大小就是getDefaultSize()的第一个参数，即getSuggestedMinimumWidth()/getSuggestedMinimumHeight()

### View # getSuggestedMinimumWidth()
```java
protected int getSuggestedMinimumWidth() {
	//如果没有设置背景那么View宽度就是mMinWidth，
	//mBackground.getMinimumWidth() 得到的就是Drawble的原始宽度
	return (mBackground == null) ? mMinWidth : max(mMinWidth,mBackground.getMinimumWidth());
}
```
getSuggestedMinimumWidth()的逻辑就是： 如果没有设置背景，那么View的宽度就是mMinWidth,而mMinWidth 对应的就是Android：minWidth,如果不指定这个属性，默认为0;如果设置了背景，那么得到的就是mMindth和背景宽度的最大哪一个。
## ViwGroup 的measure过程
因为ViewGroup是一个抽象类， 不同的ViewGroup ,布局方式不同，测量细节也就不同的，比如LinearLayout，RelativeLayout，所以并没有重写onMeaure(), 要具体实现类实现该方法。但是它提供了一个叫measureChildren()的方法

### ViewGroup # measureChildren()
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
从代码上看，ViewGroup在measure时,会对每一个子View进行measure,也就是执行measureChild()，这个方法也很好理解的。就是measure 子View呗。

### ViewGroup # measureChild()
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
代码里面都带有注释，很好理解，就不多说了，getChildMeasureSpec()方法参照上面的表格就可以了  ，然后就到了child.measure()，如果这个child 是一个ViewGroup，继续上的，如果是一个View，就执行View的 measure流程。这个上面已经讲过了。

### LinearLayout  # onMeasure()
以LinearLayout 为例子看一个ViewGroup的measure过程。

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
代码很简单，根据不同的方向，选择不同的measure流程。比如选择竖直方向的LinearLayout的测量过程。即 measureVertical()，源码比较长，分看来看。

#### LinearLayout  # measureVertical()
```java
// See how tall everyone is. Also remember max width.
for (int i = 0; i < count; ++i) {
	final View child = getVirtualChildAt(i);
		...
		//遍历每个子元素，并对每个子元素执行measureChildBeforeLayout()，这个方法内部调用measure()
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

void measureChildBeforeLayout(View child, int childIndex, int widthMeasureSpec, int totalWidth, int heightMeasureSpec, int totalHeight) {
		measureChildWithMargins(child, widthMeasureSpec, totalWidth, heightMeasureSpec, totalHeight);
}
```
从代码看，
1. 系统会遍历子元素，并对每个子元素执行measureChildBeforeLayout()，然后到了 measureChildWithMargins() ,虽然不是 measureChild(),但是看代码其实内部逻辑是一样的。在该方法内部，也会执行View的measure()方法，这样子view就依次开始执行measure过程。
```java
protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed, int parentHeightMeasureSpec, int heightUsed) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        //得到子元素的MeasureSpec。显然子元素的MeasureSpec的创建与父容器的MeasureSpec和子元素本身的LayoutParams有关，此外还和View的margin和padding有关
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec, mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec, mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin + heightUsed, lp.height);
        //如果当前child也是ViewGroup，则measure()方法就又会调用相应的onMeasure（）继续遍历它的子View,
        //如果当前child是View，便根据这个MeasureSpec测量自己
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```
执行某个子View 的measure()后，就可以通过child.getMeasuredHeight()得到该View的高度，
2. 通过mTotalLength来存储系在竖直方向的高度。每测量一个，mTotalLength就会增加。增加的部分主要包括子View的高度以及子元素在竖直方向的margin，当子元素测量完后，LinearLayout开始测量自己的大小。
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
LinearLayout 会根据子元素的情况，来测量自己的大小。针对竖直方向而言，他的水平方向遵循View的测量过程。但是竖直方向则和View不同，具体来说
	1. 如果布局中高度采用的match_parent或者具体的值，那么它的测量过程和View 的一致
	2. 如果布局高度采用wrap_content,他的高度就是所有子元素高度的总和，但是不能超过它父容器的剩余空间。当然还要考虑他的竖直方向的padding，可以参考resolveSizeAndState(),这是View的方法，LinearLayout 和ViewGroup 都没有重新该方法
```java
public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
	int result = size;
	int specMode = MeasureSpec.getMode(measureSpec);
	int specSize = MeasureSpec.getSize(measureSpec);
	switch (specMode) {
		case MeasureSpec.UNSPECIFIED:
			result = size;
			break;
		case MeasureSpec.AT_MOST:
		//采用wrap_content,他的高度就是所有子元素高度的总和，但是不能超过它父容器的剩余空间。
			if (specSize < size) {
				result = specSize | MEASURED_STATE_TOO_SMALL;
			} else {
				result = size;
			}
			break;
		case MeasureSpec.EXACTLY:
		//采用的match_parent或者具体的值,那么返回specSize
			result = specSize;
			break;
	}
	return result | (childMeasuredState & MEASURED_STATE_MASK);
}
```

看到了我们熟悉的setMeasuredDimension()了吗。因为ViewGroup中没有处理setMeasuredDimension(),所以还是只想到了View中的setMeasuredDimension()中
这就是LinearLayout 的 Measure 流程，再看看DecorView的Measure过程。
### DecorView #onMeasure()
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
2. 调用了 measureChildWithMargins() 。在  measureChildWithMargins()里面会结合父View的MeasureSpec和自己的LayoutParams 计算出自己的MeasureSpce，然后执行measure()

### 在Activity中得到View的宽高的四种方式
measure()的流程就说完了，但是如果有这样一个任务，在Activity一启动，就想要得到View的宽高，这个怎么处理啊，因为View的measure() 过程是和Activity的生命周期不是同步执行的。这样就无法保证在Activity执行到onStart()/onResume()的时候，能准确拿到View的宽高，我们可以通过下面的四种方式得到。
1. onWindowFocusChanged()  View 已经初始化完毕，会被调用多次，当Activity的窗口得到焦点和失去焦点的时候，都会执行一次，如果频繁的进行onResume()和onPause()那么onWindowFocusChanged()也会被循环调用
2. View.post(Runnable) 通过post把一个Runnable对象加到添加队列中，然后等待Looper调用次Runnable的时候，View已经初始化好了，
3. ViewTreeObserver  使用ViewTreeObserver的众多回调可以完成此功能l，比如使用OnGlobalFocusChangeListener 这个借口，当View的状态树的状态发生改变或者View树内部的可见性发送改变，onGlobalFocusChanged（）这个方法被回调，但是需要注意，伴随View状态树改变，onGlobalFocusChanged（）会被调用多次。
4. 手动对View进行measure的宽和高。这种情况比较复杂。


整体的流程图如下
![Alt text](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCE751f50bcba2207aab617f8ca29d9083e/2444)


[View 的绘制 - 概览](../../../../2018/06/09/view_draw_procress_performTraversals/)   
[View 的绘制 - Measure 流程](../../../../2018/06/12/view_draw_procress_measure/)   
[View 的绘制 - Layout 流程](../../../../2018/06/20/view_draw_procress_layout/)   
[View 的绘制 - Draw 流程，invalidate 的流程 以及 requestLayout 流程](../../../../2018/06/29/view_draw_procress_draw/)

---
搬运地址：   
Android开发艺术探索
