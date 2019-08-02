
[参考](https://github.com/joeyguo/blog/issues/2)<br/>

# 简易路由实现
``` 
function Router() {
    this.routes = {};
    this.currentUrl = '';
}
Router.prototype.route = function(path, callback) {
    this.routes[path] = callback || function(){};
};
Router.prototype.refresh = function() {
    this.currentUrl = location.hash.slice(1) || '/';
    this.routes[this.currentUrl]();
};
Router.prototype.init = function() {
    window.addEventListener('load', this.refresh.bind(this), false);
    window.addEventListener('hashchange', this.refresh.bind(this), false);
}
```
[简易路由实现](testDemo/simple-router.html)<br/>

# react-router分析
react-router的使用：[官方文档](https://github.com/ReactTraining/react-router)<br/>

2019.8.2 先粗略说下整个流程，以后再细看这里的流程<br/>

以下面代码为例，说下路由渲染和切换的流程
```
var Routers = <HashRouter>
    <Switch>
        <Route path="/404" component={_404Page}/>
        <Route path="/home" component={App} a="test" exact/>

        <Redirect from="/" to="/home" exact/>
        <Redirect to={{pathname: "/404"}}/>
    </Switch>
</HashRouter>;

ReactDOM.render(Routers, document.getElementById('root'));
```

## 首先<HashRouter>作为组件被渲染到页面中
>node_modules/react-router-dom/esm/react-router-dom.js

HashRouter组件实现<br/>
```
var HashRouter = 
function (_React$Component) {
  _inheritsLoose(HashRouter, _React$Component);

  function HashRouter() {
    var _this;
    //...
    _this.history = createHashHistory(_this.props); //保存
    return _this;
  }

  var _proto = HashRouter.prototype;

  _proto.render = function render() {
    return React.createElement(Router, {
      history: this.history,
      children: this.props.children
    });
  };

  return HashRouter;
}(React.Component);
```

1. createHashHistory：创建histroy对象
>node_modules/history/esm/history.js

```
function createHashHistory(props) {
    function listen(listener) {
        var unlisten = transitionManager.appendListener(listener);
        checkDOMListeners(1);
        return function () {
          checkDOMListeners(-1);
          unlisten();
        };
    }

    var history = {
        length: globalHistory.length,
        action: 'POP',
        location: initialLocation,
        createHref: createHref,
        push: push,
        replace: replace,
        go: go,
        goBack: goBack,
        goForward: goForward,
        block: block,
        listen: listen // 事件监听入口
    }; 
}
```

2. 组件创建
```
React.createElement(Router, {
      history: this.history,
      children: this.props.children
    }); 
```
