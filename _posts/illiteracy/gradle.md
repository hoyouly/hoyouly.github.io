---
layout: post
title: 扫盲系列之---build.gradle 的基本介绍
category: 扫盲系列
tags: RecycleView
---
* content
{:toc}

## project 和task

每一次构建，至少得有一个project完成，每一个project至少有一个task，每一个build.gradle 文件代表一个project，task定义在build.gradle中，当初始化构建的时候，gradle 会基于build文件，集合所有的project和tasks,一个taks包含一系列动作，然后他们将会按照顺序执行，一个动作就是一段代码块，很像java中的方法，

## 构建声明周期
一旦一个task被执行，那么它就不会被再次执行了，不包含依赖的tasks总是优先执行，一次构建将会经历以下三个阶段
1. 初始化阶段  project实例在这里创建，如果有多个模块，即有多个build.gradle文件，多个project实例被创建，
2. 配置阶段  该阶段，build.gradle脚本被执行，为每个project创建和配置所有的tasks
3. 执行阶段  这一阶段，gradle会决定哪一个task被执行，哪一个task会被执行完全依赖开始构建时传入的参数和当前所在文件夹的位置

## 基本的构建命令
* assemble 针对每个版本创建一个apk
* build  是check和assemble的集合体
* clean  删除所有的构建任务，包含apk文件
* check  执行Lint检查并且能够在Lint检测到错误后停止执行脚本

`gradlew assemble`

## Android Stuido task
Android Studio面板右侧 Gradle Project 栏目，就能看到，双击就可以了
## 使用aar
1. 在目录中建立一个文件 例如在根目录下面，和app同一级目录，建立一个aars文件夹
2. project 中的build.gradle 中添加下面代码，就可以

```java
allprojects {
    repositories {
        ...
        flatDir {
            dirs '../aar'
        }
    }
}
```
3.然后就可以在具体的module的build.gradle中的dependencies中使用了
```
dependencies{
  compile(name: 'libraryname', ext: 'aar')
}    
```
这个会告诉Gradle 在aars文件夹下，添加一个叫做libraryname的文件，且其后缀是aar的作为依赖。

##


## applicationId和Package name 不同
在gradle被用来作为Android构建工具之前，package
name在AndroidManifest.xml有两个作用：其作为一个app的唯一标示，并且其被用在了R资源文件的包名。



## implementation 和compile 区别

---
搬运地址：   

[Build.gradle详细配置说明](https://jingyan.baidu.com/article/bea41d4389bdc3b4c51be6be.html)  
[Gradle入门学习---认识buildeTypes和dependencies](https://www.cnblogs.com/wenjiang/p/6638927.html)   
[Android 使用gradle打包的各种配置](https://www.jianshu.com/p/1a320062aedd)  
[Android Studio 中build.gradle文件的详细解析](https://blog.csdn.net/huanchengdao/article/details/51472628)  
[从Eclipse到AndroidStudio（四）Gradle基本配置](https://blog.csdn.net/scott2017/article/details/51799597)  
[gradle入门](http://www.androidchina.net/2155.html)  
[Gradle for Android 第一篇( 从 Gradle 和 AS 开始 )](http://www.androidchina.net/6636.html)
