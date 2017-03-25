---
layout: post
title:  "filter、map、reduce"
date:   2017-03-25 09:00:00
categories: Data
---
** 介绍 **

&emsp;&emsp;对于数据，不同的语言都有异曲同工的处理方式，比较直观的就是使用控制流和循环语句，将有同一处理需求的数据按照一定的规则进行处理，从而得到一定规则的数据。但这对于常常需要进行数据操作的人来说，一些常用的数据处理方式往往需要重复使用，于是便出现了像`filter`、`map`、`reduce`这种灵活通用的数据处理方法，并且可以发现，这三个处理，在多种语言中都有提供，如Swift、Python、Javascript等等。

&emsp;&emsp;那么，这里将以Swift为例阐述`filter`、`map`、`reduce`这三种处理。

** filter **

&emsp;&emsp;从名字来看，`filter`就是过滤的意思。定义如下：

```
public func filter(_ isIncluded: (Element) throws -> Bool) rethrows -> [Element]
```

&emsp;&emsp;可以看到，参数是一个`closure`，其返回值为Bool类型，实际上很好理解，当一个`Array`对象调用`filter`方法后，需要传入一个过滤条件，`Array`对象内的`Element`符合这个过滤条件（即返回值为`true`）时，那么这个`Element`就通过了过滤，被放到`[Element]`这个返回值中，重复遍历完整个`Array`对象后，组成了完整的`[Element]`返回值对象，此时，`filter`执行完毕。

举个栗子，如下：

```
let array = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

// filter1 = [5, 6, 7, 8, 9]
let filter1 = array.filter { (data) -> Bool in
    if data > 4 {
        return true
    } else {
        return false
    }
}

// filter2 = [5, 6, 7, 8, 9]
let filter2 = array.filter{$0 > 4}

// filter3 = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
let filter3 = array.filter{_ in true}

```

&emsp;&emsp;从栗子中的`filter1`中，基本已经可以看出`filter`处理的方式了，而`filter2`、`filter3`则是非常方便的`closure`表达式简写了过滤条件。

** map **

&emsp;&emsp;从名字来看，`map`就是映射的意思。定义如下：

```
public func map<T>(_ transform: (Element) throws -> T) rethrows -> [T]
```

&emsp;&emsp;可以看到，参数也是一个`closure`，其返回值是一个泛型，可以自定义，要理解也不难，这种处理方式，就是把`Array`对象内的`Element`按照一定规则映射成另一个`[T]`类型的对象。

举个栗子，如下：

```
let array = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

//	map1 = ["0", "1", "2", "3", "4", "5", "6", "7", "8", "9"]
let map1 = array.map { (data) -> String in
    return "\(data)"
}

//	map2 = ["0", "1", "2", "3", "4", "5", "6", "7", "8", "9"]
let map2 = array.map{"\($0)"}

```

&emsp;&emsp;从栗子中的`map1`中，可以看出`map`处理的方式，`closure`中的返回值被定义为`String`了，并在`closure`进行了`Element`类型的转换，所以在这里可以理解为把数字类型的数据转换成字符类型的数据。

** reduce **

&emsp;&emsp;单从名字来看，这个处理方法有点难理解，`reduce`其实是把`Array`内的`Element`个数减少、降低的意思。定义如下：

```
public func reduce<Result>(_ initialResult: Result, _ nextPartialResult: (Result, Element) throws -> Result) rethrows -> Result
```

&emsp;&emsp;可以看到，参数有2个，并且这里的`Result`也是泛型，用来指定初始化和最终返回值的类型，而对于返回值，更可以看出返回值不一定是`Array`，而第二个参数也是一个`closure`，所做的处理便是以`Result`为返回值容器，把`Element`按照一定规则处理后放到`Result`容器中，返回`Result`值为`reduce`的返回值。

举个栗子，如下：

```
let array = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

//	reduce1 = "0123456789"
let reduce1 = array.reduce(String()) { (result: String, data: Int) -> String in
    return result + "\(data)"
}

//	reduce2 = "0123456789"
let reduce2 = array.reduce(String()) {$0 + "\($1)"}

```

&emsp;&emsp;从栗子中的`reduce1`中，可以想象`reduce`的处理方式，`String()`把`Result`泛型定义为`String`类型，之后`Element`定义为名为`data`的`Int`参数，在`closure`中通过反复转换类型并把其值传入闭包返回值`Result`中，最终的返回值就是最终的`Result`返回值。

** 总结 **

&emsp;&emsp;通过上面的几个栗子，应能明白这三种数据处理方式，而对于这些处理数据的返回值，亦可互相组合使用，从而产生更灵活的处理，在一些语言中，也是有更高级的处理方式，例如Swift，还会有类似如下的用法：

```
public func flatMap<SegmentOfResult : Sequence>(_ transform: (Element) throws -> SegmentOfResult) rethrows -> [SegmentOfResult.Iterator.Element]
public func flatMap<ElementOfResult>(_ transform: (Element) throws -> ElementOfResult?) rethrows -> [ElementOfResult]
```

&emsp;&emsp;灵活处理数据，可以有效地提升开发效率，而这三种处理方式也是多语言支持的，所以不妨放下数不尽的相同的循环语句，尝试使用这些处理方式。但当然，不能纯粹为了使用而使用，好好思考使用条件与处理机制，处理后的数据才会成为真正需要的。







