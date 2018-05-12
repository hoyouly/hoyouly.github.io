---
layout: post
title: View 的事件分发机制
category: 读书笔记
tags: View 事件分发 Android开发艺术探索 
---
* content
{:toc}

## 触摸事件
点击事件，也称为触摸事件，是捕获触摸屏幕产生后的事件，所谓点击事件的分发，其实就是对MotionEvent事件的分发过程，即当一个MotionEvent产生后，系统需要把这个时间传递给具体的view ,而这个传递的过程其实就是分发过程
## MotionEvent 类
触摸事件封装的类，可以得到触摸的坐标，getX()和getRawX()，得到触摸的类型，例如ACTION_DOWN，ACTION_UP,ACTION_MOVE等
* getX()是表示view 相对于自身左上角的x坐标,
* getRawX()是表示相对于屏幕左上角的x坐标值(注意:这个屏幕左上角是手机屏幕左上角,不管activity是否有titleBar或是否全屏幕),
* getY(),getRawY()一样的道理   

![](http://ww3.sinaimg.cn/large/9dd25cfdgw1f4cal6hv8gj20ci09kmxk.jpg)

## 主要涉及到的方法

* <font color="#ff000" > boolean dispatchTouchEvent(MotionEvent event)</font>  当进行事件分发的时候，如果事件能传递到当前View,那么此方法一定会被调用，返回的结果受到当前View 的onTouchEvnet()和下级View的dispatchTouchEvent()的影响，返回值表示是否消耗当前事件。
* <font color="#ff000" > boolean onInterceptTouchEvent(MotionEvent ev) </font>
在dispatchToucheEvent() 内部调用，用来判断是否拦截某个事件，如果当前View拦截了某个事件，那么在同一个事件序列中，此方法不会被再次调用，返回结果表示是否拦截当前事件
* <font color="#ff000" >boolean onTouchEvent(MotionEvent event)</font>
在dispatchToucheEvent() 方法中调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果表示不消耗，则在同一个事件序列中，当前View无法再次接受到事件

### 伪代码
```java
public boolean dispatchTouchEvent(MotionEvent event) {
    boolean consume = false;        
    if (onInterceptTouchEvent(event)) {
        consume = onTouchEvent(event);
    } else {
        consume = child.dispatchTouchEvent(event);
    }
    return consume;
}
```

**传递规则**&#160;&#160;&#160;&#160;  对于一个根ViewGroup,点击事件产生后，首先会传递给它，这个时候他的dispatchTouchEvent() 就会被调用，如果这个ViewGroup的onInterceptTouchEvent() 方法返回了true，表示它要拦截这个事件，接着这个事件就会交给这个ViewGroup处理，即调用它的onTouchEvent方法，如果onInterceptEvent()  返回false，则就会继续传递给子View，即调用子View的dispatchTouchEvent() 方法，如果反复直到这个事件最终处理  
**View 处理一个事件**
  * 如果设置了onTouchListener，那么onTouchListener中的onTouch() 方法就会被调用，
    * 如果onTouch()  返回false，则当前View的onTouchEvent方法才会被调用，
    * 如果返回true，那么onTouchEvent()  方法不会被调用，  
    **所以 onToucherListener 优先级要比onTouchEvent高**   
  * 如果设置了onClickListener,那么它的onClick()  方法会被调用，由此 onClicklistener 优先级最低

**优先级由低到高**&#160;&#160;&#160;&#160; onClickListener < onTouchEvent < onTouchListener  
**事件的传递顺序**:&#160;&#160;&#160;&#160; Activity -> Window ->View   

事件总是先传递给Activity，Activity再传递给Window，最后传递给View，其中View的根View就是decoer View ( decoer:装饰 布置)  
如果View 的onTouchEvent() 返回false,即表示不处理，那么他的父容器的onTouchEvent()  就会被调用，如果都不处理这个事件，那么最终就会传递给Activity，即Activity的onTouchEvent()  事件被调用，

### 事件传递机制的一些结论
1. 同一个事件序列是指从手指接触屏幕的那一刻起，到手指离开屏幕的那一刻结束，在这个过程中所产生的一系列事件，这个事件序列以down事件开始，中间包括了一个或者多个move事件，最终以up事件结束
2. 正常情况下，一个事件序列只能被一个View拦截并且消耗，这一条的原因可以参考（3），因为一旦一个元素拦截了某个事件，那么同一个事件序列内的所有事件都会直接交给它处理，因此同一个事件序列中的事件不能分别有两个View同时处理， **但是通过特殊的手段可以做到，比如一个View将本该自己处理的事件，通过onTouchEvent强行传递给其他View处理**
3. 某个View 一旦决定拦截，那么这个事件序列都只能由他来处理（如果事件能够传递给它的话），并且他的onInterceptTouchEvent() 不会再被调用,这条也很好理解，就是说当一个View决定拦截一个事件后，那么系统就会把同一个事件序列内的其他方法都直接交给它来处理，因此就不用再调用这个View的 onInterceptTouchEvent() 去询问是否要拦截
4. 某个View一旦开始处理事件，如果它不消耗ACTION_DOWN事件（onTouchEvent() 返回了false）,那么同一事件序列的其他事件都不会再交给他来处理，并且事件将重新交给由它的父控件去处理，即父控件的onTouchEvent() 会被调用，意思是说一旦一个事件交给了View处理，那么他就必须消耗掉，否则同一事件序列中剩下的事件就不再交给它来处理。
5. 如果View不消耗ACTION_DOWN以外的其他事件，那么这个事件就会消失，此时父元素的onTouchEvent() 并不会调用，并且当前View可以持续收到后续事件，最终这些消失的点击事件会传递到Activity处理
6. ViewGroup 默认不会拦截任何事件的，Android源码中ViewGroup的onInterceptTouchEvent() 方法默认返回false，
7. View 没有onInterceptTouchEvent() 方法，一旦有点击事件传递给他，那么它的onTouchEvent()  就会被调用，
8. View的onTouchEvent() 默认都会消耗事件，（返回true），除非他是 **不可点击的（clickable和longClickable同时为false）** .View的longClickable属性默认都是false，clickable属性要分情况，比如Button的clickable属性默认是true,而TextView的clickable属性默认为false
9. View的enable属性不影响onTouchEvent() 的默认返回值，哪怕一个View是disable状态，只要它的clickable或者longClickable有一个为true,那么他的onTouchEvent就返回true
10. onClick会发生的嵌套是当前View是可点击的，并且它收到了down事件
11. 事件传递过程是由外向内的，即事件总是传递给父元素，然后在有父元素分发给子View,通过requestDisallowInterceptTouchEvent() 方法可以在子元素中干预父元素的事件分发过程，但是ACTION_DOWN事件除外

## 源码分析
### Activity对点击事件的分发过程
当点击事件发生，事件最先传递给当前的Activity，由Activity的dispatchTouchEvent() 进行分发，具体工作是由Activity中的Window来完成，Window会将事件传递给decor view，decor view一般就是当前界面装饰容器，即setContentView()  所设置的View 的父容器，通过Activity.getWindow().getDecorView()可以获得

### Activity # dispatchTouchEvent()
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();//空实现，不管
    }
    //getWindow()得到的是一个Window对象，而Window是一个抽象类，唯一的实现类是 PhoneWindow，
    // 而superDispatchTouchEvent又是一个抽象方法，
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```
所以根据上面的源码可知，如果PhoneWindow中的superDispatchTouchEvent() 返回true 整个事件结束，如果返回false,那么Activity中的onTouchEvent() 会被执行，

### PhoneWindow # superDispatchTouchEvent()
```java
public  boolean superDispatchTouchEvent(MotionEvent event){
  return mDecor.superDispatchTouchEvent(event)
}
```
直接把事件传递给了DecorView 的 superDispatchTouchEvent() 方法
### DecorView 是个啥东东
继承FrameLayout，是一个父View，我们通常在Activity中setContentView() 其实是把我们自己写的布局文件作为一个子View 设置到这个DecorView中，所以才有了，
```java
ViewGroup viewGroup =  (ViewGroup)getWindow().getDecorView().findViewById(android.R.id.content);
viewGroup.getChildAt(0);//得到Activity所设置的View，
```
总的来说，事件从Activity的dispatchTouchEvent() 事件，经过phoneWindow，然后传递给了我们通常设置的View的父控件DecorView的superDispatchTouchEvent() ，由于DecorView是一个ViweGroup,ViewGroup的onIntecerpteTouchEvent() 事件是false，所以肯定会传递到我们说的View中

### 顶级View 对点击事件的分发过程
点击事件到顶级View 上，一般是一个ViewGroup，会调用ViewGroup的dispatchTouchEvent 方法,接下来的伪代码如下
```java
public boolean dispatchTouchEvent(MotionEvent event) {
    //执行onInterceptTouchEvent 方法
    if (onInterceptTouchEvent(event)) {//返回true,事件由该ViewGroup处理，事件被拦截
        //判断是否设置了onTouchListener。
        if (mTouchListener != null) {//设置了，调用onTouch事件
            return mTouchListener.onTouch(this, event);
        } else {//调用onTouchEvent 事件，也就是说onTouch会屏蔽到onTouchEvent
            if (mOnClickListener != null) {//如果设置了onClickListener，
                mOnClickListener.onClick(this);//调用onClick方法
            }
            return onTouchEvent(event);
        }
    } else {//不拦截，则交给子元素的dispatchTouchEvent 处理
        return child.dispatchTouchEvent(event);
    }
}
```
### ViewGrop # dispatchTouchEvent()
ViewGrop的dispatchTouchEvent()方法比较长，接下来分段讲解

```java
if (actionMasked == MotionEvent.ACTION_DOWN) {
        cancelAndClearTouchTargets(ev);//取消和清理mFirstTouchTarget
        resetTouchState();//对FLAG_DISALLOW_INTERCEPT 进行重置
}

