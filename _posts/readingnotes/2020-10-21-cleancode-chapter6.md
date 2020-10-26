---
layout: post
title: 代码整洁之道 - 对象和数据结构
category: 读书笔记
tags: 代码整洁之道
---
<!-- * content -->
<!-- {:toc} -->

看了一遍，没搞明白这里面说的对象和数据结构的具体意思，但是还是决定下记录下来，后面再仔细品味这一部分。

## 对象和数据结构区分
对象和数据结构的区别如下
* 对象暴露行为，隐藏数据，便于添加新的对象而无需修改既有行为，同时也难以在既有对象添加新行为。
* 数据结构暴露数据，没有明显的行为，便于想既有数据结构添加新的行为，同时也难于在既有函数添加新的数据结构。


如下面代码，就是使用数据结构类型代码，也叫过程式代码
```java
public class Squre {
    public Point topleft;
    public double side;
}

public class Retangle {
    public Point topleft;
    public double height;
    public double width;
}


public class Geometry {
    public final double PI = 3.14;

    public double area(Object shape) {
        if (shape instanceof Squre) {
            Squre squre = (Squre) shape;
            return squre.side * squre.side;
        } else if (shape instanceof Retangle) {
            Retangle retangle = (Retangle) shape;
            return retangle.height * retangle.width;
        } else {
            return -1;
        }
    }
}
```
这种如果添加一个新的数据结构，例如，添加一个Circle
```java
public class Circle {
       public Point center;
       public double radius;
   }
```
那么Geometry中所有的函数都需要修改，可是如果想在Geometry中添加一个函数，那么就只需要修改这Geometry一个类就行了，其他类就可以不需要修改。这就是所谓的数据结构易于修改新的行为，难于添加新的数据结构。

可是如果是面向对象类型的代码，也就是使用多态形式表现
```java
public interface Shape {
    public double area();
}

public class Squre extends Shape {
    public Point topleft;
    public double side;

    @Override
    public double area() {
        return side * side;
    }
}

public class Retangle extends Shape {
    public Point topleft;
    public double height;
    public double width;

    @Override
    public double area() {
        return height*width;
    }
}
```
这样的话，如果想要添加一个新的类型Circle的话，就只需要让Circle实现这个Shape接口就行了，其他地方不需要修改，可是如果Shape中想要添加新的函数，例如计算周长，那么所有实现Shape接口的类型都需要修改。这就是所谓的易于添加新的数据结构，难于添加新的行为。


## 德墨忒尔律
得墨忒尔律认为：类C的方法f只应该调用一下对象的方法
* C
* 由f创建的对象
* 作为参数传递给f的对象
* 由C的实体变量持有的对象

换而言之：只更朋友谈话，不与陌生人谈话。


## 其他

* 将变量设置为私有的一个理由：我们不想其他人依赖这些变量，我们还想再心血来潮时能自由修改其类型或实现。

* 隐藏实现并非只是在变量之间放上一层函数这么简单。隐藏实现关乎于抽象，类并不是简单的用set/get方法将变量推向外间。而是暴露抽象接口。以便用户无需了解数据的实现就能操作数据本体。

* 很多时候，我们不愿意暴露数据细节，更愿意以抽象形态表述数据。这并不只是用接口或者set/get方法就万事大吉，要以最好的方式呈现某个对象包含的数据。需要做严肃的思考。傻乐着乱加set/get方法，是最坏的选择。

* 连串的调用通常被认为是肮脏的风格。应该避免。这个和build模式不一样，因为build模式返回的还是对象本身，而这种连串的调用，每次返回的都是不同的对象。

* 隐藏内部结构，防止当前函数因浏览它不该知道的对象而违反得墨忒尔律

* 最为精炼的数据结构是一个只有公共变量，没有函数的类。这种数据结构有时被称为数据传递对象。或者DTO（data transfer object）,常见的是javabean结构。



---
搬运地址：    

代码整洁之道
