---
title: javascript原型深入解析1-prototype 和原型链、js面向对象
tags:
  - js
categories:
  - 前端
date: 2017-04-08 23:42:24
---

### 用prototype 封装类
创建的每个函数都有一个prototype(原型属性)，他是个指针，指向的对象，这个对象的用途就是包含了这个类型所有实例共享的属性和方法。

回味这句，想想java或者C++吧，如果func是class 类，`类的类属性和类方法都放在了prototype`中了
```
var func = function (argument) {
　　// body...
};
 ```
 ![](https://images2015.cnblogs.com/blog/564050/201703/564050-20170321004553096-1350416176.png)

Ok，现在就用c++面向对象的思想建立个Person类，里面有一些类成员
```
function Person(){

 }

 Person.prototype.say = function(first_argument) {
   // body...
   console.info('i say...')

 };

 Person.prototype.age = 10;
 var sy = new Person(); // new 会创建个对象，不加new 就是执行这个函数


 console.info(sy.age)
 sy.say()
```
成功实现了面向对象，sy 可以say，还有了age，当然age不应该是类属性，

Say和age 是所有 Person 所有实例共享的，假如age是数组，sy.age[1] = 10 会导致所有实例的age都变化

### 原理：定义Person function，实例化一个Person，中间有啥奥秘
定义：
```
function Person(){

}
```
- 只要创建个新函数，js引擎会根据一组特定的规则为该函数创建一个原型对象， prototype属性指向这个原型对象；
- 默认情况下，所有原型对象都会自动获取一个constructor(构造函数)属性，这个constructor指向了prototype属性所在的函数；

实例化：
new的过程拆分成以下三步：
- var p={}; 也就是说，初始化一个对象p
- p.__proto__ = Person.prototype;
- Person.call(p); 也就是说构造p，也可以称之为初始化p，过程中关键字 this 被设定为该实例。
- 返回实例

```
function New (f) {

    var n = { '__proto__': f.prototype }; /*第1,2步*/

    return function () {

        f.apply(n, arguments);            /*第3步*/

        return n;                         /*第4步*/

    };

}

var p2 = New (Point)(10, 20);

p2.print(); // 10 20

console.log(p2 instanceof Point); // true
```
sy = new Person()，创建一个新对象sy，sy实例的内部将包含一个Prototype类型的指针__proto__，指向Person函数的原型对象，实例sy和Person没直接关系，和Person.prototype才有直接关系

这里我们可以发现，prototype太重要了
![](https://images2015.cnblogs.com/blog/564050/201703/564050-20170321004619924-872439005.png)
![](https://images2015.cnblogs.com/blog/564050/201703/564050-20170321004612705-1756541328.png)

sy.say() 看来是通过查询__proto__指向的原型来调用方法的；访问对象属性时，js会搜索改该属性，从实例sy开始，沿着原型搜索；

需要注意的是sy.age = 100, 不会修改原型的值，而是给sy添加了个属性age，以后就屏蔽了原型的age，这点和python类似。当然你可以 delete sy.age，就可以重新访问到原型属性了。当然原型的属性age不属于实例，实例也delete不掉。

有的浏览器无法访问__proto__，可以通过Person.prototype.isPrototypeOf(sy)  等于true，或者Object.getPrototypeOf(sy) == Person.prototype 判断原型和实例之间的关系

### 类：现在我们来实现面向对象的封装功能

```
function Person(name, age){
  this.name = name; // 调用new的时候，创建个新的this指向当前正在构造的新实例，其他时候的this都是指向调用者s
  this.age = age;
}

Person.prototype.say = function(first_argument) {
  console.info(this.name + ' say...')
};

var sy = new Person('suyuan', 26);
sy.say()

var zl = new Person('zhuli', 27)
zl.say()
```
一句话，方法用原型链，类属性(数据)用构造函数

### 既然面向对象编程了，那么继承呢？

Js中的函数没有声明，只有定义，所有不支持接口继承，那我们就用原型链来实现定义（实现）继承
```
function Person(){
  this.live = true;
}

Person.prototype.say = function(first_argument) {// (4) Person的原型添加函数say
  console.info('i am ' + this.live)

};



function Man(){
  this.manLive = true;
}

var tmp_person = new Person();//(3) 实例tmp_person 的__proto__ 指向Person原型
Man.prototype = tmp_person //(2) temp_person 这个对象有say方法，Man的原型（也就是temp_person）有say方法，Man实例化出来的对象sy当然有say

var sy = new Man();//(1)
sy.say()//(5)
```
sy是Man实例，

- sy的__proto__指向Man的原型，
- Man原型指向Person的实例tmp_person，
- tmp_person的__proto__指向Person原型,
- Person原型有函数say
- 然后sy 就有了say方法。

在此继承就实现了。




来自我的博客园(不更新)
http://www.cnblogs.com/suyuan1573/p/prototype.html