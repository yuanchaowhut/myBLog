<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [继承的方式](#%E7%BB%A7%E6%89%BF%E7%9A%84%E6%96%B9%E5%BC%8F)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


# 继承的方式
1. 原型链实现继承
    子类原型指定为父类实例（因此会执行一次父构造函数并多出父类实例属性）
2. 借用构造函数
3. 组合继承（伪经典继承）<br/>
    原型【链】 + 借用构造函数<br/>
    缺点：调用两次超类构造函数<br/>
4. 原型式继承<br/>
    Object.create()<br/>
    借助原型可以基于已有对象创建新对象，同时还不需要指定类型<br/>
    
    原型式继承相较于原型链继承的优势？<br/> 
    区别于原型链继承：原型链继承的是父类的实例（会执行一次父构造函数），但是原型式继承是直接继承已有对象甚至是父类原型，而不需要执行父类构造函数<br/><br/> 
5. 寄生式继承<br/> 
    在原型式继承的继承上封装增强对象的过程<br/> 
6. 寄生组合继承<br/> 
    寄生式继承 + 借用构造函数 + 原型式继承<br/> 
    ```
    function inheritPortotype(SubType,SuperType){
        //原型式继承
        var prototype = Object.create(SuperType.prototype);
        //寄生式继承
        SubType.constructor = SubType;
        SubType.prototype = prototype;
    }   
    
    //借用构造函数
    var SubType = function(name,age){
        SuperType.call(this,name);
        this.age = age;
    }
    
    ```
    