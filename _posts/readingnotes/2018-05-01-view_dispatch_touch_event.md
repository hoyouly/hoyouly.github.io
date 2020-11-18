---
layout: post
title: View 的事件分发机制
category: 读书笔记
tags: View 事件分发 Android开发艺术探索
---
<!-- * content -->
<!-- {:toc} -->

## 触摸事件
点击事件，也称为触摸事件，是捕获触摸屏幕产生后的事件。    
所谓事件的分发，其实就是对 MotionEvent 事件的分发过程，即当一个 MotionEvent 产生后，系统需要把这个事件传递给具体的 view ,这个传递的过程就是分发过程

### MotionEvent
触摸事件封装的类，可以得到触摸的坐标， getX() 和 getRawX() ；得到触摸的类型，例如 ACTION_DOWN ， ACTION_UP , ACTION_MOVE 等
* getX()是表示 view 相对于自身左上角的 x 坐标,
* getRawX()是表示相对于屏幕左上角的 x 坐标值(注意:这个屏幕左上角是手机屏幕左上角,不管 activity 是否有 titleBar 或是否全屏幕),
* getY(), getRawY() 一样的道理   

![](http://ww3.sinaimg.cn/large/9dd25cfdgw1f4cal6hv8gj20ci09kmxk.jpg)

## 主要涉及到的方法

* <font color="#ff000" > boolean dispatchTouchEvent(MotionEvent event)</font>  当进行事件分发的时候，如果事件能传递到当前 View ,那么此方法一定会被调用，返回的结果受到当前 View 的 onTouchEvnet() 和下级 View 的 dispatchTouchEvent() 的影响，返回值表示是否消耗当前事件。
* <font color="#ff000" > boolean onInterceptTouchEvent(MotionEvent ev) </font>
在 dispatchToucheEvent() 内部调用，用来判断是否拦截某个事件，如果当前 View 拦截了某个事件，那么在同一个事件序列中，此方法不会被再次调用，返回结果表示是否拦截当前事件
* <font color="#ff000" >boolean onTouchEvent(MotionEvent event)</font>
在 dispatchToucheEvent() 方法中调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果表示不消耗，则在同一个事件序列中，当前 View 无法再次接受到事件

### 伪代码
```java
public boolean dispatchTouchEvent(MotionEvent event) {
    boolean consume = false;        
    if (onInterceptTouchEvent(event)) {
        // onTouchEvent() 是执行在 super.dispathTouchEvent()里面的
        consume = onTouchEvent(event);
    } else {
        consume = child.dispatchTouchEvent(event);
    }
    return consume;
}
```

**传递规则**    
对于一个根 ViewGroup ,点击事件产生后，首先会传递给它，这个时候他的 dispatchTouchEvent() 就会被调用，如果这个 ViewGroup 的 onInterceptTouchEvent() 返回了 true ，表示它要拦截这个事件，这个事件就会交给这个 ViewGroup 处理，即调用它的 onTouchEvent()，如果 onInterceptTouchEvent() 返回 false ，则就会继续传递给子 View ，即调用子 View 的 dispatchTouchEvent() 方法，反复直到这个事件最终处理 。<font color="#ff000" > 实质就是一个大的递归函数。 并且使用的是DFS，深度优先遍历</font>  


**View 处理一个事件**    
**优先级由低到高**&#160;&#160;&#160;&#160; onClickListener < onTouchEvent < onTouchListener  

1. 如果设置了 onTouchListener ，那么 onTouchListener 中的 onTouch() 就会被调用，
  * 如果 onTouch() 返回 false ，那么当前 View 的 onTouchEvent() 才会被调用。
  * 如果 onTouch() 返回 true ， 那么当前 View 的 onTouchEvent() 不会被调用。
2. 如果设置了 onClickListener ,那么onClickListener 中的onClick() 方法会被调用。     

**事件的传递顺序**:&#160;&#160;&#160;&#160; Activity -> Window ->View   

事件总是先传递给 Activity ， Activity 再传递给 Window ，最后传递给 View ，其中 View 的根 View 就是decoer View ( decoer:装饰 布置)  
如果 View 的 onTouchEvent() 返回 false ,即表示不处理，那么他的父容器的 onTouchEvent() 就会被调用，如果都不处理这个事件，那么最终就会传递给 Activity ，即 Activity 的 onTouchEvent() 事件被调用，

### 事件传递机制的一些结论
1. 同一个事件序列是指从手指接触屏幕的那一刻起，到手指离开屏幕的那一刻结束，在这个过程中所产生的一系列事件，这个事件序列以 down 事件开始，中间包括了一个或者多个 move 事件，最终以 up 事件结束
2. 正常情况下，一个事件序列只能被一个 View 拦截并且消耗，这一条的原因可以参考（3），因为一旦一个元素拦截了某个事件，那么同一个事件序列内的所有事件都会直接交给它处理，因此同一个事件序列中的事件不能分别有两个 View 同时处理， **但是通过特殊的手段可以做到，比如一个 View 将本该自己处理的事件，通过 onTouchEvent 强行传递给其他 View 处理**
3. 某个 View 一旦决定拦截，那么这个事件序列都只能由他来处理（如果事件能够传递给它的话），并且他的 onInterceptTouchEvent() 不会再被调用。
4. 某个 View 一旦开始处理事件，如果它不消耗 ACTION_DOWN 事件（  即onTouchEvent() 返回了false）,那么同一事件序列的其他事件都不会再交给他来处理，并且事件将重新交给由它的父控件去处理，即父控件的 onTouchEvent() 会被调用，意思是说一旦一个事件交给了 View 处理，那么他就必须消耗掉，否则同一事件序列中剩下的事件就不再交给它来处理。
5. 如果 View 不消耗 ACTION_DOWN 以外的其他事件，那么这个事件就会消失，此时父元素的 onTouchEvent() 并不会调用，并且当前 View 可以持续收到后续事件，最终这些消失的点击事件会传递到 Activity 处理
6. ViewGroup 默认不会拦截任何事件的， Android 源码中 ViewGroup 的 onInterceptTouchEvent() 方法默认返回 false ，
7. View 没有 onInterceptTouchEvent() 方法，一旦有点击事件传递给他，它的 onTouchEvent() 就会被调用，
8. View的 onTouchEvent() 默认都会消耗事件，（返回true），除非他是 **不可点击的（clickable和 longClickable 同时为false）.** View的 longClickable 属性默认都是 false ， clickable 属性要分情况，比如 Button 的 clickable 属性默认是 true ,而 TextView 的 clickable 属性默认为false
9. View的 enable 属性不影响 onTouchEvent() 的默认返回值，哪怕一个 View 是 disable 状态，只要它的 clickable 或者 longClickable 有一个为 true ,那么他的 onTouchEvent 就返回true
10. onClick会发生的嵌套是当前 View 是可点击的，并且它收到了 down 事件
11. 事件传递过程是由外向内的，即事件总是传递给父元素，然后在由父View分发给子 View ,通过 requestDisallowInterceptTouchEvent() 方法可以在子View中干预父View的事件分发过程，但是 ACTION_DOWN 事件除外



## 源码分析
当点击事件发生，事件最先传递给当前的 Activity ，由 Activity 的 dispatchTouchEvent() 进行分发，具体工作是由 Activity 中的 Window 来完成， Window 会将事件传递给 DecorView ， DecorView 一般就是当前界面装饰容器，即 setContentView() 所设置的 View 的父容器，通过Activity.getWindow().getDecorView()可以获得

### Activity # dispatchTouchEvent()
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();//空实现，不管
    }
    //getWindow()得到的是一个 Window 对象，而 Window 是一个抽象类，唯一的实现类是 PhoneWindow ，
    // 而 superDispatchTouchEvent 又是一个抽象方法，
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```
所以根据上面的源码可知，如果 PhoneWindow 中的 superDispatchTouchEvent() 返回 true 整个事件结束，如果返回 false ,那么 Activity 中的 onTouchEvent() 会被执行，

### PhoneWindow # superDispatchTouchEvent()
```java
public  boolean superDispatchTouchEvent(MotionEvent event){
  return mDecor.superDispatchTouchEvent(event)
}
```
直接把事件传递给了 DecorView 的 superDispatchTouchEvent() 方法

### DecorView
继承 FrameLayout ，是一个父 View ，我们通常在 Activity 中 setContentView() 其实是把我们自己写的布局文件作为一个子 View 设置到这个 DecorView 中，所以才有了，
```java
ViewGroup viewGroup =  (ViewGroup)getWindow().getDecorView().findViewById(android.R.id.content);
viewGroup.getChildAt(0);//得到 Activity 所设置的 View ，
```
总的来说，事件从 Activity 的 dispatchTouchEvent() 事件，经过 phoneWindow ，然后传递给了我们通常设置的 View 的父控件 DecorView 的 superDispatchTouchEvent() ，由于 DecorView 是一个 ViweGroup , ViewGroup 的 onIntecerpteTouchEvent() 事件是 false ，所以肯定会传递到我们说的 View 中

### ViewGrop # dispatchTouchEvent()
ViewGrop 的 dispatchTouchEvent() 方法比较长，接下来分段讲解

```java
if (actionMasked == MotionEvent.ACTION_DOWN) {
  cancelAndClearTouchTargets(ev);//取消和清理mFirstTouchTarget
  resetTouchState();//对 FLAG_DISALLOW_INTERCEPT 进行重置
}

