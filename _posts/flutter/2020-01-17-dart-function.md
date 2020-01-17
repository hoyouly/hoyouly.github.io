---
layout: post
title:  Flutter学习之Dart 中的Function
category: Flutter 学习
tags: Flutter  Dart
---
* content
{:toc}

最近开始学习Flutter，然后学道了Dart的函数，发现挺有意思的。
Dart里面所有的东西都是对象，包括 int,函数 这些对象的父类是Object，这倒是有意思，如果一个函数也是对象，那么类型是啥呢？ 答案是Function，这是Dart中的一种类型。
Function 格式 ：  返回类型 函数名（参数列表）{函数体}
## 返回类型
可以省略，因为返回值类型可以通过return的值类型推断出来的，如果没有return，那么返回类型自然就是void了，但是不建议这么做。

## 函数名
这个没啥好说的，符合命名规则就好，尽量名称能概括函数体的内容

## 参数
分以下几种
1. 正常的参数
2. 可选命名参数
3. 可选位置参数
4. 默认值参数

### 正常的参数
这个和java一样，没啥好说的

###  可选命名参数
可以在函数传递参数时，指定参数名，这个在Flutter中经常看到。
定义的时候使用{}将形参包含起来，如下：
```java
void sayHello({String name, String msg}) => print("$name: $msg");
```
{}中的参数可传可不传，使用就如下了
```java
void main() {
    sayHello(name: "david", msg: "你好啊"); // david: 你好啊
    sayHello(msg: "你好啊", name: "david",);  // david: 你好啊
    sayHello(msg: "hello");         //null: hello
    sayHello(name: "david");     //david: null
    sayHello();
}
```

### 可选的位置参数
语法格式 使用[] 进行包裹，如下
```java
String say(String from, String msg, [String device, String time]) {
    var result = "$from says $msg";
    if (device != null) {
      result = "$result with a $device";
    }
    if (time != null) {
      result = "$result at $time";
    }
    return result;
}

void main(){
    print(say("david", "hello", "mobile", "2019.2.29.20:08:08"));
    // david says hello with a mobile at 2019.2.29.20:08:08
    print(say("david", "hello", "2019.2.29.20:08:08", "mobile"));
    // david says hello with a 2019.2.29.20:08:08 at mobile
    print(say("david", "hello"));
    // david says hello
}
```

1. 同样作为可选参数，故在调用时可以不传
2. 是通过位置来确定实参和形参的对应关系的

### 默认值参数
这个很简单，就是给 可选的命名参数或者 可选的位置参数指定默认值，因为他们都是可选的，
```java
String say(String from, String msg, [String device = "mobile", String time = "2019.2.23.18:30:00"]) {
  var result = "$from says $msg";
  if (device != null) {
    result = "$result with a $device";
  }
  if (time != null) {
    result = "$result at $time";
  }
  return result;
}
```
给device 和time 都设置了默认值，如果不填写，就使用默认值了

然后我们需要再介绍几个新鲜词语 @required ,  匿名函数，typedef

###  @required  
可选参数列表中某个参数添加该注解，表示该参数为必须设置，有点像是必填项

### 匿名函数
语法 ([type] param){}   参数 类型可选，例如

```java
void main(){
    var list = [1, 2, 3];
    list.forEach((item) {
       print(item);
    });

}
```
### typedef
给某种特定的函数类型起一个名字，可以认为是一个类型的别名，
可以类比class和对象这样理解：自己定义了一种数据类型，不过这种数据类型是一个函数类型，一个一个的具体实现的函数就相当于按照这种类型实例化的对象

```java
//声明参数为int,无返回值的callback函数 MenuCallBack
typedef MenuCallBack = void Function(int position);
//声明一个参数为 context 和int,返回值是Widget 的callback函数 IndexedWidgetBuilder
typedef IndexedWidgetBuilder = Widget Function(BuildContext context, int index);

//声明一个 参数为 BuildContext，返回值 Widget 的callback 函数 WidgetBuilder
typedef WidgetBuilder = Widget Function(BuildContext context);
```

