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

接口是无状态的，即时提供了默认的方法实现也是很简单的，不能实现复杂的逻辑，也不推荐在接口中实现复杂的方法逻辑

### 构造函数
kotlin 没有new 关键字，直接声明一个对象，创建对象的时候，如下
`val bird= Brid()`

构造函数默认参数

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
<font color="#ff000" >  构造函数中参数带上val 或者var 的话，等价于在Brid类内部声明了一个同名的属性,这些属性可以直接在函数中使用的。</font>例如
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

正常情况下，kotlin规定类中的所有非抽象属性成员都必须在对象创建时被初始化。但也可以在使用到该属性的时候进行初始化，这就是我们说的延迟初始化。

#### 延迟初始化 by lazy 和lateinit

* by lazy : 主要用于 val 声明的变量，在首次调用时候，才会进行赋值操作，一旦被赋值，后续将不能被更改。
* lateinit: 主要用于var 声明的变量，然后不能用于基本数据类型。如Long，Int等
* Delagate.notNull<T>： 可用于 var 声明的基本数据类型实现延迟加载

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

### 限定修饰符
* open   
kotlin 默认的类和方法是final的，是不可继承或重写的，如果想要继承或者重载，所以必须加上 open 修饰符.    
Google官方主要通过kotlin中的扩展语法对Android标准库进行扩展。即android-ktx, 而不是通过继承原始类的手段。

* sealed
通过sealed关键字来修饰一个类为密封类，若要继承则需要将子类定义在同一个文件中。其他文件中的类将无法继承它。

### 可见性修饰符
1. kotlin 与java的默认修饰符不同，kotlin中是public，而java中是default，只允许包访问。
2. kotlin 中有一个独特的修饰符internal （模块内访问）
3. kotlin 可以在一个文件内单独声明方法以及常量，同样支持可见性修饰符
4. java中除了内部类可以用private修饰符以外，其他类都不允许private修饰，而kotlin可以，他的作用域就是当前这个kotlin文件。
5. kotlin和java中的protected的访问访问不同，java是包，类，以及子类可以访问，而kotlin只允许子类。

那么什么是模块呢？ 一个模块可以看做一起编译的kotlin文件组成的聚合。以下情况可以看做一个模块
* 一个Eclipse项目
* 一个Intelij IDEA项目
* 一个Maven项目
* 一个Gradle项目
* 一组由一次Ant任务执行编译的代码

## 多继承的问题

### 接口实现多继承

1. 两个接口，定义了相同的方法名，并且都带有默认实现

```kotlin
interface Flyer {
    fun kind() = "Flyer kind"
    fun fly()
}

interface Animal {
    val name: String
    fun eat()
    fun kind() = "Animal kind"
}

class Brid(override val name: String) : Animal, Flyer {
    override fun eat() {
        println("I can eat")
    }

    override fun kind() = super<Flyer>.kind()

    override fun fly() {
        println("I can fly")
    }
}

fun main() {
    val brid = Brid("sparrow")
    println(brid.kind()) //Flyer kind
    brid.eat()//I can eat
    brid.fly() //I can fly
  }
```
虽然两个接口都定义并实现了kind()方法，Brid类同时实现了这两个类，这种情况必须override，指明要使用哪个接口中的kind,格式就是 `super<T>.`,T就是拥有该方法的接口名。  
super<Flyer>.kind()，代表使用Flyer中的kind()

2. 两个接口，虽然方法名相同，但是一个带有默认实现，一个是没有，如下

```kotlin
interface Flyer {
    fun kind(): String
    fun fly()
}
interface Animal {
    val name: String
    fun eat()
    fun kind() = "Animal kind"
}

class Brid(override val name: String) : Animal, Flyer {
    override fun eat() {
        println("I can eat")
    }

    override fun kind(): String {
        return super.kind()
    }

    override fun fly() {
        println("I can fly")
    }
}

fun main() {
    val brid = Brid("sparrow")
    println(brid.kind())  //Animal kind
    brid.eat() //I can eat
    brid.fly() //I can fly
  }
```
Flyer中kind()是抽象的，而Animal中kind()方法是带有默认实现的。

这种情况下，可以把kind()当做Flyer类中的抽象方法，直接写实现类，也可以当做Animal 中的，执行super.kind()

```kotlin
class Brid(override val name: String) : Animal, Flyer {
    override fun eat() {
        println("I can eat")
    }

    override fun kind(): String {
        val kind = super.kind()
        println(kind)
        return "Brid kind"
    }

    override fun fly() {
        println("I can fly")
    }
}

fun main() {
    val brid = Brid("sparrow")
    println(brid.kind())
}
//结果
Animal kind
Brid kind
```
重写的kind()方法中执行了super.kind()，其实这个super指的就是Animal 了。

<span style="border-bottom:1px solid red;"> 注意：在实现类的属性和方法时候，都必须带上overrde关键字。不能省略。</span>



### 内部类
内部类可以继承一个与外部类无关的类，所以这就保证了内部类的独立性

kotlin 中的内部类必须加上inner关键字。否则就是一个嵌套类，刚好和java相反，java中默认的是内部类，加上static后是嵌套类

* 内部类  包含对外部类实例的引用，在内部类中我们可以使用外部类的属性
* 嵌套类   不包含其外部类实例的引用，无法调用外部类的属性

```kotlin
class OuterKotlin {
    val name = "This is outer name"

    inner class inKotlin {
        fun printName() {
            println("the name is ${name}") //去掉inner关键字，这个地方就会报错。
        }
    }
}
```

## 数据类
使用 data class  表示一个数据类，可以理解为javabean，包括了equals()和hashcode().

```kotlin
data class Brid(val name: String)
```

* 数据类必须拥有一个构造方法，改方法只是包含一个参数，一个没有数据的数据类是没有任何用处的
* 与普通的类不同，数据量的构造方法的参数强制使用var或者val进行声明
* data class 之前不能使用abstract,open,sealed或者inner进行修饰
* 在kotlin1.1 版本之后，数据类既可以实现接口，也可以继承类

```kotlin
interface Flyer {
    fun kind(): String
    fun fly()
}

data class Brid(val name: String) :Flyer{
    override fun kind(): String {
        return "brid kind"
    }
    override fun fly() {
    }
}
```


## 从static到object

kotlin中没有static 关键字，引入了全新的object关键字。它可以完美的替代static的所有场景，还能实现更多的功能，比如单例对象，简化匿名表达式等

### 伴生对象
使用companion object 关键字，根java中的static修饰效果性质是一样的。全局只有一个单例。需要声明在类的内部，在类被装载时会被初始化。

```kotlin
class Prize(val name: String, val type: Int) {
    companion object {

        val TYPE_RED = 1
        val TYPE_BLUE = 2;

        fun isRed(prize: Prize):Boolean {
            return prize.type == TYPE_RED
        }
    }
}

fun main() {
  val prize=Prize("红包",Prize.TYPE_RED)
  println(Prize.isRed(prize))
}
```
companion object 用{}包裹了所有静态属性的行为，是的它可以与Prize类的普通方法和熟悉清晰区分开。

  任何在java类部可以使用static定义的内容都可以使用kotlin中的伴生对象来实现

### objec 天生的单例
如果想定义一个单例的类，只需要在类名前面加上object 即可

```kotlin
object DatabaseConfig {
    val host: String = "127.0.0.1"
    val prot: Int = 8000
    val userName: String = "root"
}
```
单例同样可以实现接口和继承类，可以看成一种不需要我们主动初始化的类，也可以拥有扩展方法







---
搬运地址：    

Kotlin 核心编程 - 水滴技术团队
