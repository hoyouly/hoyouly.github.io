---
layout: post
title: Android 性能优化 -- 布局优化
category: 性能优化
tags: Android开发艺术探索 性能优化
description: Android 性能优化
---

<!-- * content -->
<!-- {:toc} -->

布局对 Android 性能的影响就是影响页面的测量和绘制时间。

对布局的优化，也主要包括以下几个方面进行
* 选择合适的布局类型
* 减少布局文件的层级，层级越少， Android 绘制的工作量越少，那么程序的性能自然越高
* 减少测量和绘制时间，
* 提高复用性

## 选择合适的布局类型

简单界面 使用 LinearLayout 和 FrameLayout   
复杂界面 使用 ConstraintLayout

LinearLayout 和 FrameLayout 能搞定的就使用这两个，
如果需要嵌套 才能实现，那就使用 ConstraintLayout ，而不是 RelativeLayout ，因为 ConstraintLayout 的性能比 RelativeLayout 更好。

## 减少布局文件层级
1. 删除布局中无用的空间和层级
2. 删除控件中无用的属性
2. 选择使用性能高较低的 ViewGroup ，如果 LinearLayout 和 RelativeLayout 都可以使用，那么就采用 LinearLayout ，因为 RelativeLayout 的功能比较复杂，他的布局过程需要花费更多的 CPU 时间，但是如果通过一个 LinearLayout 不能实现，需要嵌套方式完成，建议使用 RelativeLayout ，因为 ViewGroup 的嵌套相当于增加了布局的层级，不过现在都建议使用 ConstraintLayout 来替换 RelativeLayout 。
3. 采用 include 标签 ， merge 标签和 ViewStub 标签,
include 主要用于布局的重用， merge 一般和 include 配合使用，可以降低布局层级， ViewStub 提供按需加载的功能，
4. 尽可能少用 wrap_content ,因为 wrap_content 会增加布局 measure 时计算的成本，在已知宽高固定的情况下，不适用 wrap_content


### include 标签
可以将一个指定的布局文件加载到当前布局文件中，
* 只支持 android:layout_开头的属性，比如android:layout_width， android:layout_height，但是android:background 这样的属性就不支持，android:id是个特例
* 如果指定了android:layout_开头的属性，那么android:layout_width， android:layout_height 这两个属性必须存在

### merge 标签
一般和 include 标签一起使用，如果当前布局是一个竖直方向的 LinearLayout ，这个时候如果包含的布局文件也是采用竖直方向的 LinearLayout ，那么显然被包含的布局文件中的 LinearLayout 是多余的，通过 merge 标签就可以去掉多有的，
* 必须放在视图的根节点上
* 不是一个 ViewGroup ，也不是一个 View ，相当于声明了一些视图，等待被添加
* 因为 merge 标签并不是一个 View ，所以<span style="border-bottom:1px solid red;"> 通过LayoutInflate.inflate方法渲染的时候， 第二个参数必须指定一个父容器，且第三个参数必须为 true ，也就是必须为 merge 下的视图指定一个父亲节点。</span>
* 因为 merge 不是 View ，所以对 merge 标签设置的所有属性都是无效的

#### merge 和 include 的总结
1. 使用 include 标签可以增加布局的复用性，提高效率
2. 使用 merge 标签可以减少视图树的节点个数，加快视图的绘制，提高 UI 的性能

### ViewStub
* 继承 View ，是一个非常轻量级并且宽高都是 0 ，因此本身不参与任何布局和绘制过程
* 存在的意义就是按需加载所需要的布局文件。
* 在实际开发过程中，很多布局文件在正常情况下是不会显示的，这个时候就没有必要再整个界面初始化的时候加载进来
* 通过 ViewStub 就可以做到在使用的时候进行加载，提高程序的初始化的性能


![Alt text](../../../../images/1465729201927.png)

stub_import 是 ViewStub 的 id ， panel_import 是 layout_network_error 这个布局的根元素的 id ，如何进行按需加载 ViewStub 的布局呢，可以有两种方式
``` java
((ViewStub)findViewById(R.id.stub_import)).setVisibility(View.VISIBLE);
//或者
View importPanel=((ViewStub)findViewById(R.id.stub_import)).inflate();
```
1. 区别就是 inflate() 返回的是一个引用布局，从而可以通过 findViewById() 方法找到对应的控件
2. 当 ViewStub 通过 setVisibility() 或者 inflate() 方法加载后， ViewStub 就会被它内部的布局替换掉，这个时候 ViewStub 就不在是整个布局的结构的一部分了，所以再次调用 inflate() 就会报错。
3. 目前不支持 merge 标签

## 常用的工具

1. 可以通过<font color="#ff000" >Hierarchy Viewer</font>查看布局的嵌套情况。 使用 [View Server](https://github.com/romainguy/ViewServer) 可以在普通的手机上使用 Hierarchy Viewer
    注： Android Studio 3.1 或更高版本中， Hierarchy Viewer 已经被废弃，推荐在运行时使用 Layout Inspector 来检查应用程序的视图层次结构。
2. 在开发者选项中的<font color="#ff000" >GPU过度可视化工具</font>，查看界面渲染情况，
3. 可以使用<font color="#ff000" > Layout Inspector </font>辅助分析。捕捉当前页面快照。生成一个.li文件，通过 AndroidStudio 查看页面视图层次结构
    打开布局检测器  
    点击 Tools > Android > Layout Inspector。在出现的 Choose Process 对话框中，选择想要检查的应用进程，然后点击 OK 。
4. 使用<font color="#ff000" >Profile GPU Rendering </font>也是在开发者选项中打开。打开后的界面如下
    ![Alt text](../../../../images/carsetting_1.png)
   各个颜色的意思
    ![Alt text](../../../../images/carsetting_2.png)
5. Lint 是 Android Studio 提供的一款代码扫描分析工具，可以扫描、发现代码结构和质量问题并提供解决方案。    


---   
搬运地址：    

Android 开发艺术探索      

Android 群英传     

[Android 布局优化之 include 与merge](https://blog.csdn.net/a740169405/article/details/50473909)

[Android 性能优化之布局优化](https://henleylee.github.io/posts/2019/d59595e2.html)
