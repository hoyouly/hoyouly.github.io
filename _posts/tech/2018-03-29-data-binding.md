---
layout: post
title: 扫盲系列之---Data Binding
category: 扫盲
tags: DataBinding
---
* content
{:toc}

# 开启DataBinding
在Moudle下面的build.gradle  中声明
```java
android {
     …………
     dataBinding{
        enabled = true
     }    
     …………
}
```
这样就开启DataBinding ,so easy
# layout 标签
作用： 作为DataBinding的标志，省去findViewById()方法



[告别findView和ButterKnife](https://www.jianshu.com/p/499c8e2b80c4)
[DataBinding实用指南](https://www.jianshu.com/p/015ad08c2c75)
