---
layout: post
title:  "解释器模式：设计模式系列（二十二）"
date:   2015-11-23 00:00:00
categories: Pattern
---
&emsp;&emsp;**解释器模式（Interpreter Pattern）**:是一种用来对自定义语言语法进行解释的设计模式，归类为行为型模式。

&emsp;&emsp;这一模式的解读分为以下几个方面:

* 语言语法可自定义。
* 解释器能对语言进行解释。

&emsp;&emsp;使用解释器模式的思考过程如下：

1. 一连串的对象通过有规则的组合能出线有意义的的存在。
2. 这些对象和规则通过进行类形式的封装，在实例化对象后能组合在一起。
3. 解释器类实例对象通过递归或循环方式解释组合在一起的对象，并返回有意义的结果。

&emsp;&emsp;明确一下结构，解释器模式结构包含待解释的上下文实现类、解释器抽象出来的接口或协议、组合规则实现类、组合对象实现类。

解释器模式示例代码如下（Swift语言）:

	class Context{
	    
	    var objects = [Interpreter]();
	    
	    func addObject(object:Interpreter){
	        self.objects.append(object);
	    }
	}

	protocol Interpreter{
	    
	    func interpret(context:Context)->String;
	}

	class Object:Interpreter{
	    
	    var object:String;
	    
	    init(object:String){
	        self.object = object;
	    }
	    
	    func interpret(context:Context)->String{
	        return self.object;
	    }
	}

	class Rule:Interpreter{
	    
	    var object1:Interpreter;
	    var object2:Interpreter;
	    
	    init(object1:Interpreter,object2:Interpreter){
	        self.object1 = object1;
	        self.object2 = object2;
	    }
	    
	    func interpret(context:Context)->String{
	        return (self.object1.interpret(context) + " means " + self.object2.interpret(context) + " in this new language.");
	    }
	}

	var context = Context();
	var obj1 = Object(object: "Z");
	var obj2 = Object(object: "Will");

	context.addObject(obj1);
	context.addObject(obj2);

	var rule = Rule(object1: context.objects[0], object2: context.objects[1]);
	print(rule.interpret(context));

&emsp;&emsp;**代码分析**:`Context`是保存上下文的实现类，这里的`objects`属性其实是把遵循`Interpreter`协议的类实例化对象以有序方式存储在一起（其实也可以使用字典，即键值对），而遵循了`Interpreter`协议的实现类必须实现`interpret`方法，`Object`实现类返回了自身的属性作为解释值，而`Rule`实现类则返回了两`Object`实例化对象与一些字符串组合起来的字符串作为解释值。最后可以看到以上代码的输出结果如下：

	Z means Will in this new language.

&emsp;&emsp;**总结**:解释器模式是比较少用的设计模式，究其原因是当需要解释的规则越来越多的时候，定义规则的实现类也会越来越多，逻辑变得越来越复杂，而且在三个及三个以上的组合对象解释上往往需要使用递归或循环，对运行性能有一定影响。