---
layout: post
title:  "Objective-C"
date:   2020-12-01 09:00:00
categories: Objective-C
---
**Objective-C**

# 1. 前言

&emsp;&emsp;广泛地，iOS开发者都会以[`objc`](https://github.com/opensource-apple/objc4)是`C++`的超集来作为这门语言的入门描述。

## 1.1. `C++`

&emsp;&emsp;在前言里，对于涉及到的`C++`特性，如下：

1. `Struct`与`Class`，
   1. 同：可继承、多态、包含成员函数
   2. 异：
      1. `Struct`默认`plublic`
      2. `Class`默认`private`且可使用泛型模版等
2. `typedef`类型定义符
    ```C++
    struct a {
        int data
    };

    // v1等效
    typedef struct a A;
    struct a v1;
    A v1;
    
    // v2等效
    typedef struct a *A_P;
    struct a *v2
    A_P v2;
    ```

# 2. 微观

&emsp;&emsp;从`NSObject`到`objc_object`，细节都在这里。

## 2.1 `NSObject`

&emsp;&emsp;声明关联路径：`Public Headers/NSObject.h`

&emsp;&emsp;实现关联路径：`Source/NSObject.mm`

&emsp;&emsp;对应地，Xcode内最新的声明文件位于以下路径内。

    /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/usr/include/objc/

```ObjC
#ifndef _OBJC_NSOBJECT_H_
#define _OBJC_NSOBJECT_H_

#if __OBJC__

#include <objc/objc.h>
#include <objc/NSObjCRuntime.h>

@class NSString, NSMethodSignature, NSInvocation;

@protocol NSObject

/*
    ...
*/

@end


OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0, 2.0)
OBJC_ROOT_CLASS
OBJC_EXPORT
@interface NSObject <NSObject> {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wobjc-interface-ivars"
    Class isa  OBJC_ISA_AVAILABILITY;
#pragma clang diagnostic pop
}

/*
    ...
*/

@end

#endif

#endif
```

&emsp;&emsp;在头文件中，可以看得出的关注点：

1. 可以发现关于`NSObject`，声明了一个`protocol`和一个`interface`，并且`interface`遵循同名`protocol`
2. 声明了是`NSObject`子类的3个类`NSString`, `NSMethodSignature`, `NSInvocation`，这是很特殊的，因为基类的方法与子类的类型有关系
3. 有若干个熟悉而陌生的类型，`id`, `Class`, `SEL`, `IMP`，它们是`objc`与`runtime`载体

&emsp;&emsp;怀着这几点，通过内在实现，理解这些设计的目的。以下是`NSObject.mm`内的`NSObject`实现代码。

```ObjC
@implementation NSObject

/*
    ...
*/


// Replaced by CF (returns an NSMethodSignature)
+ (NSMethodSignature *)instanceMethodSignatureForSelector:(SEL)sel {
    _objc_fatal("+[NSObject instanceMethodSignatureForSelector:] "
                "not available without CoreFoundation");
}

// Replaced by CF (returns an NSMethodSignature)
+ (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    _objc_fatal("+[NSObject methodSignatureForSelector:] "
                "not available without CoreFoundation");
}

// Replaced by CF (returns an NSMethodSignature)
- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    _objc_fatal("-[NSObject methodSignatureForSelector:] "
                "not available without CoreFoundation");
}

+ (void)forwardInvocation:(NSInvocation *)invocation {
    [self doesNotRecognizeSelector:(invocation ? [invocation selector] : 0)];
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    [self doesNotRecognizeSelector:(invocation ? [invocation selector] : 0)];
}

+ (id)forwardingTargetForSelector:(SEL)sel {
    return nil;
}

- (id)forwardingTargetForSelector:(SEL)sel {
    return nil;
}


// Replaced by CF (returns an NSString)
+ (NSString *)description {
    return nil;
}

// Replaced by CF (returns an NSString)
- (NSString *)description {
    return nil;
}

+ (NSString *)debugDescription {
    return [self description];
}

- (NSString *)debugDescription {
    return [self description];
}


/*
    ...
*/

@end
```

&emsp;&emsp;分别看看从声明中关注的点透露出什么信息：

1. `interface`与`protocol`中的声明，透露出的信息：
   1. `interface`含有
      1. 对象的初始化
      2. 消息的分发
   2. `protocol`含有`ObjC`里`id`、`SEL`、`Class`关系的标准方法声明
2. 3个在声明中的类`NSString`, `NSMethodSignature`, `NSInvocation`，在实现中是没有作用的，因为相关返回类型或参数类型在实现中并没有调用，并且可以看到一些类的方法实现，需要依赖`CoreFoundation`内的代码才有实现的意义
3. `objc`与`runtime`载体，`id`, `Class`, `SEL`, `IMP`，它们的声明在源码中是有分叉的，很容易让人迷惑。
   1. 在`NSObject.h`引入的头文件`objc.h`中，有一份相关的声明，如下，作为一份对外声明的头文件来说，显然声明了`id`, `Class`, `SEL`, `IMP`这4个名字是什么数据类型的别名。
    ```ObjC
        #include <objc/objc-api.h>

        // ...

        #if !OBJC_TYPES_DEFINED
        /// An opaque type that represents an Objective-C class.
        typedef struct objc_class *Class;

        /// Represents an instance of a class.
        struct objc_object {
            Class isa  OBJC_ISA_AVAILABILITY;
        };

        /// A pointer to an instance of a class.
        typedef struct objc_object *id;
        #endif

        /// An opaque type that represents a method selector.
        typedef struct objc_selector *SEL;

        /// A pointer to the function of a method implementation. 
        #if !OBJC_OLD_DISPATCH_PROTOTYPES
        typedef void (*IMP)(void /* id, SEL, ... */ ); 
        #else
        typedef id (*IMP)(id, SEL, ...); 
        #endif
    ```
   2. 在`NSObject.mm`引入的头文件`objc-private.h`中，却有特别的引入头文件方式，如下，先引入`objc-private.h`，再引入`NSObject.h`。
      1. `NSObject.mm`源码：
        ```ObjC
            #include "objc-private.h"
            #include "NSObject.h"
        ```
      2. `objc-private.h`源码：
        ```ObjC
            // ...
            #ifdef _OBJC_OBJC_H_
            #error include objc-private.h before other headers
            #endif

            #define OBJC_TYPES_DEFINED 1

            // ...

            struct objc_class;
            struct objc_object;

            typedef struct objc_class *Class;
            typedef struct objc_object *id;
        ```
      3. 如此一来，`objc-private.h`中定义的宏`OBJC_TYPES_DEFINED`，使得`NSObject.mm`中关于`id`, `Class`的别名声明不是使用`objc.h`中的，而是`objc-private.h`内的再次声明的别名了。
      4. 从前面几个注意点可以看出，实际的这几个`objc`与`runtime`载体，有效的声明如下：
        ```ObjC
            // 结构体声明别名
            typedef struct objc_class *Class;
            typedef struct objc_object *id;
            // objc.h中的声明是有效的
            typedef struct objc_selector *SEL;
            // 声明 IMP 为类型别名的函数指针
            // 第一个参数是 id 类型，即 struct objc_object *
            // 第二个参数是 SEL 类型， 即 struct objc_selector *
            // 后续参数可变
            typedef id (*IMP)(id, SEL, ...); 
        ```
## 2.2 `objc_object`

&emsp;&emsp;通过上面看到的`NSObject`实现，可以发现，他们都在操作一种数据，就是`objc_objcet`。

&emsp;&emsp;`objc_objcet`结构体的声明，位于`objc-private.h`文件中。

&emsp;&emsp;`objc_object`结构体内的函数，内联部分实现位于`objc-object.h`文件中（引入于`objc-private.h`文件末尾，目的是优化内存管理，减少内存消耗），剩余部分位于`NSObject.mm`文件中。

&emsp;&emsp;先来看`objc-private.h`中的声明，如下，关于`objc_object`我们需要了解的是`isa`是`union isa_t`类型的私有变量，`objc_object`对于内存的管理都与此变量有关。

```ObjC
struct objc_object {
private:
    isa_t isa;

public:

/*
    ...
*/
};

// Inlined parts of objc_object's implementation
#include "objc-object.h"

```

## 2.3 `isa_t`

&emsp;&emsp;`isa_t`是个`union`，一方面各成员共用内存，节省内存空间，一方面出于优化内存目的而方便改变数据类型。代码如下：

**备注：本文提及的知识点仅限于`arm64`的`iOS`系统，其他平台有所差异。**

```ObjC
union isa_t 
{
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;

#if SUPPORT_NONPOINTER_ISA

    // extra_rc must be the MSB-most field (so it matches carry/overflow flags)
    // indexed must be the LSB (fixme or get rid of it)
    // shiftcls must occupy the same bits that a real class pointer would
    // bits + RC_ONE is equivalent to extra_rc + 1
    // RC_HALF is the high bit of extra_rc (i.e. half of its range)

    // future expansion:
    // uintptr_t fast_rr : 1;     // no r/r overrides
    // uintptr_t lock : 2;        // lock for atomic property, @synch
    // uintptr_t extraBytes : 1;  // allocated with extra bytes

# if __arm64__
#   define ISA_MASK        0x0000000ffffffff8ULL
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
    struct {
        uintptr_t indexed           : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 19;
#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };
/*
    ...
*/

# else
    // Available bits in isa field are architecture-specific.
#   error unknown architecture
# endif

// SUPPORT_NONPOINTER_ISA
#endif

};
```

&emsp;&emsp;对于`isa_t`，有2点重要的内存优化知识需要掌握:

1. Tagged Pointer
2. nonpointer isa
   
### 2.3.1 Tagged Pointer

*参考文章*：
1. https://blog.devtang.com/2014/05/30/understand-tagged-pointer/
2. https://juejin.cn/post/6844904132940136462

&emsp;&emsp;`Tagged Pointer`的出现，使特定类型数据的内存存取获得了倍数级的提升。

&emsp;&emsp;在`objc-config.h`中，可以看到决定是否支持使用`Tagged Pointer`的宏，如下，

```ObjC
// Define SUPPORT_TAGGED_POINTERS=1 to enable tagged pointer objects
// Be sure to edit tagged pointer SPI in objc-internal.h as well.
#if !(__OBJC2__  &&  __LP64__)
#   define SUPPORT_TAGGED_POINTERS 0
#else
#   define SUPPORT_TAGGED_POINTERS 1
#endif
```

&emsp;&emsp;在`objc-private.h`中，可以看到一个头文件的引入，如下：

```ObjC
#include "objc-internal.h"
```

&emsp;&emsp;在`objc-internal.h`头文件中，有`Tagged Pointer`的支持，如下：

```ObjC
// Tagged pointer objects.

#if __LP64__
#define OBJC_HAVE_TAGGED_POINTERS 1
#endif

#if OBJC_HAVE_TAGGED_POINTERS

// Tagged pointer layout and usage is subject to change 
// on different OS versions. The current layout is:
// (MSB)
// 60 bits  payload
//  3 bits  tag index
//  1 bit   1 for tagged pointer objects, 0 for ordinary objects
// (LSB)

#if __has_feature(objc_fixed_enum)  ||  __cplusplus >= 201103L
enum objc_tag_index_t : uint8_t
#else
typedef uint8_t objc_tag_index_t;
enum
#endif
{
    OBJC_TAG_NSAtom            = 0, 
    OBJC_TAG_1                 = 1, 
    OBJC_TAG_NSString          = 2, 
    OBJC_TAG_NSNumber          = 3, 
    OBJC_TAG_NSIndexPath       = 4, 
    OBJC_TAG_NSManagedObjectID = 5, 
    OBJC_TAG_NSDate            = 6, 
    OBJC_TAG_7                 = 7
};
#if __has_feature(objc_fixed_enum)  &&  !defined(__cplusplus)
typedef enum objc_tag_index_t objc_tag_index_t;
#endif

OBJC_EXPORT void _objc_registerTaggedPointerClass(objc_tag_index_t tag, Class cls)
    __OSX_AVAILABLE_STARTING(__MAC_10_9, __IPHONE_7_0);

OBJC_EXPORT Class _objc_getClassForTag(objc_tag_index_t tag)
    __OSX_AVAILABLE_STARTING(__MAC_10_9, __IPHONE_7_0);

static inline bool 
_objc_taggedPointersEnabled(void)
{
    extern uintptr_t objc_debug_taggedpointer_mask;
    return (objc_debug_taggedpointer_mask != 0);
}

#if TARGET_OS_IPHONE
// tagged pointer marker is MSB

static inline void *
_objc_makeTaggedPointer(objc_tag_index_t tag, uintptr_t value)
{
    // assert(_objc_taggedPointersEnabled());
    // assert((unsigned int)tag < 8);
    // assert(((value << 4) >> 4) == value);
    return (void*)((1UL << 63) | ((uintptr_t)tag << 60) | (value & ~(0xFUL << 60)));
}

static inline bool 
_objc_isTaggedPointer(const void *ptr) 
{
    return (intptr_t)ptr < 0;  // a.k.a. ptr & 0x8000000000000000
}

static inline objc_tag_index_t 
_objc_getTaggedPointerTag(const void *ptr) 
{
    // assert(_objc_isTaggedPointer(ptr));
    return (objc_tag_index_t)(((uintptr_t)ptr >> 60) & 0x7);
}

static inline uintptr_t
_objc_getTaggedPointerValue(const void *ptr) 
{
    // assert(_objc_isTaggedPointer(ptr));
    return (uintptr_t)ptr & 0x0fffffffffffffff;
}

static inline intptr_t
_objc_getTaggedPointerSignedValue(const void *ptr) 
{
    // assert(_objc_isTaggedPointer(ptr));
    return ((intptr_t)ptr << 4) >> 4;
}

// TARGET_OS_IPHONE
#else
// not TARGET_OS_IPHONE
// tagged pointer marker is LSB

/*
    ...
*/

// not TARGET_OS_IPHONE
#endif


OBJC_EXPORT void _objc_insert_tagged_isa(unsigned char slotNumber, Class isa)
    __OSX_AVAILABLE_BUT_DEPRECATED(__MAC_10_7,__MAC_10_9, __IPHONE_4_3,__IPHONE_7_0);

#endif
```

&emsp;&emsp;从这段代码中，我们可以看明白`Tagged Pointer`是怎么优化内存的。

1. 别名为`objc_tag_index_t`的枚举内，对`Tagged Pointer`的类型进行了细分，只有枚举中的类型可以在少量数据的情况下使用`Tagged Pointer`方式读写。从数字的最大值可以发现，类型的区分，最多只需要3bits，即`111`。
2. 对于iPhone设备，数据使用MSB，即大端，读写。
3. 在看创建`Tagged Pointer`的内联静态函数的实现，如下，知道一个`Tagged Pointer`的`isa`变量是怎么放置数据的。
   ```ObjC
    static inline void *
    _objc_makeTaggedPointer(objc_tag_index_t tag, uintptr_t value)
    {
        // assert(_objc_taggedPointersEnabled());
        // assert((unsigned int)tag < 8);
        // assert(((value << 4) >> 4) == value);
        return (void*)((1UL << 63) | ((uintptr_t)tag << 60) | (value & ~(0xFUL << 60)));
    }
   ```
4. 
5. 从目前的实验来看，需要注意几点：
   1. 对于目前iOS14的情况，`Tagged Pointer`数据的放置安排与代码不相符，`tag`数据变更了放置在0～2下标的位中。
   2. Xcode中编译运行的代码里，对`Tagged Pointer`的内存地址进行了混淆，需要设置运行环境变量`OBJC_DISABLE_TAG_OBFUSCATION`为`false`才能发现`Tagged Pointer`的真正内存情况。
   3. 基于前2点的表述，需要明白的是，实际最新的系统运行着的与开源的代码，存在着差异。但是`Tagged Pointer`这种内存优化手段的设计目的与实现情况还是运行着在所有设备中。唯一可能有阻碍的，大概是逆向的动态调试时，需要改变一下数据的查看规则了。对于这种数据情况的不符合，具体各个细分类型的细节暂不深究。
   
### 2.3.2 nonpointer isa

&emsp;&emsp;`nonpointer isa`是在非`Tagged Pointer`情况下，优化数据存取的方式，可以直观地在`union isa_t`中看到具体的数据有什么，得益于内部的结构体，我们能清楚地知道`isa`数据的放置情况的，如下：

```ObjC
#   define ISA_MASK        0x0000000ffffffff8ULL
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
    struct {
        uintptr_t indexed           : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 19;
#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };
```

&emsp;&emsp;在iOS设备中，以MSB（大端）的方式排列数据，这个结构体把64bit中的每个bit都安排得妥妥当当。

&emsp;&emsp;其中，真正的`ISA`地址只占用了64bit中的33bit，在需要获取的时候，则是通过`ISA_MASK`这个宏中的掩码进行与操作获取。

&emsp;&emsp;其他的位代表的意思，大部分能从表面的字意明白，如果有疑惑，可以通过`struct objc_object`内的函数操作了解。

&emsp;&emsp;首先，从初始化切入，`objc_object`的初始化函数实现（位于`objc-object.h`）如下：

```ObjC
inline void 
objc_object::initIsa(Class cls)
{
    initIsa(cls, false, false);
}

inline void 
objc_object::initClassIsa(Class cls)
{
    if (DisableIndexedIsa) {
        initIsa(cls, false, false);
    } else {
        initIsa(cls, true, false);
    }
}

inline void
objc_object::initProtocolIsa(Class cls)
{
    return initClassIsa(cls);
}

inline void 
objc_object::initInstanceIsa(Class cls, bool hasCxxDtor)
{
    assert(!UseGC);
    assert(!cls->requiresRawIsa());
    assert(hasCxxDtor == cls->hasCxxDtor());

    initIsa(cls, true, hasCxxDtor);
}

inline void 
objc_object::initIsa(Class cls, bool indexed, bool hasCxxDtor) 
{ 
    assert(!isTaggedPointer()); 
    
    if (!indexed) {
        isa.cls = cls;
    } else {
        assert(!DisableIndexedIsa);
        isa.bits = ISA_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.indexed is part of ISA_MAGIC_VALUE
        isa.has_cxx_dtor = hasCxxDtor;
        isa.shiftcls = (uintptr_t)cls >> 3;
    }
}
```


# 3. 宏观

&emsp;&emsp;从`objc_object`到`NSObject`，联系都在这里。

## 3.1 `SEL` & `IMP`

## 3.2 Property

## 3.3 Category

## 3.4 Association

## 3.5 Autoreleasepool