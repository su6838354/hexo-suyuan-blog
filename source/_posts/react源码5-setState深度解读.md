---
title: react源码5-setState深度解读
tags:
  - reactjs
categories:
  - 前端
date: 2017-07-06 00:45:15
---

React虽然只是MVVM中的View部分，但还是实现了View和model的绑定。修改数据的同时，可以实现View的刷新。
这大大简化了我们的逻辑，只用关心数据流的变化。这个特性则要归功于`setState()`方法。
React中利用`队列机制`来管理state，`避免了很多重复`的View刷新。下面我们来从源码角度探寻下setState机制。

### 时序流程
为了研究setState导致的页面更新更简单，我们通过定时器来触发setState，避免onClick 等事件机制等引入带来过多困扰。
```
    componentDidMount() {
        window.setTimeout(()=>{
            console.log('did mount set state')
            this.setState({count: 100})
        }, 1000)
    }
```
这段代码显然会引起页面更新，当然页面有用到count变量。我们先从全局看下整个流程。
![](/images/react/state_10.png)

可以看出setState主要做了以下几件事情：
- 更新队列中添加变更内容
- 执行一堆事务，事务中会做实际dom diff，页面更新渲染等内容

### setState
看下面例子，都知道count加了1次。
```
    add = () => {
        this.setState({count: this.state.count+1})
        this.setState({count: this.state.count+1})
        console.log('set state', this.state)
    }
```

我们先来跟着setState 进去看看
```
ReactComponent.prototype.setState = function (partialState, callback) {
  this.updater.enqueueSetState(this, partialState);// *********1
  if (callback) {
    this.updater.enqueueCallback(this, callback, 'setState');
  }
};

//*********1
enqueueSetState: function (publicInstance, partialState) {
    var internalInstance = getInternalInstanceReadyForUpdate(publicInstance, 'setState');
    if (!internalInstance) {
      return;
    }
  
  	// 如果_pendingStateQueue为空,则创建它。可以发现队列是数组形式实现的
    var queue = internalInstance._pendingStateQueue || (internalInstance._pendingStateQueue = []);
    queue.push(partialState);
    
	// 将要更新的ReactComponent放入数组中
    enqueueUpdate(internalInstance);//*************2
  },
  
  //**************2
  function enqueueUpdate(component) {
  ensureInjected();

  // 如果不是正处于创建或更新组件阶段,则处理update事务
  //**************3
  if (!batchingStrategy.isBatchingUpdates) {
    batchingStrategy.batchedUpdates(enqueueUpdate, component);
    return;
  }

  // 如果正在创建或更新组件,则暂且先不处理update,只是将组件放在dirtyComponents数组中
  dirtyComponents.push(component);
}
```
setState做的主要的事件就是提交了一个更新到队列，没有实际的数据或者页面更新；就行 git add一样，没有做真正的git commit。
如下图可以看出确实往队列中插入了两条数据cout:3，因为我们setState了两次
![](/images/react/state_1.jpg)

