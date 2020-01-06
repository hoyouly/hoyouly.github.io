---
layout: post
title: 工作填坑系列---resultCode =0  或许并不是你想的那样
category: 工作填坑
tags: resultCode
---
* content
{:toc}

最近又遇到一个挺好玩的问题，得记录下来。
就是我调用相机，得到一张照片，可是不知道为啥，onActivityResult 中 resultCode 一直返回0，也查了很多资料，什么和启动模式有关啊，什么不能执行finish()啊，什么没添加权限啊，
1. 首先 我这个Activity 是Standard 模式的，所以和启动模式有关的就不可能
2. 我调用相机的时候，没有执行finish()方法，所以这个也不可能
3. 权限问题，我动态申请了，
![添加图片](https://github.com/hoyouly/BlogResource/raw/master/imges/permission_code_1.png)

提示框也出现了啊

![添加图片](https://github.com/hoyouly/BlogResource/raw/master/imges/permission_1.png)

这些都都检查了，可是还是出现这种情况。这到底是什么问题呢，而且程序也不崩溃，也没有什么log，然后我就开始懵逼了，之前一直只是打印当前进程的log，那就看看所有log吧，看不不会有异常输出，还真有，然后就看到了这样的信息，
![添加图片](https://github.com/hoyouly/BlogResource/raw/master/imges/permission_denied.png)
权限被拒，可是我明明添加了权限啊，这是怎么回事啊。
然后我又检查了一遍代码，猛地看到，我写的是READ_EXTERNAL_STORAGE，读的权限，并不是写的权限，难道是这里的问题， 那就试一下吧 WRITE_EXTERNAL_STORAGE。

![添加图片](https://github.com/hoyouly/BlogResource/raw/master/imges/permission_code_2.png.png)

注意红框里面的东西，WRITE_EXTERNAL_STORAGE。然后再次运行，

![添加图片](https://github.com/hoyouly/BlogResource/raw/master/imges/permission_2.png)
同意弹出了申请权限的对话框，感觉没啥区别啊，但是允许的结果却是resultCode = [-1]。看log，也没有那个权限被拒的异常了，然后我有对比了一下这两次动态申请权限的，发现了一个不同点，一个是读取，一个是读写。

![添加图片](https://github.com/hoyouly/BlogResource/raw/master/imges/permission_3.png)


通过这件事，感受了两点
1. copy代码的时候一定要注意，动态读取权限那一部分并不是自己手敲的，而是直接从网上copy过来的
2. 出了问题，如果当前进程中没有异常信息，可以看整个的log信息，没准就能柳暗花明了。


---
搬运地址：
