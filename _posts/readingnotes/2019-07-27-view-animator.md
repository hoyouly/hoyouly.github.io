---
layout: post
title: View 属性动画
category: 读书笔记
tags: ValueAnimator
---
* content
{:toc}

上篇介绍了 [ View动画 ](../../../../2019/07/25/view-animation/)，这篇就主要说 属性动画

# 属性动画
可以对任意对象的属性进行动画而不仅仅是View。动画的默认时间是300ms，默认帧率是10ms/帧，

其可以达到的效果就是在一个时间间隔内，完成对象从一个属性值到另外一个属性值的改变，因此，属性动画几乎是无所不能的。

只要对象有这个属性，就能实现动画效果。

属性动画是从API11 开始的

ValueAnimator,ObjectAnimator 和 AnimatorSet, ObjectAnimator 继承 ValueAnimator,
AnimatorSet 是一组动画集合，可以定义一组动画。  
属性动画的XML定义在res/animator/下面   
如果想要object 对象的属性abc 做动画，必须满足两个条件：
1. object 类必须提供setAbc方法，如果动画的时候没有传递初始值，你们还要求提供getAbc()方法，因为系统要去object中属性的初始值，如果不满足，直接crash
2. object 类中setAbc()方法对属性abc所做的改变必须通过某种方法反映出来，比如带来UI的改变等，如果不满足这条，动画无效但是不会crash











---
搬运地址：    

Android 开发艺术探索
