---
layout: post
title: 扫盲系列 - AppBarLayout
category: 扫盲系列
tags: AppBarLayout  CoordinatorLayout  ConstraintLayout
---
* content
{:toc}

## CoordinatorLayout 和 ConstraintLayout 区别

### CoordinatorLayout
Material Design 的根布局，作为协调 Material 下所有控件的动画联动，所以被称为协调者布局    
继承ViewGroup，是加强 FramLayout,可以协调其他控件，并且实现控件之间的联动，通过在其直接子View中设置behevior来实现子 View 的不同交互效果，  
一般作为一个界面的根部局，用来协调 AppBarLayout ,ToolBarLayout,以及 Scrollview 之间的联动


### ConstraintLayout
google 为了将布局扁平化，减少嵌套而设计的约束布局。


## AppBarLayout 特性
是一种支持响应滚动手势APP bar 布局



AppBarLayout中所有的子View，如果没有设置 layout_scrollFlags 这个属性，都会悬停到顶部

### app:layout_scrollFlags

* scroll
设置为scroll的View 会随着滚动事件一起发生移动，就是当指定的Scrollview发生滚动时，改View也跟着一起滚动，就好像这个View也是属于这个Scrollview一样
* enterAlawys
当任何时候Scrollview往下滚动的时候，该View会直接往下滚动，而不用考虑Scrollview 是否在滚动到顶部位置还是其他位置。  
* exitUntilCollapsed
Collapse   坍塌，瓦解
当这个View要往上逐渐消逝时候，会一直往上滑动，知道剩下的高度达到他的最小高度，再响应Scrollview的内部滑动事件
简单来说，当Scrollview滑动的时候，首先是View把滑动事件夺走，当View执行滑动，当滑动最小高度后，再把这个滑动事件还回去，让Scrollview内部上滑
* enterAlawysCollapsed
是enterAlways 的附加选项，一般跟enterAlways一起使用，它指View在往下出现的时候，首先是enterAlways效果，当View的高度达到最小高度是，View 就暂时不去往下滚动，知道Scrollview滚动到顶部再滑动，View再继续往下滑动，知道滑动View 顶部结束
* snap
Child View 滚动比例的一个吸附效果，也就是说，ChildView 不会存在局部显示的情况，要么向上全部滚出屏幕，要么向下滚进屏幕，有点类似Viewpager的左右滑动


## CollapsingToolBarLayout  
则是专门用来实现子布局内不同元素响应滚动细节的布局
对AppBarLayout进行的再次包装的ViewGroup，需要放到AppBarLayout布局里面，并且作为直接子View，
主要功能包括
* 折叠Title (Collapsing Title) 当布局内容全部显示出来时，title 是最大的，但是随着View逐步移除屏幕顶部，title变得越来越小，可以通过setTitle()方法来设置title
* 内容纱布(Content scrim) 根据滚动的位置是否到达一个阈值，来决定是否对View 盖上纱布，可以通过setContentScrim(Drawable) 来设置纱布的图片，默认contentScrim是colorPrimary的色值
* 状态栏纱布（status bar scrim） 根据滚动位置是否到达一个阈值决定是否对状态栏覆盖纱布，你可以通过setStatusBarScrim(Drawable)来设置纱布的图片，但是只能在LOLLIPOP设备上有作用，默认是statusBarScrim是colorPrimaryDark
* 视差滚动子View（Parallax scrolling children ） 子View可以选择在当前的布局当时是否以视差的方式跟随滚动。其实就是让这个View的滚动速度比其他正常滚动的View速度稍微慢一点，将布局参数app:layout_collapseMode设为parallax
* 将子View位置固定（Pinned position children） 子View可以选择是否在全局空间上固定位置，这对于Toolbar 来说非常有用，因为在当布局移动时，可以将Toolbar固定位置而不受移动的影响，将app:layout_collapseMode设为pin



## Behavior 的原理

CoordinatorLayout 的使用核心是behavior,
理解两个核心
1. Child    子View，就是CoordinatorLayout父布局下要执行动作的子View，是 观察者
2. Dependency  值Child 依赖的View，也是 被观察者
简单说。如果Dependcy 这个View 发生了变化，那么Child这个View也就要发生相应的变化。具体变化就是Behavior引入

Child 发生的变化的具体执行代码就是放在Behavior类里面的

### 自定义Behavior

1. 定义一个类，继承Coordinator.Behavior<T> ,其中泛型参数T 就是我们要执行的动作 View 类，也就是Child
2. 实现Behavior的两个方法

```java
/**
* 判断child的布局是否依赖dependency
*/
    @Override
public boolean layoutDependsOn(CoordinatorLayout parent, T child, View dependency) {
    boolean rs;
    //根据逻辑判断rs的取值
    //返回false表示child不依赖dependency，ture表示依赖
    return rs;
}

/**
* 当dependency发生改变时（位置、宽高等），执行这个函数
* 返回true表示child的位置或者是宽高要发生改变，否则就返回false
*/
    @Override
public boolean onDependentViewChanged(CoordinatorLayout parent, T child, View dependency) {
    //child要执行的具体动作
    return true;
}
```










---
搬运地址：    


[AppbarLayout的简单用法](https://www.jianshu.com/p/bbc703a0015e)

[Android 开发 CoordinatorLayout 协调者布局 与 ConstraintLayout 约束布局 两者的关系](https://www.cnblogs.com/guanxinjing/p/10158562.html)

[Android 详细分析AppBarLayout的五种ScrollFlags](https://www.jianshu.com/p/7caa5f4f49bd)

[Android CoordinatorLayout Behavior](https://www.jianshu.com/p/4ebb7bfa1228)
