---
layout: post
title: View 动画
category: 读书笔记
tags: 动画
---
* content
{:toc}
Android 动画分三种
* View 动画  通过对场景里面的对象不断做图像转换（平移，缩放，旋转，透明度）从而产生动画效果，是一种渐进式动画
* 帧动画  属于View动画的一种，只不过他和平移，旋转等常见的View动画表现形式略有不同。通过顺序播放一系列图像从而产生动画效果，可以简单的理解为图片的切换动画，如果图片过大，可能导致OOM
* 属性动画 通过动态改变对象的属性从而达到动画效果，API11 新特性，低版本可以使用兼容库

这篇主要介绍 View 动画，下篇介绍 [ View 属性动画 ](../../../../../article-detail/2019/07/27/view-animator/)

# View 动画
支持四种效果，平移动画，旋转动画，缩放动画和透明度动画，分别对应四个类，TranslateAnimation，RotateAnimation， ScaleAnimation，AlphaAnimation
可以通过代码动态创建，也可以通过XML定义，建议使用XML定义，

|名称|标签 |子类|效果|
|:----|:------|:------|:------|
|平移动画|`<translate>`|TranslateAnimation|移动View|
|旋转动画|`<rotate>`|RotateAnimation|放大或者缩小View|
|缩放动画|`<scale>`|ScaleAnimation|旋转View|
|透明度动画|`<alpha>`|AlphaAnimation|改变View的透明度|

使用View动画，创建创建XML文件，路径res/anim/filename.xml  
View 动画的固定语法,既可以是单个动画，也可以是组合动画，
![](../../../../../article-detail/images/animation_all.png)

这里面就使用到了 set 标签，translate 标签，alpa 标签 ，scale 标签，rotate 标签
## 常用的动画标签
###  `<set>` 标签
表示动画集合，对应AnimationSet类，可以保护若干动画，并且内部可以嵌套其他动画集合，他的两个属性的含义如下

* android：interpolator
表示动画集合采用的差值器，插值器影响动画的速度，如果非均匀的动画，需要通过插值器控制动画的播放过程，这个属性可以不指定，默认的是`@android:anim/accelerate_decelerate_interpolator`,即加速减速插值器，
* android:shareInterpolator
表示集合中动画和集合共享一个插值器，如果集合不指定插值器，那么动画就需要单独指定所需要的插值器或者使用默认值。

### `<translate>` 标签
表示平移动画，对应TranslationAnimation类，它可以使一个View在水平和竖直方向完成动画效果，一系列属性如下
* android:fromXDelta="" ---- 表示X的起始值，比如0；
* android:toXDelta=""   ---- 表示X的结束值 比如 2；
* android:fromYDelta="" ---- 表示Y的起始值
* android:toYDelta=""   ---- 表示Y的结束值

### `<scale>` 标签
表示缩放动画，对应ScaleAnimation类，它可以使一个View具有放大或者缩小的动画效果，一系列属性如下
* android:fromXScale=""  ---- 水平方向缩放的起始值 比如05.
* android:toXScale=""    ---- 水平方向缩放的结束值 比如2.0
* android:fromYScale=""  ---- 竖直方向缩放的起始值
* android:toYScale=""    ---- 竖直方向缩放的结束值
* android:pivotX=""      ---- 缩放的轴点的X坐标，它会影响缩放效果
* android:pivotY=""      ---- 缩放的轴点的Y坐标，它会影响缩放效果

轴点:  默认情况下来，轴点是View的中心点，这个时候在水平方向缩放会导致View向左右两个方向同时进行缩放，但是如果设置了轴点设置为View的右边界，那么View就只能向左进行缩放，反之亦如此。

### `<rotate>` 标签
表示旋转动画，对应RotateAnimation类，它可以使一个View具有旋转动画效果，一系列属性如下
* android:pivotY=""       ---- 旋转的轴点的X坐标
* android:pivotX=""       ---- 旋转的轴点的、坐标
* android:fromDegrees=""  ---- 旋转开始的角度 比如0
* android:toDegrees=""    ---- 旋转结束的角度 比如 180

轴点，会影响旋转效果，扮演旋转轴的角色，默认轴点是View的中心点，不同的轴点的旋转轨迹不同。

### `<alpha>` 标签
表示透明度动画，对应AlphaAnimation类，它可以改变View的透明度，一系列属性如下
* android:fromAlpha="" ---- 表示透明度的起始值 ，比如0.1
* android:toAlpha=""   ---- 表示透明度的结束值 ，比如1

### 其他常用属性
* android:duration="" ---- 动画持续时间
* android:fillAfter="" ---- 动画结束后是否停留在结束位置，false 表示不停留，

一个实际的例子

