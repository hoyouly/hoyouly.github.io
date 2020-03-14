
在计算机系统中，图片显示需要CPU，GPU和显示器一起配合完成，CPU负责图像数据的计算，GPU负责图像数据的渲染，显示器负责最终的图像显示

CPU 把计算好的、需要显示的内容交给 GPU，由 GPU 完成渲染后放入帧缓冲区，随后视频控制器根据垂直同步信号（VSync）以每秒 60 次的速度，从帧缓冲区读取帧数据交由显示器完成图像显示。


Flutter通过Skia交给GPU渲染，UI线程通过Dart语言来构建视图结构数据，这些数据会在GPU线程进行图层合并，随后交给Skia引擎加工成GPU数据，而这些数据会通过OpenGl最终提供给GPU渲染

## Skia

由C++ 开发，性能彪悍的2D图像绘制引擎，在图像转换，文字渲染，位图渲染方面都表现卓越，并提供了开发者友好的API，通过与Skia的深度定制以及优化，Flutter可以最大限度的磨平平台的差异，提高渲染效率和性能

底层渲染能力统一了，上层开发接口和功能体验也就随之统一了，开发者不必担心平台相关的渲染特性了，也就是说Skia 保证了一套代码在Android和IOS上渲染效果是一直的。


Flutter 工程实际上就是一个同时内嵌了 Android 和 iOS 原生子工程的父工程：我们在 lib 目录下进行 Flutter 代码的开发，而某些特殊场景下的原生功能，则在对应的 Android 和 iOS 工程中提供相应的代码实现，供对应的 Flutter 代码引用。


Flutter 把视图树进行了扩展，把视图数据和渲染抽象分为三部分，Widget，Element，RenderObject

Flutter 渲染过程，可以分为这么三步：
首先，通过 Widget 树生成对应的 Element 树；
然后，创建相应的 RenderObject 并关联到 Element.renderObject 属性上；
最后，构建成 RenderObject 树，以完成最终的渲染

* Widget 对视图树的一种结构化描述，是控件实现的基本单位，里面存储的是有关视图渲染的配置信息，包括布局，渲染属性，事件相应信息等
* Element Widget 的一个实例化对象，承载这视图构建的上下文数据，是连接结构化的信息和最终渲染的桥梁


* RenderObject


## Widget

Widget 是组件视觉效果的封装，是 UI 界面的载体

描述一个UI元素的配置数据，并不是最终绘制在设备屏幕上的显示元素，只是描述显示元素的一个配置数据

真正代表屏幕上显示元素的是Element，  

**Widget 只是UI元素的一个配置数据，并且一个Widget可以对应多个Element**
同一份配置（Widget），可以创建多个实例（Element）


分两种
* 无状态 StatelessWidget   只能用来展示信息，不能有动作即用户交互，使用直接继承，实现build()即可,整个生命周期都不会变，所以build()只执行一次，
* 有状态  StatefulWidget   可以通过改变状态使得UI发生变化，可以保护用户交互 ，除了继承，需要一个State，只要状态改变，就会调用build()重新创建UI，可以通过setState()来触发，
```java
class BarWidget extends StatefulWidget {
  @override
  State<StatefulWidget> createState() {
    return _BarWidgetState();
  }
}

class _BarWidgetState extends State<BarWidget> {
  @override
  Widget build(BuildContext context) {
    return null;
  }
}
```
定义Widget的惯例
1. widget的构造函数参数应使用命名参数，命名参数中的必要参数要添加@required标注，这样有利于静态代码分析器进行检查。
2. 在继承widget时，第一个参数通常应该是Key，
3. 如果Widget需要接收子Widget，那么child或children参数通常应被放在参数列表的最后。
4. Widget的属性应尽可能的被声明为final，防止被意外改变。

//声明参数为int,无返回值的callback函数 MenuCallBack
typedef MenuCallBack = void Function(int position);


### Context
一个BuildContext 的实例，表示当前Widget在widget树中的上下文，每一个Widget都会对应一个Context对象，

//声明一个参数为 context 和int,返回值是Widget 的callback函数 IndexedWidgetBuilder
typedef IndexedWidgetBuilder = Widget Function(BuildContext context, int index);


### State

一个StatefulElemet对应一个State 实例

