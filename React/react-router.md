<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [简易路由实现](#%E7%AE%80%E6%98%93%E8%B7%AF%E7%94%B1%E5%AE%9E%E7%8E%B0)
- [react-router使用](#react-router%E4%BD%BF%E7%94%A8)
  - [withRouter](#withrouter)
- [react-router分析](#react-router%E5%88%86%E6%9E%90)
  - [<HashRouter>作为组件被渲染到页面中](#hashrouter%E4%BD%9C%E4%B8%BA%E7%BB%84%E4%BB%B6%E8%A2%AB%E6%B8%B2%E6%9F%93%E5%88%B0%E9%A1%B5%E9%9D%A2%E4%B8%AD)
    - [执行HashRouter构造函数](#%E6%89%A7%E8%A1%8Chashrouter%E6%9E%84%E9%80%A0%E5%87%BD%E6%95%B0)
      - [创建histroy对象](#%E5%88%9B%E5%BB%BAhistroy%E5%AF%B9%E8%B1%A1)
  - [Router组件渲染](#router%E7%BB%84%E4%BB%B6%E6%B8%B2%E6%9F%93)
    - [hashChange事件的监听](#hashchange%E4%BA%8B%E4%BB%B6%E7%9A%84%E7%9B%91%E5%90%AC)
      - [history.listen](#historylisten)
      - [事件注销](#%E4%BA%8B%E4%BB%B6%E6%B3%A8%E9%94%80)
  - [跨组件数据共享](#%E8%B7%A8%E7%BB%84%E4%BB%B6%E6%95%B0%E6%8D%AE%E5%85%B1%E4%BA%AB)
    - [Router](#router)
    - [Route、Switch](#routeswitch)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


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

# react-router使用
http://react-guide.github.io/react-router-cn/index.html
## withRouter

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

## <HashRouter>作为组件被渲染到页面中  
```
//node_modules/react-dom/cjs/react-dom.development.js

function constructClassInstance(workInProgress, ctor, props, renderExpirationTime) {
    //...
    var instance = new ctor(props, context); // 构造组件实例
    //...
}
```
![avatar](../images/react/constructClassInstance.png)<br/>


### 执行HashRouter构造函数
```
//node_modules/react-router-dom/esm/react-router-dom.js

var HashRouter = 
function (_React$Component) { //不是构造函数哦
  _inheritsLoose(HashRouter, _React$Component);

  function HashRouter() { // 这才是构造函数
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
}(React.Component); //立即执行函数
```

注意这里的render方法可以进行改写
```
  _proto.render = function render() {
    return React.createElement(Router, {
      history: this.history,
      children: this.props.children
    });
  };
```

等价于下面代码，因此在渲染HashRouter的过程中，会去渲染Router组件
``` 
render(){
    return <Router history={this.history} children={this.props.children}></Router>
}
```
 
#### 创建histroy对象
在执行HashRouter构造函数时会调用createHashHistory：创建histroy对象，并作为属性传递到Router组件中 
```
//node_modules/history/esm/history.js

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

## Router组件渲染
```
//node_modules/react-router/esm/react-router.js

var Router =
/*#__PURE__*/
function (_React$Component) {
  _inheritsLoose(Router, _React$Component);

  Router.computeRootMatch = function computeRootMatch(pathname) {
    return {
      path: "/",
      url: "/",
      params: {},
      isExact: pathname === "/"
    };
  };

  function Router(props) { // 构造函数
    var _this;

    _this = _React$Component.call(this, props) || this;
    _this.state = {
      location: props.history.location
    }; 

    _this._isMounted = false;
    _this._pendingLocation = null;

    if (!props.staticContext) {
      _this.unlisten = props.history.listen(function (location) { 
        if (_this._isMounted) {
          _this.setState({ 
            location: location
          });
        } else {
          _this._pendingLocation = location;
        }
      });
    }

    return _this;
  }

  var _proto = Router.prototype;

  _proto.componentDidMount = function componentDidMount() {
    this._isMounted = true;

    if (this._pendingLocation) {
      this.setState({
        location: this._pendingLocation
      });
    }
  };

  _proto.componentWillUnmount = function componentWillUnmount() {
    if (this.unlisten) this.unlisten();
  };

  _proto.render = function render() {
    return React.createElement(context.Provider, {
      children: this.props.children || null,
      value: {
        history: this.props.history,
        location: this.state.location,
        match: Router.computeRootMatch(this.state.location.pathname),
        staticContext: this.props.staticContext
      }
    });
  };

  return Router;
}(React.Component); 
```

### hashChange事件的监听
props.history.listen：props.history来源于父组件HashRouter<br/>
```
 _this.unlisten = props.history.listen(function (location) { // hashChange回调
    if (_this._isMounted) {
      _this.setState({  // 关键之处：引起子组件重新渲染，解释了hash值的变化导致组件的重新渲染的过程
        location: location
      });
    } else {
      _this._pendingLocation = location;
    }
  });
```
注意这里的返回值 _this.unlisten：用于注销注册的事件

#### history.listen 
>node_modules/history/esm/history.js

```
function listen(listener) {
    var unlisten = transitionManager.appendListener(listener);
    checkDOMListeners(1);
    return function () {
      checkDOMListeners(-1);
      unlisten();
    };
}
```

checkDOMListeners：监听hashChange事件
```
  var HashChangeEvent$1 = 'hashchange';
  function checkDOMListeners(delta) {
    listenerCount += delta;

    if (listenerCount === 1 && delta === 1) {
      window.addEventListener(HashChangeEvent$1, handleHashChange);
    } else if (listenerCount === 0) {
      window.removeEventListener(HashChangeEvent$1, handleHashChange);
    }
  }
```

#### 事件注销
```
//node_modules/react-router/esm/react-router.js

var Router =
/*#__PURE__*/
function (_React$Component) {
   //...
  _proto.componentWillUnmount = function componentWillUnmount() {
    if (this.unlisten) this.unlisten();
  };
  //...
}
```


## 跨组件数据共享
### Router

```
// //node_modules/react-router/esm/react-router.js
var context = createNamedContext("Router"); //关键代码：共享router

var Router =
/*#__PURE__*/
function (_React$Component) {
    //...
     _proto.render = function render() {
        return React.createElement(context.Provider, {
          children: this.props.children || null,
          value: { // 共享的router的值
            history: this.props.history,
            location: this.state.location,
            match: Router.computeRootMatch(this.state.location.pathname),
            staticContext: this.props.staticContext
          }
        });
      };
    //...
}
```

### Route
使用context.Consumer

### Switch

### Link
<Link />的核心就是渲染<a>标签，拦截<a>标签的点击事件，然后通过<Router />共享的router对history进行路由操作，进而通知<Router />重新渲染

### Route
<Route />有一部分源码与<Router />相似，可以实现路由的嵌套，但其核心是通过Context共享的router，判断是否匹配当前路由的路径，然后渲染组件


# 总结
整个react-router其实就是围绕着<Router />的Context来构建的