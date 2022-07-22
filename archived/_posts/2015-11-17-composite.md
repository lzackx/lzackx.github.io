---
layout: post
title:  "组合模式：设计模式系列（十六）"
date:   2015-11-17 00:00:00
categories: Pattern
---
&emsp;&emsp;**组合模式（Composite Pattern）**:是一种让实例对象关系按照树形图结构进行关联的设计模式，归类为结构型模式。

&emsp;&emsp;这一模式的解读分为以下几个方面:

* 实例对象本质相似或相同。
* 实例对象之间有同级或上下关系。

&emsp;&emsp;使用组合模式的思考过程如下：

1. 一个唯一的对象与很多相似的对象是父关系。
2. 除了唯一对象外的其它对象也有可能是其它相似对象的父关系对象。

&emsp;&emsp;明确一下结构，组合模式结构包含本质相同的实现类、实现类抽象出来的接口或协议。

组合模式示例代码如下（Swift语言）:

	protocol WidgetBluePrint{
	    
	    var father:WidgetBluePrint?{get set};
	    var children:[WidgetBluePrint]?{get set};
	    var id:Int{get set};
	    
	    func from()->WidgetBluePrint?;
	    func have()->[WidgetBluePrint]?;
	    
	    func belongTo(father:WidgetBluePrint);
	    func contain(child:WidgetBluePrint);
	    func doNotContain(child:WidgetBluePrint);
	    
	    func show();
	}

	class Widget:WidgetBluePrint{
	    
	    var father:WidgetBluePrint?;
	    var children:[WidgetBluePrint]?;
	    var id:Int;
	    
	    init(id:Int){
	        self.id = id;
	    }
	    
	    func from()->WidgetBluePrint?{
	        return self.father;
	    }
	    func have()->[WidgetBluePrint]?{
	        return self.children;
	    }
	    
	    func belongTo(father:WidgetBluePrint){
	        self.father = father;
	    }
	    func contain(child:WidgetBluePrint){
	        child.belongTo(self);
	        if(self.children == nil){
	            self.children = [WidgetBluePrint]();
	        }
	        self.children!.append(child);
	    }
	    func doNotContain(child:WidgetBluePrint){
	        for(var i = 0;i < self.children?.count;i++){
	            if(self.children![i].id == child.id){
	                self.children?.removeAtIndex(i);
	            }
	        }
	    }
	    
	    func show(){
	        if(self.father == nil){
	            print("I am \(self.id),I am ancestor,my Children are:");
	        }else{
	            print("I am \(self.id),my father is \(self.father?.id),my Children are:");
	        }
	        for(var i = 0;i < self.children?.count;i++){
	            print("\(self.children![i].id)");
	        }
	    }
	}

	var object0:WidgetBluePrint = Widget(id: 0);

	var objectA1:WidgetBluePrint = Widget(id: 1);
	var objectA2:WidgetBluePrint = Widget(id: 2);
	var objectA3:WidgetBluePrint = Widget(id: 3);

	var objectB1:WidgetBluePrint = Widget(id: 4);
	var objectB2:WidgetBluePrint = Widget(id: 5);

	object0.contain(objectA1);
	object0.contain(objectA2);
	object0.contain(objectA3);

	objectA1.contain(objectB1);
	objectA2.contain(objectB2);

	object0.show();
	objectA1.show();
	objectA2.show();
	objectA3.show();
	objectB1.show();
	objectB2.show();

&emsp;&emsp;**代码分析**:可以看到，此模式结构中只有一个遵循类`WidgetBluePrint`协议的实现类`Widget`,类中有属性`father`和`children`分别被方法`belongTo`和`contain`设置，这样以上下级的关系从唯一的对象以树形结构扩展开来，形成类对象与对象之间的关系。以下是上面代码的输出结果：

	I am 0,I am ancestor,my Children are:
	1
	2
	3
	I am 1,my father is Optional(0),my Children are:
	4
	I am 2,my father is Optional(0),my Children are:
	5
	I am 3,my father is Optional(0),my Children are:
	I am 4,my father is Optional(1),my Children are:
	I am 5,my father is Optional(2),my Children are:

&emsp;&emsp;**总结**:组合模式是一种扩展性很强的设计模式，其中对象与对象之间的关系常常与现实中的关系相对应，但是当组合起来的对象越来越多的时候，结构就会变得越来越复杂，想要从祖先开始搜索子孙对象也会变得越来越复杂。但其层次结构对于整体来说却很清晰。

