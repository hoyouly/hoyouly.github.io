---
layout: post
title: Android 性能优化 -- APK 瘦身
category: 性能优化
tags: 性能优化
description: Android 性能优化
---

* content
{:toc}

想了解APK 打包流程的，可以参考这篇文章。[扫盲系列 - APK 打包流程](http://hoyouly.fun/2020/04/15/apk-build/)

# APK瘦身
应用安装包大小对应用使用没有影响，但应用的安装包越大，用户下载的门槛越高
减小安装包大小可以让更多用户愿意下载和体验产品。
## 代码混淆
在gradle使用minifyEnabled进行Proguard混淆的配置，可大大减小APP大小,它包括压缩、优化、混淆等功能。
```java
android {
    buildTypes {
        release {
            minifyEnabled true
        }
    }
}
```
然后就是配置要那些需要混淆，那些不混淆。主要包括一下几种
###  参数
* -include {filename}    从给定的文件中读取配置参数   
* -basedirectory {directoryname}    指定基础目录为以后相对的档案名称   
* -injars {class_path}    指定要处理的应用程序jar,war,ear和目录   
* -outjars {class_path}    指定处理完后要输出的jar,war,ear和目录的名称   
* -libraryjars {classpath}    指定要处理的应用程序jar,war,ear和目录所需要的程序库文件   
* -dontskipnonpubliclibraryclasses    指定不去忽略非公共的库类。   
* -dontskipnonpubliclibraryclassmembers    指定不去忽略包可见的库类的成员。

### 保留选项
* -keep {Modifier} {class_specification}    保护指定的类文件和类的成员   
* -keepclassmembers {modifier} {class_specification}    保护指定类的成员，如果此类受到保护他们会保护的更好   
* -keepclasseswithmembers {class_specification}    保护指定的类和类的成员，但条件是所有指定的类和类成员是要存在。   
* -keepnames {class_specification}    保护指定的类和类的成员的名称（如果他们不会压缩步骤中删除）   
* -keepclassmembernames {class_specification}    保护指定的类的成员的名称（如果他们不会压缩步骤中删除）   
* -keepclasseswithmembernames {class_specification}    保护指定的类和类的成员的名称，如果所有指定的类成员出席（在压缩步骤之后）   
* -printseeds {filename}    列出类和类的成员-keep选项的清单，标准输出到给定的文件  

### 压缩
* -dontshrink    不压缩输入的类文件   
* -printusage {filename}   
* -whyareyoukeeping {class_specification}  

### 优化
* -dontoptimize    不优化输入的类文件   
* -assumenosideeffects {class_specification}    优化时假设指定的方法，没有任何副作用   
* -allowaccessmodification    优化时允许访问并修改有修饰符的类和类的成员  

### 混淆
* -dontobfuscate    不混淆输入的类文件   
* -printmapping {filename}   
* -applymapping {filename}    重用映射增加混淆   
* -obfuscationdictionary {filename}    使用给定文件中的关键字作为要混淆方法的名称   
* -overloadaggressively    混淆时应用侵入式重载   
* -useuniqueclassmembernames    确定统一的混淆类的成员名称来增加混淆   
* -flattenpackagehierarchy {package_name}    重新包装所有重命名的包并放在给定的单一包中   
* -repackageclass {package_name}    重新包装所有重命名的类文件中放在给定的单一包中   
* -dontusemixedcaseclassnames    混淆时不会产生形形色色的类名   
* -keepattributes {attribute_name,...}    保护给定的可选属性，例如LineNumberTable, LocalVariableTable, SourceFile, Deprecated, Synthetic, Signature, and InnerClasses.  
* -renamesourcefileattribute {string}    设置源文件中给定的字符串常量  

### 注意
1. 一定别把自己的 model实体混淆，因为如果这样的话，Json解析就会失败。
2. 混淆后一定要多测试，很多时候一些功能没混淆之前是正常的，但是混淆后就不正常了。

## 去除无用资源
在gradle使用shrinkResources去除无用资源，效果非常好。
```java
android {
    buildTypes {
        release {
            shrinkResources true
        }
    }
}
```
## 清理无用资源
shrinkResources true 只能去除没有任何父函数调用的情况
而要清理调用的父类函数最终是废弃的代码，就要用到 AndroidStudio 自带的Remove Unused Resources 插件了,
* 使用：AndroidStudio -> Refactor -> Remove Unused Resources  

## 删除无用的语言资源
如果不需要语言几十种语言支持的话，强大的gradle支持语言的配置，比如只保留中文和英文
```java
android {
    defaultConfig {
        resConfigs "zh","en"
    }
}
```
## 使用tinypng有损压缩
TinyPNG工具只支持上传PNG图片到官网上压缩，然后下载保存，在保持alpha通道的情况下对PNG的压缩可以达到1/3之内，而且用肉眼基本上分辨不出压缩的损失.
Tinypng的官方网站：http://tinypng.com/

## 使用webp格式
webp支持透明度，压缩比比jpg更高但显示效果却不输于jpg，
* 使用：在AndroidStudio选中要转换的图片，右键 最下面  就看到了Convert to Webp

注意：webp格式对版本有要求，
![](../../../../images/webp_1.png)

minSdkVersion 必须大于14，也就是 Android 4.0才行，不过现在的APP一般都大于这个吧，微信的新版本要求的最低版本还是Android 5.0呢。
## 手动lint检查，手动删除无用资源
在Android Studio中打开“Analyze” 然后选择"Inspect Code…"，范围选择整个项目，然后点击"OK"。

## 清理第三方库和冗余代码
避免重复功能的库
## so库的优化
```java
ndk{
      //设置支持的so库架构
      abiFilters "armeabi-v7a"
    }
```
armeabi-v7主要不支持ARMv5(1998年诞生)和ARMv6(2001年诞生)
许多基于 x86 的设备也可运行 armeabi-v7a 和 armeabi NDK 二进制文件。对于这些设备，主要 ABI 将是 x86，辅助 ABI 是 armeabi-v7a。
如果适配版本高于4.1版本，可以直接像我上面这样写，


# 实践
先看来原始APK的大小吧。
使用的工具就是AndroidStudio 自带的 Analyze app   
步骤：Android Studio下 ——> Build——> Analyze app
![添加图片](../../../../images/analyze_app_1.png)
可以看到，apk 大小是6.9M

## 1.清理无用资源
步骤： AndroidStudio -> Refactor -> Remove Unused Resources  
运行后，看看有啥改变。
![](../../../../images/remove_unused_res.png)
删除了这么多图片和xml文件，然后再次打包吧，
![添加图片](../../../../images/analyze_app_2.png)
现在的apk 大小是 6.7M，竟然只优化了0.2M ,我还以为能优化很多的，也罢，苍蝇也是肉啊。能瘦一点是一点。

我们可以看到，res目录 和class.dex都变小了。
## 2.代码混淆
`minifyEnabled true`
然后再次打包吧，
![添加图片](../../../../images/analyze_app_3.png)
发现又小了0.3M，这次减小的地方就是 class.dex文件

## 3.移除无用的资源
shrinkResources true
![添加图片](../../../../images/analyze_app_4.png)
这一步没效果，可能是因为项目小，无用资源又被清理过的缘故吧
## 4.删除无用的语言资源
这里我只保留的两个 `resConfigs "zh","en"`
再次打包
![添加图片](../../../../images/analyze_app_5.png)
发现包体积又变小了，这次瘦身的是 resources.ares ,

然后我就又试了一把，只保留一中语言，即 `resConfigs "zh"`
![添加图片](../../../../images/analyze_app_6.png)

发现 resources.ares 又瘦了一点点，虽然只有小小的1.5kb,
因为项目比较小，没有涉及到，估计项目大的话，会瘦多一些。

至于其他的瘦身计划，因为项目小，就不再实践了。
从6.9M瘦到6.2M，虽然瘦的少，但是体积也降低了10%啊。能瘦一些是一些。


---   
搬运地址：    
[Android性能优化系列之apk瘦身](https://blog.csdn.net/u012124438/article/details/54958757)

[Android性能优化之APK瘦身详解(瘦身73%)](https://blog.csdn.net/qq_32175491/article/details/80071987)
