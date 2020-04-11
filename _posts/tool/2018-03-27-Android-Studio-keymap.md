---
layout: post
title: Android Studio 使用技巧集合
category: 工具
tags: AndroidStudio 快捷键
---

* content
{:toc}

## 快捷键

|平台|Mac |Win/Linux|
|:----|:------|:------|
|自动生成变量|Cmd + Alt + V|Ctrl + Alt + V|
|自动提取参数|Cmd + Alt + P|Ctrl + Alt + P|
|自动提取方法|Cmd + Alt + M|Ctrl + Alt + M|
|内联变量/参数/方法|Cmd + Alt + N|Ctrl + Alt + N|
|万能重构键|Ctrl + T|Ctrl + Alt + Shift + T|
|重命名|shift + F6|shift + F6|
|参数提示|CMD + P|Ctrl + P|
|隐藏所有窗口|CMD + Shift + F12|Ctrl + shift + F12|
|Surround With..|cmd + alt + t|ctrl + alt + t|

## 修改 toString() 模板为 JSON 格式
参考： [IDEA修改 toString 方法模板为 JSON 格式](https://blog.csdn.net/masonqaq/article/details/77975030)

## 自定义模板

以打印调用栈信息为例
1. 以此打开，setting -> Editor -> Live Templates
2. 点击右侧加号 选择 Live Templates
  * Abbreviation 中填写模板快捷键 例如 logz
  * Description 中是描述，可以随便写
  * Template text 中就是模板内容
  ```java
  android.util.Log.d("hoyouly",getClass().getName()+" -> "+android.util.Log.getStackTraceString(new Throwable()));`
  ```

3. 右侧勾选 Reformat according to style 和  Shorten RA names
4. 点击下面的 Define 选择 java 中的 statment ，然后点击保存就可以了。

这样在编辑框中，输入 logz 就直接输出来 android.util.Log.d("hoyouly",getClass().getName()+" -> "+android.util.Log.getStackTraceString(new Throwable())); 这行代码了。

当然也可以修改 Android studio 的代码，例如 logd 中的模板我已经改成了 android.util.Log.d("hoyouly", getClass().getName()+" -> $METHOD_NAME$: $content$");
这样输出 log 的时候我就可以不用再考虑 TAG 的事情了


搬运地址：    


[Android Studio你不知道的快捷键](http://weishu.me/2015/12/17/shortcut-of-android-studio-you-may-not-know-3/)
