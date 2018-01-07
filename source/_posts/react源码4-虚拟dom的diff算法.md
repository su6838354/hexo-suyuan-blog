---
title: react源码4-虚拟dom的diff算法
tags:
  - reactjs
categories:
  - 前端
date: 2017-06-20 21:46:24
---

网上有位同学自己实现了一个dom diff 算法
https://github.com/creeperyang/blog/issues/33

### 什么是dom diff
Web界面由DOM树来构成，当其中某一部分发生变化时，其实就是对应的某个DOM节点发生了变化。
在React中，构建UI界面的思路是由当前状态决定界面。前后两个状态就对应两套界面，然后由React来比较两个界面的区别，这就需要对DOM树进行Diff算法分析。

即给定任意两棵树，找到最少的转换步骤。但是[标准的的Diff算法](http://grfia.dlsi.ua.es/ml/algorithms/references/editsurvey_bille.pdf)复杂度需要O(n^3)，这显然无法满足性能要求。
要达到每次界面都可以整体刷新界面的目的，势必需要对算法进行优化。这看上去非常有难度，然而Facebook工程师却做到了，他们结合Web界面的特点做出了两个简单的假设，使得Diff算法复杂度直接降低到O(n)
	- 两个相同组件产生类似的DOM结构，不同的组件产生不同的DOM结构；
	- 对于同一层次的一组子节点，它们可以通过唯一的id进行区分。
    
### 不同节点类型的比较
为了在树之间进行比较，我们首先要能够比较两个节点，在React中即比较两个虚拟DOM节点，当两个节点不同时，应该如何处理。这分为两种情况：
	- 节点类型不同 
	- 节点类型相同，但是属性不同。本节先看第一种情况。

- 当在树中的同一位置前后输出了`不同类型的节点`，React直接`删除前面的节点`，然后`创建并插入新的节点`。假设我们在树的同一位置前后两次输出不同类型的节点。

```
renderA: <div />
renderB: <span />
=> [removeNode <div />], [insertNode <span />]
```

当一个节点从div变成span时，简单的直接删除div节点，并插入一个新的span节点。这符合我们对真实DOM操作的理解。

需要注意的是，删除节点意味着彻底销毁该节点，而不是再后续的比较中再去看是否有另外一个节点等同于该删除的节点。如果该删除的节点之下有子节点，那么`这些子节点也会被完全删除`，它们也不会用于后面的比较。这也是算法复杂能够降低到O（n）的原因。

上面提到的是对虚拟DOM(原始tag)节点的操作，而同样的逻辑也被用在虚拟DOM(React组件)的比较，例如：

```
renderA: <Header />
renderB: <Content />
=> [removeNode <Header />], [insertNode <Content />]
```

当React在同一个位置遇到不同的组件时，也是简单的销毁第一个组件，而把新创建的组件加上去。这正是应用了第一个假设，不同的组件一般会产生不一样的DOM结构，`与其浪费时间去比较它们基本上不会等价的DOM结构`，还不如完全创建一个新的组件加上去。

由这一React对不同类型的节点的处理逻辑我们很容易得到推论，那就是React的DOM Diff算法实际上只会对树进行逐层比较，如下所述。

### 逐层进行节点比较
提到树，相信大多数同学立刻想到的是二叉树，遍历，最短路径等复杂的数据结构算法。而在React中，树的算法其实非常简单，那就是两棵树只会对`同一层次的节点进行比较`。如下图所示：
![](https://res.infoq.com/articles/react-dom-diff/zh/resources/0909000.png)

React只会对相同颜色方框内的DOM节点进行比较，即同一个父节点下的所有子节点。当发现节点已经不存在，则该节点及其子节点会被完全删除掉，不会用于进一步的比较。这样只需要对树进行一次遍历，便能完成整个DOM树的比较。

例如，考虑有下面的DOM结构转换：

![](https://res.infoq.com/articles/react-dom-diff/zh/resources/0909001.png)

A节点被整个移动到D节点下，直观的考虑DOM Diff操作应该是

```
A.parent.remove(A); 
D.append(A);
```

但因为React只会简单的考虑同层节点的位置变换，对于不同层的节点，只有简单的创建和删除。当根节点发现子节点中A不见了，就会直接销毁A；而当D发现自己多了一个子节点A，则会创建一个新的A作为子节点。因此对于这种结构的转变的实际操作是：
```
A.destroy();
A = new A();
A.append(new B());
A.append(new C());
D.append(A);
```
可以看到，以A为根节点的树被整个重新创建。

虽然看上去这样的算法有些“简陋”，但是其基于的是第一个假设：两个不同组件一般产生不一样的DOM结构。根据[React官方博客](http://facebook.github.io/react/docs/reconciliation.html)，这一假设至今为止没有导致严重的性能问题。这当然也给我们一个提示，在实现自己的组件时，保持稳定的DOM结构会有助于性能的提升。例如，我们有时可以通过CSS隐藏或显示某些节点，而不是真的移除或添加DOM节点。

### 相同类型节点的比较
第二种节点的比较是相同类型的节点，算法就相对简单而容易理解。React会对属性进行重设从而实现节点的转换。例如：
```
renderA: <div id="before" />
renderB: <div id="after" />
=> [replaceAttribute id "after"]
```
虚拟DOM的style属性稍有不同，其值并不是一个简单字符串而必须为一个对象，因此转换过程如下：
```
renderA: <div style={{color: 'red'}} />
renderB: <div style={{fontWeight: 'bold'}} />
=> [removeStyle color], [addStyle font-weight 'bold']
```


### 由DOM Diff算法理解组件的生命周期
React组件的生命周期，其中的每个阶段其实都是和DOM Diff算法息息相关的。例如以下几个方法：

- constructor: 构造函数，组件被创建时执行；
- componentDidMount: 当组件添加到DOM树之后执行；
- componentWillUnmount: 当组件从DOM树中移除之后执行，在React中可以认为组件被销毁；
- componentDidUpdate: 当组件更新时执行。

为了演示组件生命周期和DOM Diff算法的关系，原笔者创建了一个示例：https://supnate.github.io/react-dom-diff/index.html ，大家可以直接访问试用。这时当DOM树进行如下转变时，即从“shape1”转变到“shape2”时。我们来观察这几个方法的执行情况：
![](https://res.infoq.com/articles/react-dom-diff/zh/resources/0909002.png)

浏览器开发工具控制台输出如下结果：
```
C will unmount.
C is created.
B is updated.
A is updated.
C did mount.
D is updated.
R is updated.
```
可以看到，C节点是完全重建后再添加到D节点之下，而不是将其“移动”过去。

### 列表节点的比较
上面介绍了对于不在同一层的节点的比较，即使它们完全一样，也会销毁并重新创建。那么当它们在同一层时，又是如何处理的呢？这就涉及到列表节点的Diff算法。相信很多使用React的同学大多遇到过这样的警告：

![](https://res.infoq.com/articles/react-dom-diff/zh/resources/0909003.png)

这是React在遇到列表时却又找不到key时提示的警告。虽然无视这条警告大部分界面也会正确工作，但这通常意味着潜在的性能问题。因为React觉得自己可能无法高效的去更新这个列表。

列表节点的操作通常包括添加、删除和排序。例如下图，我们需要往B和C直接插入节点F，在jQuery中我们可能会直接使用$(B).after(F)来实现。而在React中，我们只会告诉React新的界面应该是A-B-F-C-D-E，由Diff算法完成更新界面。

![](https://res.infoq.com/articles/react-dom-diff/zh/resources/0909004.png)

这时如果每个节点都没有唯一的标识，React无法识别每一个节点，那么更新过程会很低效，即，将C更新成F，D更新成C，E更新成D，最后再插入一个E节点。效果如下图所示：

![](https://res.infoq.com/articles/react-dom-diff/zh/resources/0909005.png)

可以看到，React会逐个对节点进行更新，转换到目标节点。而最后插入新的节点E，涉及到的DOM操作非常多。而如果给每个节点唯一的标识（key），那么React能够找到正确的位置去插入新的节点，入下图所示：

![](https://res.infoq.com/articles/react-dom-diff/zh/resources/0909006.png)

对于列表节点顺序的调整其实也类似于插入或删除，下面结合示例代码我们看下其转换的过程。仍然使用前面提到的示例：https://supnate.github.io/react-dom-diff/index.html ，我们将树的形态从shape5转换到shape6：

![](https://res.infoq.com/articles/react-dom-diff/zh/resources/0909007.png)

即将同一层的节点位置进行调整。如果未提供key，那么React认为B和C之后的对应位置组件类型不同，因此完全删除后重建，控制台输出如下：
```
B will unmount.
C will unmount.
C is created.
B is created.
C did mount.
B did mount.
A is updated.
R is updated.
```
而如果提供了key，如下面的代码：
```
shape5: function() {
  return (
    <Root>
      <A>
        <B key="B" />
        <C key="C" />
      </A>
    </Root>
  );
},

shape6: function() {
  return (
    <Root>
      <A>
        <C key="C" />
        <B key="B" />
      </A>
    </Root>
  );
},
```
那么控制台输出如下：
```
C is updated.
B is updated.
A is updated.
R is updated.
```
可以看到，对于列表节点提供唯一的key属性可以帮助React定位到正确的节点进行比较，从而大幅减少DOM操作次数，提高了性能。

 
### 小结
本文分析了React的DOM Diff算法究竟是如何工作的，其复杂度控制在了O（n），这让我们考虑UI时可以完全基于状态来每次render整个界面而无需担心性能问题，简化了UI开发的复杂度。而算法优化的基础是文章开头提到的两个假设，以及React的UI基于组件这样的一个机制。理解虚拟DOM Diff算法不仅能够帮助我们理解组件的生命周期，而且也对我们实现自定义组件时如何进一步优化性能具有指导意义。

本文来自http://www.infoq.com/cn/articles/react-dom-diff/



