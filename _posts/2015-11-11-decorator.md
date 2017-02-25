---
layout: post
title:  "装饰模式：设计模式系列（十）"
date:   2015-11-11 00:00:00
categories: Pattern
---
&emsp;&emsp;**装饰模式（Decorator Pattern）**:是把一实体对象类抽象出来的父类或接口或协议由另一接口或协议封装起来，在不改变实际对象的方法与属性的情况下，扩展实体对象类的功能的设计模式，归类为结构型模式。

&emsp;&emsp;这一模式的解读分为以下几个方面:

* 被扩展的实现类有一抽象出来的接口或协议。
* 装饰抽象协议或接口属性包含被扩展的实现类对象。
* 装饰实现类遵循类抽象协议或接口后，可扩展相关功能的属性和方法。

&emsp;&emsp;使用装饰模式的思考过程如下：

1. 有一目标对象无法直接从内部结构扩展其功能，或扩展后会影响其在其它方面的功能。
2. 建立一个装饰用的接口或协议，包含目标对象的父类或接口或协议。
3. 再建立一遵循装饰用的接口或协议的实现类，并把需要扩展的功能添加进去。

&emsp;&emsp;明确一下结构，装饰模式结构包含被装饰的实现类、被装饰的实现类的父类或接口或协议、装饰接口或协议、装饰实现类。

装饰模式示例代码如下（Swift语言）：

	protocol ObjectBluePrint{
	    
	    func show();
	}

	class Object:ObjectBluePrint{
	    
	    func show(){
	        print("I am Object");
	    }
	}

	protocol DecoratorBluePrint:ObjectBluePrint{
	    
	    var object:ObjectBluePrint{get set};
	    init(object:ObjectBluePrint);
	    func tell();
	    func show();
	}

	class Decorator:DecoratorBluePrint{
	    
	    var object:ObjectBluePrint;
	    required init(object:ObjectBluePrint){
	        self.object = object;
	    }
	    
	    func tell() {
	        print("I am Decorator");
	    }
	    func show(){
	        
	        self.object.show();
	    }
	}

	var object:ObjectBluePrint = Object();
	var decorator:DecoratorBluePrint = Decorator(object:object);

	object.show();
	decorator.show();
	decorator.tell();

&emsp;&emsp;**代码分析**:`ObjectBluePrint`就是被装饰对象实现类`Object`的抽象出来的协议，就只具有`show`一个方法，现在不破坏它的继承遵循关系，创建一个遵循`ObjectBluePrint`协议的装饰协议`DecoratorBluePrint`，协议除了必须有`show`方法外，还需要有`ObjectBluePrint`的变量、构造器和扩展了的方法（这里是`tell`），在装饰实现类`Decorator`中，`show`方法就可以通过自身初始化了的`object`属性调用没有修改的`Object`类的`show`方法，同样的可以通过`Decorator`类实例化对象调用扩展方法`tell`，但调用方式却是一样的。一下是上面代码的输出结果：

	I am Object
	I am Object
	I am Decorator

&emsp;&emsp;**总结**:装饰模式就是在不破坏原目标类的结构进行功能扩展的结构模式，在不希望影响目标类的对象功能的时候这么做是非常有效的解决方法，同样的，在无法修改的代码中也是。但却不利于频繁使用，由于此这几模式会大量增加类，松散虽然灵活，但是容易出错。
