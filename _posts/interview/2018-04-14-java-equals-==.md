---
layout: post
title: 街题系列 - java 中 "==" 和 equals() 的区别
category: 街题系列
tags: Java  equals
---
<!-- * content -->
<!-- {:toc} -->

我相信这个也是绝对的街题一个，在面试中没有被直接问道，也肯定会间接遇到，今天就一块整理出来吧

## "==" 和 equals() 的区别
Java 语言中， equals() 方法是交给开发者自己去复写的，让开发者自己定义满足什么条件下两个 Object 对象相等的，默认的是比较引用地址，但是很多类都重写了这个方法，例如 String ,返回 true 表示两个对象内容相同，而不是两个地址相等。所以很多情况下，两 String 变量 equals() 返回 true ,而“==”结果却是 false ,那是因为"=="对比是基于两个对象的内存引用，如果两个对象的内存引用完全相同，即指向同一个对象，那么“==”返回 true ，否则，返回false

* “==”用于比较基本数据类型，例如 int ， char , boolean 等， equals() 用于比较对象，
* 如果两个变量指向同一个对象，“==”返回 true ,而 equals() 的返回结果依赖于业务逻辑，因为我们可以重写 equals() 方法，
* 在 String 类中， equals() 方法被重写，返回 true 表示两个对象内容相同，但是他们的引用地址并不一定相同。“==”返回 ture ,表明两个 String 对象的引用地址相同

```java

    String s3=new String("abc");
    String s4=new String("abc");
    System.out.println("s3==s4 : "+(s3==s4));
    System.out.println("s3.equals(s4) : "+s3.equals(s4));


    String s5="abc";
    String s6="abc";
    System.out.println("s5==s6 : "+(s5==s6));//ture
    System.out.println("s5.equals(s6) : "+s5.equals(s6));

```
运行结果是：  
```java
s3==s4 : false
s3.equals(s4) : true
s5==s6 : true
s5.equals(s6) : true
```
虽然 equals() 返回的 true ,但是“==”却有区别，因为 s3 和 s4 变量分别被赋予不同的对象，所以引用地址不相同，而因为 JVM 中字符常量池的缘故，“abc”是被放到常量池中的， s5 和 s6 都是指向常量池中的这个“abc”,指向相同地址，所以“==”为true

---
搬运地址：    

[如何“记住” equals 和 == 的区别？](https://www.zhihu.com/question/26872848)
