---
layout: post
title:  "单例模式：设计模式系列（一）"
date:   2015-11-02 00:00:00
categories: Pattern
---
&emsp;&emsp;**单例模式（Singleton Pattern）**是让一个类确保只有一个实例，并能向全局提供这个实例的访问方法的设计模式，归类为创建型模式。

&emsp;&emsp;这一设计模式作为解读，使用时要注意几个方面：

* 一个类只能实例一个对象。（体现了SRP）
* 实例对象是被代码自行创建的。
* 全局代码都能方便地访问调用这个实例对象。

&emsp;&emsp;单例模式适用情景有以下几个，置身于这些情景时，几乎就能把握使用此模式的时机。

* 需要经常使用，而频繁创建和销毁会消耗较多时间或资源的对象的时候。
* 需要保留状态供全局访问的对象的时候。
* 需要实例惟一一个对象，或因为安全，或由于冲突的时候。

单例模式示例代码如下（Swift语言）：

    class Singleton{
    
        private static let singleton = Singleton();
        
        private init(){}
    
        static func getInstance()->Singleton{
            return singleton;
        }
    }


&emsp;&emsp;**代码分析**：这是一个线程安全的单例模式定义，`singleton`是类型属性(Type Properties)，`getIntance()`是类型方法(Type Methods),全局可以通过调用`Singleton.getInstance()`来获取单例实例对象（假如代码是以库的形式存在，调用时可能需要在其前面添加`public`关键字），但要注意的是`init()`这个类构造函数需要显式为`private`，因为swift默认类中的方法都是internal访问级别，而为了单例模式中“惟一”这个定义，一旦构造函数被私有化了，那么就可以防止其它地方创建，而只能通过`getIntance()`类型方法获取。

&emsp;&emsp;**总结**:单例模式在适当的情景中能很好地提供服务，被访问的方式也很简便，但也有缺点，有时候因为会因为把太多的逻辑放在单例模式中，以至于违反SRP，成为一个臃肿的存在，所以在使用之前，需要好好权衡斟酌。
