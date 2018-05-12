---
layout: post
title: Activity的生命周期和启动模式
category: 读书笔记
tags: Android开发艺术探索 生命周期 启动模式
description: Activity的生命周期和启动模式
keywords: 关键字
---

* content
{:toc}

* onStart  和onStop是从Activity是否可见这个角度来回调的
* onResume和onPause是从Activity是否位于前台这个角度来回调的
* 不能在onPause中做重量级操作
* onSaveInstanceState方法是在onStop之后，可能在onPause之前，也可能在onPause之后
## 启动模式
* **standard  标准模式**，也是系统默认模式，每次启动都会创建一个新的 实例，不管该实例是否存在。被创建的实例的生命周期符合典型情况下的生命周期，谁启动这个Activity，那么该Activity就和这个在同一个栈里面
* **singleTop  栈顶复用模式**，如果该Activity在栈顶，则再次启动的该Activity的时候，不创建，onCreate和onStart也不会执行，但是onNewIntent方法会执行，如果不在栈顶但是存在这个栈里面，那么就仍然创建新的实例，
* **singleTask  栈内复用模式**  也是一种单实例模式，只要该Activity存在一个栈中，那么多次启动该Activity都不会重新创建该实例，也不会执行onCreat 和onStart方法，但是执行onNewIntent，如果Activity A 是singleTask
if(A 所在的栈不存在){
	创建A所需的任务栈 taskA
}
if(taskA 中不存在A的实例){
	创建A的实例，
	放到任务栈 taskA 中
}
到这里说明存在包含A实例的任务栈
调用A的实例到栈顶并执行onNewIntent 方法
例如  任务栈S1 中有ABC，Activity D以singleTask模式启动
1. 如果D所需要的栈是S2 由于D和S2都不存在，系统先创建任务栈S2，然后在创建D的实例并将其压入到S2 中
2. 如果D所需要的任务栈是S1,由于S1已经存在，那么就直接创建D的实例并将其压入到S1中
3. 如果D所需要的任务栈是S1，并且S1内是ADBC，那么此时D不会创建，而是把BC移除栈，因为singleTask具有clearTop 的效果，最终S1的情况是AD 这两个
* **singleInstance  单实例模式**，是一种加强的singleTask模式，除了具有singleTask的模式的所有特性，还有一点就是此模式的Activity只能单独位于一个任务栈中。由于独自一个栈，就没有clearTop 一说了，

## TaskAffinity 任务相关性
标识了一个Activity所需要的任务栈的名字，默认情况下，所有的Activity所需要的任务栈的名字都为应用程序的包名，
主要是和singleTask 启动模式或者allowTaskReparenting属性配对使用，
任务栈分为前台任务栈和后台任务栈，可以通过切换将后台任务栈再次调到前台
可以通过android:taksAffinity=""来设置改Activity的任务栈，注意属性必须是字符串，并且必须包含分隔符“..”


## 设置Activity的启动模式
1.  在AndroidManifest为Activity指定启动模式：android：launchMode=""
2.  通过Intent中设置标志来为Activity指定启动模式
intent.addFlags(Intent.FlAG_ACTIVITY_NEW_TASK);

二者区别：
1. 第二种方式优先级高于第一种，两种同时存在的时候，以第二种为准
2. 范围不同，第一种方式无法设定FlAG_ACTIVITY_CLAER_TOP,第二种方式无法指定Activity为singleInstance模式

## Activity 的Flags
* FLAG_ACTIVITY_NEW_TAKS     和在xml中设置 singleTask模式相同
* FLAG_ACTIVITY_SINGLE_TOP  和在xml中设置singleTop 模式相同
* FLAG_ACTIVITY_CLEAR_TOP   启动该Activity的时候，在同一个任务栈中位于他上面的Activity都要出栈，一般和FLAG_ACTIVITY_NEW_TAKS 配合使用，在这种情况，如果已经存在该实例，系统会调用onNewIntent，如果被启动的Activity采用standard模式启动，那么连他同它之上的Activity都要出栈，系统会创建新的Activity实例并放入栈顶，
singleTask启动模式默认就有次标记的效果
* FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS
具有这个标记的Activity不会出现在历史Activity列表中，等同于android:excludeFromRecents="true"
## IntentFilter 的匹配规则
启动Activity方法
1. 显式调用 需要明确指定被启动对象的组件信息，包括包名和类名，
2. 隐式调用 不需要明确组件信息，
原则上一个Intent不应该即使显示调用，又是隐式调用，如果二者共存以显式调用为主
### 隐式调用
需要Intent能够匹配目标组件的IntentFilter所设置的过滤信息，包括action，category，data,
一个Intent-filter可以包含多个action，category，data，一个Intent只有同时匹配action类别，category类别，data类别，才能算完全匹配，才能启动目标Activity
一个Activity中可以有多个IntentFilter,一个Intent只要匹配任何一组intent-filter就可以启动对应的Activity
![Alt text](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCEf352487d6ecc2802b5fdcad12816fbea/3044)

### action的匹配规则
action是一个字符串，一个过滤规则可以有多个action，只要Intent中的action能够和过滤规则的任何一个action相同就可以匹配成功。如果没有指定action，那么匹配失败，**action的匹配要求Intent中的action存在并且必须和过滤规则中的其中一个action相同** action区分大小写，
### category的匹配规则
要求Intent如果含有category，那么所有的category都必须和过滤中的其中一个category相同，换句话说，如果Intent中出现了category，不管有几个category，对于每个category来说，它必须是过滤规则中已经定义的category，
**Intent中可以没有category，但是一旦有category，不管有几个，每个都要能够和过滤规则中的任何一个category相同**
不设置category，系统在调用startActivity的时候，会默认加上android.intent.category.DEFAULT这个category，

