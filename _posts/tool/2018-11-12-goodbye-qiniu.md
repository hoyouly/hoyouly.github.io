---
layout: post
title: 拜拜了，七牛。GitHub 才是图床的王道
category: 工具
tags: GitHub  图床
---
* content
{:toc}

刚开始搭建博客的时候，就在想着图床从哪里找，最开始是使用有道云笔记当图床，后来觉得麻烦，然后网上很多人都建议使用七牛，什么10G免费存储之类的，觉得还挺好，就用七牛了，可是谁知道，最近收到七牛的邮件，说什么我的测试域名回收通知，如果正式域名，还要什么公安网备案，太麻烦。我就自己写个博客（尽管我好长时间不更新一次），偶尔贴张图片，又不想那么商业化，咋找个免费的这么难啊。难道真的满足不了我这小小需求吗？忽然想到，GitHub好像就行。Google吧。然后就看到了使用GitHub当做图床教程，方法很简单，五部搞定。
1. 在github创建repository。   
例如我建的项目就是 BlogResource ，博客资源，清楚明了，
2. clone到本地，使用GitHub客户端，当然，命令行也行。   
`git clone git@github.com:hoyouly/BlogResource.git`
3. 将需要上传的文件放入本地的文件夹里面.    
因为我想着可能还会有其他资源，所以我又建了一个images文件夹，用来专门放图片。这样如果有视频的话，可以建一个videos,分类管理，方便清洗。
4. 提交到GitHub上。
5. 网页浏览复制图片URL。   
如 https://github.com/hoyouly/BlogResource/blob/master/imges/mvp.png    
`将URL中blob替换为raw`,即 ../../../../images/mvp.png  
注意：   
`将URL中blob替换为raw`   
`将URL中blob替换为raw`   
`将URL中blob替换为raw`   
重要的事情说三遍，我刚开始就是因为不知道这个，所以弄了好长时间都显示不出来，还以为自己弄错了呢。   

可是之前已经往七牛上弄了好多图片了，咋办啊，没办法，`有些人看不着前门就去找后门，得！一下撞到骗子的怀里了！`韩春明这话一点不错，好好的GitHub不用，干嘛非得去用七牛呢。折腾一下吧。全部下载，放到BlogResource/images下面。然后commit。
至于替换之前的连接，这个就好办了，使用Atom，加载整个blog存放的目录，然后全局替换就行了，比如我之前的图片连接都是 以 http://p5sfwb51p.bkt.clouddn.com/ 开头的，那样全局替换成 ../../../../images/ 就可以了，然后再提交，搞定。
当然，如果你有好几个域名，那没办法了，慢慢折腾吧。生命贵在折腾。

这样以后写blog的时候，想贴图片，就简单了，先弄张图片，扔到BlogResource/images下面，那么他最后的路径就是../../../../images/图片名， 路径有了，那就插入图片吧。 最后一块提交，齐活。  
GitHub 才是王道，如果连GitHub都不能用了，那还敲个写个啥博客啊，直接回家种地不就行了。

---
搬运地址：
[github做Markdown图床](https://www.jianshu.com/p/33eeacac3344)