final boolean intercepted;
/**
  * 判断是否拦截当前事件，有两种情况，事件类型为 ACTION_DOWN 或者 mFirstTouchTarget ！=null
  * 第一种情况好理解，第二张情况是当子元素成功处理时， mFirstTouchTarget 会被赋值并指向子元素，
  * 换句理解，一旦当前 ViewGroup 拦截处理， mFirstTouchTarget 就为 null 当 ACTION_MOVE 和 ACTION_UP 事件到来，
  * 该判断就为 false ,就导致 onInterceptTouchEvent() 不会被调用。即同一事件序列的件都会默个它处理，
  */
if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
  /**
    * FLAG_DISALLOW_INTERCEPT 是由子 View 的 requestDisallowInterceptTouchEvent() 设置的，
    * 也就是说一旦子元素设置了 FLAG_DISALLOW_INTERCEPT 后，父 View 除了 ACTION_DOWN 以外其件序列都无法拦截，
    * 这是因为在 ACTION_DOWN 事件， ViewGroup 总对 FLAG_DISALLOW_INTERCEPT 进行重置，
    * 因此子 View 设置的 FLAG_DISALLOW_INTERCEP 在 ACTION_DOWN 中是无效的
    * 即当面对 ACTION_DOWN 事件的时候， ViewGroup 总是调的 onInterceptTouchEvent() 问是否拦截的
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
1. 在 action_down 事件中重置状态操作，主要是一些子元素对父元素的设置，包括 mFirstTouchTarget 和FLAG_DISALLOW_INTERCEPT  
2. 开始判断是否要拦截事件，通过   
`if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null)`进行判断，MotionEvent.ACTION_DOWN 在整个事件序列中是第一个并且只执行一次，所以肯定会执行到代码块中的，而里面还有一个条件判断，这个就是子元素调用 requestDisallowInterceptTouchEvent() 设置的，但是因为在(1)的时候已经进行了重置，所以这个设置在 ACTION_DOWN 中是无效的，也就是说在 ACTION_DOWN 事件中，肯定会执行到 onInterceptTouchEvent 方法，  
3. 如果该 ViewGroup 拦截了该 ACTION_DOWN 事件，事件就不能传递到子View(**或者以 CANCEL 的方式通知子 View**)，那么 mFirstTouchTarget 就为 null ，因为 mFirstTouchTarget 是由子元素处理事件后赋值的.那么这个事件序列中的其他事件（ACTION_MOVE 和ACTION_UP）就直接交个这个 ViewGroup 处理
4. 因为默认情况下， ViewGroup 中的 onInterceptTouchEvent 方法返回的 false ，即不拦截，所以事件会传递到子元素中

当 ViewGroup 不拦截事件的时候，事件会下发交由他的子 View 进行处理。

```java
final int childrenCount = mChildrenCount;
if (newTouchTarget == null && childrenCount != 0) {
    ...
    final View[] children = mChildren;
    for (int i = childrenCount - 1; i >= 0; i--) {//遍历整个子元素
      final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
      ...
      //判断子元素是否能接收点击事件：子元素是否在播放动画和点击坐标点是否在子元素内
      if (!canViewReceivePointerEvents(!isTransformedTouchPointInView(x, y , child , null))) {
          ev.setTargetAccessibilityFocus(false);
          continue;
      }

      newTouchTarget = getTouchTarget(child);
      if (newTouchTarget != null) {
          newTouchTarget.pointerIdBits |= idBitsToAssign;
          break;
      }

      resetCancelNextUpFlag(child);
      if (dispatchTransformedTouchEvent(ev, false , child , idBitsToAssign)) {
        //子元素的 dispatchTouchEvent() 返回的是true
        ...
        //mFirstTouchTarget 被赋值，并且跳出循环
        newTouchTarget = addTouchTarget(child, idBitsToAssign);
        alreadyDispatchedToNewTouchTarget = true;
        break;
      }
    }
  }
```
1. 遍历整个 ViewGroup 的子元素，然后判判断子 View 是否能够接受点击事件,判断条件有两个
  * 子元素是否在播放动画
  * 点击事件的左边是否落在子元素区域内   
2. 如果能够接收点击事件，那么事件交由他来处理，执行 view 的 dispatchTransformedTouchEvent() ，其实该方法内部，同样是调用的子 View 的 dispathTouchEvent() 方法，这样就完成了一轮事件分发

### View # dispatchTransformedTouchEvent()
```java
if (child == null) {
    handled = super.dispatchTouchEvent(event);
} else {
    handled = child.dispatchTouchEvent(event);
}
```            
因为传递过来的 View 不为 null ，所以调用了子 View 的child.dispatchTouchEvent 方法，完成了一轮分发

如果 dispatchTransformedTouchEvent() 返回 true ,那么 mFirstTouchTarget 就会被赋值，同时跳出循环
```java
if (dispatchTransformedTouchEvent(ev, false , child , idBitsToAssign)) {
  ...
  newTouchTarget = addTouchTarget(child, idBitsToAssign);
  alreadyDispatchedToNewTouchTarget = true;
  break;
}
```
这个 mFirstTouchTarget 前面说了一点点，现在又遇到了，就好好看看这家伙吧。
### mFirstTouchTarget
mFirstTouchTarget 其实是一个单链表结构。他的赋值过程是在 addTouchTarget() 中完成的,addTouchTarge()会根据传递过来的子 View 和 pointerIdBits 创建一个 target ,然后把这个 target 赋值给mFirstTouchTarget。

```java
private TouchTarget addTouchTarget(View child, int pointerIdBits) {
    TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
```
原来，子 View 调用 dispatchTouchEvent 返回 true ，就会根据该子 View ，创建一个 Target ，并把这个 target 赋值给 mFirstTouchTarget ， mFirstTouchTarget 是否赋值，直接就影响到 ViewGroup 的拦截策略，如果 mFirstTouchTarget 为 null ,那么 ViewGroup 就默认拦截下来同一事件序列所有的点击事件

如果遍历所有的子元素后，事件都没有被合适的处理，包含两种情况
1. 该 ViewGroup 没有子元素**或者子元素没有处理点击事件**，
2. 有子元素并且处理了点击事件，但是在 dispatchTouchEvent() 中返回了 false ，这一般就是因为子元素在 onTouchEvent() 中返回了false      

这两种情况下， ViewGroup 会自己处理点击事件，即调用
dispatchTransformedTouchEvent 方法，不过 View 传递的是 null ，则调用 super.dispatchTouchEvent(event);
```java
if (mFirstTouchTarget == null) {
  handled = dispatchTransformedTouchEvent(ev, canceled , null , TouchTarget.ALL_POINTER_IDS);
}
```
因为 child 是 null ，所以会执行 super.dispathTouchEvent(),这个 super 也就是 View ， ViewGroup extends View ,也就是说最终会执行到 View 的dispatchTouchEvent()

### 容易被遗漏的 CANCEL 事件
当父视图的 onInterceptTouchEvent 先返回 false，然后在子 View 的 dispatchTouchEvent 中返回 true（表示子 View 捕获事件），关键步骤就是在接下来的 MOVE 的过程中，父视图的 onInterceptTouchEvent 又返回 true，intercepted 被重新置为 true，此时子控件就会收到 ACTION_CANCEL 的 touch 事件

我们平时自定义View时，尤其是有可能被ScrollView或者ViewPager嵌套使用的控件，不要遗漏对CANCEL事件的处理，否则有可能引起UI显示异常。

### View对点击事件的处理过程
需要说明的两点
1. 这里的 View 不包括ViewGroup
2. 没有 onIntercepeTouchEvent ,因为不需要拦截。只能由自己处理

dispatchTouchEvent() 主要源码如下
```java
if (onFilterTouchEventForSecurity(event)) {
  ListenerInfo li = mListenerInfo;
  if (li != null && li.mOnTouchListener != null
    && (mViewFlags & ENABLED_MASK) == ENABLED//可用状态
    && li.mOnTouchListener.onTouch(this, event)) {//onTouch()返回true
      result = true;
  }
  if (!result && onTouchEvent(event)) {
      result = true;
  }
}
```
1. 首先判断是否设置了 mOnTouchListener ,如果设置了并且是 View 是可用状态（(mViewFlags & ENABLED_MASK) == ENABLED&&），则调用 onTouch()，如果该方法返回了 true ,则事件不在往下传递，直接返回，如果 View 不可用，即 enable 为 false ，那么就会执行 onTouchEvent() ,因为 result 默认就是 false 。
2. 如果没有设置 mOnTouchListener 或者 onTouch() 返回了 false ，则调用 onTouchEvent() ,这也说明了 mOnTouchListener 优先级高于 onTouchEvent() ,这样做的是为了方便外界处理点击时间

### View # onTouchEvent()

1. 首先处理不可用状态下的点击事件。不可用情况下，照样可以消耗点击事件，只要 CLICKABLE 或者 LONG_CLICKABLE 有一个为 true ，但是 onClick() , onLongClick() 等实质性逻辑不会执行，此时 onTouchEvent() 的返回值由该 View 是否可点击(包括长按，短按等点击事件)来决定。
```java
if ((viewFlags & ENABLED_MASK) == DISABLED) {//不可用，
    if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
        setPressed(false);
    }
    return (((viewFlags & CLICKABLE) == CLICKABLE
            || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
            || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
}
```            
2. 如果设置了代理，执行 TouchDelegate 的 onTouchEvent 方法，这个和 onToucheListener 类似
```java
if (mTouchDelegate != null) {
    if (mTouchDelegate.onTouchEvent(event)) {
        return true;
    }
}
if (((viewFlags & CLICKABLE) == CLICKABLE || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
    //会执行 onClick()  onLongClick()
    return true;
}
return false;
```
当 View 是不可点击的时候，想要消耗掉该事件唯一的做法就是 设置 setTouchDelegate() ,并且 mTouchDelegate. onTouchEvent(event)为 true 。否则 onTouchEvent() 必定返回 false ，具体逻辑，如 onClick() , onLongClick() 也不会执行。


3. 接下来看一下 onTouchEvent() 中对点击事件的具体处理,只要 CLICKABLE 或者 LONG_CLICKABLE 有一个为 true ，那么最后就会消耗这个事件，因为最后返回的是 true 。
```java
if (((viewFlags & CLICKABLE) == CLICKABLE ||
      (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
      (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
      return true;
}
```
4. 如果能进入判断内，说明肯定会消耗这个事件，但是在事件 ACTION_UP 上，会触发 performClick() ，    
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
5.  如果 View 中设置了 onClickeListener 那么就会触发 onCLick 方法， performClick() 方法源码，
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
6. View的 LONG_CLICKABLE 属性默认为 false ， CLICKABLE 属性与具体的 View 有关，确切的说，可点击的 View 其 CLICKABLE 为 true ，比如 button ，不可点击的 View ，其 CLICKABLE 为 false ，例如 TextView ，可以通过 setOnClickListener 和 setOnLongClickListener 然后进行设置源码如下,   
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

## enable 属性和 Clickable 属性
* enable 可不可用    
* clickable 可不可点击   

都是对 mPrivateFlags 进行操作的，判断 View 是否可用或者是否可点击，也是通过 mPrivateFlags 进行操作的，
mPrivateFlags & CLICKABLE == CLICKABLE  代表控件是可点击的，
mViewFlags & ENABLED_MASK == ENABLED   代表控件是可用的。

1. 当 View 不可用的时候，通过 setOnTouchListener设置的OnTouchListener中的 onTouch()方法将不会执行。参考 View 中的 dispatchTouchEvent() 中的代码
2. 当 View 不可用的时候， onTouchEvent() 会执行，但是 onClick() , OnLongClick() 等实质性逻辑不会执行，此时 onTouchEvent() 的返回值由该 View 是否可点击(包括长按，短按等点击事件)来决定。参考 View 中的onTouchEvent()
3. 当 View 是不可点击的时候，除非调用了 View 的 setTouchDelegate() ,传入 mTouchDelegate ,否则 onTouchEvent() 必定返回 false ，具体逻辑，如 onClick() , onLongClick() 不会执行。

## 总结
大致的流程，伪代码如下：
```java
public boolean dispatchTouchEvent(MotionEvent event) {
    //执行 onInterceptTouchEvent 方法
    if(event=ACTION_DOWN){
      //做一些清理操作，包括重置FLAG_DISALLOW_INTERCEPT
      reset();
    }
    boolean consume=false;
    if(event==ACTION_DOWN||mFristTarget!=null){
      //没有设置 FLAG_DISALLOW_INTERCEPT 为true
      if(!requestDisallowInterceptTouchEvent()){
        consume=onInterceptTouchEvent();
      }
    }
    if(!consume){
      for(){//循环遍历，找到是否有 View 能消耗，
        //条件，在该 View 的范围内，并且没有执行动画
        if(child !=null&&child.dispathTouchEvent()){
          mFristTarget=child;
          break;
        }
      }
    }

    if(mFristTarget==null){
      // 执行 super.dispathTouchEvent(); 这也是 View 的 dispatchTouchEvent() 逻辑
      //判断是否设置了 onTouchListener 。
      if (mTouchListener != null&&ENABLED && mTouchListener.onTouch(this, event)) {//设置了，调用 onTouch 事件并且返回 true ，消耗时间
          return true;
      } else {//调用 onTouchEvent() 事件，也就是说 onTouch 会屏蔽到onTouchEvent
          if(DISABLED){//不可点击
            return clickable;
          }
          if(clickable){
            if (mOnClickListener != null) {//如果设置了 onClickListener ，
                mOnClickListener.onClick(this);//调用 onClick 方法
            }
            if(mOnLongClickListener!=null){
              mOnLongClickListener.setOnClickListener(this);
            }
            return true;
          }
          return false;
      }
    }
}
```

---
搬运地址：    

Android 开发艺术探索    

[Android中 View 的 Clickable 和 Enabled 的区别与原理](https://blog.csdn.net/weixin_37077539/article/details/54895485)