## 例子说明
```java
FlatButton(
  onPressed: () {
    Navigator.push(
      context,
      MaterialPageRoute(
        builder: (context) {
          return NewRoute();
        },
      ),
    );
  },
  child: Text("点击跳转"),
)
```

相信这样的代码应该会经常看到，现在我们就来解剖一下这段代码的意思，还原一下他们的真实面貌

### 省略new 关键字
因为我们知道在Dart创建对象的时候，可以省略到new 关键字，上面代码已经省略了，现在我们把他们添加上去,
```java
new FlatButton(
  onPressed: () {
    Navigator.push(
      context,
      new MaterialPageRoute(
        builder: (context) {
          return new NewRoute();
        },
      ),
    );
  },
  child: new Text("点击跳转"),
)
```
FlatButton 和 MaterialPageRoute 以及 NewRoute,Text 前面都添加了，这是要创建三个对象，这三个也都是Widget
### onPressed
我们继续看看第二行，也就是onPressed这一行
在看第二行之前，我们先看看FlatButton的构造函数
```java
const FlatButton({
    Key key,
    @required VoidCallback onPressed,
    ... 省略
     @required Widget child,
  })...

  typedef VoidCallback = void Function();
```
我们可以得知
* 因为有{} 包裹 onPressed和 child ,所以是可选命名参数
* 因为签名有 @required注解，所以 onPressed 和 child 又是必填参数，
* 因为 typedef VoidCallback = void Function(); 可知onPressed 的类型是 一个返回值是void，无参数的callback函数 VoidCallback，

所以综上所述，onPressed  是一个必填的参数，类型是一个返回值是void，无参数的函数，  child 是一个类型是 Widget 的必填参数，这个后面说到，现在继续说onPressed
onPressed: 后面是一个匿名函数，格式就是 (){}，参数无

在点击的FlatButton 的时候，会执行 这个匿名函数的函数体，也即时执Navigator.push()函数
### Navigator.push()
push()有两个参数，一个是context，一个是 Route
context 上面有，
Route 就需要重新创建了一个,即 new MaterialPageRoute(),

### 创建MaterialPageRoute 对象
先看MaterialPageRoute 的构造函数
```java
MaterialPageRoute({
   @required this.builder,
   RouteSettings settings,
   this.maintainState = true,
   bool fullscreenDialog = false,
 })：...省略
```
看到了{},可选参数，看到了 @required 注解，所以builder 就是必填可选参数，
然后我们看看 builder的定义吧
```java
final WidgetBuilder builder;
typedef WidgetBuilder = Widget Function(BuildContext context);
```
类型是WidgetBuilder,其实就是返回类型是Widget ，参数是BuildContext 的 callBack 函数 ,既然他们要的是这样一个callback 函数，那么就创建一个这样的函数

### 匿名函数
创建一个参数类型是BuildContext， 返回值是 Widget的 匿名函数，那么应该是
(BuildContext context) {return new Widget的();}

* 匿名函数中 参数类型可省略。
* 创建对象的时候，new 关键字 可以省略，
* NewRoute 是 继承StatelessWidget ，属于一个 Widget。

根据上面这三点，这个匿名函数就变成了 (context) {return NewRoute();} 这个鸟样。

然后还剩下一个child参数
### child参数
 创建了一个Text()对象，看看Text 的构造函数
```java
const Text(
     this.data, {
     Key key,
     ...
   }) ...
final String data;
```
看到了没，data 在{} 之外，也就是正常的参数，类型是String的，那么就需要给他传递值，即"点击跳转"，后面都是可选命名参数，所以可以不写。


---
搬运地址：
[Flutter和Dart系列四：Function](https://blog.csdn.net/xlh1191860939/article/details/87895616)      
[Flutter学习日记：Dart语言学习之typedef](https://blog.csdn.net/FreeAndWake/article/details/88979769)
