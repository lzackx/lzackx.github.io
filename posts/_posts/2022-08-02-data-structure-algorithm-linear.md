---
layout: post
title:  "数据结构与算法(1.线性结构)"
date:   2022-08-02 08:00:00
categories: Algorithm
---
**数据结构与算法(2.线性结构)**

- [0. 前言](#0-前言)
- [1. 线性表](#1-线性表)
- [2. 链表](#2-链表)
  - [2.1 单向链表](#21-单向链表)
  - [2.2 单向循环链表](#22-单向循环链表)
  - [2.3 双向链表](#23-双向链表)
  - [2.4 双向循环链表](#24-双向循环链表)
- [3. 栈](#3-栈)
  - [3.1 线性栈](#31-线性栈)
  - [3.2 链式栈](#32-链式栈)
- [4. 队列](#4-队列)
  - [4.1 线性队列](#41-线性队列)
  - [4.2 链式队列](#42-链式队列)
- [5. 总结](#5-总结)
  - [5.1 线性表](#51-线性表)
  - [5.2 链表](#52-链表)
    - [5.2.1 单向链表](#521-单向链表)
    - [5.2.2 单向循环链表](#522-单向循环链表)
    - [5.2.3 双向链表](#523-双向链表)
    - [5.2.4 双向循环链表](#524-双向循环链表)
  - [5.3 栈](#53-栈)
    - [5.3.1 线性栈](#531-线性栈)
    - [5.3.2 链式栈](#532-链式栈)
  - [5.4 队列](#54-队列)
    - [5.4.1 线性队列](#541-线性队列)
    - [5.4.2 链式队列](#542-链式队列)

# 0. 前言

线性结构在`存储结构`上, 有以下2种:

* `顺序存储`
* `链式存储`

而多种逻辑结构皆可使用这2种`存储结构`实现.

`顺序结构`, 是在内存中开辟连续的空间存放数据, 再对数据进行操作, 有以下特性:

1. 存储空间大小固定, 从初始化时就被决定好了.
2. `insert`操作可能需要腾挪其他数据的位置.
3. `delete`操作可能要腾挪其他数据的位置.
4. `get`操作可通`index`直接获得.

`链式存储`, 是在内存中离散地存放数据, 再对数据进行操作, 有以下特性:

1. 存储空间大小不固定, 可灵活调整数据数量.
2. `insert`操作需要目标节点的`previous`和`next`一起操作指向.
3. `delete`操作需要目标节点的`previous`和`next`一起操作指向.
4. `get`操作需要通过遍历获取.

# 1. 线性表

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
7. 基于存储结构的`reverse`是通过首尾元素互换来实现的.

图解:
![](/assets/images/2022-08-01-data-structure-algorithm-definition-linear-list.png)

# 2. 链表

* 存储结构: 链式存储.

傻瓜式快速实现一个链表时, 会用到的一些归纳总结:
1. 有效节点的逻辑索引`index`总是从`0`开始.
2. 总是创建`Header`节点来**标记**首节点, 注意: 仅作标记作用.
3. 重点处理`get`和`length`方法, 能让逻辑更加清晰, 代码更加简单可读.
4. `insert`和`delete`操作可以总是通过`index - 1`的节点开始处理.
5. 基于链式结构的`reverse`是通过首插遍历法来实现的.

## 2.1 单向链表

**例子: [SinglyLinkedList](https://github.com/lzackx/Zone/tree/master/Demo/DataStructureAlgorithm/SinglyLinkedList)**

归纳总结单向链表: 离散排序的数据对象, 如按权重排序的数据表.

1. 可创建标记节点`Header`, `Header`的数据可以被额外利用为记录链表长度, 可以让获取链表长度的实现时间复杂度更少, 仅作标记用.
2. 出于健壮度的考虑, 实现一个链表的时候, 可以先实现`length`和`empty`方法. 为后面实现节点操作方法提供防止越界的判断方便.
3. 添加节点的方式有2种:
   1. 头插法, 新节点在`Header`节点后插入添加.
   2. 尾插法, 新节点在链表末尾插入, 通常这种方式更符合使用习惯.
4. `get`方法逐个遍历节点, 可被`insert`, `delete`, `traverse`方法方便使用, 在`insert`的时候, 由于需要处理`index=0`和`index - 1`的情况, 所以准入条件需要额外处理`header`.
5. `insert`, `delete`的规则是找到`index-1`位置再进行操作.
6. 删减方法的实现要注意语言的内存管理问题, 可能需要手动释放内存(如C、C++).

图解:
![](/assets/images/2022-08-01-data-structure-algorithm-definition-linear-singly-linked-list.png)

## 2.2 单向循环链表

**例子: [SinglyCircularLinkedList](https://github.com/lzackx/Zone/tree/master/Demo/DataStructureAlgorithm/SinglyCircularLinkedList)**

使用场景: 扑克发牌流程.

归纳总结单项循环链表:

1. 因单向链表的操作都是从`index-1`位置开始操作`index`节点的, 所以单向循环列表需要特殊处理`index=0`的情况.
   1. `insert`, 要移动`this->header->next = insertNode`的指向.
   2. `delete`时, 要移动`this->header->next = this->header->next->next`的指向.
2. 在`index = length`时`insert`, 或`add`时, 与正常插入无区别.
3. 通过`length`来实现`traverse`或通过判断`node->next == this->header->next`来结束`traverse`循环.

图解:
![](/assets/images/2022-08-01-data-structure-algorithm-definition-linear-singly-circular-linked-list.png)

## 2.3 双向链表

**例子: [DoublyLinkedList](https://github.com/lzackx/Zone/tree/master/Demo/DataStructureAlgorithm/DoublyLinkedList)**

归纳总结双向链表:

1. 在单向链表的基础上节点添加了前驱节点的指针, 使得`insert`和`delete`时, 需要多处理`previouse`指针的指向.
2. 有`previous`和`next`的指针后, 节点的操作会变得更便捷.

图解:
![](/assets/images/2022-08-01-data-structure-algorithm-definition-linear-doubly-linked-list.png)

## 2.4 双向循环链表

**例子: [DoublyCircularLinkedList](https://github.com/lzackx/Zone/tree/master/Demo/DataStructureAlgorithm/DoublyCircularLinkedList)**

归纳总结双向循环链表:

1. 注意`index=0`时的`insert`和`delete`.

图解:
![](/assets/images/2022-08-01-data-structure-algorithm-definition-linear-doubly-circular-linked-list.png)

# 3. 栈

栈是一种操作受限的线性数据结构, 数据操作遵循先进后出的规则.

## 3.1 线性栈

**例子: [LinearStack](https://github.com/lzackx/Zone/tree/master/Demo/DataStructureAlgorithm/LinearStack)**

归纳总结线性栈:

1. 由于栈的特性是只能操作栈顶, 线性栈的数据操作无需挪移数据.
2. `traverse`操作是从栈顶往栈底的顺序进行.
3. `clear`操作只需要重置栈顶索引.
4. 初始化的`top`为-1, `top + 1`就是`length`

图解:
![](/assets/images/2022-08-01-data-structure-algorithm-definition-linear-stack.png)


## 3.2 链式栈

**例子: [LinkedStack](https://github.com/lzackx/Zone/tree/master/Demo/DataStructureAlgorithm/LinkedStack)**

归纳总结链式栈:

1. `push`操作其实就是头插法.
2. `pop`操作无需释放节点.

图解:
![](/assets/images/2022-08-01-data-structure-algorithm-definition-linear-linked-stack.png)

# 4. 队列

队列是一种操作受限的线性数据结构, 数据操作遵循先进先出的规则.

## 4.1 线性队列
 
**例子: [LinearQueue](https://github.com/lzackx/Zone/tree/master/Demo/DataStructureAlgorithm/LinearQueue)**

归纳总结线性队列:

1. 数据最大数量固定, 且需要操作`Header`和`Tailer`, 所以可将线性队列看作一个循环
   1. `Enqueue`时, `tailer = (tailer + 1) % capacity`自增
   2. `Dequeue`时, `header = (header + 1) % capacity`自减
   3. `Length`时, `(tailer - header + capacity) % capacity`计算
2. 数据为空时, `header == tailer`
3. 数据为满时, `(tailer + 1) % capacity == header`, 从最大数据量中空出一个位置, 用于差异化满和空状态
4. 重置时, `header = tailer = 0`

图解:
![](/assets/images/2022-08-01-data-structure-algorithm-definition-linear-queue.png)

## 4.2 链式队列

**例子: [LinkedQueue](https://github.com/lzackx/Zone/tree/master/Demo/DataStructureAlgorithm/LinkedQueue)**

归纳总结链式队列:

1. 无节点数量限制
2. 使用尾插法时, `header`指向为空, 即为空队列
3. 得益于`header`和`tailer`的指向, `Enqueue`和`Dequeue`只需要操作首末数据的指向

图解:
![](/assets/images/2022-08-01-data-structure-algorithm-definition-linked-queue.png)

# 5. 总结

线性逻辑结构的特点是: 数据是1对1关系的

之后按照存储结构, 条件限制, 使用场景等情况, 衍生出了以下的实现.

## 5.1 线性表

特点:

1. 存储结构连续
2. 数据最大长度已知
3. 可通过索引直接获取数据
4. 数据插入删除涉及位置调整
5. 数据操作需要找到 `index = n - 1` 的数据节点进行操作

## 5.2 链表

特点:

1. 存储结构离散
2. 数据最大长度可变
3. 数据需要遍历获取
4. 数据的插入删除等操作, 需要依赖数据遍历
5. 总是利用Header节点能更容易实现链表的遍历

### 5.2.1 单向链表

特点:

1. 拥有`5.2`的特点
2. 数据遍历只能从首至末遍历

### 5.2.2 单向循环链表

特点:

1. 拥有`5.2.1`的特点
2. 限制: 末数据节点指向首数据节点
   1. 尾插法时, 注意指向首数据节点
   2. 头插法时(一般不使用), 注意末数据节点的指向

### 5.2.3 双向链表

特点:

1. 拥有`5.2`的特点
2. 数据可以双向遍历, 相当于增加了数据查找的方式

### 5.2.4 双向循环链表

特点:

1. 拥有`5.2.3`的特点
2. 限制: 末数据节点与首数据节点相互指向
   1. 尾插法时, 注意指向首数据节点

## 5.3 栈

特点:

1. 先入后出
2. 限制: 只能操作栈顶数据

### 5.3.1 线性栈

特点:

1. 拥有`5.3`的特点
2. 线性存储结构
3. 限制: 需要维护栈顶指针, 栈空时栈顶索引为`< 0`
4. 限制: 栈的最大长度固定, 栈满不可入栈

### 5.3.2 链式栈

特点:

1. 拥有`5.3`的特点
2. 离散存储结构
3. 限制: 使用链表的头插法入栈(出栈)

## 5.4 队列

特点:

1. 先进先出
2. 限制: 数据处理按照有序方式执行

### 5.4.1 线性队列

特点:

1. 拥有`5.4`的特点
2. `假溢出`的情况, 需要让队尾预留一个节点的空间为空作为标记.
3. 存储空间大小固定.
4. 出入队列可以循环看待, 以`%`方式实现索引循环.

### 5.4.2 链式队列

特点:

1. 拥有`5.4`的特点
2. 存储空间大小无限制
3. `header`和`tailer`让`Enqueue`和`Dequeue`更便捷
