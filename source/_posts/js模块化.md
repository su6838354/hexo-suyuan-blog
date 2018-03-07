---
title: js模块化
tags:
  - js
categories:
  - 前端
date: 2017-03-09 00:08:42
---

### es6
定义了新的模块标准化，提供了modules
```
export xxx;
import xxx from file
 ```

没有采用nodejs的CommonJS，

没有实现require，require和module.exports只是node的私有全局方法和对象属性



建议写法：
```
export {fun as default,a,b,c};
```
 

### nodejs
模块的规范是CommonJS
```
module.exports = data;
var data = require(file)
 ```

commonjs 对应 浏览器，nodejs的概念位置
```
 

  |---------------浏览器----- ------------------|        |--------------------------CommonJS----------------------------------|

  |  BOM  |       | DOM |        | ECMAScript |         | FS |           | TCP |         | Stream |        | Buffer |   |........|

  |-------W3C-----------|       |---------------------------------------Node--------------------------------------------------|

 ```

#### CommonJS 是同步加载模块
```
var math = require('math'); //（同步） math.js 引去进来后，才会执行后面的add，文件都在磁盘上，同步没有任何问题

math.add(2,3); // 5
```
 

#### 浏览器也能用
其为nodejs专有，由于浏览器没有全局变量module、require等，不过可以借助一些工具完成浏览器端的实现，

其原理是现将所有模块都定义好并通过 id 索引，这样就可以方便的在浏览器环境中解析了，

如：Browserify
```
// foo.js
module.exports = function(x) {
  console.log(x);
};
 
// main.js
var foo = require("./foo");
foo("Hi");
```
$ browserify main.js > compiled.js
$ browser-unpack < compiled.js
 ```
[
  {
    "id":1,
    "source":"module.exports = function(x) {\n  console.log(x);\n};",
    "deps":{}
  },
  {
    "id":2,
    "source":"var foo = require(\"./foo\");\nfoo(\"Hi\");",
    "deps":{"./foo":1},
    "entry":true
  }
]
```
执行的时候，浏览器遇到 require('./foo') 语句，就自动执行1号模块的 source 属性，并将执行后的 module.exports 属性值输出。

CommonJS 已经过时，Node.js 的内核开发者已经废弃了该规范。

 
### 现在用了webpack和babel编译
- es6的import 是编译时的，选择import的内容编译进来，性能更好

- commonjs的require是运行时的，运行的时候，会执行require 模块后赋值给某个变量

- 目前不管是nodejs中，还是写前端代码，为了支持es6都是用babel编译，babel其实是把把es6转成es5,import编译成require,

- 所以用babel编译的代码可以混搭 import和require

 

### 浏览器端
#### 为什么commonjs 不适合
CommonJS 是`同步`加载模块
```
var math = require('math'); //（同步） 等待math.js 引去进来后，才会执行后面的add，文件都在磁盘上，同步没有任何问题
math.add(2,3); // 5
 ```

这时候浏览器端不适合的，js是通过网络加载的，因为模块都放在服务器端，等待时间取决于网速的快慢，可能要等很长时间，浏览器处于"假死"状态。


因此，浏览器端的模块，不能采用"同步加载"（synchronous），只能采用"异步加载"（asynchronous）。这就是AMD规范诞生的背景。

AMD是"Asynchronous Module Definition"的缩写

它采用异步方式加载模块，模块的加载不影响它后面语句的运行。所有依赖这个模块的语句，都定义在一个回调函数中，等到加载完成之后，这个回调函数才会运行。

 ```
require(['math'], function (math) {
　　　　math.add(2, 3);// math 加载好了后执行这句
　　});
 ```

math.add()与math模块加载不是同步的，浏览器不会发生假死。

说白了，js的执行没有提高和改变，主要是不想让加载js影响到代码；所以需要我们换种回调类型的写法，加载好了通过回调执行js，没有谁会因为加载而卡住。

 
>目前，主要有两个Javascript库实现了AMD规范：require.js和curl.js。

