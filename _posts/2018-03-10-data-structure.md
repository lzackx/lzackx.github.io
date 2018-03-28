---
layout: post
title:  "数据结构（持续更新）"
date:   2018-03-10 09:00:00
categories: DataStructure 
---
**数据结构（Data Structure）**

&emsp;&emsp;开发中，常见有线性、树、图等数据结构，本文将结合代码着重描述线性表的数据结构。

* 平台：iOS
* 语言：Objective-C
* 源码地址：[源码](https://github.com/lzackx/Zone/tree/master/iOS/DataStructure)

# 1. 线性表

&emsp;&emsp;线性表在逻辑上，是有序的，例如数组、链表等。

&emsp;&emsp;即便逻辑上线性表的数据结构是有序的，但在存储上，却不一定是连续的，所以在实现细节上，略有不同。

### 1.1 代码实现

&emsp;&emsp;以下代码为数组的实现：

##### 1.1.1 数组

&emsp;&emsp;数组是一种逻辑上有序，存储上连续的线性表，在创建数组和增加元素时注意内存释放问题，在删改查时注意越界问题，总体上就没什么问题了。

```
//
//  array.h
//  LinearList
//
//  Created by lzackx on 2018/3/10.
//  Copyright © 2018年 lzackx. All rights reserved.
//

#ifndef array_h
#define array_h

#include <stdio.h>

/*
 .c源码中因经过array_private.h的封装，所以已经有array类型的声明，
 但在这头文件中，因为有相关函数的声明，所以相应地，在没有ARRAY宏定义的这个文件中，
 可以简单声明，使编译期间不报错，但其实在编译器链接时，起作用的还是私有头文件中的声明。
 */
#ifndef ARRAY
typedef void array;
typedef void * array_value;
#endif

array *array_create(int capacity);
void array_destory(array *a);
void array_clear(array *a);
int array_get_length(array *a);
array_value array_get_value(array *a, int index);
void array_insert(array *a, int index, array_value av);
void array_add(array *a, array_value av);
void array_set_value(array *a, int index, array_value av);
void array_remove(array *a, int index);
void array_remove_value(array *a, array_value av);
void array_print(array *a);

#endif /* array_h */
```

```
//
//  array_private.h
//  LinearList
//
//  Created by lzackx on 2018/3/10.
//  Copyright © 2018年 lzackx. All rights reserved.
//

/*
 这是一个私有头文件，目的是为了封装结构体内的变量，使外部调用时，无法直接访问修改。
 #define ARRAY
 这个宏是重点。
 */

#ifndef array_private_h
#define array_private_h

typedef void * array_value;

typedef struct {
    
    int capacity;
    int length;
    array_value *value;
    
} array;
#define ARRAY

#endif /* array_private_h */
```

```
//
//  array.c
//  LinearList
//
//  Created by lzackx on 2018/3/10.
//  Copyright © 2018年 lzackx. All rights reserved.
//

#include "array_private.h"
#include "array.h"
#include <stdlib.h>
#include "string.h"

// 创建
array *array_create(int capacity) {
    
    if (capacity < 0) {
        return NULL;
    }
    array *a = malloc(sizeof(array));
    if (a) {
        a->capacity = capacity;
        a->length = 0;
        a->value = capacity ? malloc(sizeof(array_value) * capacity) : NULL;
    }
    return a;
}

/* 销毁
 销毁过程需要注意对应创建过程，由于创建过程中，a->value与a的开辟内存堆空间是分开的，
 所以在销毁过程，要分步释放内存堆空间，防止内存泄漏。
 */
void array_destory(array *a) {
    
    if (a == NULL) {
        return;
    }
    free(a->value);
    free(a);
}

/* 清除
 清除过程并不是把内存中的数据擦除，而是让内存知道旧数据不可用就可以了。
 */
void array_clear(array *a) {
    
    if (a == NULL) {
        return;
    }
    a->length = 0;
}

// 获取长度
int array_get_length(array *a) {
    
    if (a == NULL) {
        return 0;
    }
    return a->length;
}

// 获取值：注意别越界
array_value array_get_value(array *a, int index) {
    
    if (a == NULL || index < 0 || index >= a->length) {
        return NULL;
    }
    // 皆为指针的使用 等同于 *((a->value)+index)
    return a->value[index];
}

/* 插入
 当数组可扩容时，步骤如下：
 1. 当堆空间容量不够了，就重新malloc一块新空间，该空间容量按不同的规则而不同，这里示范2倍空间。
 2. 复制旧数据到新数堆空间
 3. 将旧空间释放掉，不需要做擦除
 */
void array_insert(array *a, int index, array_value av) {
    
    if (a == NULL || index < 0 || index > a->length) {
        return;
    }
    // 数组空间满时
    if (a->capacity == a->length) {
        int newCapacity = a->capacity * 2; //2倍旧空间大小，就是扩大一倍空间
        array_value newValue = malloc(sizeof(array_value) * newCapacity);
        if (newValue == NULL) {
            return; // 堆空间分配失败，就只能返回了
        }
        /*
         直接使用内存复制比较方便(不然就得for循环逐条复制了)，
         意思是将旧内存 a->value 的 sizeof(array_value) * a->length 那么多字节，
         复制到 newValue 内存里
         */
        memcpy(newValue, a->value, sizeof(array_value) * a->length);
        // 最后就是释放旧的，保留新的，并更新新容量
        free(a->value);
        a->value = newValue;
        a->capacity = newCapacity;
    }
    
    for (int i = a->length;i > index;i--) {
        a->value[i + 1] = a->value[i];
    }
    a->value[index] = av;
    a->length += 1;
}

// 增加：等同于末尾插入
void array_add(array *a, array_value av) {
    
    if (a == NULL) {
        return;
    }
    
    array_insert(a, a->length, av);
}

// 修改
void array_set_value(array *a, int index, array_value av) {
    if (a == NULL) {
        return;
    }
    a->value[index] = av;
}

// 删除
void array_remove(array *a, int index) {
    
    if (a == NULL || a->length <= index || index < 0) {
        return;
    }
    
    for (int i = index;i < a->length; i++) {
        a->value[i] = a->value[i + 1];
    }
    a->length -= 1;
}

// 删除
void array_remove_value(array *a, array_value av) {
    
    if (a == NULL) {
        return;
    }
    int count = 0;
    for (int i = 0; i < a->length; i++) {
        if (a->value[i] == av) {
            count++;
        } else {
            a->value[i - count] = a->value[i];
        }
    }
    a->length -= count;
}

// 打印
void array_print(array *a) {
    
    printf("array\n");
    printf("{\n");
    printf("\tcapacity: %d\n", a->capacity);
    printf("\tlength: %d\n", a->length);
    printf("\tvalue: [");
    for (int i = 0; i < a->length; i++) {
        printf("%p", *(a->value));
        if (i < a->length - 1) {
            printf(",");
        }
    }
    printf("]\n}\n");
}
```









