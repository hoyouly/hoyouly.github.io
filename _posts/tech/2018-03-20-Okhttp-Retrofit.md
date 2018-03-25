---
layout: post
title: Okhttp和Retrofit
category: 技术
tags: Okhttp,Retrofit
description: 介绍这两个框架的作用和联系
---

话题：Okhttp和Retrofit

1、介绍这两个框架的作用和联系
PS：这个问题几乎Android面试必问

## Retrofit
Retrofit是Square公司开发的一款针对Android网络请求的框架，**Retrofit2底层基于OkHttp实现的**，OkHttp现在已经得到Google官方认可，大量的app都采用OkHttp做网络请求，

retrofit封装了okhttp,okhttp使用的还是 httpurlconnection.不过retrofit非常适合于restful url格式的请求，更多使用注解的方式提供功能。

Retroifit是基于okhttp的封装，Retrofit利用注解来方便拼装请求参数（比如headers或者params，就是通过注解，然后拼接到网络请求中），对于返回结果也可以完美适配Rxjava，如果用的其他结构的数据，也可以自己写相对应的Covers来解析。总而言之，Retrofit是基于okhttp的封装，更加方便利用okhttp的使用。
如果看源码会发现其实质上就是对okHttp的封装，使用面向接口的方式进行网络请求，利用动态生成的代理类封装了网络接口请求的底层,

是一个RESTful的HTTP网络请求框架的封装，网络请求的本质是OkHttp 完成的，Retrofit 只是进负责网络请求接口的封装

其将请求返回javaBean，对网络认证 REST API进行了很好对支持此，使用Retrofit将会极大的提高我们应用的网络体验。

### REST
既然Retrofit是一个RESTful的HTTP网络请求的框架的封装，那么什么是RESTful架构呢
REST REpresentational State Transfer的意思 ，是一组架构约束条件和原则
RESTful 都满足以下条件
1. 每一个URL代表一个资源
2. 客户端和服务器之间，传递这种资源的某种表现层
3. 客户端通过四个HTTP动词，对服务端资源进行操作，实现表层状态转换
更多关于REST的介绍：[]什么是REST - GitHub讲解的非常详细](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/08.3.md)



![](http://p5sfwb51p.bkt.clouddn.com/QQ20180325-205301@2x.png)

几种网络框架的对比
![](http://p5sfwb51p.bkt.clouddn.com/%E5%87%A0%E7%A7%8D%E7%BD%91%E7%BB%9C%E6%A1%86%E6%9E%B6%E7%9A%84%E5%AF%B9%E6%AF%94.png)


搬运地址：

[Retrofit2 完全解析 探索与okhttp之间的关系](https://blog.csdn.net/lmj623565791/article/details/51304204)


[这是一份很详细的 Retrofit 2.0 使用教程（含实例讲解）](https://blog.csdn.net/carson_ho/article/details/73732076)

[Retrofit2.0使用总结](https://www.jianshu.com/p/3e13e5d34531)
