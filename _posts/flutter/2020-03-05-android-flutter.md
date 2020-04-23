---
layout: post
title: Flutter 填坑 - Android 原生项目嵌入 Flutter
category: Flutter 学习
tags: Flutter  Android
---
<!-- * content -->
<!-- {:toc} -->
在学习 Android 原生项目嵌套 Flutter ，发现一篇好文章，[Flutter学习小计：Android原生项目引入Flutter  ](https://www.jianshu.com/p/7b6522e3e8f1) ，就按照这个跑了一遍，竟然遇到了好几个坑。记录一下。

* 声明 各个版本号
```
Flutter        1.12.13
Dart           2.7.0
Android Studio 3.5
```

接下来就是记录坑的过程
# 坑一
在 Android 项目中引入Flutter Module
也按照 上面写的，
* 在 app/build.gradle文件中的 android {} 添加 以下代码
  ```java
  compileOptions {
        sourceCompatibility 1.8
        targetCompatibility 1.8
    }
  ```
* 在 setting.gradle 中添加以下代码
```java
// 加入下面配置
setBinding(new Binding([gradle: this]))
evaluate(new File(
        settingsDir.parentFile,
        'my_flutter/.android/include_flutter.groovy'
))
```

然后同步，结果报错了  
```java
Caused by: java.io.FileNotFoundException: /Users/hoyouly/llll/my_flutter/.android/include_flutter.groovy (/Users/hoyouly/llll/my_flutter/.android/include_flutter.groovy)
  at groovy.lang.GroovyCodeSource.<init>(GroovyCodeSource.java:112)
  at groovy.lang.GroovyShell.evaluate(GroovyShell.java:487)
  at groovy.lang.Script.evaluate(Script.java:221)
  at groovy.lang.Script$evaluate$0.callCurrent(Unknown Source)
  at org.codehaus.groovy.runtime.callsite.CallSiteArray.defaultCallCurrent(CallSiteArray.java:51)
  at org.codehaus.groovy.runtime.callsite.AbstractCallSite.callCurrent(AbstractCallSite.java:156)
  at org.codehaus.groovy.runtime.callsite.AbstractCallSite.callCurrent(AbstractCallSite.java:168)
```

文件没找到，不可能啊，但是确实报错了，最后查到得知，是路径写的有问题，改成 <font color="#ff000" >  rootProject.name + '/my_flutter/.android/include_flutter.groovy' </font> 就可以了，完整的 文件如下

```java
include ':app'
rootProject.name = 'AndroidFlutter'

// 加入下面配置
setBinding(new Binding([gradle: this]))
evaluate(new File(
        settingsDir.parentFile,
        rootProject.name + '/my_flutter/.android/include_flutter.groovy'
))
```

然后我们就得到了一个 Flutter 的 module ，在 my_flutter/.android/下面。在app/build.gradle 下面添加这个 module 的依赖  **implementation project(':flutter')**

然后在 Sync now ，第二个坑就出现了
# 坑二
**错误: 不兼容的类型: FlutterPluginRegistry无法转换为FlutterEngine**

出现问题的代码是在 Flutter.java 中的第 99 行，如图红框位置

![添加图片](../../../../images/flutter_quesioint_one.jpg)

查了好久不知道为啥，这文件是自动生成的，怎么会出现问题呢，搞不懂，但是问题解决了：<font color="#ff000" >直接注释掉改行代码即可。</font>   简单粗暴。但是好使。

原因不知道，先记录下来，没准后面学习中，就柳暗花明了。

目前没发现有啥影响。试验在有序进行。然而在最后关头，又遇见坑了

# 坑三
Flutter 栈管理中有一个 Flutter 页面跳转到另外一个 Flutter ，然后我就在第一个页面中写了一个 RaiseButton ,然后在 onpress 中通过 Navigator 进行跳转，本以为很简单的事情，可是谁知道点击后无反应，查看 log 发现是这个

```java
Navigator operation requested with a context that does not include a Navigator.
    The context used to push or pop routes from the Navigator must be that of a widget that is a
    descendant of a Navigator widget.
 When the exception was thrown , this was the stack:
I/flutter: #0      Navigator.of.<anonymous closure> (package:flutter/src/widgets/navigator.dart:1495:9)
    #1      Navigator.of (package:flutter/src/widgets/navigator.dart:1502:6)
```

这是为啥呢，继续查找咯，原来<font color="#ff000" > 直接在 MaterialApp 中 push 是不行的，要加一层 。 </font>

详情参考这篇文章[  flutter页面间跳转和传参-Navigator的使用](https://segmentfault.com/a/1190000015150843)

眼看就大功告成，试验结束了，可是谁知道。还有一个坑再等我
# 坑四
 从第一个 Flutter 页面跳转到第二个 Flutter 页面，点击返回，可以返回到第一个页面，但是再点击返回，就没任何反应了，感觉第一个 Flutter 页面没有接收到通知一样。最后发现，如果要是把那段代码放到 build() 中就可以了，放到 initState() 中就不行
如这样
```java
@override
 Widget build(BuildContext context) {
   Future<dynamic> handler(MethodCall call) async {
     var canPop = Navigator.canPop(context);
     print("MyApp  call.method  ${call.method}    canPop： $canPop");
     switch (call.method) {
       case "onActivityResult":
         name = call.arguments["message"];
         setState(() {
           value = "route1?{\"name\":\"$name\"}";
         });
         break;
       case "goback":
         if (canPop) {
           Navigator.pop(context);
         } else {
           nativeChannel.invokeMethod("goback");
         }
     }
   }
   nativeChannel.setMethodCallHandler(handler);
   return _widgetForRoute(value);
 }
```
并且这个页面，也需要处理 goback 事件才行，因为第二个页面 pop 的时候，已经 dispose() 了，
所以需要在第一个页面监听，处理才行。这样就能实现栈管理了。

坑终于填完了。可以安心睡觉了。

---
搬运地址：    