#### 生命周期
* initState  
当Widget第一次插入到Widget树时调用，对于每一个State对象，只会调用一次该回调。
通常在该回调中做一些初始化操作，比如状态初始化，订阅子树的事件通知等。
* didChangeDependencies()
当State对象的依赖发生变化时调用，
* build()
在initState()/didChangeDependencies()/setState()/didUpdateWidget()/从一个树中位置移除又重新插入树对其他位置之后都会调用
* reassmble()
为开发屌丝提供，在热重载是会被调用，在Release模式下永远不会被调用
* didUpdateWidget
在Widget重新构建是，会先调用Widget.canUpdate来检测树中的同一位置新旧节点，如果Widget.canUpdate返回true，调用该函数。Widget.canUpdate 在新旧widget的key和runtimeType同时相等时候会返回true，
* deactivate()
从树中被移除会调用
* dispose()
从树中永久移除时候调用

#### 获取State对象
1. 通过context.ancestorStateOfType(TypeMatcher)对象  参数是指定类型的StatefulWidget
```java
// 查找父级最近的Scaffold对应的ScaffoldState对象
ScaffoldState _state = context.ancestorStateOfType(TypeMatcher<ScaffoldState>());
```
2. 通过GloblKey

GloblKey 是Flutter提供的一种在整个APP中引用该Element的机制，如果一个widget设置的Globalkey,那么我们就可以通过gloabkey.currentWidget 获得改widget对象，globalKey.currentElement来获得widget对应的element对象，如果当前widget是StatefulWidget，则可以通过globalKey.currentState来获得该widget对应的state对象。

使用方法
1. 给目标StatefulWidget添加GlobalKey。
```java
//定义一个globalKey, 由于GlobalKey要保持全局唯一性，我们使用静态变量存储
static GlobalKey<ScaffoldState> _globalKey= GlobalKey();
...
Scaffold(
    key: _globalKey , //设置key
    ...  
)
```
2. 通过GlobalKey来获取State对象
```java
_globalKey.currentState.openDrawer()
```

### Element


## TextSpcan

可以对一个内容不同的部分按照不同样式显示，代表文本的一个片段
```java
Text.rich(
    TextSpan(
      children: [
        TextSpan(text: "home:"),
        TextSpan(
          text: "https://ddddddd", style: TextStyle(color: Colors.green),
          //在该片段上用于收拾进行识别处理
//             recognizer: ()=>print(""),
        ),
      ],
    ),
  )
```
home 和  https://ddddddd 显示的颜色就不一致，并且还可以设置 该片段的点击效果等




# Tween  与Curve
一般的Animation 会在给定的时间内线性的产生0.0到1.0 的值
Tween 可以把这些变成我们想要的类型或者范围，
Curve 是一个抽象类，表示生成值的曲线，Curves已经定义了许多常用的曲线


## 路由管理，
也就是页面跳转

### Navigator
#### 入栈，也就是打开页面
Future push(BuildContext context, Route route)

```java
Navigator.push(
    context,
    new MaterialPageRoute(
      builder: (BuildContext context) {
        return new NewRoute();
      },
    ),
);
```
MaterialPageRoute 继承 PageRoute

#### 出栈，也就是返回
bool pop(BuildContext context, [ result ])
result为页面关闭时返回给上一个页面的数据
例如
```java
 Navigator.pop(context, "我是返回值"),

 // 在上一个界面 点击的地方，
 onPressed: () async {
    var result = await Navigator.push(
      context,
      MaterialPageRoute(
        builder: (context) {
          return TipRoute(
            text: "我是提示xxxx",
          );
        },
      ),
    );
```
注意 匿名函数是 async ，push()方法签名有 await，这样就可以得到result 就是传递过来的值，即 我是返回值
但是如果按返回键的话，result 的值就是null

##  路由命名
1. 在MyApp类的build方法中找到MaterialApp 添加 routes 属性




##  捕获全局异常

```
void main() {
  runZoned(() => runApp(MyApp()), zoneSpecification:
      // ZoneSpecification 可以自定义一些代码行为，比如拦截日志输出行为等
      new ZoneSpecification(print: (Zone self, ZoneDelegate parent, Zone zone, String line) {
    parent.print(zone, "Intercepter : $line");
    collectLog(line);
  }), onError: (Object object, StackTrace stack) {
    //捕获我们Flutter应用中全部错误了
    var detail = makeDetails(object, stack);
    reportErrorAndLog(detail);
  });
}

//收集日志
void collectLog(String line) {
//  print(line);
}

//上报错误和日志逻辑
void reportErrorAndLog(FlutterErrorDetails details) {
  print(details.toString());
}

//构建错误信息
FlutterErrorDetails makeDetails(Object obj, StackTrace stack) {
  print("obj= $obj   ");
}
```

### WillPopScope
返回按钮拦截


---
