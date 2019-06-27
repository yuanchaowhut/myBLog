## 3.1 value:双向绑定
- 结构


```
ko.bindingHandlers['value'] = {
    'after': ['options', 'foreach'],
    'init': function (element, valueAccessor, allBindings) {
    },
    'update': function() {} //空实现
};
```

- 双向绑定的两个方向：1. dom -> viewModel 2. viewModel -> dom

- ko.bindingHandlers['value'].init，该方法较长，分为以下几个部分
    1. input[tyep=checkbox]、input[type=radio]的处理
    2. 事件名称处理
    3. 事件订阅 
    >这部分重要性在于：给input[type=text]/select元素注册change事件；完成双向绑定的一侧即dom -> viewModel
    4. 注册依赖 
    >完成双向绑定的另一侧即 viewModel -> dom

### 3.1.1 input[tyep=checkbox]、input[type=radio]


```
ko.bindingHandlers['value'] = { 
    'init': function (element, valueAccessor, allBindings) { 
        if (element.tagName.toLowerCase() == "input" && (element.type == "checkbox" || element.type == "radio")) {
            ko.applyBindingAccessorsToNode(element, { 'checkedValue': valueAccessor });
            return;
        } 
        //...
    }
}
```


- ko.bindingHandlers['value']是针对input[type=text]、select两种情况的

### 3.1.2 事件名称处理
```
ko.bindingHandlers['value'] = { 
    'init': function (element, valueAccessor, allBindings) {  
        var eventsToCatch = ["change"];
        var requestedEventsToCatch = allBindings.get("valueUpdate");
        var propertyChangedFired = false;
        var elementValueBeforeEvent = null;
    
        if (requestedEventsToCatch) {
            if (typeof requestedEventsToCatch == "string")  
                requestedEventsToCatch = [requestedEventsToCatch];
            ko.utils.arrayPushAll(eventsToCatch, requestedEventsToCatch);
            eventsToCatch = ko.utils.arrayGetDistinctValues(eventsToCatch);
        }
    }
}
```


1. 获取事件名称，change事件必须的，因为要通过该事件实现dom的更新 -> viewModel的更新
2. 保证事件名称的唯一性


### 3.1.3 事件订阅 

```
ko.bindingHandlers['value'] = { 
    'init': function (element, valueAccessor, allBindings) {  
        //...
         var valueUpdateHandler = function() {
            elementValueBeforeEvent = null;
            propertyChangedFired = false;
            var modelValue = valueAccessor();
            var elementValue = ko.selectExtensions.readValue(element);
            ko.expressionRewriting.writeValueToProperty(modelValue, allBindings, 'value', elementValue);
        }
 
        var ieAutoCompleteHackNeeded = ko.utils.ieVersion && element.tagName.toLowerCase() == "input" && element.type == "text"
            && element.autocomplete != "off" && (!element.form || element.form.autocomplete != "off");
        if (ieAutoCompleteHackNeeded && ko.utils.arrayIndexOf(eventsToCatch, "propertychange") == -1) {
            ko.utils.registerEventHandler(element, "propertychange", function () { propertyChangedFired = true });
            ko.utils.registerEventHandler(element, "focus", function () { propertyChangedFired = false });
            ko.utils.registerEventHandler(element, "blur", function() {
                if (propertyChangedFired) {
                    valueUpdateHandler();
                }
            });
        }

        ko.utils.arrayForEach(eventsToCatch, function(eventName) { 
            var handler = valueUpdateHandler;
            if (ko.utils.stringStartsWith(eventName, "after")) {
                handler = function() { 
                    elementValueBeforeEvent = ko.selectExtensions.readValue(element);
                    ko.utils.setTimeout(valueUpdateHandler, 0);
                };
                eventName = eventName.substring("after".length);
            }
            ko.utils.registerEventHandler(element, eventName, handler);
        });
        //...
    }
}
```


