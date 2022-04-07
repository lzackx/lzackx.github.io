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

```objc
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

```objc
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

```objc
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

##### 1.1.2 链表

&emsp;&emsp;链表是一种逻辑上有序，存储上不连续的线性表，在创建数组和增加元素时除了需要注意内存释放问题，最关键的还是需要关注内存和指针的配合使用。

```objc
//
//  list.h
//  LinearList
//
//  Created by lzackx on 2018/3/10.
//  Copyright © 2018年 lzackx. All rights reserved.
//

#ifndef list_h
#define list_h

#include <stdio.h>
/*
 .c源码中因经过list_private.h的封装，所以已经有LIST类型的声明，
 但在这头文件中，因为有相关函数的声明，所以相应地，在没有宏定义的这个文件中，
 可以简单声明，使编译期间不报错，但其实在编译器链接时，起作用的还是私有头文件中的声明。
 */
#ifndef LIST
typedef void list;
typedef void list_node;
typedef void * list_node_value;
#endif

list* list_create(void);
list_node* list_node_create(list_node_value lnv);
void list_clear(list *l);
void list_destory(list *l);
int list_get_length(list *l);
list_node* list_node_get(list *l, int index);
list_node_value list_node_get_value(list_node *ln);
void list_node_value_insert(list *l, int index, list_node_value *nln);
void list_node_value_add(list *l, list_node_value *ln);
void list_node_set(list *l, int index, list_node *nln);
void list_node_set_value(list_node *ln, list_node_value lnv);
void list_node_remove(list *l, int index);
void list_node_value_remove(list *l, list_node_value lnv);
void list_print(list *l);

#endif /* list_h */
```

```objc
//
//  list_private.h
//  LinearList
//
//  Created by lzackx on 2018/3/28.
//  Copyright © 2018年 lzackx. All rights reserved.
//

/*
 这是一个私有头文件，目的是为了封装结构体内的变量，使外部调用时，无法直接访问修改。
 #define LIST
 这个宏是重点。
 */

#ifndef list_private_h
#define list_private_h

//typedef void * list_node;
typedef void * list_node_value;

typedef struct _list_node list_node;
struct _list_node {
    
    list_node_value value;
    list_node *next;
    
};

typedef struct {
    
    int length;
    list_node *header;
    
} list;


#define LIST

#endif /* list_private_h */
```

```objc
//
//  list.c
//  LinearList
//
//  Created by lzackx on 2018/3/10.
//  Copyright © 2018年 lzackx. All rights reserved.
//

#include "list_private.h"
#include "list.h"
#include <stdlib.h>
#include "string.h"

// 创建
list* list_create() {
    
    list *l = malloc(sizeof(list) + sizeof(list_node));
    if (l) {
        l->length = 0;
        l->header = (list_node *)(l + 1);
        /*
         这里有个比容易中但较难发现的坑，初始化时，需要把header内的next指向显式指定为NULL，
         如果没有这步，因为内存中可能会存在莫名其妙的值，
         会导致链表的最后一个node的指向认为这个值是个指针，最后访问错误导致crash。
         */
        l->header->next = NULL;
    }
    return l;
}

list_node* list_node_create(list_node_value lnv) {
    
    list_node *ln = malloc(sizeof(list_node));
    if (ln == NULL) {
        return NULL;
    }
    ln->next = NULL;
    ln->value = lnv;
    return ln;
}

// 清除
void list_clear(list *l) {
    
    if (l == NULL) {
        return;
    }
    list_node *lhn = l->header;
    while (lhn) {
        list_node *lnn = lhn->next;
        if (lnn == NULL) {
            break;
        }
        lhn->next = lnn->next;
        free(lnn);
    }
    l->length = 0;
}

// 销毁：链表中，因为存储结构的不连续，导致清除数据需要把每个node都拿出来free掉。
void list_destory(list *l) {
    
    if (l == NULL) {
        return;
    }
    list_clear(l);
    free(l);
    l = NULL;
}


// 获取长度
int list_get_length(list *l) {
    
    if (l == NULL) {
        return 0;
    }
    return l->length;
}

list_node* list_node_get(list *l, int index) {
    
    if (l == NULL) {
        return NULL;
    }
    list_node *ln = l->header;
    for (int i = 0; i < index; i++) {
        ln = ln->next;
    }
    
    return ln;
}

list_node_value list_node_get_value(list_node *ln) {
    
    if (ln == NULL) {
        return NULL;
    }
    return ln->value;
}

// 插入：注意指针操作
void list_node_value_insert(list *l, int index, list_node_value *lnv) {
    
    if (l == NULL || index < 0 || index > l->length) {
        return;
    }

    list_node *nln = malloc(sizeof(list_node));
    if (nln == NULL) {
        return;
    }
    nln->next = NULL;
    nln->value = lnv;
    
    list_node *ln = l->header;
    for (int i = 0;i < index;i++) {
        ln = ln->next;
    }
    nln->next = ln->next;
    ln->next = nln;
    l->length += 1;
}

// 增加：等同于末尾插入
void list_node_value_add(list *l, list_node_value *lnv){
    
    if (l == NULL) {
        return;
    }
    
    list_node_value_insert(l, l->length, lnv);
}

// 修改
void list_node_set(list *l, int index, list_node *nln) {
    if (l == NULL || nln == NULL) {
        return;
    }
    list_node *ln = l->header;
    for (int i = 0; i <= index; i++) {
        ln = ln->next;
    }
    nln->next = ln->next;
    free(ln->next);
    ln->next = nln;
}

void list_node_set_value(list_node *ln, list_node_value lnv) {
    if (ln == NULL) {
        return;
    }
    ln->value = lnv;
}

// 删除
void list_node_remove(list *l, int index) {
    
    if (l == NULL || l->length <= index || index < 0) {
        return;
    }
    list_node *ln = l->header;
    for (int i = 0;i < index; i++) {
        ln = ln->next;
    }
    list_node *rln = ln->next;
    ln->next = rln->next;
    free(rln);
    rln = NULL;
    l->length -= 1;
}

void list_node_value_remove(list *l, list_node_value lnv) {
    
    if (l == NULL || l->length < 0) {
        return;
    }
    
    list_node *ln = l->header;
    while (ln) {
        list_node *lnn = ln->next;
        if (lnn == NULL) {
            return;
        }
        if (lnn->value == lnv) {
            ln->next = lnn->next;
            free(lnn);
            lnn = NULL;
            l->length -= 1;
        } else {
            ln = lnn;
        }
    }
    
}

// 打印
void list_print(list *l) {
    
    printf("list\n");
    printf("{\n");
    printf("\tlength: %d\n", l->length);
    printf("\tvalue: [");
    list_node *ln = l->header;
    for (int i = 0; i < l->length; i++) {
        ln = ln->next;
        printf("%p", (ln->value));
        if (i < l->length - 1) {
            printf(",");
        }
    }
    printf("]\n}\n");
}
```



