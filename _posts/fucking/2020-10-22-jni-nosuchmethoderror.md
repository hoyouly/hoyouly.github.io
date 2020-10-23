---
layout: post
title: 卧槽系列 - JNI NewStringUTF called with pending exception java.lang.NoSuchMethodError 或许并不是你想的那样
category: 卧槽系列
tags: NoSuchMethodError
---

<!-- * content -->
<!-- {:toc} -->

前两天接到一个新的任务，研究百度的PaddleOCR，准备把现有的讯飞OCR替换掉。下载了一个DEMO运行一下，然后就想着替换试试，可是刚开始就出现错误了，已运行就闪退，查看log，发现竟然是一个Error
```
JNI DETECTED ERROR IN APPLICATION:
JNI NewStringUTF called with pending exception java.lang.NoSuchMethodError:
no non-static method "Lcom/baidu/ai/edge/core/base/i;.getInt(Ljava/lang/String;)I
```

全部log如下


![添加图片](../../../../images/jni_nosuchmethoderror.png)


为啥出现这个问题，集成PaddleOCR，文档很简单啊，而且在刚开始初始化的时候，就直接报错了，不应该啊。

刚开始我怀疑是和包名有关，可是修改包名后还是不行，难道和我项目中之前的依赖库有冲突，然后我新建了一个项目，里面就一个Helloword，可是还有这个问题，又开始郁闷了。为毛线啊。不应该啊。没理由啊，各种疑问全部涌现脑子里。

可是遇到问题得解决啊。不会解决问题的程序猿不是高级程序猿。

既然DEMO是可以正常运行的，那么先精简DEMO吧。

把无关紧要的地方全部干掉，只留下初始化那一部分。为了看出来我修改了什么，我特意新建了一个git 库，把DEMO放进去。 删一些代码，运行正常就提交上去。这样一点点的删除，运行，提交，终于精简到整个项目中里面就只是剩下一个MainActivity，而这里面的操作就是开启一个线程，初始化OcrInterface,如下图所示

![添加图片](../../../../images/paddle_ocr_demo.png)

这足够很简单了吧，也能正常初始化。

然后开始把这一部分代码copy到新的项目中，包括lib库，so库，还是报这个错。问题到底出现在哪里了呢。

把两个build.gradle也copy过来了啊，还是报错。

开始崩溃了。到底哪里不一样呢。哪里出的问题了呢？百思不得其解。那就祭出来领一个神器吧。Beyond Compare,对比一下吧。

对比果真还是发现了一些不一样的地方，竟然是混淆文件 proguard-rules.pro。之前一直没太留意的，在DEMO中添加了

**-keep class com.baidu.ai.edge.core.*.*{ *; }**

而我新建的项目中却没有，难道和这个有关，先试试把，死马当活马医咯！添加后，果然正常了。卧槽。这是什么操作。一万只羊驼从百度误我啊。集成文档上明明写的是可选性的。如下图

![添加图片](../../../../images/paddle_ocr_baidu.png)

因为看到是可选项，我就没添加，谁知道这个TM是必选项啊，坑爹啊。害我忙活了一天。不过总算解决了。现在想想，这个问题出现的原因是混淆导致了getInt()这个方法找不到的。


这也给我开拓了解决思路：
1. 怀疑代码
2. 怀疑依赖库
3. 怀疑混淆


---
搬运地址：    
