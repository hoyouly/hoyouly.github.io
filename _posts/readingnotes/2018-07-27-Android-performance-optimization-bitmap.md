---
layout: post
title: Android 性能优化 -- Bitmap 优化
category: 读书笔记
tags: Android开发艺术探索 性能优化
description: Android 性能优化
---

* content
{:toc}


# 基础知识

## 资源目录分辨率对照表

|密度类型|代表的分辨率（px）|屏幕密度（dpi）|dpi范围|换算（dp/px）|比例|
|:----|:------:|:------|:------|:------|
|低密度（ldpi）|240x320|120|0dpi ~ 120dpi|1dp=0.75px|3|
|中密度（mdpi）|	320x480|160|120dpi ~ 160dpi|1dp=1px|4|
|高密度（hdpi）|480x800|240|160dpi ~ 240dpi|1dp=1.5px|6|
|超高密度（xhdpi）|720x1280	|320|240dpi ~ 320dpi|1dp=2px	|8|
|超超高密度（xxhdpi）|1080x1920|480|320dpi ~ 480dpi|1dp=3px|12|
|超超超高密度（xxxhdpi）| |640|480dpi ~ 640dpi|1dp=4px|16|

加载资源图片的流程

![添加图片](../../../../images/drawable_image.png)

自己的设备的dpi
```java
float xdpi = getResources().getDisplayMetrics().xdpi;
float ydpi = getResources().getDisplayMetrics().ydpi;
```

## 一张图片占用的内存
```
一张图片占用的内存= 图片的原始高度 * 图片的宽度 * 单位像素点占用的字节数
```

Bitmap.Config正是单位像素占用的字节数的重要依据
* ALPHA_8  表示8个alpha位图，即A=8，一个像素点占用一个字节，他没有颜色，只有透明度
* ARGB_4444 表示16位的ARGB位图，即A=4,R=4，G=4,B=4，一个像素点占用4+4+4+4=16个位，即2字节 ，已废弃
* ARGB_8888  表示32位的ARGB位图，即A=8,R=8，G=8,B=，一个像素点占用8+8+8+8=32个位，即4字节，24位真彩色
* RGB_565  表示16位的ARGB位图，即R=5，G=6,B=4，一个像素点占用5+6+4=16个位，即2字节 ，简易RGB色调
* RGBA_F16 占用8个字节大小，Android 8.0 新增（更丰富的色彩表现HDR）
* HARDWARE   Android 8.0 新增 （Bitmap直接存储在graphic memory）

其中 A 表示透明度，R表示红色，G表示绿色，B表示蓝色

如果以一个 100*100 像素的图片计算所占用内存大小


|Bitmap.Config|单位像素占用的字节数|分辨率100*100的图片所占用内存大小|
|:----|:------:|:------|
|ALPHA_8|1|100x100x1=10000B~=9.77KB|
|ARGB_4444|2|100x100x2=20000B~=19.53KB|
|RGB_565|2|100x100x2=20000B~=19.53KB|
|ARGB_8888|4|100x100x4=40000B~=39.06KB|
|RGBA_F16|8|100x100x2=20000B~=78.1KB|


一张 1920 X 1080 放入不同的 mipmap 文件下面，

<img src="../../../../images/mipmap_mdpi.png" alt="mipmap_mdpi" width="200" height="300" align="left" />
<img src="../../../../images/mipmap_xhdpi.png" alt="mipmap_xhdpi" width="200" height="300" align="right" />
<img src="../../../../images/mipmap_hdpi.png" alt="mipmap_hdpi" width="200" height="300" align="center" />
<img src="../../../../images/mipmap_xxhdpi.png" alt="mipmap_xxhdpi" width="200" height="300" align="left" />
<img src="../../../../images/mipmap_nohdpi.png" alt="mipmap_nohdpi" width="200" height="300" align="right" />
<img src="../../../../images/mipmap_xxxhdpi.png" alt="mipmap_xxxhdpi" width="200" height="300" align="center" />


由上可知:
* 同一张图片，放在不同资源目录下，其所在内存大小是不同的，并且随着资源目录分辨率的变大而减小
* 资源目录分辨率越高，其解析后的宽高越小，甚至会小于图片原有的尺寸（即缩放），从而内存占用也相应减少
* 图片不特别放置任何资源目录时，其默认使用mdpi分辨率：160
* 资源目录分辨率和设备分辨率一致时，图片尺寸不会缩放
* mipmap_nohdpi 目录下面，宽高就是原始图片宽高。

### 不同目录下的宽高计算公式

图片的宽高计算公式 就是 原始的宽 * 设备分辨率 / 资源目录分辨率  
例如在 mipmap_mdpi， 设备分辨率是 420，资源目录分辨率是 160，图片的宽是1080，所以在 mipmap_mdpi 下面的得到的宽= 1080*420/160 = 2835

所以,前面的图片所占内存计算公式，也可以写成这样。
```
一张图片所占的内存 = 图片的宽 * 图片的高 * （设备分辨率 / 资源目录分辨率）^2 * 单位像素点占用的字节数
```

# 优化方案
## 降低色彩解析模式
使用低色差的解析模式，比如 RGB565  ，减少单个像素的字节大小

Android 默认使用 ARGB8888 配置来处理色彩，如果改为 RGB565，内存大概减少一半，代价是显示的色彩减少，适用于对色彩丰富程度要求不高的场景

## 资源文件合理放置
资源文件合理放置，高分辨率图片可以放到高分辨率目录下面

Android 默认放置的资源目录对应的是 160 dpi, 理论上，图片放置的资源分辨率越高，其占用的内存会越小，但分辨率的图片也会因此被拉伸，显示出现失真。另一方面，高分辨率的图片意味着其占用的本地存储也变大。

## 图片尺寸压缩
图片缩放，减少尺寸。就是我们经常说的图片质量和图片体积压缩。典型代码如下
典型的代码
```java
BitmapFactory.Options options = new BitmapFactory.Options();
options.inPreferredConfig = Bitmap.Config.RGB_565;
options.inJustDecodeBounds = true;
BitmapFactory.decodeResource(getResources(), resId,options);
options.inJustDecodeBounds = false;
options.inSampleSize = BitmapUtil.computeSampleSize(options, -1, imageView.getWidth() * imageView.getHeight());
Bitmap newBitmap = BitmapFactory.decodeResource(getResources(), resId, options);
```
## 复用和缓存
就是所谓的 LRU 算法，详情可以查看 [ Bitmap 的加载和 Cache 处理 ](http://hoyouly.fun/article-detail/2018/03/17/Bitmap-loading-and-Cache/)

---   
搬运地址：    

Android 开发艺术探索      

Android 群英传     

[Android性能优化系列之Bitmap图片优化](https://blog.csdn.net/u012124438/article/details/66087785)   

[ Android中Bitmap内存优化 ](https://www.jianshu.com/p/3f6f6e4f1c88)

[Android mipmap技巧](https://www.jianshu.com/p/7fa3417d2ca4)

[Android 适配（drawable文件夹）图片适配（二）](https://www.cnblogs.com/huihuizhang/p/9473698.html)
