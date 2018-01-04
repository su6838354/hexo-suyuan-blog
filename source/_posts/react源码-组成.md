---
title: react源码1--页面是怎么出来的
tags:
    - reactjs
categories:
    - 前端
date: 2017-10-09 20:40:55
---

![](/images/react/react-1.png)

## 创造组件 Component

上帝创造了世界，浏览器创造了 div、h1、a...这些标签。
你可以用他们来创造自定义的组件，当然还有你最外面到那个页面，他是你最顶层到组件。
```
class PicList extends Component {
     render () {
     	return <div>123</div>
     }
}

```
上面到写法也就是调用 `createClass`来实现到创建组件

## 创建虚拟dom
是不是感觉一下到虚拟dom有点快，这里我们还是省略了很多内容。
真实情况是先createElement api创建出元素，元素就是通过一堆标签组件组织起来的，反应父子关系的结构体，可以认为元素就是虚拟dom。元素会递归的根据各个节点的类型生产真实的DOM结构。

有了字符串、原始标签（div）、的组件（PicList） 这些模版；
react 就可以根据他们到类型构建出虚拟dom。


## 创建真实dom
这个也是react做到事情，每个虚拟dom都会调用自己到mountComponent，会返回真实的标签，构建出真实dom


### 参考
[比较与理解React的Components，Elements和Instances](https://github.com/creeperyang/blog/issues/30)
