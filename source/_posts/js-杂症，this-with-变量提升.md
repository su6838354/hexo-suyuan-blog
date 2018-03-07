---
title: js 杂症，this with 变量提升
tags:
  - js
categories:
  - 前端
date: 2016-10-17 23:35:24
---

### this.xx 和 xx 是两回事

受后端语言影响，总把this.xx 和xx 当中一回事，认为在function中，xx 就是this.xx，其实完全两回事；

this.xx 是沿着this 原型链找变量，xx是沿着作用域链找变量

```
var func = function(){
console.info(this);
}

var func1 = function(){
function func11(){
console.info(this) // 应该是func1
this.func(); // this 是window
func(); // windows 输出    
}
console.info(this)
func11(); // 全局调用，func11 里面的this是全局对象window, 见下面的2
//this.func11(); //失败，this是window，没有func11方法
}

func1()
```
 

元原型链：this.xxx 的时候会沿着原型链查找，继承可以通过原型链实现

作用域链：xxx 的时候会沿着作用域链查找，with 会设置作用域链最底层


### this代表啥
this对象根据不同的调用方式，所绑定的对象也是不同的。 
函数调用有四种：

1. 方法模式的调用：当一个函数被保存为一个对象的属性时，我们称这个函数为一个方法。当一个方法被调用时，this绑定到该对象。
```
var p = {func: func1}
p.func()// func1中的this就p
```
2. 函数模式的调用：当一个函数并非一个对象的属性时，那么它就被当作一个函数来调用，this被绑定到全局对象。

3. 构造器模式的调用：如果一个函数前面带上new来调用，那么将创建一个隐藏连接到该函数的prototype成员的新对象，同时this被绑定到这个新对象上。
```
function person(){
console.info(this);
this.age = 10;
}
var a = new person()
```
 

4. apply模式的调用：apply方法接收两个参数，第一个被绑定到this，第二个是参数数组。什么也不传时，默认this绑定到全局对象。

```
var person = function(){
this.age = 10;
this.say = function(){console.info(this.age)}
}

var zl = function(){
person.call(this)
console.info(this)
}

var a = new zl()
```
 

// 下面这个其实是错误的继承，涉及到变量提升，

```
var person；
var zl;
person = function(){
  this.age = 10;
  this.say = function(){
  	console.info(this.age)
	}
}
zl = function(){
  console.info(zl);//其实是给zl 加了属性、方法
  person.call(zl);
}
// 执行
var a = new zl() // 执行时候，zl， person 都已经定义出来了，在执行zl() 的时候，会沿着作用域链找到zl
```
 

三、变量提升

```
say()
console.info(a);


function say(){
console.info(111)
}

var a = 100;


-----------js解析后为，然后其实是执行下面的代码---------
function say(){
console.info(111)
}
var a;

say()
console.info(a)
a = 100;
```
 

### 一个奇怪的事情，with，变量提升，this

```
({
x: 10,
foo: function () {
function bar() {
console.log(x);
console.log(y);
console.log(this.x);
}
with (this) {
var x = 20;
var y = 30;
bar.call(this);
}
}
}).foo();
```
 


----js引擎解析翻译后----

```
({
x: 10,
foo: function () {
var x;
var y;
function bar() {
console.log(x); // 沿着作用域找到了foo 下面的x=undefined
console.log(y); // 沿着作用域链找到 foo 下面的 y=30
console.log(this.x);// this 是最外面的对象，x 已经改为20
}
with (this) {
x = 20; //this是最外面的对象,有x，其实是赋值给了this.x, 10变成20
y = 30; // this 没有y，沿着作用域链找到 foo 下面的y， undefine 变成30

bar.call(this);
}
}
}).foo();
```
 题目来自  http://luopq.com/2016/02/14/js-with-keyword/ 

 

输出:

undefine

30

20