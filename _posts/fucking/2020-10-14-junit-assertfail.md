---
layout: post
title: 卧槽系列 - junit.framework.AssertionFailedError No tests found in ** 或许并不是你想的那样
category: 卧槽系列
tags: junit
---

<!-- * content -->
<!-- {:toc} -->

我们知道，新建一个Android项目的时候，都会有下面红框中的文件，

![添加图片](../../../../images/android_test.png)

虽然Android开发了有好几年了，可是一直都没用过，测试都是直接运行到手机中的，没写过测试用例的，没那个习惯，而且我一般都会把那几个文件给删掉的

可是今天一个偶然的机会，看到了写测试用例，然后就想着玩玩，可是我的项目中都已经没有这些东西了，没办法，重新添加一份吧。
红框中的就是新添加的部分。

![添加图片](../../../../images/android_test_a.png)

想着没啥问题的，可是在MainPresenterTest类中

```java
@RunWith(MockitoJUnitRunner.class)
public class MainPresenterTest {

    @Test
    public void testMock() {
        Assert.assertEquals("dd","d");
    }
```
这个最简单了吧，可是还是一直报错，错误日志如下

```java
junit.framework.AssertionFailedError: No tests found in top.hoyouly.wechatmonents.presenter.MainPresenterTest
at android.test.AndroidTestRunner.runTest(AndroidTestRunner.java:198)
at android.test.AndroidTestRunner.runTest(AndroidTestRunner.java:183)
at android.test.InstrumentationTestRunner.onStart(InstrumentationTestRunner.java:560)
at android.app.Instrumentation$InstrumentationThread.run(Instrumentation.java:2074)
```
在网上搜了一大堆，都是说啥方法名要以test开头，并且区分大小写之类的，可是还是不行，我就纳闷了，到底是哪里出问题了呢？

已经按照网络上的说的，方法名是以test开头的，

在自己百思不得其解中，看到了 [获取junit.framework.AssertionFailedError：使用Unit和Mockito时，[package]中没有找到任何测试](https://www.it1352.com/993791.html) 这个blog,豁然开朗了，柳暗花明啊。

原来是自己少添加引用了,在 defaultConfig 中添加

 `testInstrumentationRunner 'android.support.test.runner.AndroidJUnitRunner'`

就行了。

其实对比上面两幅图片，也能看出问题了。但在那个时候，却纠结了半天。

不熟悉的东西，稍微不注意，就能折腾你半天啊。不过还好，算是解决了。


---
搬运地址：    

[获取junit.framework.AssertionFailedError：使用Unit和Mockito时，[package]中没有找到任何测试](https://www.it1352.com/993791.html)
