<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [event](#event)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# event
```
ko.bindingHandlers['event'] = {
    'init' : function (element, valueAccessor, allBindings, viewModel, bindingContext) {
        var eventsToHandle = valueAccessor() || {};
        ko.utils.objectForEach(eventsToHandle, function(eventName) {
            if (typeof eventName == "string") {
                ko.utils.registerEventHandler(element, eventName, function (event) {
                    var handlerReturnValue;
                    var handlerFunction = valueAccessor()[eventName];
                    if (!handlerFunction)
                        return;

                    try { 
                        var argsForHandler = ko.utils.makeArray(arguments);
                        viewModel = bindingContext['$data'];
                        argsForHandler.unshift(viewModel);
                        handlerReturnValue = handlerFunction.apply(viewModel, argsForHandler);
                    } finally {
                        if (handlerReturnValue !== true) {
                            if (event.preventDefault)
                                event.preventDefault();
                            else
                                event.returnValue = false;
                        }
                    }

                    var bubble = allBindings.get(eventName + 'Bubble') !== false;
                    if (!bubble) {
                        event.cancelBubble = true;
                        if (event.stopPropagation)
                            event.stopPropagation();
                    }
                });
            }
        });
    }
};
```

用于事件注册，没啥好说的，提下特别处理的地方

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