enqueueUpdate包含了React避免重复render的逻辑。
mountComponent和updateComponent方法在执行的最开始，会调用到batchedUpdates进行批处理更新，此时会将isBatchingUpdates设置为true，也就是将状态标记为现在正处于更新阶段了。
之后React以事务的方式处理组件update，事务处理完后会调用wrapper.close(), 而TRANSACTION_WRAPPERS中包含了RESET_BATCHED_UPDATES这个wrapper，故最终会调用RESET_BATCHED_UPDATES.close(), 它最终会将isBatchingUpdates设置为false。
 
 现在来轻松下，走段进入****3的case
 ```
   componentDidMount() {
       window.setTimeout(()=>{
       console.log('did mount set state')
       this.setState({count: 100})
       }, 1000)
   }
 ```
 ![](/images/react/state_2.jpg)
 突然发现我们进入了react真正有意义的地方，这一次意外的setState让react直接去做批量更新了。
 
 真的勇士敢于直面真正挑战，那就让我们继续走下去吧。
 
 ```
 batchedUpdates: function (callback, a, b, c, d, e) {
    var alreadyBatchingUpdates = ReactDefaultBatchingStrategy.isBatchingUpdates;

    ReactDefaultBatchingStrategy.isBatchingUpdates = true;

    // The code is written this way to avoid extra allocations
    if (alreadyBatchingUpdates) {
      return callback(a, b, c, d, e);
    } else {
    // *******4 目前看来玄机应该就在这里了
      return transaction.perform(callback, null, a, b, c, d, e);
    }
  }
 ```
 
 ### transaction
 上一节最后的transaction 到底是什么？先看下上下文代码
 ```
 // ********6  重置批量更新
 var RESET_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: function () {
    ReactDefaultBatchingStrategy.isBatchingUpdates = false;
  }
};
// ********7  激发批量更新
var FLUSH_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: ReactUpdates.flushBatchedUpdates.bind(ReactUpdates)
};

var TRANSACTION_WRAPPERS = [FLUSH_BATCHED_UPDATES, RESET_BATCHED_UPDATES];

function ReactDefaultBatchingStrategyTransaction() {
  this.reinitializeTransaction();
}

_assign(ReactDefaultBatchingStrategyTransaction.prototype, Transaction, {
  getTransactionWrappers: function () {
    return TRANSACTION_WRAPPERS;
  }
});
// *******5 这个就是给*******4 用的transaction
var transaction = new ReactDefaultBatchingStrategyTransaction();
 ```
 可以猜测出6，7两个应该是伴随着4 要去做的事情，是不是像redux中间件那样呢？
 > 感悟：猜测是科学发展的必要条件，大量的伟大发现／发明都是先猜测出来的，什么物理学／数学／量子力学的猜想，多年后才验证其正确性。所以我们看代码，写代码也需要伴随大量猜测
 
 ![](/images/react/state_3.jpg)
 可以看出事务transation 先执行所有的initialize，然后执行真正的事情 enqueueUpdate，最后再执行所有的close；
 
 看来没有想象的那么复杂，事务通过wrapper进行封装。一个wrapper包含一对initialize和close方法。比如RESET_BATCHED_UPDATES
 ```
 var RESET_BATCHED_UPDATES = {
  // 初始化调用
  initialize: emptyFunction,
  // 事务执行完成，close时调用
  close: function () {
    ReactDefaultBatchingStrategy.isBatchingUpdates = false;
  }
};
```
transcation被包装在wrapper中，比如
```
var TRANSACTION_WRAPPERS = [FLUSH_BATCHED_UPDATES, RESET_BATCHED_UPDATES];
```
transaction是通过transaction.perform(callback, args…)方法进入的，它会先调用注册好的wrapper中的initialize方法，然后执行perform方法中的callback，最后再执行close方法。
 而且transation 代码中用了一幅图说明了事务的执行时序。
 
 ```
 * <pre>
 *                       wrappers (injected at creation time)
 *                                      +        +
 *                                      |        |
 *                    +-----------------|--------|--------------+
 *                    |                 v        |              |
 *                    |      +---------------+   |              |
 *                    |   +--|    wrapper1   |---|----+         |
 *                    |   |  +---------------+   v    |         |
 *                    |   |          +-------------+  |         |
 *                    |   |     +----|   wrapper2  |--------+   |
 *                    |   |     |    +-------------+  |     |   |
 *                    |   |     |                     |     |   |
 *                    |   v     v                     v     v   | wrapper
 *                    | +---+ +---+   +---------+   +---+ +---+ | invariants
 * perform(anyMethod) | |   | |   |   |         |   |   | |   | | maintained
 * +----------------->|-|---|-|---|-->|anyMethod|---|---|-|---|-|-------->
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | +---+ +---+   +---------+   +---+ +---+ |
 *                    |  initialize                    close    |
 *                    +-----------------------------------------+
 * </pre>
 *
 ```
 我们继续沿着执行代码分析
- initializaAll中没有实际操作
- ret = method.call(scope, a, b, c, d, e, f);
这行代码又去执行enqueueUpdate了，看来setState 是不执行下面的代码不死心
```
  dirtyComponents.push(component);
  if (component._updateBatchNumber == null) {
    component._updateBatchNumber = updateBatchNumber + 1;
  }
```
这段代码应该就是标记下这个组件数据变化了，需要更新。
- this.closeAll(0);
这个里面有个flush_batched_updates,可以发现里面又是一个事务。

