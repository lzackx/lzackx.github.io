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

归纳总结线性表:

1. 连续存储的存储结构使得部分高级语言无法手动指定实现
   1. 例如`Python`, 没有指针操作的存在, 导致无法手动开辟连续存储的内存空间
   2. `C`, `C++`, `Objective-C`, `Swift`等, 都能通过指针操作, 手动开辟连续的内存空间.
2. 开辟连续的存储空间, 有3个`C`/`C++`比较重要的开辟内存空间的函数, 理解他们可以让开辟内存空间的思路跟清晰.
   1. `malloc`,  仅仅开辟内存空间, 需要配合`memset`函数初始化内存空间数据.
   2. `calloc`, 开辟内存空间并初始化内存.
   3. `realloc`, 重新开辟内存空间, 并把原内存空间的内容拷贝到新的内存空间中, 并释放原内存空间.
3. 出于健壮性与语言特性的考虑, 表的空间大小需要进行管理与判断. 超出最大内存空间时, 可以通过重新开辟内存空间存放数据的方式来优化表, 即`mutable`.
4. 通过数据类型进行内存对齐后, `get`可以通过下标方式实现.
5. `insert`与`delete`需要遍历腾挪的内存空间
   1. `insert`由于需要往后挪位置, 所以需要从末尾开始遍历.
   2. `delete`由于需要往前挪位置, 所以可以从目标位置开始遍历.
6. `clear`只需要将`length`重置为0, 在开辟过的内存空间中的数据会被认为是无效数据.

图解:
![](/assets/images/2022-08-01-data-structure-algorithm-definition-linear-list.png)

# 2. 链表

* 逻辑结构: 1对1, 有序.

* 存储结构: 链式存储.

## 2.1 单向链表

**例子: [SinglyLinkedList](https://github.com/lzackx/Zone/tree/master/Demo/DataStructureAlgorithm/SinglyLinkedList)**

归纳总结单向链表:

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

**例子: [SinglyCircularLinkedList](https://github.com/lzackx/Zone/tree/master/Demo/DataStructureAlgorithm/SinglyCircularLinkedList)**

归纳总结单项循环链表:

## 2.3 双向链表

**例子: [DoublyLinkedList](https://github.com/lzackx/Zone/tree/master/Demo/DataStructureAlgorithm/DoublyLinkedList)**

归纳总结双向链表:

## 2.4 双向循环链表

**例子: [DoublyCircularLinkedList](https://github.com/lzackx/Zone/tree/master/Demo/DataStructureAlgorithm/DoublyCircularLinkedList)**

归纳总结双向循环链表:


# 3. 栈

# 4. 队列

# 5. 字符串