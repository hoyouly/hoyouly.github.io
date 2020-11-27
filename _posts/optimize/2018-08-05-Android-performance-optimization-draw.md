---
layout: post
title: Android 性能优化 -- 绘制优化
category: 性能优化
tags: 性能优化
description: Android 性能优化
---

<!-- * content -->
<!-- <!-- {:toc} -->
绘制优化，主要从两个方面进行优化
1. 降低View.onDraw()的复杂度
2. 避免过度绘制（OverDraw）


# 降低View.onDraw()的复杂度
从两个方面优化降低复杂度。
1. onDraw() 中不要创建新的局部对象     
因为 onDraw 可能会被频繁调用，这样就会一瞬间产生大量临时对象，这样不仅占用过多的内存而且还导致系统频繁 GC ，降低程序的执行效率
2. 避免 onDraw() 执行大量耗时的工作    
因为这样会抢占 CPU 时间片，从而导致 View 的绘制过程不流畅。

# 避免过度绘制（OverDraw）
过渡绘制就是屏幕上某一个像素，在同一帧的时间内，被绘制了多次

原因： 在多层次的 UI 中，若不可见的 UI 也在做绘制操作，则会导致某些像素被绘制多次。
## 检测工具
### 调试 GPU 过度绘制
打开： 开发出选项 -> 调试 GPU 过度绘制 -> 显示 GPU 过度绘制。
可以查看过度绘制会导致屏幕显示的色块不同。

![](../../../../images/overdraw.png)

* 紫色： 代表 1 层覆盖，像素绘制了两次，大片蓝色还是可以接受，若整个窗口都是蓝色，可以摆脱一层
* 绿色： 代表 2 层覆盖，像素绘制了三次，中等大小的绿色还是可以接受，但是应该尝试优化，减少他们
* 粉色： 代表 3 层覆盖，像素绘制了四次，小范围可以接受
* 红色： 代表 4 层覆盖，像素绘制了至少 5 次，这是错误的，要修复他们。

* 过度绘制优化的原则
1. 尽可能的控制过度绘制的次数小于等于 2 次，绿色以下，紫色最好
2. 尽可能避免过度绘制的粉色和红色情况
3. 不允许 3 次以上的过度绘制(淡红色)面积超过屏幕大小的1/4

### GPU呈现模式分析
打开：  开发出选项 -> GPU呈现模式分析 -> 在屏幕上显示为条形图。
<img src="../../../../images/gpu_modle.png"  height="600" width="400">
1. Activity 区域有一根绿色横线，代表 16ms ，我们需要确保每一帧花费都低于这根横线，这样才能避免卡顿的问题。   
2. 界面显示垂直的柱状图表示每一帧画面渲染需要的时间，柱状图越高，表示花费渲染的时间越长。每种颜色的意义如下
<img src="../../../../images/gpu_color_expl.png"  height="400" width="600">
如上图，我们主要关注的就是蓝色区域。蓝色代表绘制所花费的时间，代表视图在界面发生更新的用时情况。<font color="#FF0000">蓝色对判断流畅度的参考意义很大，当它越长，说明当前视图比较复杂或者无效重复绘制，即我们通常说的卡了。</font>

## 优化方案
### 移除默认的 Window 背景

* 背景
一般情况下，默认继承主题 windowBackground ,如默认的 Light 主题

```xml
<style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
    <!-- Customize your theme here. -->
    <item name="colorPrimary">@color/colorPrimary</item>
    <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
    <item name="colorAccent">@color/colorAccent</item>
</style>
<style name="Theme.AppCompat.Light.DarkActionBar" parent="Base.Theme.AppCompat.Light.DarkActionBar"/>

<style name="Base.Theme.AppCompat.Light.DarkActionBar" parent="Base.Theme.AppCompat.Light">
  ...
</style>

<style name="Base.Theme.AppCompat.Light" parent="Base.V7.Theme.AppCompat.Light"/>

<style name="Base.V7.Theme.AppCompat.Light" parent="Platform.AppCompat.Light">
  ...
</style>

<style name="Platform.AppCompat.Light" parent="android:Theme.Holo.Light">
  ...
  <item name="android:windowBackground">@color/background_material_light</item>
  ...
</style>
```
如上，我们常用的 AppTheme ,经过多层的继承后，发现了设置的 windowBackground 属性。

* 问题
一般情况下，该默认的 window 背景基本用不上，因为背景都会在具体的 Layout 布局中设置，若不移除，则导致所有界面都多绘制 1 层

* 解决方案
移除默认的 Window 背景

```xml
<!-- 方式1：在应用的主题中添加如下的一行属性-->
<item name="android:windowBackground">@android:color/transparent</item>
<!-- 或者 -->
<item name="android:windowBackground">@null</item>

<!--方式2：在 BaseActivity 的 onCreate() 方法中使用下面的代码移除 -->
getWindow().setBackgroundDrawable(null);
<!-- 或者 -->
getWindow().setBackgroundDrawableResource(android.R.color.transparent);
```

### 移除控件中不必要的背景
子 View 和父 View 如果背景相同，可移除子 View 的布局中的背景设置。

例如 ListView 和 Item 之间，如果背景相同，就没必要设置 Item 的背景了

