---
layout: post
title:  "数据结构与算法(1.线性结构)"
date:   2022-08-02 08:00:00
categories: Algorithm
---
**数据结构与算法(2.线性结构)**

- [1. 线性表](#1-线性表)
- [2. 链表](#2-链表)
  - [2.1 单向链表](#21-单向链表)
  - [2.2 单项循环链表](#22-单项循环链表)
  - [2.3 双向链表](#23-双向链表)
  - [2.4 双向循环链表](#24-双向循环链表)
- [3. 栈](#3-栈)
- [4. 队列](#4-队列)
- [5. 字符串](#5-字符串)

# 1. 线性表

* 逻辑结构: 1对1, 有序.

* 存储结构: 连续存储.

**例子: [LinearList](https://github.com/lzackx/Zone/tree/master/Demo/DataStructureAlgorithm/LinearList)**

一个容易实现的线性表:

1. 

# 2. 链表

* 逻辑结构: 1对1, 有序.

* 存储结构: 链式存储.

## 2.1 单向链表

**例子: [SinglyLinkedList](https://github.com/lzackx/Zone/tree/master/Demo/DataStructureAlgorithm/SinglyLinkedList)**

一个容易实现的单向链表的特性:

1. 创建首节点`Header`, 作为起始点, `Header`的数据可以被额外利用为记录链表长度, 可以让获取链表长度的实现时间复杂度更少.
2. 出于健壮度的考虑, 实现一个链表的时候, 可以先实现`length`和`empty`方法. 为后面实现节点操作方法提供防止越界的判断方便.
3. 添加节点的方式有2种
   1. 头插法, 新节点在`Header`节点后插入添加
   2. 尾插法, 新节点在链表末尾插入
4. `get`, `insert`, `delete`, `traverse`等需要遍历的方法, 可以记作 `index == length`, `Header`的`index`是`0`.
5. 删减方法的实现要注意语言的内存管理问题, 可能需要手动释放内存(如C、C++).

图解:
![](/assets/images/2022-08-01-data-structure-algorithm-definition-linear-singly-linked-list.png)

## 2.2 单项循环链表


## 2.3 双向链表


## 2.4 双向循环链表



# 3. 栈

# 4. 队列

# 5. 字符串