---
layout: post
title: Kotlin核心编程 - 面向对象
category: 读书笔记
tags: Kotlin核心编程
---
<!-- * content -->
<!-- {:toc} -->

## 类和构造方法
kotlin中类声明与java不同之处
* 不可变属性成员。kotlin支持用val 在类中声明引用不可变的属性成员，使用var 声明的属性则引用可变属性成员
* 属性默认值。在kotlin中，除非显示声明延迟初始化，否则就需要制定属性的默认值
* 不同的可访问修饰符。kotlin类中成员默认都是全局可见的。而java中默认可见域是包作用域。

```kotlin
class Brid  {
    val weight: Double = 500.0
    val color: String = "blue"
    val age: Int = 1
    fun fly() {}
}
```


### 接口
kotlin 中的接口可以带有方法的实现，同时支持抽象属性。

```kotlin
interface Flyer {
    //抽象属性，同方法一样 若没有指定默认行为，则在实现该接口的类中必须对该属性进行初始化
    val speed: Int
    fun kind()

    fun fly() {//带有方法的实现
        println("I can fly")
    }
}
```

kotlin 是基于java6，

如果一个类实现该接口，方式如下，注意。 属性 speed 没有指定默认行为，则在实现该接口的类中必须对该属性进行初始化

```kotlin
class Plane : Flyer {
    override fun fly() {}

    override val speed: Int //对抽象属性进行初始化。
        get() = 10

    override fun kind() {
        println(" brid kind ")
    }
}
```

也可以在接口中直接对属性进行初始化，那样就在实现类中就可以省略对该属性的初始化操作。
```kotlin
interface Flyer {
    val speed: Int //直接对属性初始化
        get() = 100

    fun kind()

    fun fly() {//带有方法的实现
        println("I can fly")
    }
}
```

```kotlin
class Plane : Flyer {
    override fun fly() {}

    //override val speed: Int //对抽象属性进行初始化。
      //  get() = 10

    override fun kind() {
        println(" brid kind ")
    }
}
```

### 构造函数
kotlin 没有new 关键字，直接声明一个对象，创建对象的时候，如下
`val bird= Brid()`

* 构造函数默认参数
上面的Brid类，可以简写成
```kotlin
class Brid(
    val weight: Double = 500.0,
    val color: String = "blue",
    val age: Int = 1
){
    fun fly() {}
}
```

在创建类对象的时候，最好指定参数的名称，否则必须按照实际参数的顺序进行赋值,指定参数名称的，顺序可以随意，如下

```kotlin
Brid(weight = 11.1, color = "black", age = 2)

Brid(weight = 11.1,  age = 2,color = "black")
Brid(weight = 11.1)

Brid(11.1,  "black", 2)//必须按照实际顺序赋值
```

构造方法中可以没有val和var ,如下所示
```kotlin
class Brid(
     weight: Double = 500.0,
     color: String = "blue",
     age: Int = 1
) {
    fun fly() {}
}
```
带上val 或者var 的话，等价于在Brid类内部声明了一个同名的属性,这些属性可以直接在函数中使用的，例如
```kotlin
class Brid(
    val weight: Double = 500.0,
    val color: String = "blue",
    val age: Int = 1
) {
    fun fly() {
        println("weight : $weight ,color : $color ,age : $age ")
    }
}
```

而如果参数前面没有使用 val 或者var 的话，需要手动创建属性的。

```kotlin
class Brid(
     weight: Double = 500.0,
     color: String = "blue",
     age: Int = 1
) {
    fun fly() {
      //  println("weight : $weight ,color : $color ,age : $age ") //不能直接使用，
    }
}
```

但是一种情况除外，那就是init 语句块
```kotlin
class Brid(
     weight: Double = 500.0,
     color: String = "blue",
     age: Int = 1
) {
    init { //可以正常使用构造函数中的属性的
        println("weight : $weight ,color : $color ,age : $age ")
    }
}
```

### init 语句块

属于构造函数的一部分，所以可以在init函数中使用构造函数中的属性，尽管该属性没有使用val 或者var 进行声明

一个类中可以拥有多个init语句块，他们会在对象被创建时按照类中从上到下的 顺序先后执行

```kotlin
class Brid(
     weight: Double = 500.0,
     color: String = "blue",
     age: Int = 1
) {
    init {
        println("weight : $weight")
    }

    init {
        println("color : $color")
    }
    init {
        println("age : $age ")
    }
    fun fly() {
//        println("weight : $weight ,color : $color ,age : $age ")
    }
}
```

正常情况下，kotlin规定类中的所有非抽象属性成员都必须在对象创建时被初始化。

#### 延迟加载 by lazy 和lateinit

* by lazy : 主要用于 val 声明的变量，在首次调用时候，才会进行赋值操作，一旦被赋值，后续将不能被更改。
* lateinit： 主要用于var 声明的变量，然后不能用于基本数据类型。如Long，Int等
* 用var声明的基本数据类型实现延迟加载，可以使用Delagate.notNull<T>,

```kotlin
class Brid(
    val weight: Double,
    val color: String
) {
    lateinit var sex: String
    fun printSex() {
        this.sex = if (this.color == "yellow") "male" else "female"
        println("sex : $sex")
    }

    val name: String by lazy {
        "weight: ${weight} ,color $color"
    }

    fun printName() {
        println("name : $name")
    }

    var age: Int by Delegates.notNull<Int>()
    fun printAge() {
        age = 10
        println("age : $age")
    }
}

fun main() {
    val brid = Brid(weight = 11.1, color = "black")

    brid.printSex() //sex : female
    brid.printName() //name : weight: 11.1 ,color black
    brid.printAge()  //age : 10
}
```
### 主从构造函数
* 通过constructor定义的一个新的构造函数，称为从构造函数

* 主构造函数： 在类外部定义的构造函数被称为主构造函数。如果主构造函数存在注解或可见性修饰符，也必须像从构造函数一样加上 constructor 关键字。

如果一个类存在主构造函数，那么每个从构造函数都要直接或间接的通过this 关键字 委托给它。

```kotlin
class KotlinView : View {

    constructor(context: Context) : this(context, null)

    constructor(context: Context, attributes: Attributes?) : this(context, attributes, 0)

    constructor(context: Context, attributes: Attributes?, defaultStyle: Int) : super(context, attributes, defaultStyle) {
    }
}

```

* 限定修饰符
kotlin 默认的类和方法是final的，是不可继承或重写的，如果想要继承或者重载，所以必须加上 open 修饰符


---
搬运地址：    

Kotlin 核心编程 - 水滴技术团队
