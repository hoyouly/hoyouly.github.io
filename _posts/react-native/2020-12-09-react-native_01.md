---
layout: post
title: React Native  - React 基础知识
category: RN 学习
tags: RN
---
<!-- * content -->
<!-- {:toc} -->


## JSX
JSX是 JavaScript 的一个扩展。可以很好的描述UI应该呈现出它应有交互的本质形式。    
JSX的基本语法规则：**遇到HTML标签（以<开头），就用HTML规则解析，遇到代码块（以{）开头，就用JavaScript规则解析。**   

注意
* 在JSX中，可以在{}中放置任何有效的JavaScript表达式，
* 凡是使用到JSX的地方，都要加上 type="text/babel"，例如
```JavaScript
<script type="text/babel">
    // ** Our code goes here! **
  </script>
```

JSX允许直接在模板插入JavaScript变量，如果这个变量是一个数组，则会展开这个数组的所有成员
```JavaScript
var arr = [
  <h1>Hello world!</h1>,
  <h2>React is awesome</h2>,
];
ReactDOM.render(
  <div>{arr}</div>,
  document.getElementById('example')
);
```
上面arr变量就是一个数组，结果JSX会把他的所有成员，添加到模板上。


JSX 也是一个表达式，在编译后，JSX表达式会被转为普通的JavaScript函数调用。并且对其取值后得到JavaScript对象。
也就是说可以在if 语句和for循环的代码中使用JSX，将JSX赋值给变量，把JSX当做参数传入，以及从函数中返回JSX

```JavaScript
function getGreeting(user) {
  if (user) {
    return <h1>Hello, {formatName(user)}!</h1>;
  }
  return <h1>Hello, Stranger.</h1>;
}
```

### JSX 特定的属性
通过引号，将属性指定为字符串字面量
```JavaScript
const element = <div tabIndex="0"></div>;
```
通过 {}，在属性值中插入一个JavaScript表达式

```JavaScript
const element = <img src={user.avatarUrl}/>;
```

注意： 在属性中嵌套JavaScript表达式时候，仅仅使用引号(对于字符串值)或大括号（对于表达式）中的一个，对于同一属性，不能同时使用者两个符号。

建议自定义组件使用大写字母开头。




### JSX中的Props

有很多方式可以在JSX中指定props的
* JavaScript 表达式可以作为props

```JavaScript
<MyComponent foo={1 + 2 + 3 + 4} />
```
在MyComponent中，props.foo的值就等于10，

注意，if语句已经for循环不是JavaScript表达式。所以不能在JSX中直接使用。但是可以在JSX以外的代码中使用。

```JavaScript
function NumberDescriber(props) {
  let description;
  if (props.number % 2 == 0) {
    description = <strong>even</strong>;
  } else {
    description = <i>odd</i>;
  }
  return <div>{props.number} is an {description} number</div>;
}
```

* 字符串字面量
下面两个JSX表达式等价
```JavaScript
<MyComponent message="hello world" />

<MyComponent message={'hello world'} />
```

* props默认值为True
下面两个表达式等价
```JavaScript
<MyTextBox autocomplete />

<MyTextBox autocomplete={true} />
```
但是通常不建议这么做，因为可能与ES6对象简写混淆。

### 属性展开

如果已经有一个pros对象，可以使用展开运算符`...`来在JSX中传递整个props对象，以下两组件是等价的

```JavaScript
function App1() {
  return <Greeting firstName="Ben" lastName="Hector" />;
}

function App2() {
  const props = {firstName: 'Ben', lastName: 'Hector'};
  return <Greeting {...props} />;
}
```

### JSX中的子元素
包含在开始和结束标签直接的JSX表达式内容将作为特定属性。props.children传递给外层组件，有几种不同的方法来传递子元素

* 字符串字面量
```JavaScript
<MyComponent>Hello world!</MyComponent>
```
通过 props.children 得到的就是一个简单的未转义的字符串`Hello world!`

JSX 会移除收尾的空格以及空行，与标签相邻的空行均会被删除。字符串直接的换行也会被压缩为一个空格。

* JSX 子元素
子元素运行有多个JSX元素组成，这对于嵌套组件非常有用，例如下面这种。
```JavaScript
<MyContainer>
  <MyFirstComponent />
  <MySecondComponent />
</MyContainer>
```

React 组件能够返回存储在数组中的一组元素
```JavaScript
render() {
  // 不需要用额外的元素包裹列表元素！
  return [
    // 不要忘记设置 key :)
    <li key="A">First item</li>,
    <li key="B">Second item</li>,
    <li key="C">Third item</li>,
  ];
}
```

* JavaScript 表达式作为子元素
JavaScript表达式可以被包裹在`{}`中作为子元素。例如,以下表达式是等价的。
```JavaScript
<MyComponent>foo</MyComponent>

<MyComponent>{'foo'}</MyComponent>
```

* 函数作为子元素
props.children 和其他 prop 一样，它可以传递任意类型的数据，而不仅仅是 React 已知的可渲染类型。例如，如果你有一个自定义组件，你可以把回调函数作为 props.children 进行传递

```JavaScript
// 调用子元素回调 numTimes 次，来重复生成组件
function Repeat(props) {
  let items = [];
  for (let i = 0; i < props.numTimes; i++) {
    items.push(props.children(i));
  }
  return <div>{items}</div>;
}

function ListOfTenThings() {
  return (
    <Repeat numTimes={10}>
      {(index) => <div key={index}>This is item {index} in the list</div>}
    </Repeat>
  );
}
```

* 布尔类型、Null 以及 Undefined 将会忽略
false,null,undefined,true 都是合法的子元素，但是他们并不会被渲染，而是被忽略,下面JSX表达式渲染结果相同。这有助于一句特定条件来渲染其他的React元素。
```JavaScript
<div />

<div></div>

<div>{false}</div>

<div>{null}</div>

<div>{undefined}</div>

<div>{true}</div>
```
如果想要渲染 false,null,undefined,true等值，需要将他们转换为字符串。
```JavaScript
<div>
  My JavaScript variable is {String(myVariable)}.
</div>
```

## 组件
React 允许将代码封装成组件（Component）,然后向插入普通HTML标签一样，在网页中插入这个组件。

React 元素可以是DOM标签，一个JavaScript函数。，也可以只用户自定义的组件。当 React 元素为用户自定义组件时，它会将 JSX 所接收的属性（attributes）以及子组件（children）转换为单个对象传递给组件，这个对象被称之为 “props”。


```JavaScript
function Welcome(props) {
  return <h1>Hello, {props.name}</h1>;
}
```
这就是一个有效的React组件，因为它接收唯一带数据的props对象，并返回一个React元素，这类组件被称为函数组件，

等效于 ES6的函数组件

```JavaScript
class Welcome extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}
```

* 组件名称必须以大写字母开头，React 会将以小写字母开头的组件视为原生 DOM 标签。例如，<div /> 代表 HTML 的 div 标签
而 `<Welcome />` 则代表一个组件，并且需在作用域内使用 Welcome。
*  所有的组件都必须有自己的render方法，用于输出组件。并且render()中只能return一个标签即只能包含一个顶层标签，否则也会报错
* 组件的用法和原生HTML标签完全一致，可以任意加入属性，`<HelloMessage name="John">`
* 组件的属性可以在组件类的this.props对象上获取。比如name 属性，可以通过this.props.name读取。
```JavaScript
var HelloMessage = React.createClass({
  render: function() {
    return <h1>Hello {this.props.name}</h1>;
  }
});
```

添加组件属性，有一个地方需要注意，就是 class 属性需要写成 className ，for 属性需要写成 htmlFor ，这是因为 class 和 for 是 JavaScript 的保留字。


### 函数组件转换成class组件
1. 创建一个同名的ES6 class ,并且继承于React.Component
2. 添加一个空的render()方法
3. 将函数体移动到render()方法之中
4. 在render()方法中使用this.props替代props
5. 删除剩余的空函数声明

例如 函数组件 Clock,转换成class组件。
```JavaScript
function Clock(props) {
  return (
    <div>
      <h1>Hello, world!</h1>
      <h2>It is {props.date.toLocaleTimeString()}.</h2>
    </div>
  );
}
```

```JavaScript
class Clock extends React.Component {
  render() {
    return (
      <div>
        <h1>Hello, world!</h1>
        <h2>It is {this.props.date.toLocaleTimeString()}.</h2>
      </div>
    );
  }
}
```

建议不要创建自己的组件基类。在React 组件中，代码重用的主要方式是组合而不是继承。

## Props
组件无论是使用函数声明还是通过class神明，都不能修改自身的props，
所有的React组件都必须像纯函数那样保护他们的props不被更改




#### this.props.children

this.props对象的属性与组件的属性一一对应，但是有一个例外，就是this.props.children属性，它表示组件的所有子节点。

值有三种可能，
1. 当前组件没有子节点，就是undefined
2. 有一个子节点： 数据类型就是object
3. 有多个子节点，数据类型就是 array

React 提供一个工具方法 React.Children 来处理 this.props.children 。我们可以用 React.Children.map 来遍历子节点，而不用担心 this.props.children 的数据类型是 undefined 还是 object。


## PropTypes

组件的属性可以接受任意值，字符串，对象，函数等都可以，
如果想验证组件提供的参数是否符合要求，可以通过组件的PropTypes属性
```JavaScript
var MyTitle = React.createClass({
  propTypes: {
    title: React.PropTypes.string.isRequired,
  },

  render: function() {
     return <h1> {this.props.title} </h1>;
   }
});


```
MyTitle 有一个title属性，这个是必须的，并且是字符串，如果设置一个数值，

```JavaScript
var data = 123;

ReactDOM.render(
  <MyTitle title={data} />,
  document.body
);
```
那么控制台就会显示一个错误信息
>Warning: Failed propType: Invalid prop `title` of type `number` supplied to `MyTitle`, expected `string`.

getDefaultProps()方法可以用来设置组件属性的默认值。
## 获取真实的DOM节点

组件并不是真实的DOM节点，而是存在于内存之中的一种数据结构，叫作虚拟DOM，只有当他插入文档后，才能变成真实的DOM

需要用到ref属性。


## this.state
将组件看成是一个状态机，一开始有一个初始状态，然后用户互动，导致状态变化，从而触发重新渲染UI，即调用this.render方法。
this.props和this.state 都用于描述组件的特性，
this.props 表示那些一旦定义，就不再改变的特性
this.state是会随着用户互动 而发生变化的特性。

state 是私有的，完全受控于当前组件。

### setState()

1. 不要直接修改state
构造函数是唯一可以给this.state赋值的地方，其他地方想要修改，需要使用setState()方法
例如
```JavaScript
this.state.comment = 'Hello';// wrong,
//而应该使用
this.setState({comment:"hello"})
```
2. State的更新可能是异步的

出于性能考虑，React可能会把多个setState()合并成一个调用。
因此this.props和this.state可能是异步更新的，所以不要依赖他们的值更新下一个状态

如果非要依赖上一个值的才能更新下一个状态的话，可以让setState()接受一个函数，而不是对象,这个函数用上上一个state作为第一个参数，将此更新被应用是的prop作为第二个参数。


```JavaScript
this.setState({//可能无法更新计数器， wrong
  counter: this.state.counter + this.props.increment,
});

//这种就可以正常更新计数器，
this.setState((state, props) => ({
  counter: state.counter + props.increment
}));
//上面这种使用了箭头函数，普通函数也是可以的

this.setState(function(state, props) {
  return {
    counter: state.counter + props.increment
  };
});
```
3. State的更新会被合并
当调用setState()的时候，React会把提供的对象合并到当前state

如果一个state包含几个独立的变量，可以分别调用setState()来单独更新，


### 数据流

组件可以选择把他的state作为props向下传递到他的子组件中

`<FormattedDate date={this.state.date} />`  FormattedDate 组件会在其props中接收参数date,但是组件本身无法知道他是来自父组件的state，或是父组件的props,亦或是手动输入的

可以把组件够长的树想象成一个props的数据瀑布，那么每一个组件的state就像是在任意一点给瀑布增加额外的水源，但是它只能向下流动。



## 组件的生命周期

分为三个状态
* Mounting: 以插入真实的DOM
* Updateing: 正在被重新渲染
* UnMounting: 已经移除真实DOM

每个状态都有两个处理函数，will函数在进入状态之前调用，did函数在进入状态之后调用。

* componentWillMount()
* componentDidMount()
* componentWillUpdate(object nextProps, object nextState)
* componentDidUpdate(object prevProps, object prevState)
*  componentWillUnmount()

两种特殊状态的函数
* componentWillReceiveProps(object nextProps)：已加载组件收到新的参数时调用
* shouldComponentUpdate(object nextProps, object nextState)：组件判断是否重新渲染时调用

当组件实例被创建并插入DOM中时，其生命周期调用顺序如下
* constructor()
* static getDervideStateFromProps()
* render()
* componentDidMount()

当组件的props或者state发生变化时，会触发更新，组件更新的生命周期调用顺序如下
* static getDervideStateFromProps()
* shouldComponentUpdate()
* render()
* getSnapShotBeforeUpdate()
* componentDidMount()


当组件从DOM中移除时会调用如下方法
* componentWillUnmount()

当渲染过程，生命周期，或子组件的构造函数中抛出错误时，会调用如下方法

* static getDervideStateFromError()
* commentDidCatch()


### 其他API

* setState()
* forceUpdate()

class 属性
* DefaultProps
* displayName

实例属性
* props
* state


`style={{opacity: this.state.opacity}}`
React 组件样式

第一重大括号表示这是JavaScript语法，
第二重大括号表示样式对象。



组件的数据来源，可以使用 componentDidMount()设置Ajax请求，等到请求成功，再用this.setState()重新渲染UI

## 事件处理

事件的命名采用小驼峰式，而不是纯小写，和java方法命名一样

在React中不能通过返回false 的方式阻止默认行为，必须显示的使用preventDefault

```JavaScript
function ActionLink() {
  function handleClick(e) {
    e.preventDefault();
    console.log('The link was clicked.');
  }

  return (
    <a href="#" onClick={handleClick}>
      Click me
    </a>
  );
}
```


在JavaScript中，class的方法默认不会绑定this,

通常情况下，如果你没有在方法后面添加 ()，例如 onClick={this.handleClick}，你应该为这个方法绑定 this。



---
搬运地址：    

[React Native 中文网](https://reactnative.cn/)

[react-native 系列](https://blog.csdn.net/zeping891103/category_8583110.html)

[开发 React Native APP —— 从改造官方Demo开始（一）](https://juejin.cn/post/6844903567845769223)

[React 中文网](https://zh-hans.reactjs.org/)