最早的时候，所有Javascript代码都写在一个文件里面，只要加载这一个文件就够了。后来，代码越来越多，一个文件不够了，必须分成多个文件，依次加载。下面的网页代码，相信很多人都见过。

```
　　<script src="1.js"></script>
　　<script src="2.js"></script>
　　<script src="3.js"></script>
　　<script src="4.js"></script>
　　<script src="5.js"></script>
　　<script src="6.js"></script>
```
这段代码依次加载多个js文件。

这样的写法有很大的缺点。首先，加载的时候，浏览器会停止网页渲染，加载文件越多，网页失去响应的时间就会越长；其次，由于js文件之间存在依赖关系，因此必须严格保证加载顺序（比如上例的1.js要在2.js的前面），依赖性最大的模块一定要放到最后加载，当依赖关系很复杂的时候，代码的编写和维护都会变得困难。

require.js的诞生，就是为了解决这两个问题：

- 实现js文件的异步加载，避免网页失去响应；

- 管理模块之间的依赖性，便于代码的编写和维护。

 

#### 浏览器目前模块化方式
浏览器端有requirejs(AMD)和seajs(CMD)之类的工具包实现模块化,他们写法差不多，执行原理过程有点区别，后面会说

```
// ----------- AMD or CMD ----------------
define(function(require, exports, module){
  module.exports = {
    a : function() {},
    b : 'xxx'
  };
});


// ------------ AMD or CMD -------------
define(function(require, exports, module){
   var m = require('./a');
   m.a();
});
```

 

AMD(异步模块定义)主要为前端js的表现指定规范
```
define(id?: String, dependencies?: String[], factory: Function|Object);
 ```

id 是模块的名字，它是可选的参数。

dependencies 指定了所要依赖的模块列表，它是一个数组，也是可选的参数，每个依赖的模块的输出将作为参数一次传入 factory 中。如果没有指定 dependencies，那么它的默认值是 ["require", "exports", "module"]。

define(function(require, exports, module) {}）

factory 是最后一个参数，它包裹了模块的具体实现，它是一个函数或者对象。如果是函数，那么它的返回值就是模块的输出接口或值。

eg:

定义一个名为 myModule 的模块，它依赖 jQuery 模块：

```
define('myModule', ['jquery'], function($) {
    // $ 是 jquery 模块的输出
    $('body').text('hello world');
});
// 使用
define(['myModule'], function(myModule) {});
```
 

在模块定义内部引用依赖：
```
define(function(require) {
    var $ = require('jquery');
    $('body').text('hello world');
});
```
这里有define，把东西包装起来啦，那require.js 和 Node实现中怎么没看到有define关键字呢，它也要把东西包装起来呀，其实吧，只是他们隐式包装了而已

 

#### SeaJs和RequireJs区别
>SeaJS对模块的态度是懒执行, 而RequireJS对模块的态度是预执行

如下模块通过SeaJS/RequireJS来加载, 执行结果会是怎样?

```
define(function(require, exports, module) {
    console.log('require module: main');

    var mod1 = require('./mod1');
    mod1.hello();
    var mod2 = require('./mod2');
    mod2.hello();

    return {
        hello: function() {
            console.log('hello main');
        }
    };
});
```
 

先试试SeaJS的执行结果，他是需要的时候执行依赖
```
    require module: main

    require module: mod1

    hello mod1

    require module: mod2

    hello mod2

    hello main
```
 

 

再来是RequireJS的执行结果，他是 所有的依赖提前执行好了，
```
    require module: mod1

    require module: mod2

    require module: main

    hello mod1

    hello mod2

    hello main
```
 
 
 参考： 

AMD http://www.cnblogs.com/chenguangliang/p/5856701.html
Commonjs http://www.cnblogs.com/skylar/p/4065455.html

 

 

参考： 

AMD http://www.cnblogs.com/chenguangliang/p/5856701.html

Commonjs http://www.cnblogs.com/skylar/p/4065455.html