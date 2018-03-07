---
title: React生命周期中应该做什么事
tags:
  - reactjs
categories:
  - 前端
date: 2016-10-25 01:29:35
---

React生命周期函数
![](https://images2015.cnblogs.com/blog/564050/201703/564050-20170301005923235-1884053087.png)
### 装载组件触发
#### 0.construct(props) 用来 props--->state
初始化state，并且把props转化为state

#### 1.componentWillMount   用来 props--->state，用构造函数就可以了，这个我们一般不用
只会在装载之前调用一次，在render之前调用，你可以在这个方法里面调用setState 改变状态，并且不会导致额外调用一次render，不能有副作用在里面

#### 2.componentDidMount    页面初次加载好后，发送网络请求，获取数据后setState(data) 触发重新渲染 or 操作DOM改变样式
只会在装载完成之后调用一次，在render之后调用，从这里开始可以通过 ReactDOM.findDOMNode(this)获取到组件的DOM节点。

### 更新组件触发
#### 1.componentWillReceiveProps(nextProps)  用来props--->state,setState不会重新渲染
有可能props没有改变的时候也触发，比如父组件更新导致的触发，有时候可能需要比较props是否发生了改变

#### 2.shouldComponentUpdate(nextProps, nextState)  判断新的props或者state是否需要重新渲染
更新前所有的setState已经完成，forceUpdate()会强刷

#### 3.componentWillUpdate(nextProps, nextState)  可以执行一些预备操作在更新前， 基本不用
此时已经不能调用setState了，

#### 4.componentDidUpdate()  页面加载好后，发送网络请求，获取数据后setState(data) 触发重新渲染 or操作DOM改变样式

### 给出一个疑问：

官网说可以再componenDidUpdate 中做一些网络请求，然后setState的事情

componentDidUpdate() is invoked immediately after updating occurs. This method is not called for the initial render.

Use this as an opportunity to operate on the DOM when the component has been updated. This is also a good place to do network requests as long as you compare the current props to previous props (e.g. a network request may not be necessary if the props have not changed).

 