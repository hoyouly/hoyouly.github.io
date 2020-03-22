---
layout: post
title: 恍然大悟 - Android Studio 中 asserts 位置
category: 恍然大悟
tags: AndroidStudio asserts
---
* content
{:toc}

不知道大家有没有遇到过这样的问题，一个bug 查了好几天，突然发现，卧槽，竟然是这里出的问题。卧槽，竟然是这样玩儿的，卧槽，竟然。。。是不是有一种   
踏破铁鞋无觅处，得来全不费工夫，  
山重水复疑无路，柳暗花明又一村。  
所有就有了这个恍然大悟系列

今天遇到的这个问题就是这样。首先说结论吧
** Android Studio 中，assets文件夹是放在src/main 下面的，和res 属于平行关系**
今天做写demo，需要从assets中加载一张图片显示出来，这对我这种有N年开发经验的工程师来说，不是小菜一碟嘛，在res下面建一个assets文件夹，图片放进去，然后读取出来成为InputStream ,然后加载到Imageview,简单，
```java
InputStream is=getAssets().open("qm.jpg");
largeImageView.setInputStream(is);
```

然而，刚开始就出问题
## 问题一 java.io.FileNotFoundException
卧槽，文件不能找到，assert下面的有文件的啊，这是毛线啊，难道我读取assets下的文件方式有问题，那么换一种方式吧，
```java
String path="file:///android_asset/qm.jpg";
InputStream is=new FileInputStream(new File(path));
```
问题依旧，NND，为啥啊，
那我就看看asserts下面有啥东西吧，这个也简单。
```java
String[] list = getAssets().list("");
```
list()参数为空字符串就可以了。
结果竟然出乎意料，竟然是 images、sounds、webkit 这三个文件，根本就没有我添加的qm.jpg,这是毛线啊，然而，这个问题还没解决，竟然又出新的问题。
## 问题二 The file name must end with .xml
不知道是毛线，我再往assets中添加一个文件，竟然出现这个错误，可是我把这个文件删除，只剩下我添加的那个图片，依旧有这个错误，这又是毛线啊，还让不让人干活啦。。
assets下面为啥必须只能放xml文件呢，不可能啊，肯定是可以放图片的啊，然后继续Google+百度吧，于是就找到了一个博客  [The file name must end with .xml or .png](https://blog.csdn.net/zhangnianxiang/article/details/76906567)
让我在项目级的gradle.properties 添加 `android.disableResourceValidation=true`，然后在clean一下，尽管我不知道这是干嘛的，是骡子是马，拉出来溜溜再说吧，添加，然后clean，然后build ,然后又出问题了

## 问题三 前言中不允许有内容。  
卧槽，还能不能行啊，这又是啥啊，百度吧，还真找到了， [Android studio assets error：前言中不允许有内容](https://blog.csdn.net/alice_1_1/article/details/70050794), 原来assets是放在src/main 下面的，不是在res 文件夹下面的，犯了经验主义错误。只需要把assets移动到src/main 下面，所有的问题都解决了，不需要添加所谓的 android.disableResourceValidation=true一样能解决问题。终于可以睡个好觉了。。

---
搬运地址：    