final boolean intercepted;
  /**
    * 判断是否拦截当前事件，有两种情况，事件类型为ACTION_DOWN mFirstTouchTarget ！=null
    * 第一种情况好理解，但是第二张情况，mFirstTouchTarget是个什么鬼东西：
    * 当子元素成功处理时，mFirstTouchTarget会被赋值并指向子元素，
    * 换句理解，一旦当前ViewGroup拦截处理，mFirstTouchTarget就为nu当ACTION_MO和ACTION_UP 事件道来，
    * 该判断就为false,就导致onInterceptTouchEvent 不会被调用。即同一事件序列的件都会默个它处理，
    */
if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
  /**
   * FLAG_DISALLOW_INTERCEPT 是有子View的requestDisallowInterceptTouchEvent（）设置的，
    * 也就是说一旦子元素设置了FLAG_DISALLOW_INTERCEPT 后，父了ACTION_DOWN以外其件序列都无法拦截，
    * 这是因为在ACTION_DOWN 事件，ViewGroup总会个FLAG_DISALLOW_INTERCEPT 标子View设置的FLAG_DISALLOW_INTERCEP效的
    * 即当面对 ACTION_DOWN 事件的时候，ViewGroup总是调的onInterceptTouchEvent 问是否拦截的
    */
  final boolean disallowIntercept = (mGroupFlagsFLAG_DISALLOW_INTERCEPT) != 0;
  if (!disallowIntercept) {
      intercepted = onInterceptTouchEvent(ev);
      ev.setAction(action); // restore action in case it was changed
  } else {
      intercepted = false;
  }
} else {
    intercepted = true;
}
```
这段代码都做了什么呢
1. 在action_down 事件中重置状态操作，主要是一些子元素对父元素的设置，包括mFirstTouchTarget 和FLAG_DISALLOW_INTERCEPT  
2. 开始判断是否要拦截事件，通过`if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null)`进行判断，MotionEvent.ACTION_DOWN 在整个事件序列中是第一个并且只执行一次，所以肯定会执行到代码块中的，而里面还有一个条件判断，这个就是子元素调用requestDisallowInterceptTouchEvent() 设置的，但是因为在(1)的时候已经进行了重置，所以这个设置在ACTION_DOWN中是无效的，也就是说在ACTION_DOWN事件中，肯定会执行到onInterceptTouchEvent 方法，  
3. 如果该ViewGroup 拦截了该ACTION_DOWN 事件，事件就不能传递到子元素，那么 mFirstTouchTarget 就为null，因为mFirstTouchTarget 是由子元素处理事件后赋值的.那么这个事件序列中的其他事件（ACTION_MOVE 和ACTION_UP）就直接交个这个ViewGroup 处理
4. 因为默认情况下，ViewGroup中的onInterceptTouchEvent方法返回的false，即不拦截，所以事件会传递到子元素中

当ViewGroup不拦截事件的时候，事件会下发交由他的子View进行处理

```java
final int childrenCount = mChildrenCount;
if (newTouchTarget == null && childrenCount != 0) {
    。。。
    final View[] children = mChildren;
    for (int i = childrenCount - 1; i >= 0; i--) {//遍历整个子元素
        final int childIndex = customOrder ? getChildDrawingOrder(childrenCount, i) : i;
        final View child = (preorderedList == null) ? children[childIndex] : preorderedList.get(childIndex);

        if (childWithAccessibilityFocus != null) {
            if (childWithAccessibilityFocus != child) {
                continue;
            }
            childWithAccessibilityFocus = null;
            i = childrenCount - 1;
        }
        //判断子元素是否能接收点击事件：子元素是否在播放动画和点击坐标点是否在子元素内
        if (!canViewReceivePointerEvents(!isTransformedTouchPointInView(x, y, child, null)) {
            ev.setTargetAccessibilityFocus(false);
            continue;
        }

        newTouchTarget = getTouchTarget(child);
        if (newTouchTarget != null) {
            newTouchTarget.pointerIdBits |= idBitsToAssign;
            break;
        }

        resetCancelNextUpFlag(child);
        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
            //子元素的dispatchTouchEvent（）返回的是true
            mLastTouchDownTime = ev.getDownTime();
            if (preorderedList != null) {
                for (int j = 0; j < childrenCount; j++) {
                    if (children[childIndex] == mChildren[j]) {
                        mLastTouchDownIndex = j;
                        break;
                    }
                }
            } else {
                mLastTouchDownIndex = childIndex;
            }
            mLastTouchDownX = ev.getX();
            mLastTouchDownY = ev.getY();
            //mFirstTouchTarget 被赋值，并且跳出循环
            newTouchTarget = addTouchTarget(child, idBitsToAssign);
            alreadyDispatchedToNewTouchTarget = true;
            break;
}

```
1. 遍历整个ViewGroup的子元素，然后判断是否能够
2. 判断子View是否能够接受点击事件
  * 子元素是否在播放动画
  * 点击事件的左边是否落在子元素区域内   
3. 如果能够接收点击事件，那么事件交由他来处理，执行view的 dispatchTransformedTouchEvent()，其实该方法内部，同样是调用的子View的 dispathTouchEvent() 方法，这样就完成了一轮事件分发

### View # dispatchTransformedTouchEvent()
```java
if (child == null) {
        handled = super.dispatchTouchEvent(event);
    } else {
        handled = child.dispatchTouchEvent(event);
    }
```            
因为传递过来的View不为null，所以调用了子View的child.dispatchTouchEvent 方法，完成了一轮分发

如果 dispatchTransformedTouchEvent()返回true,那么mFirstTouchTarget就会被赋值，同时跳出循环，break,这个 mFirstTouchTarget到底是啥东西，再一次遇到了，
```java
if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
	...
	...
  newTouchTarget = addTouchTarget(child, idBitsToAssign);
	alreadyDispatchedToNewTouchTarget = true;
	break;
}
```
### mFirstTouchTarget
mFirstTouchTarget 其实是一个单链表结构，
mFirstTouchTarget 的赋值过程是在addTouchTarget()中完成的，
### View # addTouchTarge()

addTouchTarge()方法又是什么呢？	它会根据传递过来的子View和pointerIdBits 创建一个target ,然后把这个target赋值给mFirstTouchTarget
```java
private TouchTarget addTouchTarget(View child, int pointerIdBits) {
    TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
```
原来，子View调用dispatchTouchEvent 返回true，就会根据该子View，创建一个Target，并把这个target赋值给mFirstTouchTarget，mFirstTouchTarget 是否赋值，直接就影响到ViewGroup的拦截策略，如果mFirstTouchTarget为null,那么ViewGroup就默认拦截下来同一事件序列所有的点击事件

如果遍历所有的子元素后，事件都没有被合适的处理，包含两种情况
1. 该ViewGroup 没有子元素**或者子元素没有处理点击事件**，
2. 有子元素并且处理了点击事件，但是在dispatchTouchEvent中返回了false，这一般就是因为子元素在onTouchEvent 中返回了false      

这两种情况下，ViewGroup会自己处理点击事件，即调用
dispatchTransformedTouchEvent方法，不过View传递的是null，则调用 super.dispatchTouchEvent(event);
```java
if (mFirstTouchTarget == null) {
  handled = dispatchTransformedTouchEvent(ev, canceled, null, TouchTarget.ALL_POINTER_IDS);
}
```
### View对点击事件的处理过程，
需要说明的几点
1. 这里的View不包括ViewGroup
2. 没有onIntercepeTouchEvent ,因为不需要拦截。只能由自己处理

dispatchTouchEvent 主要源码如下
```java
if (onFilterTouchEventForSecurity(event)) {
      //noinspection SimplifiableIfStatement
      ListenerInfo li = mListenerInfo;
      if (li != null && li.mOnTouchListener != null&& (mViewFlags & ENABLED_MASK) == ENABLED&& li.mOnTouchListener.onTouch(this, event)) {
          result = true;
      }
      if (!result && onTouchEvent(event)) {
          result = true;
      }
}
```
1. 首先判断是否设置了mOnTouchListener,如果设置了，则调用onTouch() 方法，如果该方法返回了true,则事件不在往下传递，直接返回
2. 如果没有设置mOnTouchListener或者onTouch() 返回了false，则调用onTouchEvent()  ,这也说明了mOnTouchListener 优先级高于onTouchEvent()  ,这样做的是为了方便外界处理点击时间

### View # onTouchEvent()

1. 首先处理不可用状态下的点击事件，不可用情况下，照样可以消耗点击事件，只要CLICKABLE或者LONG_CLICKABLE 有一个为true
```java
if ((viewFlags & ENABLED_MASK) == DISABLED) {
    if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
        setPressed(false);
    }
    // A disabled view that is clickable still consumes the touch
    // events, it just doesn't respond to them.
    return (((viewFlags & CLICKABLE) == CLICKABLE
            || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
            || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
}
```        		
2. 如果设置了代理，执行TouchDelegate的onTouchEvent 方法，这个和onToucheListener 类似
```java
if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
}
```        
3. 接下来看一下onTouchEvent() 中对点击事件的具体处理,只要CLICKABLE或者LONG_CLICKABLE 有一个为true，那么最后就会消耗这个事件，因为最后返回的是true，
```java
if (((viewFlags & CLICKABLE) == CLICKABLE ||
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
            (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
      return true;
}
```
4. 如果能进入判断内，说明肯定会消耗这个事件，但是在事件ACTION_UP上，会触发performClick()，		
```java
case MotionEvent.ACTION_UP:
    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
        ...
        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
            removeLongPressCallback();

            if (!focusTaken) {
                if (mPerformClick == null) {
                    mPerformClick = new PerformClick();
                }
                if (!post(mPerformClick)) {
                    performClick();
                }
            }
        }
        ...
    break;
```
5. 触发onCLick方法  如果View中设置了onClickeListener 那么就会触发onCLick方法，performClick()方法源码，
```java
public boolean performClick() {
    final boolean result;
    final ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnClickListener != null) {
        playSoundEffect(SoundEffectConstants.CLICK);
        li.mOnClickListener.onClick(this);
        result = true;
    } else {
        result = false;
    }
    sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
    return result;
}
```      
6. View的LONG_CLICKABLE 属性默认为false，CLICKABLE 属性与具体的View有关，确切的说，可点击的View 其CLICKABLE为true，比如button，不可点击的View，其CLICKABLE 为false，例如TextView，可以通过setOnClickListener和setOnLongClickListener 然后进行设置源码如下,   
```java
public void setOnLongClickListener(@Nullable OnLongClickListener l) {
  if (!isLongClickable()) {
      setLongClickable(true);
  }
  getListenerInfo().mOnLongClickListener = l;
}
public void setOnClickListener(@Nullable OnClickListener l) {
    if (!isClickable()) {
      setClickable(true);
    }
    getListenerInfo().mOnClickListener = l;
}
```
## 第一份流程图

```flow
st=>start: 触摸事件
e=>end
opActivity=>operation: Activity#dispatchTouchEvent()
opPhoneWindow=>operation: PhoneWindow#superDispatchTouchEvent()
opDecorView=>operation: DecorView#superDispatchTouchEvent()
opDecorViewDispatch=>operation: DecorView是一个ViewGroup,事件会传递给我们设置的view,即setContentView()设置的布局
condIsViewGroup=>condition: setContentView()设置的布局是否是一个ViewGroup
opChildeView=>operation: View#dispatchTouchEvent()
opSetContentView=>operation: DecorView#superDispatchTouchEvent()

opViewGroupDispath=>operation: ViewGroup#dispatchTouchEvent()
condIntecepte=>condition: ViewGroup#onInterceptTouchEvent()
condOnTouchlistener=>condition: mOnTouchListenr!=null
opOnTouch=>operation: mOnTouchListenr.onTouch()
opTouchEvent=>operation: ViewGroup#onTouchEvent()

st->opActivity
opActivity->opPhoneWindow
opPhoneWindow->opDecorView
opDecorView->opDecorViewDispatch
opDecorViewDispatch->condIsViewGroup
condIsViewGroup(yes)->opViewGroupDispath
opViewGroupDispath->condIntecepte
condIntecepte(yes)->condOnTouchlistener
condIsViewGroup(no)->opChildeView
condIntecepte(no)->opChildeView
condOnTouchlistener(yes)->opOnTouch
condOnTouchlistener(no)->opTouchEvent
opOnTouch->e;

```

---
搬运地址：   
Android 开发艺术探索   
