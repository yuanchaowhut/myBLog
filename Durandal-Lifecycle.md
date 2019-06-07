<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [<div style='text-align:center'>Durandal-生命周期<span style='font-size:0.6em;color:grey'>（余 松）</span></div>](#div-styletext-aligncenterdurandal-%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9Fspan-stylefont-size06emcolorgrey%E4%BD%99-%E6%9D%BEspandiv)
  - [准备](#%E5%87%86%E5%A4%87)
    - [system.acquire](#systemacquire)
    - [system.defer](#systemdefer)
  - [生命周期](#%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F)
    - [setRoot入口](#setroot%E5%85%A5%E5%8F%A3)
    - [整体过程](#%E6%95%B4%E4%BD%93%E8%BF%87%E7%A8%8B)
      - [生命周期-activate的入口：tryActivate](#%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F-activate%E7%9A%84%E5%85%A5%E5%8F%A3tryactivate)
      - [生命周期-binding和bindingComplete的入口：binder.bind](#%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F-binding%E5%92%8Cbindingcomplete%E7%9A%84%E5%85%A5%E5%8F%A3binderbind)
      - [生命周期-attached，compositionComplete的入口：composition.finalize](#%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F-attachedcompositioncomplete%E7%9A%84%E5%85%A5%E5%8F%A3compositionfinalize)
        - [attached回调](#attached%E5%9B%9E%E8%B0%83)
        - [compositionComplete回调](#compositioncomplete%E5%9B%9E%E8%B0%83)
      - [生命周期-detached](#%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F-detached)
  - [补充](#%E8%A1%A5%E5%85%85)
    - [composition.addBindingHandler](#compositionaddbindinghandler)
    - [获取html](#%E8%8E%B7%E5%8F%96html)
        - [加载html文件入口](#%E5%8A%A0%E8%BD%BDhtml%E6%96%87%E4%BB%B6%E5%85%A5%E5%8F%A3)
        - [requirejs模块加载完毕后的相关代码](#requirejs%E6%A8%A1%E5%9D%97%E5%8A%A0%E8%BD%BD%E5%AE%8C%E6%AF%95%E5%90%8E%E7%9A%84%E7%9B%B8%E5%85%B3%E4%BB%A3%E7%A0%81)
    - [ko移除dom相关代码](#ko%E7%A7%BB%E9%99%A4dom%E7%9B%B8%E5%85%B3%E4%BB%A3%E7%A0%81)
  - [总结](#%E6%80%BB%E7%BB%93)
      - [执行过程](#%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# <div style='text-align:center'>Durandal-生命周期<span style='font-size:0.6em;color:grey'>（余 松）</span></div> 
> ko的缺点：组件无生命周期；durandal在其基础上实现了自己的‘组件’并提供了生命周期
## 案例代码
main.js
```
 app.setRoot('hello-index')
```

index.html
```
<div data-bind='hello/index'></div>
```

index.js
```
define(['durandal/composition', 'durandal/app', 'durandal/system', 'knockout'],
    function (composition, app, system, ko) {
        var that = null;
        composition.addBindingHandler('hello-index', {
            init: function () {
                that.init();
                system.log('custom BindingHandler:hello-index');
            }
        });

        function HelloModel() {
            that = this;
            $.extend(this, {
                activate: function () {
                    system.log('Lifecycle : activate : hello');
                },
                binding: function () {
                    system.log('Lifecycle : binding : hello');
                },
                bindingComplete: function () {
                    system.log('Lifecycle : bindingComplete : hello');
                },
                attached: function (view, parent) {
                    system.log('Lifecycle : attached : hello');
                },
                compositionComplete: function (view) {
                    system.log('Lifecycle : compositionComplete : hello');
                },
                detached: function (view) {
                    system.log('Lifecycle : detached : hello');
                }
            })
        }

        HelloModel.prototype.init = function () {
            //...
        };

        return HelloModel;
    });
```
执行结果

```
Lifecycle : activate : hello
Lifecycle : binding : hello 
Lifecycle : bindingComplete : hello
Lifecycle : attached : hello
custom BindingHandler:hello-index
Lifecycle : compositionComplete : hello
```

## 准备
system.js - 工具类
### system.acquire
通过require.js获取资源(js\html\css)
```
system.acquire(settings.model).then(function(module) {
      ...
    }).fail(function(err) {
      ...
    });
```
### system.defer
基于jQuery的Deferred对象<br/>
返回一个promise对象，处理异步问题
```
system.defer(function(dfd){
    ...
    dfd.resolve();
});
```
## 生命周期
### setRoot入口

```
//...
function finishComposition() {
    //...
    if (settings.model.canActivate) { // canActivate
        var result = settings.model.canActivate();
        if (result && result.then) {
            result.then(function (actualResult) {
                if (actualResult) {
                    composition.compose(hostElement, settings); //关键
                }
            }).fail(function (err) {
                ///...
            });
        }
        // ...
    }
}

//settings.model：model对象的路径，在本例中为hello/index
system.acquire(settings.model).then(function (module) { // module:HelloModel
    settings.model = system.resolveObject(module);
    finishComposition();
}).fail(function (err) {
    // ...
});
```

### 整体过程
1. composition.compose
2. composition.inject
3. composition.executeStrategy：获取html内容<br/>
 3.1 composition.defaultStrategy
4. composition.bindAndShow<br/>
 4.1 binder.bind<br/>
 4.2 composition.finalize
```
composition = {
    //  ...
    compose: function (element, settings, bindingContext, fromBinding) {
        //添加 compositionComplete 回调
        compositionCompleteCallbacks.push(function () {
            if (settings.composingNewView && settings.model && settings.model.compositionComplete) {
                settings.model.compositionComplete(settings.child, settings.parent, settings);
            }
        });
        // 用于处理生命周期：attached/detached
        settings.triggerAttach = triggerAttach;
        //...
        composition.inject(settings, element); 
        //...
    },
    inject: function (context, element) {
        //...
        context.strategy = this.defaultStrategy;
        this.executeStrategy(context, element);
        //...
    },
    defaultStrategy: function (context) {
        return viewLocator.locateViewForObject(context.model, context.area, context.viewElements);
    },
    executeStrategy: function (context, element) {
        context.strategy(context).then(function (child) { // context.strategy :defaultStrategy,这步用于返回index.html中的dom
            composition.bindAndShow(child, element, context);
        });
    },
    bindAndShow: function (child, element, context, skipActivation) {
        //...
        tryActivate(context, function () {
            //...
            var modelToBind = context.model || dummyModel;
            //...
            hide(child);//隐藏
            ko.virtualElements.prepend(context.parent, child);//挂载
            binder.bind(modelToBind, child);//绑定
            composition.finalize(context, element);
            // ...
        })
        //...
    },
    finalize: function (context, element) {
        //...
        removePreviousView(context);
        show(context.child);
        context.triggerAttach(context, element);
        endComposition(context, element);
        //...
    }
}
```

#### 生命周期-activate的入口：tryActivate
context.model.activate.apply：回调实际调用的地方
```
function tryActivate(context, successCallback, skipActivation, element) {
    //...
    if (system.isArray(context.activationData)) {
        result = context.model.activate.apply(context.model, context.activationData);
    } else {
        result = context.model.activate(context.activationData);
    }

    if (result && result.then) {
        result.then(successCallback, 
        function (reason) {// activate回调:dfd.reject()
            onError(context, reason, element);
            successCallback();
        });
    } else if (result || result === undefined) {
        successCallback();
    } else {// 显示返回false
        endComposition(context, element);
    }
    //...
}
```

composition.bindAndShow在正式绑定dom与model之前，先是执行tryActivate方法来调用HelloModel中的activate回调（支持返回promsie）<br/>
如果activate返回的是promise,当promis拒绝时 以及 activate不是promise即同步显示返回false时，会直接结束绑定

activate拒绝时，走拒绝回调，然后调用onError，然后endComposition
```
function onError(context, error, element) {
    try {
        if (context.onError) {
            try {
                context.onError(error, element);
            } catch (e) {
                system.error(e);
            }
        } else {
            system.error(error);
        }
    } finally {
        endComposition(context, element, true);
    }
}
```

#### 生命周期-binding和bindingComplete的入口：binder.bind
```
bind: function(obj, view) {
    return doBind(obj, view, obj, obj);
}
```

```
function doBind(obj, view, bindingTarget, data){ // obj:model:HelloModel的实例
    try {
        var instruction;
        //生命周期-binding
        if (obj && obj.binding) {
            instruction = obj.binding(view);
        }
    
        instruction = normalizeBindingInstruction(instruction);
        binder.binding(data, view, instruction);//空实现
    
        if(instruction.applyBindings){
            system.log('Binding', viewName, data);
            ko.applyBindings(bindingTarget, view);
        }else if(obj){
            ko.utils.domData.set(view, koBindingContextKey, { $data:obj });
        }
    
        binder.bindingComplete(data, view, instruction);//空实现
        
        //生命周期-bindingComplete
        if (obj && obj.bindingComplete) {
            obj.bindingComplete(view);
        }
    
        ko.utils.domData.set(view, bindingInstructionKey, instruction);
        return instruction;
    } catch (e) {//...}
}
```
> 注：上面代码中的两个空实现，有点像模板模式，通过obj.binding + binder.binding，可以避过默认绑定实现自定义绑定策略（默认采用ko.applyBindings(bindingTarget, view)方法进行绑定）

> 模板模式<br/>
> https://www.cnblogs.com/qq-361807535/p/6854191.html<br/>
> 定义一个操作中算法的骨架，而将一些步骤延迟到子类中，模板方法使得子类可以不改变算法的结构即可重定义该算法的某些特定步骤。<br/>
> 通俗点的理解就是 ：完成一件事情，有固定的数个步骤，但是每个步骤根据对象的不同，而实现细节不同；就可以在父类中定义一个完成该事情的总方法，按照完成事件需要的步骤去调用其每个步骤的实现方法。每个步骤的具体实现，由子类完成
> 


#### 生命周期-attached，compositionComplete的入口：composition.finalize
调用finalize之前的步骤是刚完成dom与model的绑定<br/>
绑定完成之后隐藏之前的页面，显示当前setRoot的页面<br/>
然后通过attached回调告诉开发者，页面的数据绑定以及显示都已经搞定了<br/>
```
finalize: function (context, element) {
    //...
    context.triggerAttach(context, element); // attached回调
    endComposition(context, element); // compositionComplete回调
    //...
}
```

##### attached回调
```
function triggerAttach(context, element) {
    //...
    //attached回调的调用
    if (context.model && context.model.attached) {
        if (context.composingNewView || context.alwaysTriggerAttach) {
            context.model.attached(context.child, context.parent, context);
        }
    }
    //因为当前页面刚完成渲染，是不会去调用detached回调，只是将detached回调添加到dom关联的domNodeDisposal回调列表中
    //当 当前的 dom 从页面被移除时，才会去调用这里的回调，也是‘组件’的最后一个生命周期
    if (context.composingNewView && context.model && context.model.detached) {
        ko.utils.domNodeDisposal.addDisposeCallback(context.child, function () {
            try{
                context.model.detached(context.child, context.parent, context);
            }catch(e2){
                onError(context, e2, element);
            }
        });
    }
}
```

##### compositionComplete回调
```
function endComposition(context, element, error) {
    compositionCount--;

    if(compositionCount === 0) {
        var callBacks = compositionCompleteCallbacks;
        compositionCompleteCallbacks = [];
        
        if (!error) {
            setTimeout(function () { // 有意思哦
                var i = callBacks.length;

                while (i--) {
                    try {
                        callBacks[i]();
                    } catch (e) {
                        onError(context, e, element);
                    }
                }
            }, 1);
        }
    }

    cleanUp(context);
}
```



#### 生命周期-detached
触发detached的前提
```
 data-bind="router: { transition:'entrance', cacheViews:false }"
```

假设现在setRoot另一个页面，那么当前的hello/index会从浏览器中被移除（前提是没有设置缓存，durandal提供了页面缓存机制）,被移除的代码如下
```
composition = {
    finalize: function (context, element) {
        ...
        removePreviousView(context);
        ...
    }
}
```


```
//composition.js
function removePreviousView(context){
    var children = ko.virtualElements.childNodes(context.parent), i, len;

    if(!system.isArray(children)){
        var arrayChildren = [];
        for(i = 0, len = children.length; i < len; i++){
            arrayChildren[i] = children[i];
        }
        children = arrayChildren;
    }

    for(i = 1,len = children.length; i < len; i++){
        ko.removeNode(children[i]); //清缓存、触发disposeCallbacks（从而调用detached回调）
    }
}
``` 

## 补充
###  composition.addBindingHandler
本质：增加自定义绑定处理器（扩展ko内置的处理器:click,value,text,foreach,...）
```
addBindingHandler:function(name, config, initOptionsFactory){
    var key,
        dataKey = 'composition-handler-' + name,
        handler;

    config = config || ko.bindingHandlers[name];
    initOptionsFactory = initOptionsFactory || function(){ return undefined;  };

    handler = ko.bindingHandlers[name] = {
         init: function(element, valueAccessor, allBindingsAccessor, viewModel, bindingContext) {
            //...
         },
         update: function(element, valueAccessor, allBindingsAccessor, viewModel, bindingContext) {
            //...
         }
    }
    //...
}
```

我预测的执行结果是，
```
Lifecycle : binding : hello 
custom BindingHandler:hello-index //（应该在这个位置啊？）
Lifecycle : bindingComplete : hello
```
也就是自定义的 hello-index 绑定处理器应该在这个同步过程中被执行
```
ko.applyBindings(bindingTarget, view);
```
事实上，在这个绑定过程中，确实执行了，handler.init方法，但是composition对我们扩展的 hello-index 绑定处理器 包装了一层，将这个逻辑移到了
compositionCompleteCallbacks
```
handler = ko.bindingHandlers[name] = {
     init: function(element, valueAccessor, allBindingsAccessor, viewModel, bindingContext) {
        //...
        //自定义的绑定处理器迁移到compositionCompleteCallbacks中
     },
     update: function(element, valueAccessor, allBindingsAccessor, viewModel, bindingContext) {
        //...
     }
}
```
这就解释了文章最开始的执行结果

```
Lifecycle : bindingComplete : hello
custom BindingHandler:hello-index
```
### 获取html
通常按照下面方式加载‘组件’，但是这种方式指定的是model的路径<br/>
然后会通过在model添加getView、viewUrl等方式显示提供html的路径来加载html文件<br/>
如果没有，才去model相同路径下去找同名html文件<br/>
```
 app.setRoot('hello/index')
```
##### 加载html文件入口
composition.js
```
defaultStrategy: function (context) {
    return viewLocator.locateViewForObject(context.model, context.area, context.viewElements);
},
```

viewLocator.js
```
locateViewForObject: function(obj, area, elementsToSearch) {
    var view;
    //...
    var id = system.getModuleId(obj); // 来自 onResourceLoad 回调
    if (id) {
        return this.locateView(this.convertModuleIdToViewId(id), area, elementsToSearch);
    }

    return this.locateView(this.determineFallbackViewId(obj), area, elementsToSearch);
},
```

system.js<br/>
这里监听了model文件的加载后的回调，比如在当前案例中，当HelloModell.js文件加载并执行完就会走这里的回调
```
// callback for require.js loader
if (typeof requirejs !== 'undefined') {
    requirejs.onResourceLoad = function(context, map, depArray) {
        system.setModuleId(context.defined[map.id], map.id); 
    };
}
```

##### requirejs模块加载完毕后的相关代码

```
Module.prototype = {
    //...
    check: function () {
        //...
        if (this.depCount < 1 && !this.defined) { // 依赖加载完后，才处理目标js
            //...
            if (this.map.isDefine && !this.ignore) {
                defined[id] = exports;
    
                if (req.onResourceLoad) {
                    req.onResourceLoad(context, this.map, this.depMaps);
                }
            }
            //...
            this.defined = true;
        }
        //...
    }
    //...
}
```


### ko移除dom相关代码 
```
ko.cleanNode = ko.utils.domNodeDisposal.cleanNode; // Shorthand name for convenience
ko.removeNode = ko.utils.domNodeDisposal.removeNode; // Shorthand name for convenience
```

```
ko.utils.domNodeDisposal = new (function () {
   removeNode : function(node) {
        ko.cleanNode(node);
        if (node.parentNode)
            node.parentNode.removeChild(node);
    }, 
}  
```

```
function cleanSingleNode(node) {
    // Run all the dispose callbacks
    var callbacks = getDisposeCallbacksCollection(node, false); //核心
    if (callbacks) {
        callbacks = callbacks.slice(0); // Clone, as the array may be modified during iteration (typically, callbacks will remove themselves)
        for (var i = 0; i < callbacks.length; i++)
            callbacks[i](node);
    }
    ...
}
```


## 总结
#### 执行过程
1. 加载model
2. 加载html：'text!xxx.html'拿到的是文本，并不是dom节点，之后通过jquery方式转为dom节点
3. 执行 activate 回调，确定是否需要进行绑定
4. 利用ko进行绑定dom,model<br/>
    4.1 执行 binding <br/>
    4.2 执行 ko.applyBindings()<br/>
    4.3 执行 bindingComplete <br/>
5. 执行 attached 回调
6. 执行compositionComplete
7. ‘组件’被替换：执行detached
