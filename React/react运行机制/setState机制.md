[参考](https://github.com/sisterAn/blog/issues/26) 
# setState 异步更新
setState 通过一个队列机制来实现 state 更新，当执行 setState() 时，会将需要更新的 state 浅合并后放入 状态队列，而不会立即更新 state，队列机制可以高效的【批量更新】 state。而如果不通过setState，直接修改this.state 的值，则不会放入状态队列，当下一次调用 setState 对状态队列进行合并时，之前对 this.state 的修改将会被忽略，造成无法预知的错误。

React通过【状态队列】机制实现了 setState 的异步更新，避免重复的更新 state。
 
在 setState 官方文档中介绍：将 nextState 【浅合并】到当前 state。这是在事件处理函数和服务器请求回调函数中触发 UI 更新的主要方法。不保证 setState 调用会同步执行，考虑到性能问题，可能会对多次调用作批处理。

何解决这个问题??<br/>
使用 setState() 的第二种形式：以一个函数而不是对象作为参数，此函数的第一个参数是前一刻的state，第二个参数是 state 更新执行瞬间的 props


## 异步案例 
setState()并不总是立即更新组件，它可能会进行批处理或者推迟更新。这使得在调用setState（）之后立即读取this.state成为一个潜在的隐患

```
add() {
    this.setState({
      count: this.state.count + 1
    });
    this.setState({
      count: this.state.count + 1
    });
  } 
```

正确的方式：
```
this.setState(preState => {
  return {
    count: preState.count + 1
  };
});
this.setState(preState => {
  return {
    count: preState.count + 1
  };
});
```

## 源码
```
ReactComponent.prototype.setState = function(partialState, callback) {
    this.updater.enqueueSetState(this, partialState)
    if (callback) {
        this.updater.enqueueCallback(this, callback, 'setState')
    }
}
```
![avatar](../../images/react/set-component-updater.png)