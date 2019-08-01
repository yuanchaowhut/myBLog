<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [特点](#%E7%89%B9%E7%82%B9)
- [事件注册](#%E4%BA%8B%E4%BB%B6%E6%B3%A8%E5%86%8C)
  - [setInitialDOMProperties](#setinitialdomproperties)
    - [ensureListeningTo](#ensurelisteningto)
    - [listenTo](#listento)
    - [trapBubbledEvent](#trapbubbledevent)
      - [addEventBubbleListener](#addeventbubblelistener)
- [事件分发](#%E4%BA%8B%E4%BB%B6%E5%88%86%E5%8F%91)
- [事件执行](#%E4%BA%8B%E4%BB%B6%E6%89%A7%E8%A1%8C)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


# 特点
1. React事件使用了【事件委托】的机制
    - 一般事件委托的作用都是为了减少页面的注册事件数量，减少内存开销，优化浏览器性能，
    - 另外可以更好的【管理事件】，
    - 实际上，React中所有的事件最后都是被委托到了 document这个顶级DOM上
2. React中就存在了自己的 合成事件(SyntheticEvent)，
    - 合成事件由对应的 EventPlugin负责合成，不同类型的事件由不同的 plugin合成，例如 SimpleEvent Plugin、TapEvent Plugin等
3. 为了进一步提升事件的性能，使用了 EventPluginHub这个东西来负责合成事件对象的创建和销毁

# 事件注册
案例代码
```
//index.js
ReactDOM.render(<App />, document.getElementById('root'));
//App.js
class App extends React.Component {
    //...
    render() {
        return <div onClick={this.clickHandler}>这是A组件</div>
    }
}
```

调用栈以及事件注册的入口
![avatar](../images/react/click-event-register.png)

## setInitialDOMProperties
>node_modules/react-dom/cjs/react-dom.development.js

```
function setInitialDOMProperties(tag, domElement, rootContainerElement, nextProps, isCustomComponentTag) {
    for (var propKey in nextProps) {
        if (!nextProps.hasOwnProperty(propKey)) {
            continue;
        }

        var nextProp = nextProps[propKey];

        if (propKey === STYLE$1) {
            //...
        } else if (propKey === DANGEROUSLY_SET_INNER_HTML) {
            //...
        } else if (propKey === CHILDREN) {
            //...
        } else if (propKey === SUPPRESS_CONTENT_EDITABLE_WARNING || propKey === SUPPRESS_HYDRATION_WARNING$1) {// Noop
        } else if (propKey === AUTOFOCUS) {
            //...
        } else if (registrationNameModules.hasOwnProperty(propKey)) {
            if (nextProp != null) {
                if (true && typeof nextProp !== 'function') {
                    warnForInvalidEventListener(propKey, nextProp);
                }

                ensureListeningTo(rootContainerElement, propKey);
            }
        } else if (nextProp != null) { 
            //...
        }
    }
}
```
变量registrationNameModules的含义，存储量一推事件映射
![avatar](../images/react/registeration-name-map.png)

### ensureListeningTo

```
function ensureListeningTo(rootContainerElement, registrationName) {
  var isDocumentOrFragment = rootContainerElement.nodeType === DOCUMENT_NODE || rootContainerElement.nodeType === DOCUMENT_FRAGMENT_NODE;
  var doc = isDocumentOrFragment ? rootContainerElement : rootContainerElement.ownerDocument;
  listenTo(registrationName, doc);
}
```
![avatar](../images/react/root-containeri-ele.png)
1. 首先判断了 rootContainerElement是不是一个 document或者 Fragment(文档片段节点)，示例中传过来的是 .box这个 div，显然不是，
2. 所以 doc这个变量就被赋值为 rootContainerElement.ownerDocument，这个东西其实就是document元素，
3. 把这个document传到下面的 listenTo里了，【事件委托也就是在这里做的】，
4. 所有的事件最终都会被委托到 document 或者 fragment上去，大部分情况下都是 document，
5. 然后这个 registrationName就是事件名称 onClick
 

### listenTo
```
switch (dependency) {
  // 省略一些代码
  default: 
    var isMediaEvent = mediaEventTypes.indexOf(dependency) !== -1;
    if (!isMediaEvent) {
      trapBubbledEvent(dependency, mountAt);
    }
    break;
} 
```
这个方法其实就是注册事件的入口<br/>
变量：registrationNameDependencies<br/>
![avatar](../images/react/registernamedpen.png)<br/>
可以看到，React是给事件名做了一些跨浏览器兼容事情的，比如传入 onChange事件，会自动对应上 blur change click focus等多种浏览器原生事件<br/>

1. 除了 scroll focus blur cancel close方法走【trapCapturedEvent】方法，
2. invalid submit reset方法不处理之外，
3. 剩下的事件类型全走default，执行【trapBubbledEvent】这个方法（注册冒泡阶段的事件）
4. trapCapturedEvent 和 trapBubbledEvent二者唯一的不同之处就在于，对于最终的合成事件，前者注册捕获阶段的事件监听器，而后者则注册冒泡阶段的事件监听器
 
>由于大部分合成事件的代理注册的都是冒泡阶段的事件监听器，也就是委托到 document上注册的是冒泡阶段的事件监听器，所以就算你显示声明了一个捕获阶段的 React事件，例如 onClickCapture，此事件的响应也会晚于原生事件的捕获事件以及冒泡事件
 实际上，所有原生事件的响应(无论是冒泡事件还是捕获事件)，都将早于 React合成事件(SyntheticEvent)，对原生事件调用 e.stopPropagation()将阻止对应 SyntheticEvent的响应，因为对应的事件根本无法到达document 这个事件委托层就被阻止掉了
  
### trapBubbledEvent
```
function trapBubbledEvent(topLevelType, element) {
  if (!element) {
    return null;
  }

  var dispatch = isInteractiveTopLevelEventType(topLevelType) ? dispatchInteractiveEvent : dispatchEvent;
  addEventBubbleListener(element, getRawEventName(topLevelType), // Check if interactive and wrap in interactiveUpdates
  dispatch.bind(null, topLevelType));
}
```

1. addEventBubbleListener这个方法接收三个参数：
    - 第一个参数 element其实就是 document元素，
    - getRawEventName(topLevelType)就是 click事件，
    - 第三个参数的 dispatch就是 dispatchInteractiveEvent，
2. dispatchInteractiveEvent其实最后还是会执行 dispatchEvent这个方法，只是在执行这个方法之前做了一些额外的事情，这里不需要关心，可以暂且认为二者是一样的
 
#### addEventBubbleListener
 ```
function addEventBubbleListener(element, eventType, listener) {
   element.addEventListener(eventType, listener, false);
}
```

## 小结
流程图如下：<br/>
![avatar](../images/react/react-event-reagister-s.png)

# 事件分发


# 事件执行