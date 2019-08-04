<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [前言](#%E5%89%8D%E8%A8%80)
- [redux 与 react-redux](#redux-%E4%B8%8E-react-redux)
  - [redux](#redux)
- [react-redux](#react-redux)
  - [Connect](#connect)
    - [Connect：使用mapStateToProps抽取数据](#connect%E4%BD%BF%E7%94%A8mapstatetoprops%E6%8A%BD%E5%8F%96%E6%95%B0%E6%8D%AE)
    - [Connect: 使用mapDispatchToProps分发actions](#connect-%E4%BD%BF%E7%94%A8mapdispatchtoprops%E5%88%86%E5%8F%91actions)
      - [bindActionCreators的作用](#bindactioncreators%E7%9A%84%E4%BD%9C%E7%94%A8)
  - [connectAdvanced、createProvider](#connectadvancedcreateprovider)
  - [Provider](#provider)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->



# 前言
1. 参考
[redux中文文档](http://cn.redux.js.org/docs/react-redux/api.html)、[学习资源](http://cn.redux.js.org/docs/introduction/LearningResources.html)
[参考2](https://segmentfault.com/a/1190000017064759) 


# redux 与 react-redux 关系
## redux
1. 动机：管理状态（不受控制 => 可预测）
2. 核心概念：state、action、reducer
    state：读取数据（只读）
    action：更改数据，唯一修改state的方式
    reducer：把 action 和 state 串起来
3. 三大原则
    单一数据源<br/>
    State 是只读的<br/>
    使用纯函数来执行修改<br/>
4.  



# react-redux 使用

## Connect
返回值：一个高阶React组件类，它能够把state和action creators传递给由所提供的参数生成的组件中去。这一高阶组件由connectAdvanced生成
### Connect：使用mapStateToProps抽取数据
Connect的第一个参数【mapStateToProps】:<br/>
>mapStateToProps的结果必须是一个纯对象，之后该对象会合并到组件的props
- 参数形式：
    - 函数：function mapStateToProps(state, ownProps?)
        - state：store.getState()，
        - ownProps：自身用的props数据，
        - 返回值：包含了组件用到的数据的纯对象
    - 对象
- 功能：用于订阅store
    - 订阅：从store中提取的部分值，当这些值改变时会重新渲染
    - 不订阅：在store改变时不能够重新渲染
- 使用指南：
    - 让mapStateToProps改造从store中取出的数据
    - 使用Selector方法去抽取和转化数据
    - mapStateToProps方法应该足够快（一旦store改变了，所有被连接组件中的所有的mapStateToProps方法都会运行）
    - mapStateToProps方法应该纯净且同步（应该仅接受state（以及ownProps）作为参数，然后以props形式返回组件所需要的数据；他不应该触发任何异步操作，如AJAX请求数据，方法也不能声明为async形式）
- 性能    
    shouldComponentUpdate使用浅比较检查mapStateToProps的结果是否改变了<br/>
    >React-Redux 内部实现了shouldComponentUpdate方法以便在组件用到的数据发生变化后能够精确地重新渲染。默认地，React-Redux使用“===”对mapStateToProps返回的对象的每一个字段逐一对比，以判断内容是否发生了改变。但凡有一个字段改变了，你的组件就会被重新渲染以便接收更新过的props值。注意到，返回一个相同引用的【突变对象】（mutated object）是一个常见错误，因为这会导致你的组件不能如期重新渲染。
    
    仅在需要时返回新的对象引用<br/>
    >很多常见的操作都会返回新对象或数组引用：<br/>
     创建新的数组：使用someArray.map()或者someArray.filter()<br/>
     合并数组：array.concat<br/>
     截取数组：array.slice<br/>
     复制值：Object.assgin<br/>
     使用扩展运算符：{...oldState,...newData}<br/>
     把这些操作放在[memeoized selector functions](https://react-redux.js.org/using-react-redux/connect-mapstate)中确保它们只在输入值变化后运行。这样也能够确保如果输入值没有改变，mapStateToProps仍然返回与之前的相同值，然后connect就能够跳过重渲染过程。
     
     仅在数据改变时运行开销大的操作<br/>
     >转化数据经常开销很大,应该仅在相关数据改变时重新运行这些复杂的转化<br/>
     有下面几种形式来达到这样的目的：<br/>
     一些转化可以在action创建函数或者reducer中运算，然后可以把转化过的数据储存在store中<br/>
     转换也可以在组件的render()生命周期中完成<br/>
     如果转化必须要在mapStateToProps方法中完成，那么我们建议使用[memeoized selector functions](https://react-redux.js.org/using-react-redux/connect-mapstate)以确保转化仅发生在输入的数据变化时。<br/>
     
     考虑Immutable.js性能<br/>
     >如果开始考虑性能了要避免使用toJS
     
     声明参数的数量影响行为<br/>
     >当仅有(state)时，每当根store state对象不同了，函数就会运行。 <br/>
      当有(state,ownProps)两个参数时，每当store state不同、或每当包装props变化时，函数都会运行。 <br/>
      这意味着你不应该增加ownProps参数，除非你实在是需要它，否则你的mapStateToProps函数会比它实际需要运行次数运行更多次。<br/>


### Connect: 使用mapDispatchToProps分发actions
dispatch是Redux store实例的一个方法，你会通过store.dispatch来分发一个action。这也是唯一触发一个state变化的途径。

Connect的第二个参数【mapDispatchToProps】<br/>
- 功能：注入actions
    - 不注入：接收一个props.dispatch以便你手动分发actions
    - 以props形式接收每个你通过mapDispatchToProps注入的action创建函数，能够在你调用后自动分发actions
- 参数形式：
    - 函数：function mapDispatchToProps(dispatch, ownProps?)
    >将mapDispatchToProps定义为一个函数使你更灵活地定义你的组件能够接收到的函数、以及这些函数如何分发actions。你对dispatch和ownProps都具有访问权限。你可以借此机会编写你的连接组件的自定义函数<br/>
    mapDispatchToProps的函数返回值会合并到你的组件props中去。你就能够直接调用它们来分发action。
    - 对象：由于上面函数形式的这个过程太过通用，因此connect支持了一个“对象简写”形式的mapDispatchToProps参数：如果你传递了一个由action creators构成的对象，而不是函数，connect会在内部自动为你调用bindActionCreators
    
    
#### bindActionCreators的作用
1. 作为返回值的action需要手动分发
```
const increment = () => ({ type: "INCREMENT" });
const decrement = () => ({ type: "DECREMENT" });
const reset = () => ({ type: "RESET" });

const mapDispatchToProps = dispatch => {
  return {
    // 分发由action creators创建的actions
    increment: () => dispatch(increment()),
    decrement: () => dispatch(decrement()),
    reset: () => dispatch(reset())
  };
};
```

2. 由bindActionCreators生成的包装函数会自动转发它们所有的参数，所以你不需要在手动操作了
```
import { bindActionCreators } from "redux";

const increment = () => ({ type: "INCREMENT" });
const decrement = () => ({ type: "DECREMENT" });
const reset = () => ({ type: "RESET" });

// 绑定一个action creator
// 返回 (...args) => dispatch(increment(...args))
const boundIncrement = bindActionCreators(increment, dispatch);

// 绑定一个action creators构成的object
const boundActionCreators = bindActionCreators({ increment, decrement, reset }, dispatch);
// 返回值：
// {
//   increment: (...args) => dispatch(increment(...args)),
//   decrement: (...args) => dispatch(decrement(...args)),
//   reset: (...args) => dispatch(reset(...args)),
// }
```

## connectAdvanced、createProvider

## Provider
<Provider/>使得每一个被connect()函数包装过的嵌套组件都可以访问到Redux store。

既然任何Redux-React app中的React组件都可以被连接，那么大多数应用都会在最顶层渲染<Provider>，从而包裹住整个组件树。

通常，你如果不把已连接组件嵌套在<Provider>中那么你就不能使用它。


# react-redux源码分析
[参考](https://juejin.im/post/59cb5eba5188257e84671aca)

## createStore 