例如 Viewpager + Fragment，如果每个 Fragment 都设置了背景色，则 Viewpager 没必要设置，可以移除。

### 减少布局文件的层级，减少不必要的嵌套
减少不必要的嵌套， UI 层级就少，过度绘制的可能性就降低    
优化方式，就是使用 merge 标签 并且选择合适的布局类型    

详情可以看 [ Android 性能优化 -- 布局优化 ](../../../../2018/07/17/Android-performance-optimization-layout/)
### 自定义控件 View 优化，
可以使用 clipRect() ,quickReject()

* clipRect() 的效果如下()
1. 给 Canvas 设置一个裁剪区域，只有在该区域的内才会被绘制，区域外的都不会被绘制

* quickReject()
判断和某个矩形相交
如果相交，则跳过相交区域，从而减少过度绘制。

## 优化的例子

自定义了一个 View ，显示三张图片，代码如下

```java
public class CardView extends View {
    private Bitmap[] mCards = new Bitmap[3];
    private int[] mImgId = new int[]{R.mipmap.carsetting_2, R.mipmap.carsetting_2, R.mipmap.carsetting_2};
    public CardView(Context context) {
        this(context, null);
    }
    public CardView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs , 0);
    }
    public CardView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs , defStyleAttr);
        for (int i = 0; i < mCards.length; i++) {
            Bitmap bm = BitmapFactory.decodeResource(getResources(), mImgId[i]);
            mCards[i] = Bitmap.createScaledBitmap(bm, 540 , 960 , false);
        }
        setBackgroundColor(Color.WHITE);
    }
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        canvas.save();
        canvas.translate(10, 200);
        for (int i = 0; i < mCards.length; i++) {
            canvas.translate(110, 0);
            canvas.drawBitmap(mCards[i], 0 , 0 , null);
        }
        canvas.restore();
    }
}
```
里面加载的同一张图片，就是 GPU 呈现模式分析 中柱状图颜色说明的那张图片。

我们先看看 xml 的布局
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/ll_root"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#FFFFFF"
    android:orientation="vertical">

    <top.hoyouly.fuckrxjava.view.CardView，
        android:layout_width="match_parent"
        android:layout_height="match_parent" />
</LinearLayout>

```
Activity 中直接加载布局。
```java
public class LayoutInflaterActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_layoutinfalter);
    }
}
```
效果如下。
<img src="../../../../images/screen_4.png"  height="600" width="400">
连紫色都没有，大片背景是绿色，说明多绘制了两层。图片显示的地方还有粉色，红色，这需要好好优化啊。
### 优化一 减少布局嵌套

我们发现 在 xml 中设置了父布局 LinearLayout 里面只有一个 CardView ，并且这个 CardView 还是 match_parent 的，其实完全可以把外层的 LinearLayout 去掉。直接加载 这个 CardView 即可。可以直接把这个 xml 文件删掉，直接在代码中加载，如下图
```java
public class LayoutInflaterActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(new CardView(this));
    }
}
```
处理后的效果如下
<img src="../../../../images/screen_1.png"  height="600" width="400">

大片背景是紫色，说明多了一层绘制。
接下来 我们处理第一步
### 优化二 默认背景移除
我们发现在 CardView 中设置了背景的 `setBackgroundColor(Color.WHITE)` 所以可以把默认背景移除。解决办法上面写道了。我们只使用其中一种，

在 setContentView() 下面添加 getWindow().setBackgroundDrawable(null) 。
```java
public class LayoutInflaterActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(new CardView(this));
        getWindow().setBackgroundDrawable(null);
    }
}
```
我们再看看效果
<img src="../../../../images/screen_2.png"  height="600" width="400">
大片的紫色背景变成了白色，说明默认背景已经移除。
但是还有一部分紫色，绿色，粉色，可以进行再次处理。那就要用到 clipRect() 了
### 优化三 使用 clipRect()

主要看 CardView 的 onDraw() 方法。
```java
public class CardView extends View {
    ...
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        canvas.save();
        canvas.translate(10, 200);
        for (int i = 0; i < mCards.length; i++) {
            canvas.translate(110, 0);
  //           canvas.drawBitmap(mCards[1], 0 , 0 , null);
            canvas.save();
            if (i < mCards.length - 1) {
                canvas.clipRect(0, 0 , 110 , mCards[i].getHeight());
            }
            canvas.drawBitmap(mCards[i], 0 , 0 , null);
            canvas.restore();
        }
        canvas.restore();
    }
}

```
注释掉的就是之前的的逻辑，下面就是新修改的。
然后再次看看效果
<img src="../../../../images/screen_3.png"  height="600" width="400">
只有白色和紫色了，就说明已经优化的很成功了。

### 总结
我们使用了三个优化措施，就处理了一个很常见的过渡绘制的问题。后续遇到这样的问题，可以参照上面的例子处理。

---   
搬运地址：    

[Android性能优化：那些不可忽略的绘制优化](https://www.jianshu.com/p/cbdaeb1bede5)   

[Android性能优化--过度绘制](https://blog.csdn.net/zxc123e/article/details/71750786?depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-3&utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-3)