```xml
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
     android:fillAfter="true">

    <translate
        android:duration="700"
        android:fromXDelta="100"
        android:fromYDelta="0"
        android:interpolator="@android:anim/linear_inerpolator"
        android:toXDelta="100"
        android:toYDelta="0"/>

    <rotate
        android:duration="400"
        android:fromDegrees="90"
        android:toDegrees="0"/>

</set>
```
在代码中使用
```java
Animation animation= AnimationUtils.loadAnimation(this,R.anim.animation_all);
mPurchaseBtn.setAnimation(animation);
```
### 代码中定义动画
```java
AlphaAnimation alphaAnimation=new AlphaAnimation(0,1);
alphaAnimation.setDuration(400);

TranslateAnimation translateAnimation=new TranslateAnimation(0,100,0,100);
translateAnimation.setInterpolator(new LinearInterpolator());
translateAnimation.setDuration(700);

set.setFillAfter(true);
set.addAnimation(alphaAnimation);
set.addAnimation(translateAnimation);

mPurchaseBtn.startAnimation(set);
```
这个效果和上边XML效果是一致的。   
### 设置动画监听
通过Animation的setAnimationListener()设置动画监听。如下：
```java
set.setAnimationListener(new Animation.AnimationListener() {
      @Override
      public void onAnimationStart(Animation animation) {

      }

      @Override
      public void onAnimationEnd(Animation animation) {

      }

      @Override
      public void onAnimationRepeat(Animation animation) {

      }
    });
```
## 自定义View动画
继承Animation,然后重写 initialize() 和 applyTransformation() 即可，
* initialize()  做一些初始化的工作
* applyTransformation() 进行相应的矩阵变幻

```java
class CustomAnimation extends Animation{
  @Override
  public void initialize(int width, int height, int parentWidth, int parentHeight) {
    super.initialize(width, height, parentWidth, parentHeight);
  }

  @Override
  protected void applyTransformation(float interpolatedTime, Transformation t) {
    super.applyTransformation(interpolatedTime, t);
  }
}
```

## 帧动画
顺序播放一组预先定义好的图片，类似电影播放    
注意： 该动画是放到 drawable 文件下面的，而是不 anim 文件夹下面，
```xml
<?xml version="1.0" encoding="utf-8"?>
<animation-list xmlns:android="http://schemas.android.com/apk/res/android">
    <item
        android:drawable="@drawable/text_color_1"
        android:duration="100"/>
    <item
        android:drawable="@drawable/text_color_2"
        android:duration="100"/>
    <item
        android:drawable="@drawable/text_color_3"
        android:duration="100"/>
    <item
        android:drawable="@drawable/text_color_4"
        android:duration="100"/>
</animation-list>
```
将上述的Drawable作为View的背景并通过Drawable来播放动画
```java
mPurchaseBtn.setBackgroundResource(R.drawable.frame_animation);
AnimationDrawable drawable= (AnimationDrawable) mPurchaseBtn.getBackground();
drawable.start();
```
注意： 帧动画使用简单，但是比较容易引起 OOM，尽量避免使用过多过大的图片
# View 动画的特殊场景
## LayoutAnimation
作用于ViewGroup，为ViewGroup指定一个动画，这样当子元素出现时就具有该动画效果，常用在 ListView 上，遵循以下步骤
1. 定义layoutAnimation，如下所示

```xml
<?xml version="1.0" encoding="utf-8"?>
<layoutAnimation
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:animation="@anim/item_animation"
    android:animationOrder="normal"
    android:delay="0.5">

</layoutAnimation>
```

* android:delay 子元素开始动画的延迟时间。如果子元素入场动画的时间是300ms,那么0.5表示延迟150ms，总的来说，第一个延迟150ms播放动画，那么第二个延迟300ms播放动画，后面依次类推
* android:animationOrder 子元素动画的顺序。有三个选项
  * normal 顺序显示   即排在前面的子元素先开始播放动画
  * reverse  倒序显示
  * random，  随机显示
* android:animation  为子元素指定具体入场动画


2. 为子元素指定具体的入场动画
```xml
<?xml version="1.0" encoding="utf-8"?>
<set
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="300"
    android:interpolator="@android:anim/linear_inerpolator"
    android:shareInterpolator="true">

    <alpha android:fromAlpha="0"
           android:toAlpha="1.0"/>
    <translate
        android:fromXDelta="500"
        android:toXDelta="0"/>
</set>
```
3. 为ViewGroup指定layoutAnimation属性,对于ListView来说，每个 item 就具有了出场动画了，这种方式适合所有的ViewGroup，
```xml
<ListView
       android:layoutAnimation="@anim/viewgroup_animation"
       android:layout_width="match_parent"
       android:layout_height="match_parent"/>
```

除了XMl中可以指定layoutAnimation之外，可以在代码中设置，通过 LayoutAnimationController 来实现，代码如下
```java
ListView listView=findViewById(R.id.lv);
Animation animation= AnimationUtils.loadAnimation(this,R.anim.viewgroup_animation);
LayoutAnimationController layoutAnimationController=new LayoutAnimationController(animation);
layoutAnimationController.setDelay(0.5f);
layoutAnimationController.setOrder(LayoutAnimationController.ORDER_NORMAL);
listView.setLayoutAnimation(layoutAnimationController);
```

## Activity的切换效果
尽管Activity有默认的切换效果，但是我们可以自定义切换效果的，主要是使用 overridePendingTransition(int enterAnim,int exitAnim),这个方法必须在 startActivity() 之前或者 finish() 之后调用才生效
* enterAnim - Activity打开时候，所需要的资源id
* exitAnim - Activity关闭的时候，所需要的资源id

```java
// 在startActivity()之前
intent.setClass(UserActivity.this, clz);
overridePendingTransition(R.anim.enter_anim,R.anim.exit_anim);
startActivity(intent);

// 或者在 finish()之后
@Override
public void finish() {
   super.finish();
   overridePendingTransition(R.anim.enter_anim,R.anim.exit_anim);
}
```

注意 Fragment也可以切换动画的，可以使用FragmentTransaction  中的 setCustomAnimation() 来添加动画的。


---
搬运地址：    

Android 开发艺术探索
