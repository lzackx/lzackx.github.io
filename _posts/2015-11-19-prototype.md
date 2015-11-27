---
layout: post
title:  "原型模式：设计模式系列（十八）"
date:   2015-11-19 00:00:00
categories: pattern
---
&emsp;&emsp;**原型模式（Prototype Pattern）**:是指定实现类原型，拷贝自身来创建实例对象的设计模式，归类为创建型模式。

&emsp;&emsp;这一模式的解读分为以下几个方面:

* 需要一实现类作为原型。
* 此实现类有拷贝自身的方法。

&emsp;&emsp;使用原型模式的思考过程如下：

1. 需要新建一个跟已有对象相似或相同的对象（例如某种由外部条件确定了状态的对象）。
2. 调用实现类中的自我拷贝方法。

&emsp;&emsp;明确一下结构，原型模式结构包含原型实现类。

原型模式示例代码如下（Swift语言）:

	class Prototype{
	    
	    var state:String;
	    
	    init(state:String){
	        self.state = state;
	    }
	    
	    func clone()->Prototype{
	        return Prototype(state: self.state);
	    }
	    
	    func setState(state:String){
	        self.state = state;
	    }
	}

	var object1 = Prototype(state: "object 1");
	print("\(object1.state)");

	var object2 = object1.clone();
	object2.setState("object 2");
	print("\(object1.state)");
	print("\(object2.state)");


&emsp;&emsp;**代码分析**:以上代码中只有一个实现类，很容易理解，在创建`Prototype`类对象的时候，调用`clone`方法，就能创建并返回一个跟现对象一样的新对象。以上代码的输出结果如下：

	object 1
	object 1
	object 2

&emsp;&emsp;**总结**:原型模式是一个看起来非常简单的设计模式，其主要动作是通过拷贝来创建一个自身类型与属性的对象，但这只是表面，本质是指定对象的原型并创建对象。在使用此模式的时候应该注意别为了模式而使用模式（这样做没有意义）。