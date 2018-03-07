---
title: javascript原型深入解析2--Object和Function，先有鸡先有蛋
tags:
  - js
categories:
  - 前端
date: 2017-04-10 23:53:34
---

### 提出两个问题：
Js 的prototype和`__proto__ `是咋回事？
先有function 还是先有object？
 

### 引用《JavaScript权威指南》的一段描述：
每个JS对象一定对应一个原型对象(`__proto__`指向的)，并从原型对象继承属性和方法。

```
function Person(){
  this.live = true;
}

var sy = new Person();
console.info(sy.__proto__ === Person.prototype) // true

Cat = {};
console.info(Cat.__proto__ === Object.prototype) // true

Dog = new Object();
console.info(Dog.__proto__ === Object.prototype) // true
```

又来一个问题：

Object 是js中的最原始的对象吗？Object.prototype是什么？

### 先谈prototype
只有函数才有prototype属性。

JS不像其它面向对象的语言，它没有类（class，ES6引进了这个关键字，但更多是语法糖）的概念。JS通过函数来模拟类。

![](https://images2015.cnblogs.com/blog/564050/201703/564050-20170321005008408-414149843.png)

当你创建（定义）函数Person时，JS会为这个函数自动添加prototype属性，其是Object类型对象，并且包含constructor和`__proto__`。

而一旦你把这个函数当作构造函数（constructor）调用（即通过new关键字调用），那么JS就会帮你创建该构造函数的实例sy，实例继承构造函数prototype的所有属性和方法（实例通过设置自己的`__proto__`指向承构造函数的prototype来实现这种继承）。

 

对象的`__proto__`指向自己构造函数的prototype。`sy.__proto__.__proto__...`的原型链由此产生，包括我们的操作符instanceof正是通过探测`obj.__proto__.__proto__... === Constructor.prototype`来验证sy是否是Person或者Object的实例。

 

### 函数和对象之间的关系（重点）
创建个函数有两种方式：

(1)
```
var foo = new Function ([arg1[, arg2[, ...argN]],] functionBody)
 ```

(2)
```
function foo() {} // var foo = function(){}
 ```

深入分析函数：

函数也是对象，是一个Function实例（Function就像是python中的元类的角色），我们称函数是特殊的对象（只有函数有prototype）。

因此每个函数对象都继承Function的原型对象，`Person.__proto__ `指向Function.prototype。下面两个黑框是同一个内存对象

![](https://images2015.cnblogs.com/blog/564050/201703/564050-20170321005156205-2060408384.png)
![](https://images2015.cnblogs.com/blog/564050/201703/564050-20170321005201815-1791381657.png)

给出两个判断：
- Object instanceof Function // true
- Function instanceof Object // true


Function是函数的起源，那Function本身是什么呢？

Object又是什么呢，先有Object还是先有Function
 

乱了。。。。。

上关系图来分析吧。

![](https://images2015.cnblogs.com/blog/564050/201703/564050-20170321010028877-885116474.png)

再来看个js开天劈地的伪代码
```
var ObjectPrototype = create( );   // 开天辟地

var FunctionPrototype = create( ObjectPrototype );  
//FunctionPrototype（后被赋值给了Function.prototype）是Object类型的
//因为其原型是ObjectPrototype

var Function = create( FunctionPrototype );

Function.prototype = FunctionPrototype;
// Function是Function类型的，也是Object类型的
//言外之意，Function对象 原型链上有Function.prototype和Object.prototype

 
Object  =  create( FunctionPrototype );

Object.prototype = ObjectPrototype;
//Object是Function类型的，也是Object类型的
//言外之意Object对象的原型链上有Function.prototype和Object.prototype
```

- 先有了Object.prototype，
- 然后有Function.prototype，
- 然后有了Function，
- 然后创建了Object，
- Object的原型又指向了Object.prototype，构成了循环指向；

从而出现了鸡和蛋的问题。

所以严格来说先有Function再有Object。

但是又先有的Object原型，再有的Function原型。

`Object.prototype.__proto__ === null`，说明原型链到Object.prototype终止

 

- Person是Function创建出来的函数，Person.prototype是个Object对象，`Person.proyotype.__proto__`指向Object.prototype
- Function是js自带的对象，可以创建函数的函数，所以Function.prototype指向的函数根源，所有创建的函数的`__proto__`都指向Function.prototype。给Object原型添加方法，Function也会有该方法，因为Function.prototype 指向Object.prototype。
- Object是js自带的对象，算基类； Object本身又是通过Function定义出来的，所以会受到Function影响。
- Function只管函数，Object什么都管。


![](https://images2015.cnblogs.com/blog/564050/201703/564050-20170321005525877-900948490.png)

可以发现黑框的Object不是普通的Object，而是原始天尊


来自我的博客（不更新）
http://www.cnblogs.com/suyuan1573/p/proto.html












