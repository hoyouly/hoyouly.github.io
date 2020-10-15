---
layout: post
title: 卧槽系列 - Fragment already added 或许并不是你想的那样
category: 卧槽系列
tags: Fragment ViewPager
---
<!-- * content -->
<!-- {:toc} -->

不知道大家有没有遇到过这样的问题，一个 bug 查了好几天，都没找出原因，郁闷，苦恼，网上各种查资料，百度Google轮番上，网上也有遇到这种问题的，可是按照上面说的，就是不能解决。就在你要缴械投降的时候，突然发现，

卧槽，竟然是这里出的问题。

卧槽，竟然是这样玩儿的，

卧槽，竟然...

一系列卧槽喷涌而出。

是不是有一种   
```
踏破铁鞋无觅处，得来全不费工夫。
山重水复疑无路，柳暗花明又一村。
```  
所有就有了这个卧槽系列系列


最近遇到一个很奇怪的问题，这个必须得记录一下。

一个布局文件，刚开始是 NestedScrollView 嵌套了一个 SmartRefreshLayout ，如下所示

```xml
<android.support.v4.widget.NestedScrollView
  <com.scwang.smartrefresh.layout.SmartRefreshLayout

```
可是因为业务需要，需要得把 SmartRefreshLayout 放到最外层， NestedScrollView 放到里层，如下所示

```xml
<com.scwang.smartrefresh.layout.SmartRefreshLayout

  <com.scwang.smartrefresh.header.MaterialHeader

  <android.support.v4.widget.NestedScrollView

```
原本挺简单的一个东西，可是只要我一改，就崩溃，原因就是 `java.lang.IllegalStateException: Fragment already added:`
![添加图片](../../../../images/fragment_alread_add.png)

我刚开始一直怀疑是因为布局嵌套太深了，因为这个 Fragment 可以说是一个孙子辈 Fragment 。一个页面，最外层是一个 ViewPager ,中间层还有一个 Viewpager ，这个 Fragment 只是中间层 Viewpager 中的一个，然后各种查百度+Google，都说什么 addFragement 之前要判断，可是我使用的是Viewpager+ Fragment，就没有直接进行 add 啊，而且布局改过来就正常了。毛线啊。也查到了一些感觉靠谱的资料.比如 [修复一个ViewPager+Fragment报的java.lang.IllegalStateException Fragment already added问题](https://blog.csdn.net/newone_helloworld/article/details/88537285)  ，感觉说的有道理，然后就按照做了，可是还是照崩不误。
还有这个 [Fragment already added问题解决方法，记录一下](https://www.jianshu.com/p/3c88629070bd)，依旧如此。

纠结了一天，最后还是同事提醒了我，既然是布局出的问题，那就在加载布局的时候 try catch 一下，把崩溃信息打印出来，可能根本原因并不是 Fragment already added 。
然后就照做了，

```java
try {
    View view = inflater.inflate(R.layout.app_market_quotation_fg_layout, container , false);
    return view;

} catch (Exception e) {
    e.printStackTrace();
    return null;
}
```

然后就找到真正,竟然是
```
java.lang.IllegalStateException:
Required view 'smartRefresh' with ID 2131297313 for field 'mRefreshLayout' was not found.
If this view is optional add '@Nullable' (fields) or '@Optional' (methods) annotation.
```

没有找到 smartRefresh ，然后我就发现，自己在移动布局的时候，不小心把 SmartRefreshLayout 中的定义的 id smartRefresh 给删掉了，而使用的 ButterKnife 去查找的时候，就出错了。看来干活还得细心啊。至于为啥没报这个错误，而是提示 Fragment already added ，这个还需要再研究。

记录下来这个，主要是想说，你看到的并不一定是真相，可能你查了半天，一天，根本原因就不在哪里，例如这个 Fragment already added ，这就需要自己看一全部 log ，而我一般崩溃了都会直接 adb logcat -b crash ,看崩溃信息，可是很不幸，真正关键的信息却被过滤了，然后查看所有 log ，就能查看出来。如下图

![添加图片](../../../../images/fragment_not_found.png)


---
搬运地址：    

[修复一个ViewPager+Fragment报的java.lang.IllegalStateException Fragment already added问题](https://blog.csdn.net/newone_helloworld/article/details/88537285)

[Fragment already added问题解决方法，记录一下](https://www.jianshu.com/p/3c88629070bd)
