---
layout: post
title:  "享元模式：设计模式系列（十四）"
date:   2015-11-15 00:00:00
categories: Pattern
---
&emsp;&emsp;**享元模式（Flyweight Pattern）**:是一种防止细粒度对象被大量复用的设计模式，归类为结构型模式。

&emsp;&emsp;这一模式的解读分为以下几个方面:

* 一些对象会被大量创建调用。
* 这些对象很相似甚至相同。

&emsp;&emsp;使用享元模式的思考过程如下：

1. 有一种本质相同的对象在很多情况下都需要被调用。
2. 创建和销毁对象的动作非常频繁。
3. 为了减少运行消耗，把这些对象保存在一个容器中并共享出来，减少创建和销毁的次数。

&emsp;&emsp;明确一下结构，享元模式结构包括相同本质对象的实现类、实现类抽象出来的接口或协议、创建与保存获取对象的工厂实现类。

享元模式示例代码如下（Swift语言）:

	protocol ObjectBluePrint{
	    
	    var state:String{get set};
	    func getState()->String;
	    func show();
	}

	class Object:ObjectBluePrint{
	    
	    var state:String;
	    
	    init(state:String){
	        self.state = state;
	    }
	    
	    func getState()->String{
	        return self.state;
	    }
	    
	    func show() {
	        print("state:\(state)");
	    }
	}

	class FlyweightFactory{
	    
	    private var object = [ObjectBluePrint]();
	    
	    func getObject(object:ObjectBluePrint)->ObjectBluePrint{
	        for(var i = 0;i < self.object.count;i++){
	            if(self.object[i].getState() == object.getState()){
	                return self.object[i];
	            }
	        }
	        self.object.append(object);
	        return self.object[self.object.endIndex-1];
	    }
	}

	var o1:ObjectBluePrint = Object(state: "object1");
	var o2:ObjectBluePrint = Object(state: "object2");

	var fwf = FlyweightFactory();

	var fw1 = fwf.getObject(o1);
	fw1.show();
	var fw2 = fwf.getObject(o2);
	fw2.show();
	var fw3 = fwf.getObject(Object(state: "object1"));
	fw3.show();

&emsp;&emsp;**代码分析**:从以上代码可以看出，`Object`实现类遵循了`ObjectBluePrint`协议，并且自身需要一不同本质的属性`state`，而`FlyweightFactory`类则是创建、保存、获取遵循`ObjectBluePrint`协议的实现类的类，通过调用`getObject`方法，使未创建的类创建并加入数组中保存，让已经被创建的`Object`对象在数组中被找到。从而减少了创建与销毁大量同一对象的目的。以下是上面代码的输出结果：

	state:object1
	state:object2
	state:object1

&emsp;&emsp;**总结**:享元模式能让大量相同或相似的对象被重复使用的时候不会被多次创建与销毁，并且能兼容不共享的属性，但也会暴露了不共享的属性，假如不共享的属性很多，会让代码的结构变得复杂。
