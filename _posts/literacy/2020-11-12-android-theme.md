---
layout: post
title: 扫盲系列 - Android 主题
category: 扫盲系列
tags:  theme
---
<!-- * content -->
<!-- {:toc} -->

# 背景

今天从一个项目中搬运Seekbar组件的时候，发现了一个很奇怪的问题

![添加图片](../../../../images/seekbar_1.png)
Seekbar 滑动的位置没有紧靠滑道，这个倒是挺奇怪，真正的效果应该是下面这种

![添加图片](../../../../images/seekbar_2.png)


代码我是CV大法弄出来的，不会出错啊，可是为啥出现这个效果呢？想不通

刚开始我怀疑是和编译版本有关，可是当我把编译版本改称一样的后，发现还有这个问题，那是什么原因呢？然后怀疑和主题有关，发现果然不一样，正常效果的是 `Theme.AppCompat.Light.NoActionBar`,而不正常效果的是` android:Theme.Holo.Light.DarkActionBar`，

这两个又啥区别呢，之前一直没太注意，Google+baidu一下吧。发现这两个还是有点嚼头的

简单来说，两句话：
* 以android开头的是Android自带的主题
* 以Theme开头的是兼容包v7带的主题，

# Android 自带的主题
常见的应该就是Holo Theme和 Material Design Theme

## Holo Theme
在4.0之后推出了Android Design，从此Android在设计上有了很大的改善，而在程序实现上相应的就是Holo风格
常见的包括以下几种

* android:Theme.Holo Holo根主题
* android:Theme.Holo.Black Holo黑主题
* android:Theme.Holo.Light Holo白主题
* android:Theme.Holo.Light.DarkActionBar

## Material Design Theme
Android在5.0版本推出了Material Design的概念，常见的是以下几种

* android:Theme.Material   Material根主题
* android:Theme.Material.Light Material亮主题
* android:Theme.Material.NoActionBar    Material样式并且没有ActionBar主题
* android:Theme.Material.Light.DarkActionBar  Material亮主题并且ActionBar是暗黑风格

下面两种不常见，只是知道即可。
## 其他

### API 1
* android:Theme 根主题
* android:Theme.Black 背景黑色
* android:Theme.Light 背景白色
* android:Theme.Wallpaper 以桌面墙纸为背景
* android:Theme.Translucent 透明背景
* android:Theme.Panel 平板风格
* android:Theme.Dialog 对话框风格

###  API 14:
android:Theme.DeviceDefault 设备默认根主题
android:Theme.DeviceDefault.Black 设备默认黑主题
android:Theme.DeviceDefault.Light 设备默认白主题

# 兼容包v7中带的主题：

兼容主题是说如果运行程序在 android 5.0的手机上则就相当于使用的 Material 主题，如果运行程序在 android 4.0 手机上相当于 Holo 主题。依次类推


兼容包会被Google不断升级，比如
*  appcompat-v7-21.0 表示升级到向 API 21 兼容
*  appcompat-v7-23.2 表示升级到向 API 23 兼容
依次类推，不过现在已经被AndroidX代替了。常见的是以下几种。

* Theme.AppCompat.NoActionBar 兼容主题的没有ActionBar风格主题
* Theme.AppCompat.DayNight 兼容主题的白昼风格主题
* Theme.AppCompat.Light 兼容主题的白色主题

所有应用于应用程序主题都是以“Theme”或者ThemeOverlay开头，
“ThemeOverlay”主题，可用于 Toolbar 控件，
以“TextAppearance”开头的主题，可用于设置文字外观，不能用于设置应用程序主题。

# 主题风格简介

下面是一些主题风格的说明，可以简单了解一下。

* Dialog 对话框风格
* Black 黑色风格
* Light 光明风格
* Dark 黑暗风格
* DayNight 白昼风格
* Wallpaper 墙纸为背景
* Translucent 透明背景
* Panel 平板风格
* Dialog 对话框风格
* NoTitleBar 没有TitleBar
* NoActionBar 没有ActionBar
* Fullscreen 全屏风格
* MinWidth 对话框或者ActionBar的宽度根据内容变化，而不是充满全屏
* WhenLarge 对话框充满全屏
* TranslucentDecor 半透明风格
* NoDisplay 不显示，也就是隐藏了
* WithActionBar 在旧版主题上显示ActionBar

下次看到一种主题，我们就能查出来到底什么意思了.

就如 `android:Theme.Holo.Light.DarkActionBar`: Android 自带的Holo主题，光明风格并且ActionBar是黑暗风格的。

再如`android:Theme.Material.NoActionBar.Fullscreen` ：Android 自带的Material主题，风格是全屏并且没有ActionBar

或者 `Theme.AppCompat.Light.NoActionBar` 兼容主题，光明风格并且没有ActionBar



- - - -
搬运地址：    

[Android关于Theme.AppCompat相关问题的深入分析](https://www.jianshu.com/p/6ad7864e005e)

[总结一下Android中主题(Theme)的正确玩法](https://www.cnblogs.com/zhouyou96/p/5323138.html)