我们直接来看setState 中的事务执行流程
![](/images/react/state_11.png)

可以发现还是挺绕的，不过真正的更新最后还是走到了runBatchedUpdates

### runBatchedUpdates

FLUSH_BATCHED_UPDATES会在一个transaction的close阶段运行`runBatchedUpdates`，从而执行update。

runBatchedUpdates循环遍历dirtyComponents数组，主要干两件事
-	首先执行performUpdateIfNecessary来刷新组件的view
-	执行之前阻塞的callback。

下面来看performUpdateIfNecessary。

```
  performUpdateIfNecessary: function (transaction) {
    if (this._pendingElement != null) {
      // receiveComponent会最终调用到updateComponent，从而刷新View
      ReactReconciler.receiveComponent(this, this._pendingElement, transaction, this._context);
    }

    if (this._pendingStateQueue !== null || this._pendingForceUpdate) {
      // 执行updateComponent，从而刷新View。这个流程在React生命周期中讲解过
      this.updateComponent(transaction, this._currentElement, this._currentElement, this._context, this._context);
    }
  },
```
是不是觉得越来越看到庐山真面目了

updateComponent就是终极boss，他会执行React组件存在期的生命周期方法，如componentWillReceiveProps， shouldComponentUpdate， componentWillUpdate，render, componentDidUpdate。 从而完成组件更新的整套流程。
```
 _performComponentUpdate: function (nextElement, nextProps, nextState, nextContext, transaction, unmaskedContext) {
    var _this2 = this;

    var inst = this._instance;

    var hasComponentDidUpdate = Boolean(inst.componentDidUpdate);
    var prevProps;
    var prevState;
    var prevContext;
    if (hasComponentDidUpdate) {
      prevProps = inst.props;
      prevState = inst.state;
      prevContext = inst.context;
    }

    if (inst.componentWillUpdate) {
      if (process.env.NODE_ENV !== 'production') {
        measureLifeCyclePerf(function () {
          return inst.componentWillUpdate(nextProps, nextState, nextContext);
        }, this._debugID, 'componentWillUpdate');
      } else {
        inst.componentWillUpdate(nextProps, nextState, nextContext);
      }
    }

    this._currentElement = nextElement;
    this._context = unmaskedContext;
    inst.props = nextProps;
    inst.state = nextState;
    inst.context = nextContext;

    this._updateRenderedComponent(transaction, unmaskedContext);

    if (hasComponentDidUpdate) {
      if (process.env.NODE_ENV !== 'production') {
        transaction.getReactMountReady().enqueue(function () {
          measureLifeCyclePerf(inst.componentDidUpdate.bind(inst, prevProps, prevState, prevContext), _this2._debugID, 'componentDidUpdate');
        });
      } else {
        transaction.getReactMountReady().enqueue(inst.componentDidUpdate.bind(inst, prevProps, prevState, prevContext), inst);
      }
    }
  },
```

### 总结

setState流程还是很复杂的，设计也很精巧，避免了重复无谓的刷新组件。它的主要流程如下

- enqueueSetState将state放入队列中，并调用enqueueUpdate处理要更新的Component
- 如果组件当前正处于update事务中，则先将Component存入dirtyComponent中。否则调用batchedUpdates处理。
- batchedUpdates发起一次transaction.perform()事务
- 开始执行事务初始化，运行，结束三个阶段
	- 初始化：事务初始化阶段没有注册方法，故无方法要执行
	- 运行：执行setSate时传入的callback方法，一般不会传callback参数
	- 结束：更新isBatchingUpdates为false，并执行FLUSH_BATCHED_UPDATES这个wrapper中的close方法
- FLUSH_BATCHED_UPDATES在close阶段，会循环遍历所有的dirtyComponents，调用updateComponent刷新组件，并执行它的pendingCallbacks, 也就是setState中设置的callback。
 
 
 
 
 
 
 














