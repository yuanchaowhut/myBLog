<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [3.5 event](#35-event)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## 3.5 event
>用于事件注册，没啥好说的

提下特别处理的地方
- 默认会preventDefault，只要事件回调返回的不是ture，则阻止默认动作，比如a标签的click事件，如果回调中没有返回true，则不会更改url、页面跳转等工作
``` 
if (handlerReturnValue !== true) { 
    if (event.preventDefault)
        event.preventDefault();
    else
        event.returnValue = false;
}
```
- 默认会stopPropagation，除非data-bind='[eventName]Bubble:true'
``` 
var bubble = allBindings.get(eventName + 'Bubble') !== false;
if (!bubble) {
    event.cancelBubble = true;
    if (event.stopPropagation)
        event.stopPropagation();
}
```