**特别说明**
```java
<intent-filter>
    <action android:name="android.intent.action.MAIN" />

    <category android:name="android.intent.category.LAUNCHER" />
</intent-filter>
```
这二者共同出现，标明该Activity是一个入口Activity，并且会出现在系统应用列表中，二者缺一不可。
#### android.intent.action.MAIN 与android.intent.category.LAUNCHER的区别
1. 区别一：
android.intent.action.MAIN决定一个应用程序最先启动那个组件
android.intent.category.LAUNCHER决定应用程序是否显示在程序列表里(说白了就是是否在桌面上显示一个图标)
* 第一种情况：有MAIN,无LAUNCHER，程序列表中无图标
原因：android.intent.category.LAUNCHER决定应用程序是否显示在程序列表里
* 第二种情况：无MAIN,有LAUNCHER，程序列表中无图标
原因：android.intent.action.MAIN决定应用程序最先启动的Activity，如果没有Main，则不知启动哪个Activity，故也不会有图标出现
所以这两个属性一般成对出现。
 如果一个应用中有两个组件intent-filter都添加了android.intent.action.MAIN和
android.intent.category.LAUNCHER这两个属性， 则这个应用将会显示两个图标， 写在前面的组件先运行。
2. 区别二：
android.intent.category.LAUNCHER：android.intent.category.LAUNCHER决定应用程序是否显示在程序列表里，就是android开机后的主程序列表。
android.intent.category.HOME：按住“HOME”键，该程序显示在HOME列表里。

### data的匹配规则
如果过滤规则中定义了data,你们Intent中必须要定义可匹配的data,
#### data的结构
![Alt text](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCE97c5e5f700d01db9404c8c5c35e6cfa5/3046)
data由两部分组成，
1. mimeType  指媒体类型，比如image/jpeg,audio/mpeg4-generic和video/*,可以表示图片，文本，视频等不同格式的媒体
2. URI
URI的结构
![Alt text](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCEc477c37c572658d99b71786ab4c78bd6/3048)

* Scheme: URI的模式，比如http，file,content等，如果URI没有指定scheme，那么整个URI的其他参数无效，这也就意味着URI无效
* Host, URI的主机名，比如www.baidu.com，如果host未指定，那么整个URI的其他参数无效，这也就意味着URI无效
* Port URI中的端口号，比如 80 ，仅当指定了Scheme和host参数的时候，port参数才有意义
* Path,pathPattern和pathPreFix  这三个表示路径信息，
1.  path表示完整路径信息，
2.  pathPattern也表示完整路径信息，但是里面可以包含通配符” * ”，由于正则表达式的规范，想要表示真实的*，要转义”\”
3.  pathPrefix  表示路径的前缀信息

匹配规则:**Intent中必须含有data数据，并且data数据能够完全匹配过滤规则中的某一个data,这里的完全匹配是指过滤规则中出现的data部分也出现在Intent中的data中**
1. 规则一
![Alt text](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCEbb9d59d2d1874d45c877f0d0a0d66d3f/3050)
![Alt text](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCEb9504cabcd0f48dedc4048f6ca6adf22/3052)
匹配规则
![Alt text](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCE679fb536aa2398ca61f21e5bcc23aa62/3054)
如果data中为指定uri，则缺省的是content或者file,intent 中的uri的scheme部分需设置为content或者file才能有效
2. 规则二
![Alt text](https://note.youdao.com/yws/public/resource/b0933b37ddd8ac810ca1d341288bbaa7/xmlnote/WEBRESOURCE75641a4844ab8869947849979cc8f5d7/3059)
3. 匹配以http开头的.pdf结尾的路径，是别的应用程序想要打开网络pdf的时候，用户能选择这个
```java
   <intent-filter>  
    <action android:name="android.intent.action.VIEW"></action>  
    <category android:name="android.intent.category.DEFAULT"></category>  
    <data android:scheme="http" android:pathPattern=".*//.pdf"></data>  
</intent-filter>   
```
4. 让别人通过Intent调用的时候显示在选择框中，只需要注册android.intent.action.SEND 与 mimeType 为 “text/plain” 或 “/” 就可以了

```java
<intent-filter>  
    <action android:name="android.intent.action.SEND" />  
    <category android:name="android.intent.category.DEFAULT" />  
    <data mimeType="*/*" />  
</intent-filter>
```
5. 当打开某个音乐文件的时候，显示在选择框中，我们只用注册 android.intent.action.VIEW 与 mimeType 为 audio/* 就可以了

```java
<intent-filter>  
     <action android:name="android.intent.action.VIEW" />  
     <category android:name="android.intent.category.DEFAULT" />  
     <data android:mimeType="audio/*" />  
</intent-filter>  
```

## 判断是否匹配隐式Intent
1. 采用PackageManager的resolveActivity，
2. Intent的resolveActivity
如果找不到就会返回null，
queryIntentActivities  返回所有成功匹配的Activity信息
resolveActivity 返回最佳匹配的Activity信息


[参考文档](http://blog.csdn.net/mynameishuangshuai/article/details/51673273)
