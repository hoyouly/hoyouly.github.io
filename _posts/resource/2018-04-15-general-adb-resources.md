---
layout: post
title: adb  常用资源
category: 资源
tags:   adb
---

<!-- * content -->
<!-- {:toc} -->

## 常用指令
### 得到设备的尺寸和密度
1. adb shell wm size    //设备的尺寸
2. adb shell wm density  // 设备的密度

## 截图，并且导出

* adb shell screencap -p /sdcard/screen.png  ; adb pull /sdcard/screen.png  ~/  
-p  后面跟的是截屏保存的 sdcard 路径
