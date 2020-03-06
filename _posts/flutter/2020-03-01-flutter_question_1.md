---
layout: post
title: Flutter疑问之 FlutterPluginRegistry 无法转换为 FlutterEngine
category: Flutter 学习
tags: FlutterPluginRegistry FlutterEngine
---
* content
{:toc}

学习 Android 项目中混合 Flutter项目，参考的是[Flutter学习小计：Android原生项目引入Flutter](https://www.jianshu.com/p/7b6522e3e8f1),  
不知道为啥，一直就报  FlutterPluginRegistry 无法转换为 FlutterEngine 这个错误，如下图所示
![添加图片](https://github.com/hoyouly/BlogResource/raw/master/imges/flutter_quesioint_one.jpg)

感觉和版本有关，可是又不知道为啥。
flutter 的版本如下
```
Flutter 1.12.13 • channel beta • https://github.com/flutter/flutter.git
Framework • revision cf37c2cd07 (3 months ago) • 2019-11-25 12:04:30 -0800
Engine • revision b6b54fd606
Tools • Dart 2.7.0

```
Flutter.java 和 GeneratedPluginRegistrant.java 都是自动生成的，为啥会出现这个问题呢，一直没搞明白
刚开始学，网上找了好多资料，也看不懂，索性就把红框里面那行 代码删掉。竟然可以了，NND，先记录下来，原因以后再找。没准后面学习中，就柳暗花明了。


---
搬运地址：
