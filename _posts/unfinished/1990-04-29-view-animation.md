---
layout: post
title: View 动画
category: 待整理
tags: DataBindng
---
* content
{:toc}
Android 动画分三种
* View 动画  通过对场景里面的对象不断做图像转换（平移，缩放，旋转，透明度）从而产生动画效果，是一种渐进式动画
* 帧动画  属于View动画的一种，只不过他和平移，旋转等常见的View动画表现形式略有不同。通过顺序播放一系列图像从而产生动画效果，可以简单的理解为图片的切换动画，如果图片过大，可能导致OOM
* 属性动画 通过动态改变对象的属性从而达到动画效果，API11 新特性，低版本可以使用兼容库


## View 动画
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
![](http://p5sfwb51p.bkt.clouddn.com/animation_all.png)

###  `<set>` 标签
表示动画集合，对应AnimationSet类，可以保护若干动画，并且内部可以嵌套其他动画集合，他的两个属性的含义如下

### android：interpolator
表示动画集合采用的差值器，插值器影响动画的速度，如果非均匀的动画，需要通过插值器控制动画的播放过程，这个属性可以不指定，默认的是`@android：anim/accelerate_decelerate_interpolator`,即加速减速插值器，
### android:shareInterpolator
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

轴点  默认情况下来，轴点是View的中心点，这个时候在水平方向缩放会导致View向左右两个方向同时进行缩放，但是如果设置了轴点设置为View的右边界，那么View就只能向左进行缩放，反之亦如此。

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
继承Animation,然后重写 initialize()和applyTransformation()即可，
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
顺序播放一组预先定义好的图片，类似电影播放，注意该动画是放到drawable文件下面的，而是不anim文件夹下面，
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
注意： 帧动画使用简单，但是比较容易引起OOM，尽量避免使用过多过大的图片


---
搬运地址：