- valueUpdateHandler 是change事件的回调函数
- 中间部分是ie下的兼容性处理，注册propertychange、focus、blur事件（propertychange等价于input事件）
    - [兼容性问题](https://github.com/knockout/knockout/pull/122)
    >IE doesn't fire "change" events on textboxes if the user selects a value from its autocomplete list
    - propertyChangedFired:true + blur = change事件
- 事件注册
    - 值得一提的是，对于 "after<eventname>" 事件的处理，这里其实是好意，为了引入'afterXxx'事件，但是引入这样的事件会带来一些问题见[issue](https://github.com/knockout/knockout/pull/1334)
    1.变量elementValueBeforeEvent只有在keyXxx等事件触发 到 valueUpdateHandler函数运行之间很短暂的时间（brief time）是非空值 <br/>
    2.数据的更新可能是异步的，比如deferUpdate/rateLimit等使用 会异步更新数据（那么可能会造成数据不同步） 
     对于这种情况，上面的说的（brief time）期间，可能会丢失用户输入的数据，因此我们需要记录用户输入的数据，以免丢失，然后使用用户输入的数据覆盖
     
- 小节：
以input[type=text]为例，
输入框发生change事件 
-> valueUpdateHandler 
->  ko.expressionRewriting.writeValueToProperty
-> 执行 modelValue（observable）
-> updateFromModel,但是 valueHasChanged 相等，所以不更新
      
### 3.1.4 注册依赖 
 ```
 ko.bindingHandlers['value'] = { 
     'init': function (element, valueAccessor, allBindings) {  
         //...
         var updateFromModel = function () { // 这里的触发是由于绑定的model更新了
             var newValue = ko.utils.unwrapObservable(valueAccessor()); // model的值
             var elementValue = ko.selectExtensions.readValue(element);  // dom的值
 
             if (elementValueBeforeEvent !== null && newValue === elementValueBeforeEvent) { // "after<eventname>" 事件可能出现情况的处理
                 ko.utils.setTimeout(updateFromModel, 0);
                 return;
             }
 
             var valueHasChanged = (newValue !== elementValue); // 只有当数据发生变化，才更新dom的值
 
             if (valueHasChanged) { //select 
                 if (ko.utils.tagNameLower(element) === "select") {
                     var allowUnset = allBindings.get('valueAllowUnset');
                     var applyValueAction = function () {
                         ko.selectExtensions.writeValue(element, newValue, allowUnset);
                     };
                     applyValueAction();
 
                     if (!allowUnset && newValue !== ko.selectExtensions.readValue(element)) { // 处理选项中没有设置值的情况
                         ko.dependencyDetection.ignore(ko.utils.triggerEvent, null, [element, "change"]);
                     } else {  // IE6 bug
                         ko.utils.setTimeout(applyValueAction, 0);
                     }
                 } else {  // input[type=text]
                     ko.selectExtensions.writeValue(element, newValue);
                 }
             }
         };
 
         ko.computed(updateFromModel, null, { disposeWhenNodeIsRemoved: element }); // 关键：添加订阅，注册依赖
         //...
     }
 }
 ```

- "after<eventname>" 情况处理
- select
    - 处理选项中没有设置值的情况 
        - 当前select的所有选项中没有设置的值，那么触发change事件 
    
    - IE6 关于select控件的bug
        - 即在IE6下，如果在当前‘线程’中动态添加选项（options）和设置选中选项不能同时进行，因此通过setTimeout在‘第二线程’中设置选中项
        - [参考](https://www.jb51.net/article/93033.htm)
- input[type=text] 
 

以下面代码为例
```
//html
<input type="text" data-bind="value:valueObservable"  />
//js
var valueObservable = ko.observable('--');
valueObservable("new-value"); 
``` 

valueObservable("new-value"); 
-> updateFromModel 
-> ko.selectExtensions.writeValue(element, newValue);
  
### 3.1.6 input[type='checkbox']、input[type='radio']说sourceBindings的作用
``` 
//ko.bindingHandlers['value'].init
ko.applyBindingAccessorsToNode(element, { 'checkedValue': valueAccessor });
```
sourceBindings的作用：通过js编码的方式代替了在html中写data-bind的过程