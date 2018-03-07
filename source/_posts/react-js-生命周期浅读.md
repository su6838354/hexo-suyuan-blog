---
title: react js 生命周期浅读
tags:
    - reactjs
categories:
    - 前端
date: 2016-04-09 00:32:32
---

react  redux 结合是目前比较流行的前端开发框架，主要基于react 中的state 树为数据模型，借助redux 来控制 state 数据；下面直接从代码层面解析该框架中一个react 组件 成员函数的生命周期；

先给出一个redux 的数据流 模型图， 只关心react 生命周期的可以忽略；
![](https://images2015.cnblogs.com/blog/564050/201611/564050-20161109010159092-764916187.png)


```
/**
 * Created by suyuan on 16/4/9.
 */

import 'style/page/PostPage.scss'
import {dispatch, changePostDataAction, getPostsData, updatePostsData} from 'action' // 定义了所有的action
import Button from 'component/Button' // 组件
import Post from 'component/Post' // 单片帖子组件

// 定义自己的页面,或者组件 PostPage, 下面安装调用次序给出了每个成员函数的解释,
class PostsPage extends React.Component { //继承后必须实现父类的方法 render

    //--------------Mounting(理解为第一次调用组件渲染)方法，目前有4个 constructor, componentWillMount, render, componentDidMount，

    // 1.1 构造
    // 在第一次构造组件时调用，继承父类，所以必须有第一句super(props),否则props不会传入父类，可能引起bug
    // 用途:一般用来初始化状态和绑定函数, 每个组件自己拥有一个state，不同的组件之间互相不影响；
    // 这里我们没有在构造函数中初始化 state,是因为用了redux 框架,真正的state 在外围的store中管理,传入的数据都在props中
    constructor(props) {
        super(props);
    }
    // 1.2 将要组装(翻译不准确)
    // 在render 之前调用，在这个方法中调用setState()不会重新渲染组建;
    // 一般不需要在该方法中做做操，用constrcutor的功能就可以了, 此方法中避免引入任何的副作用
    componentWillMount() {
    }

    // 1.3 渲染
    // 检查 props 或者state， 然后返回一个react对象,或者又其它组件组成的对象；也可以返回 null 或者false 表明不做渲染；
    // 该函数不可以有副作用，意思就是不能修改state,即不可以调用setState()，
    // 不可以和浏览器进行交互, 如果需要和浏览器交互，比如获取cookie 数据，应该在其它的函数中，比如componentWillMount 或者componentDidMount
    render(){
        return (<div className="post-parent">
                    <h2>帖子</h2>
                    {this.props.posts_data.map((item, index, items)=>{
                        return (<Post index={index} title={item.title} uri={item.uri} image={item.image} type={item.type} onDataChange={(post)=>this.onChange(index,post)} />)
                    })}
                    <div className="btn">
                        <Button type="secondary" onClick={this.onUpdate}>更新</Button>
                    </div>
                </div>)
    }

    // 自定义的内部函数,当页面的数据被用户修改后,发送changeIconDataAction, 在reducer 中更新状态,这里不是本节重点,暂时略过
    onChange=(i, post)=>{
        dispatch(changePostDataAction(i, post));
    }

    // 自定义内部函数, 当需要保存修改过的信息时,调用
    onUpdate=()=>{
        dispatch(updatePostsData(this.props.posts_data));
    }

    // 1.4 完成组装后
    // 初始化后,渲染后才触发, react 中继承的方法有一个特点，Did是表示在动作完成后执行的函数， will是指在动作完成前执行的函数
    // 这个方法很有用， 一般进行网络请求获取数据在这里进行，里面的setState 会导致render 重新执行
    componentDidMount() {
        dispatch(getPostsData()); // 发出post 请求帖子数据, 这里的action 发出后会去做post网络请求, 一般由中间件完成post请求
    }

    //-----到这里整个页面自动的渲染工作就结束了,后面当对页面做一些数据修改的操作,会用到下面的成员函数
    //-----以下称为updating方法, 当改变了props 或者state 后，下面这些方法会被执行

    //2.1 接受props前
    // 每次接受新的props触发, 这里有个注意点,state发生变化后是不会触发函数调用的,该函数调用关心的是props,
    // 注意：在接受到新的props 后，如果你需要更新state， 可以在此处执行setState， 这里可以看作完成状态变化的工作
    // state 和props 可以做个理解,props 是父组件传给自己的参数,但是自己也可以维护一个state, 部分数据用props, 部分数据用state, 比如计算props里面的post 数量放在state中
    componentWillReceiveProps(nextProps) {

    }

    //2.2 是否需要重新渲染
    // 如果该返回false， render将不会被调用，表示组件不需要更新, 默认情况下每次state变化后返回true， 初始化渲染或者forceUpdate 的时候是不会调用该成员函数
    // 当返回false的时候，子组件的state变化还是会继续渲染； 但是本组件的 componentWillUpdate(),render(), and componentDidUpdate() 不会被调用
    shouComponentUpdate(nextProps, nextState){

    }

    // 2.3 将要重新渲染
    // 第一次初始渲染的时候不会调用，当接受到新的 props 或者state的时候，使用该方法来做更新组件前的准备工作
    // 注意，在这里不要调用this.setState(), 如果需要更新state，在 componentWillReceiveProps 中调用，我的理解是 此时的nextState 已经ok， 就不要再改变了
    componentWillUpdate(nextProps, nextState){

    }

    // 2.4 render() 重新执行1.3 的渲染函数

    // 2.5 组件更新后
    componentDidUpdate(){

    }

    // 2.* Unmounting 当改组件从DOM 中移除时，调用此方法
    componentWillUnmount(){

    }

    /* OtherApis 每个组件都提供了一些成员函数调用接口
    1.setState(nextState, callback) 第一个参数是需要更新的状态，第二个参数是一个回调，表示状态更新完成后执行动作，一般不需要，直接用componentDidUpdate 代替回调函数作用即可
    注意：setState 类似一个异步接口，不会立即修改state，这里不实现同步的原因应该是 setState 里面是一个批处理，以改善setState的性能
         setState 一定会重新render组件，所以要保证需要的时候才调用，或者通过控制shouldComponentUpdate() 返回bool值,避免无意义刷新

    2.forceUpdate() 如果组件依赖与其它数据，此时希望强制重新渲染，调用后跳过shouldComponentUpdate(), 其它生命周期的函数正常调用
    */

    /* Class 成员属性
        1.defaultProps 默认属性值，
        2.displayName 一般不关心,react自动赋值
        3.propTypes 需要属性类型
    */

    /* 对象实例 成员属性, 所有有意义的数据都在他们里面, 设计好每个组件的state和props 是成功的第一步
        1.props
        2.state
    */

}


// redux 会自动调用,通过下面的connect 绑定到组件上, 所有的reducer 操作后返回的newstate 都会进入这个mapStateToProps 函数,
// 次函数把感兴趣的数据取出来, 以props的形式给绑定的组件
const mapStateToProps=(state, props)=>{
    var posts_data = state['api'][APIS.HOME.GET_POSTS_DATA] || [
        {
            'id': 1,
            'title': '',
            'type': 1,
            'url': '',
            'icon': ''
        },
        {
            'id': 2,
            'title': '',
            'type': 1,
            'url': '',
            'icon': ''
        },
        {
            'id': 3,
            'title': '',
            'type': 1,
            'url': '',
            'icon': ''
        },
        {
            'id': 4,
            'title': '',
            'type': 1,
            'url': '',
            'icon': ''
        },
    ];
    // 注意使用切片操作，这样才能更新 render，不然内容地址不变，里面的内容变了也不会刷新， 感觉这是redux 的坑
    return Object.assign({}, {'post_data': posts_data.slice()});
}

// redux 框架中非常重要的 connect 函数, mapStateToProps映射函数绑定到组件, 组件接受store的影响, store 通过reducer 创建出来, reducer 接受 action 处理state
export default connect(mapStateToProps)(PostsPage);
```