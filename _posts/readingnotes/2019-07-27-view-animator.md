---
layout: post
title: View 属性动画
category: 读书笔记
tags: ValueAnimator
---
<!-- * content -->
<!-- {:toc} -->

上篇介绍了 [ View动画 ](../../../../2019/07/25/view-animation/)，这篇就主要说 属性动画

## 属性动画

可以对任意对象的属性进行动画而不仅仅是 View 。动画的默认时间是 300ms ，默认帧率是10ms/帧，

其可以达到的效果就是在一个时间间隔内，完成对象从一个属性值到另外一个属性值的改变，因此，属性动画几乎是无所不能的。

只要对象有这个属性，就能实现动画效果。

属性动画是从 API11 开始的,可以使用 nineoldandroids 来兼容以前的版本。

常用的动画类
 ValueAnimator , ObjectAnimator 和 AnimatorSet   
其中。 ObjectAnimator 继承 ValueAnimator , AnimatorSet 是一组动画集合，可以定义一组动画。  

## 几个例子
* 让 mBt2 沿 Y 轴向上移动一段距离。
```java
ObjectAnimator.ofFloat(mBt2,"translationY",-mBt2.getHeight()).start();
```
* 修改背景色， 3 秒内 从 0xFFFF8080 这个渐变到 0xFF9090FF ，然后再渐变回来，并且无限循环
```java
private void animator() {
    ValueAnimator animator = ObjectAnimator.ofInt(mBt2, "backgroundColor", 0xFFFF8080 , 0xFF9090FF);
    animator.setDuration(3000);
    animator.setEvaluator(new ArgbEvaluator());
    //逆向动画效果
    animator.setRepeatMode(ValueAnimator.REVERSE);
    //执行次数，无限循环
    animator.setRepeatCount(ValueAnimator.INFINITE);
    animator.start();
}
```
* 旋转 平移 缩放，透明度 一块执行
```java
private void animator(View view) {
    AnimatorSet set = new AnimatorSet();
    set.playTogether(
        //旋转
        ObjectAnimator.ofFloat(view, "rotationX", 0 , 360),
        ObjectAnimator.ofFloat(view, "rotationY", 0 , 180),
        ObjectAnimator.ofFloat(view, "rotation", 0 , -90),
        //平移
        ObjectAnimator.ofFloat(view, "translationX", 0 , 90),
        ObjectAnimator.ofFloat(view, "translationY", 0 , 90),
        //缩放
        ObjectAnimator.ofFloat(view, "scaleX", 1 , 1.5f),
        ObjectAnimator.ofFloat(view, "scaleY", 1 , 0.5f),
        //透明度
        ObjectAnimator.ofFloat(view, "alpha", 1 , 0.25f, 1)

    );
    set.setDuration(3000).start();
}
```

属性动画的 XML 定义在res/animator/下面   

![](../../../../images/set_animator_xml.png)

* together 动画集合一起播放，是android:ording 的默认值
* sequentially  按照前后顺序播放

## `<objectAnimator>` 标签

* android:propertyName  属性动画的对象的属性名称
* android:duration     动画时长
* android:valueFrom   属性的起始值
* android:valueTo     属性的结束值
* android:startOffset  动画的延迟时间，当动画开始后，需要多长时间才真正播放该动画
* android:repeatCount   重复次数  -1 表示无限循环
* android:repeatMode    重复模式， 有两个模式， repeat 连续重复， reverse 逆向重复
* android:valueType   android:propertyName 所指定 的属性的类型，有 intType 和 floatType ,表示属性的的类型是整型或者浮点型，如果 android:propertyName 是 颜色，就不需要指定 android:valueType 了

使用

```java
AnimatorSet set1= (AnimatorSet) AnimatorInflater.loadAnimator(this,R.animator.pro_animator);
       set1.setTarget(view);
       set1.start();
```

## 插值器和估值器

插值器： 根据时间流逝的百分比来计算出当前属性改变的百分比
估值器： 根据当前属性改变的百分比来计算改变后的属性

系统预制的插值器
* LinearInterpolator 线性插值器，匀速动画
* AccelerateInterpolator 加速减速插值器，动画两头慢，中间块
* DecelerateInterpolator 减速插值器，动画越来越慢

系统预制的估值器
* IntEvaluator 针对整型属性
* FloatEvaluator 针对 float 属性
* ArgbEvaluator 针对 Color 属性

插值器和估值器 在属性动画中很重要，是实现非匀速动画的重要手段。

## 动画监听器

```java
set.addListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationCancel(Animator animation) {
                super.onAnimationCancel(animation);
            }

            @Override
            public void onAnimationEnd(Animator animation) {
                super.onAnimationEnd(animation);
            }

            @Override
            public void onAnimationRepeat(Animator animation) {
                super.onAnimationRepeat(animation);
            }

            @Override
            public void onAnimationStart(Animator animation) {
                super.onAnimationStart(animation);
            }

            @Override
            public void onAnimationPause(Animator animation) {
                super.onAnimationPause(animation);
            }

            @Override
            public void onAnimationResume(Animator animation) {
                super.onAnimationResume(animation);
            }
        });
```

可以只实现中一部分，需要哪个实现哪个即可

## 对任意属性做的动画

属性动画要求动画作用的对象提供该属性的 set 和 get 方法

如果想要 object 对象的属性 abc 做动画，必须满足两个条件：
1. object 类必须提供 setAbc 方法，如果动画的时候没有传递初始值，你们还要求提供 getAbc() 方法，因为系统要去 object 中属性的初始值，如果不满足，直接crash
2. object 类中 setAbc() 方法对属性 abc 所做的改变必须通过某种方法反映出来，比如带来 UI 的改变等，如果不满足这条，动画无效但是不会crash



setWidth() 是 TextView 和其子类的专属方法，它的作用不是设置 View 的宽度，而是设置 TextView 的最大宽度和最小宽度。
getWidth() 是获取 View 的宽度，

TextView 和 Button 的 setWidth() 和 getWidth() 干的不是同一件事

三种解决办法
* 给你的对象添加上set/get方法，如果有权限的话
* 用一个类包装原始对象，间接为其提供set/get 方法
* 采用 ValueAnimator ,监听动画过程，自己实现属性的改变

## 使用动画的注意事项

* OOM的问题
* 内存泄漏
尤其在属性动画无限循环的时候，这类要在 Activity 退出时及时停止。
* 兼容性问题
* View 动画的问题
View 动画是对 View 影像的改变，并不是真正改变 View 的状态，因此有时候出现动画完成后， View 无法隐藏的的现象，即setVisibily(View.GONE)失效，这个时候只需要调用view.clearAnimation()即可。
* 不要使用px
使用 px 会导致不同的设备出现不同的效果
* 动画元素的交互
View 动画移动后，点击事件还是在原来的位置，属性动画则是移动后的位置，
* 硬件加速
建议开始硬件加速，这样提高动画的流畅性。






---
搬运地址：    

Android 开发艺术探索
