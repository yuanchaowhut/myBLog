

# 前言
1. 参考
[redux中文文档](http://cn.redux.js.org/docs/react-redux/api.html)
[参考2](https://segmentfault.com/a/1190000017064759) 

# redux 与 react-redux


# react-redux 

## Connect
### Connect：使用mapStateToProps抽取数据
Connect的第一个参数【mapStateToProps】:<br/>
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
    >将mapDispatchToProps定义为一个函数使你更灵活地定义你的组件能够接收到的函数、以及这些函数如何分发actions。你对dispatch和ownProps都具有访问权限。你可以借此机会编写你的连接组件的自定义函数
    - 对象：
