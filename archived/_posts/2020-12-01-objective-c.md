---
layout: post
title:  "Objective-C"
date:   2020-12-01 09:00:00
categories: Objective-C
---
**Objective-C**

# 0. 序

&emsp;&emsp;全文较长，为方便，添加序：

- [0. 序](#0-序)
- [1. 前言](#1-前言)
  - [1.1. `C++`](#11-c)
- [2. 微观](#2-微观)
  - [2.1 `NSObject`](#21-nsobject)
  - [2.2 `objc_object`](#22-objc_object)
  - [2.3 `isa_t`](#23-isa_t)
    - [2.3.1 Tagged Pointer](#231-tagged-pointer)
    - [2.3.2 nonpointer isa](#232-nonpointer-isa)
  - [2.4 `objc_class`](#24-objc_class)
  - [2.5 `SideTable`](#25-sidetable)
- [3. 宏观](#3-宏观)
  - [3.1 Reference count](#31-reference-count)
  - [3.2 `SEL` & `IMP`](#32-sel--imp)
  - [3.3 Property](#33-property)
  - [3.4 Extension & Category](#34-extension--category)
  - [3.5 Association object](#35-association-object)
  - [3.6 Autoreleasepool](#36-autoreleasepool)
- [4. ObjC初始化流程](#4-objc初始化流程)

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
3. `using type = other_type;`可理解为类型别名

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

&emsp;&emsp;在`NSObject.h`引入的头文件`objc.h`中，有一份相关的声明，如下，作为一份对外声明的头文件来说，显然声明了`id`, `Class`, `SEL`, `IMP`这4个名字是什么数据类型的别名

```ObjC
#include <objc/objc-api.h>

// ...

#if !OBJC_TYPES_DEFINED
/// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class;

/// Represents an instance of a class.
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
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
typedef id _Nullable (*IMP)(id _Nonnull, SEL _Nonnull, ...); 
#endif
```

&emsp;&emsp;在`NSObject.mm`引入的头文件`objc-private.h`中，却有特别的引入头文件方式，如下，先引入`objc-private.h`，再引入`NSObject.h`。
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

&emsp;&emsp;`isa_t`是个`union`，一方面各成员共用内存，节省内存空间，一方面出于优化内存目的而方便读取数据。位于`objc-private.h`，代码如下：

**备注：本文提及的知识点仅限于`arm64`的`iPhoneOS`系统，其他平台有所差异。**

```ObjC
#include "isa.h"

union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
#if defined(ISA_BITFIELD)
    struct {
        ISA_BITFIELD;  // defined in isa.h
    };
#endif
};
```

&emsp;&emsp;对于`isa_t`，除了正常的64位指针，还有有2点重要的内存优化知识需要掌握:

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

// Tagged pointer layout and usage is subject to change on different OS versions.

// Tag indexes 0..<7 have a 60-bit payload.
// Tag index 7 is reserved.
// Tag indexes 8..<264 have a 52-bit payload.
// Tag index 264 is reserved.

#if __has_feature(objc_fixed_enum)  ||  __cplusplus >= 201103L
enum objc_tag_index_t : uint16_t
#else
typedef uint16_t objc_tag_index_t;
enum
#endif
{
    // 60-bit payloads
    OBJC_TAG_NSAtom            = 0, 
    OBJC_TAG_1                 = 1, 
    OBJC_TAG_NSString          = 2, 
    OBJC_TAG_NSNumber          = 3, 
    OBJC_TAG_NSIndexPath       = 4, 
    OBJC_TAG_NSManagedObjectID = 5, 
    OBJC_TAG_NSDate            = 6,

    // 60-bit reserved
    OBJC_TAG_RESERVED_7        = 7, 

    // 52-bit payloads
    OBJC_TAG_Photos_1          = 8,
    OBJC_TAG_Photos_2          = 9,
    OBJC_TAG_Photos_3          = 10,
    OBJC_TAG_Photos_4          = 11,
    OBJC_TAG_XPC_1             = 12,
    OBJC_TAG_XPC_2             = 13,
    OBJC_TAG_XPC_3             = 14,
    OBJC_TAG_XPC_4             = 15,
    OBJC_TAG_NSColor           = 16,
    OBJC_TAG_UIColor           = 17,
    OBJC_TAG_CGColor           = 18,
    OBJC_TAG_NSIndexSet        = 19,

    OBJC_TAG_First60BitPayload = 0, 
    OBJC_TAG_Last60BitPayload  = 6, 
    OBJC_TAG_First52BitPayload = 8, 
    OBJC_TAG_Last52BitPayload  = 263, 

    OBJC_TAG_RESERVED_264      = 264
};
#if __has_feature(objc_fixed_enum)  &&  !defined(__cplusplus)
typedef enum objc_tag_index_t objc_tag_index_t;
#endif


// Returns true if tagged pointers are enabled.
// The other functions below must not be called if tagged pointers are disabled.
static inline bool 
_objc_taggedPointersEnabled(void);

// Register a class for a tagged pointer tag.
// Aborts if the tag is invalid or already in use.
OBJC_EXPORT void
_objc_registerTaggedPointerClass(objc_tag_index_t tag, Class _Nonnull cls)
    OBJC_AVAILABLE(10.9, 7.0, 9.0, 1.0, 2.0);

// Returns the registered class for the given tag.
// Returns nil if the tag is valid but has no registered class.
// Aborts if the tag is invalid.
OBJC_EXPORT Class _Nullable
_objc_getClassForTag(objc_tag_index_t tag)
    OBJC_AVAILABLE(10.9, 7.0, 9.0, 1.0, 2.0);

// Create a tagged pointer object with the given tag and payload.
// Assumes the tag is valid.
// Assumes tagged pointers are enabled.
// The payload will be silently truncated to fit.
static inline void * _Nonnull
_objc_makeTaggedPointer(objc_tag_index_t tag, uintptr_t payload);

// Return true if ptr is a tagged pointer object.
// Does not check the validity of ptr's class.
static inline bool 
_objc_isTaggedPointer(const void * _Nullable ptr);

// Extract the tag value from the given tagged pointer object.
// Assumes ptr is a valid tagged pointer object.
// Does not check the validity of ptr's tag.
static inline objc_tag_index_t 
_objc_getTaggedPointerTag(const void * _Nullable ptr);

// Extract the payload from the given tagged pointer object.
// Assumes ptr is a valid tagged pointer object.
// The payload value is zero-extended.
static inline uintptr_t
_objc_getTaggedPointerValue(const void * _Nullable ptr);

// Extract the payload from the given tagged pointer object.
// Assumes ptr is a valid tagged pointer object.
// The payload value is sign-extended.
static inline intptr_t
_objc_getTaggedPointerSignedValue(const void * _Nullable ptr);

// Don't use the values below. Use the declarations above.

#if (TARGET_OS_OSX || TARGET_OS_IOSMAC) && __x86_64__
    // 64-bit Mac - tag bit is LSB
#   define OBJC_MSB_TAGGED_POINTERS 0
#else
    // Everything else - tag bit is MSB
#   define OBJC_MSB_TAGGED_POINTERS 1
#endif

#define _OBJC_TAG_INDEX_MASK 0x7
// array slot includes the tag bit itself
#define _OBJC_TAG_SLOT_COUNT 16
#define _OBJC_TAG_SLOT_MASK 0xf

#define _OBJC_TAG_EXT_INDEX_MASK 0xff
// array slot has no extra bits
#define _OBJC_TAG_EXT_SLOT_COUNT 256
#define _OBJC_TAG_EXT_SLOT_MASK 0xff

#if OBJC_MSB_TAGGED_POINTERS
#   define _OBJC_TAG_MASK (1UL<<63)
#   define _OBJC_TAG_INDEX_SHIFT 60
#   define _OBJC_TAG_SLOT_SHIFT 60
#   define _OBJC_TAG_PAYLOAD_LSHIFT 4
#   define _OBJC_TAG_PAYLOAD_RSHIFT 4
#   define _OBJC_TAG_EXT_MASK (0xfUL<<60)
#   define _OBJC_TAG_EXT_INDEX_SHIFT 52
#   define _OBJC_TAG_EXT_SLOT_SHIFT 52
#   define _OBJC_TAG_EXT_PAYLOAD_LSHIFT 12
#   define _OBJC_TAG_EXT_PAYLOAD_RSHIFT 12
#else
/*
    ...
*/
#endif

extern uintptr_t objc_debug_taggedpointer_obfuscator;

static inline void * _Nonnull
_objc_encodeTaggedPointer(uintptr_t ptr)
{
    return (void *)(objc_debug_taggedpointer_obfuscator ^ ptr);
}

static inline uintptr_t
_objc_decodeTaggedPointer(const void * _Nullable ptr)
{
    return (uintptr_t)ptr ^ objc_debug_taggedpointer_obfuscator;
}

static inline bool 
_objc_taggedPointersEnabled(void)
{
    extern uintptr_t objc_debug_taggedpointer_mask;
    return (objc_debug_taggedpointer_mask != 0);
}

static inline void * _Nonnull
_objc_makeTaggedPointer(objc_tag_index_t tag, uintptr_t value)
{
    // PAYLOAD_LSHIFT and PAYLOAD_RSHIFT are the payload extraction shifts.
    // They are reversed here for payload insertion.

    // ASSERT(_objc_taggedPointersEnabled());
    if (tag <= OBJC_TAG_Last60BitPayload) {
        // ASSERT(((value << _OBJC_TAG_PAYLOAD_RSHIFT) >> _OBJC_TAG_PAYLOAD_LSHIFT) == value);
        uintptr_t result =
            (_OBJC_TAG_MASK | 
             ((uintptr_t)tag << _OBJC_TAG_INDEX_SHIFT) | 
             ((value << _OBJC_TAG_PAYLOAD_RSHIFT) >> _OBJC_TAG_PAYLOAD_LSHIFT));
        return _objc_encodeTaggedPointer(result);
    } else {
        // ASSERT(tag >= OBJC_TAG_First52BitPayload);
        // ASSERT(tag <= OBJC_TAG_Last52BitPayload);
        // ASSERT(((value << _OBJC_TAG_EXT_PAYLOAD_RSHIFT) >> _OBJC_TAG_EXT_PAYLOAD_LSHIFT) == value);
        uintptr_t result =
            (_OBJC_TAG_EXT_MASK |
             ((uintptr_t)(tag - OBJC_TAG_First52BitPayload) << _OBJC_TAG_EXT_INDEX_SHIFT) |
             ((value << _OBJC_TAG_EXT_PAYLOAD_RSHIFT) >> _OBJC_TAG_EXT_PAYLOAD_LSHIFT));
        return _objc_encodeTaggedPointer(result);
    }
}

static inline bool 
_objc_isTaggedPointer(const void * _Nullable ptr)
{
    return ((uintptr_t)ptr & _OBJC_TAG_MASK) == _OBJC_TAG_MASK;
}

static inline objc_tag_index_t 
_objc_getTaggedPointerTag(const void * _Nullable ptr) 
{
    // ASSERT(_objc_isTaggedPointer(ptr));
    uintptr_t value = _objc_decodeTaggedPointer(ptr);
    uintptr_t basicTag = (value >> _OBJC_TAG_INDEX_SHIFT) & _OBJC_TAG_INDEX_MASK;
    uintptr_t extTag =   (value >> _OBJC_TAG_EXT_INDEX_SHIFT) & _OBJC_TAG_EXT_INDEX_MASK;
    if (basicTag == _OBJC_TAG_INDEX_MASK) {
        return (objc_tag_index_t)(extTag + OBJC_TAG_First52BitPayload);
    } else {
        return (objc_tag_index_t)basicTag;
    }
}

static inline uintptr_t
_objc_getTaggedPointerValue(const void * _Nullable ptr) 
{
    // ASSERT(_objc_isTaggedPointer(ptr));
    uintptr_t value = _objc_decodeTaggedPointer(ptr);
    uintptr_t basicTag = (value >> _OBJC_TAG_INDEX_SHIFT) & _OBJC_TAG_INDEX_MASK;
    if (basicTag == _OBJC_TAG_INDEX_MASK) {
        return (value << _OBJC_TAG_EXT_PAYLOAD_LSHIFT) >> _OBJC_TAG_EXT_PAYLOAD_RSHIFT;
    } else {
        return (value << _OBJC_TAG_PAYLOAD_LSHIFT) >> _OBJC_TAG_PAYLOAD_RSHIFT;
    }
}

static inline intptr_t
_objc_getTaggedPointerSignedValue(const void * _Nullable ptr) 
{
    // ASSERT(_objc_isTaggedPointer(ptr));
    uintptr_t value = _objc_decodeTaggedPointer(ptr);
    uintptr_t basicTag = (value >> _OBJC_TAG_INDEX_SHIFT) & _OBJC_TAG_INDEX_MASK;
    if (basicTag == _OBJC_TAG_INDEX_MASK) {
        return ((intptr_t)value << _OBJC_TAG_EXT_PAYLOAD_LSHIFT) >> _OBJC_TAG_EXT_PAYLOAD_RSHIFT;
    } else {
        return ((intptr_t)value << _OBJC_TAG_PAYLOAD_LSHIFT) >> _OBJC_TAG_PAYLOAD_RSHIFT;
    }
}

// OBJC_HAVE_TAGGED_POINTERS
#endif
```

&emsp;&emsp;从这段代码中，我们可以看明白`Tagged Pointer`是怎么优化内存的。

1. 别名为`objc_tag_index_t`的枚举内，对`Tagged Pointer`的类型进行了细分，只有枚举中的类型可以在少量数据的情况下使用`Tagged Pointer`方式读写。
2. 对于iPhone设备，数据使用MSB，即大端，读写。
3. 再看创建`Tagged Pointer`的内联静态函数的实现，`Tagged Pointer`的`isa`根据数据类型枚举，区分数据的`payloads`位数，把数据放置到`isa`相应的位里。
4. `Tagged Pointer`的内存地址会被`objc_debug_taggedpointer_obfuscator`进行混淆，需要设置运行环境变量`OBJC_DISABLE_TAG_OBFUSCATION`为`false`才能发现`Tagged Pointer`的真正内存情况。
5. 从不同版本的`objc4`开源代码中对比，可以发现支持`Tagged Pointer`优化的数据类型越来越多了，内部的数据哪些bit代表的是哪些标识可以通过`OBJC_MSB_TAGGED_POINTERS`内定义的宏来位移获取，在优化内存这一方面，Apple真的是玩出花来了。

**总结：**`Tagged Pointer`对内存的优化，本质上，是把数据量少的变量数据存放在`isa`中，这时候`isa`就不是正常意义上的指针了，它本身就包含了数据以及数据类型等信息，在内存读写的时候，可以省略掉通过指针获取数据的步骤直接获取数据，在微观上虽然省得不多，但在宏观大量的读写内存数据的情况下，可以达到很惊人的优化效果。
   
### 2.3.2 nonpointer isa

&emsp;&emsp;`ISA_BITFIELD`是宏，在`isa.h`中，是这么定义的：
&emsp;&emsp;`nonpointer isa`是在非`Tagged Pointer`情况下，优化数据存取的方式，可以直观地在`union isa_t`中看到具体的数据有什么，得益于内部的结构体（`ISA_BITFIELD`），我们能清楚地知道`isa`数据的放置情况的。

&emsp;&emsp;`ISA_BITFIELD`是宏，在`isa.h`中，是这么定义的：

```ObjC
#ifndef _OBJC_ISA_H_
#define _OBJC_ISA_H_

#include "objc-config.h"


#if (!SUPPORT_NONPOINTER_ISA && !SUPPORT_PACKED_ISA && !SUPPORT_INDEXED_ISA) ||\
    ( SUPPORT_NONPOINTER_ISA &&  SUPPORT_PACKED_ISA && !SUPPORT_INDEXED_ISA) ||\
    ( SUPPORT_NONPOINTER_ISA && !SUPPORT_PACKED_ISA &&  SUPPORT_INDEXED_ISA)
    // good config
#else
#   error bad config
#endif


#if SUPPORT_PACKED_ISA

    // extra_rc must be the MSB-most field (so it matches carry/overflow flags)
    // nonpointer must be the LSB (fixme or get rid of it)
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
#   define ISA_BITFIELD                                                      \
      uintptr_t nonpointer        : 1;                                       \
      uintptr_t has_assoc         : 1;                                       \
      uintptr_t has_cxx_dtor      : 1;                                       \
      uintptr_t shiftcls          : 33; /*MACH_VM_MAX_ADDRESS 0x1000000000*/ \
      uintptr_t magic             : 6;                                       \
      uintptr_t weakly_referenced : 1;                                       \
      uintptr_t deallocating      : 1;                                       \
      uintptr_t has_sidetable_rc  : 1;                                       \
      uintptr_t extra_rc          : 19
#   define RC_ONE   (1ULL<<45)
#   define RC_HALF  (1ULL<<18)

# elif __x86_64__

/*
    ...
*/

# else
#   error unknown architecture for packed isa
# endif

// SUPPORT_PACKED_ISA
#endif


#if SUPPORT_INDEXED_ISA

# if  __ARM_ARCH_7K__ >= 2  ||  (__arm64__ && !__LP64__)
    // armv7k or arm64_32

/*
    ...
*/

# else
#   error unknown architecture for indexed isa
# endif

// SUPPORT_INDEXED_ISA
#endif


// _OBJC_ISA_H_
#endif
```

&emsp;&emsp;在`isa.h`中，对于不同平台，有不同的宏用于控制`isa`各个bit的意义。本文在只讨论`iPhoneOS`的前提下，只贴出部分，排除掉了英特尔CPU的电脑部分与运行32位的arm64架构的嵌入式设备（如Apple Watch）。

&emsp;&emsp;在iOS设备中，以MSB（大端）的方式排列数据，这个结构体把64bit中的每个bit都安排得妥妥当当。

&emsp;&emsp;在理解`nonpointer isa`前，还需要区分2个概念：

1. indexed isa
2. packed isa

&emsp;&emsp;它们通过在`objc-config.h`中的宏控制`nonpointer isa`的支持：

```ObjC
// Define SUPPORT_INDEXED_ISA=1 on platforms that store the class in the isa 
// field as an index into a class table.
// Note, keep this in sync with any .s files which also define it.
// Be sure to edit objc-abi.h as well.
#if __ARM_ARCH_7K__ >= 2  ||  (__arm64__ && !__LP64__)
#   define SUPPORT_INDEXED_ISA 1
#else
#   define SUPPORT_INDEXED_ISA 0
#endif

// Define SUPPORT_PACKED_ISA=1 on platforms that store the class in the isa 
// field as a maskable pointer with other data around it.
#if (!__LP64__  ||  TARGET_OS_WIN32  ||  \
     (TARGET_OS_SIMULATOR && !TARGET_OS_IOSMAC))
#   define SUPPORT_PACKED_ISA 0
#else
#   define SUPPORT_PACKED_ISA 1
#endif

// Define SUPPORT_NONPOINTER_ISA=1 on any platform that may store something
// in the isa field that is not a raw pointer.
#if !SUPPORT_INDEXED_ISA  &&  !SUPPORT_PACKED_ISA
#   define SUPPORT_NONPOINTER_ISA 0
#else
#   define SUPPORT_NONPOINTER_ISA 1
#endif
```

&emsp;&emsp;在`iPhoneOS`中，`SUPPORT_PACKED_ISA`为1，`SUPPORT_INDEXED_ISA`为0，所以`SUPPORT_NONPOINTER_ISA`为1，是支持`nonpointer isa`的，并且是`packed isa`。

&emsp;&emsp;

&emsp;&emsp;其中，真正的`ISA`地址只占用了64bit中的33bit，在需要获取的时候，则是通过`ISA_MASK`这个宏中的掩码进行与操作获取。

&emsp;&emsp;其他的位代表的意思，大部分能从表面的字意明白，如果有疑惑，可以通过`struct objc_object`内的函数操作了解。

&emsp;&emsp;从`ISA_BITFIELD`中，逐个查看他们的作用。

1. `nonpointer`，[0]bit，区分`isa`是否使用`nonpointer isa`，`isa`是个`union`，在使用`nonpointer`的情况下，先以0值初始化，然后根据`nonpointer isa`内结构体的位进行默认赋值。没有使用的情况下，`isa`的初始化就是正常的一个指针。在`struct objc_object`的私有初始化函数中可以看到其实现：

```ObjC
inline void 
objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) 
{ 
    ASSERT(!isTaggedPointer()); 
    
    if (!nonpointer) {
        isa = isa_t((uintptr_t)cls);
    } else {
        ASSERT(!DisableNonpointerIsa);
        ASSERT(!cls->instancesRequireRawIsa());

        isa_t newisa(0);

#if SUPPORT_INDEXED_ISA
        ASSERT(cls->classArrayIndex() > 0);
        newisa.bits = ISA_INDEX_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.indexcls = (uintptr_t)cls->classArrayIndex();
#else
        newisa.bits = ISA_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.shiftcls = (uintptr_t)cls >> 3;
#endif

        // This write must be performed in a single store in some cases
        // (for example when realizing a class because other threads
        // may simultaneously try to use the class).
        // fixme use atomics here to guarantee single-store and to
        // guarantee memory order w.r.t. the class index table
        // ...but not too atomic because we don't want to hurt instantiation
        isa = newisa;
    }
}
```

2. `has_assoc`，[1]bit，在使用`nonpointer isa`的情况下，区分是否有联合对象（Association Object），相关函数如下：

```ObjC
inline bool
objc_object::hasAssociatedObjects()
{
    if (isTaggedPointer()) return true;
    if (isa.nonpointer) return isa.has_assoc;
    return true;
}


inline void
objc_object::setHasAssociatedObjects()
{
    if (isTaggedPointer()) return;

retry:
    isa_t oldisa = LoadExclusive(&isa.bits);
    isa_t newisa = oldisa;
    if (!newisa.nonpointer  ||  newisa.has_assoc) {
        ClearExclusive(&isa.bits);
        return;
    }
    newisa.has_assoc = true;
    if (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)) goto retry;
}
```

3. `has_cxx_dtor`，[2]bit，在使用`nonpointer isa`的情况下，区分是否有C++的析构函数，出于改变`isa`指针、初始化与释放等场景的需要
4. `shiftcls`，[3]~[36]bit，在使用`nonpointer isa`的情况下，使用33bit空间存储指向类的指针
5. `magic`，[37]～[42]bit，在使用`nonpointer isa`的情况下，存放魔术数，也用于区分数据是否已初始化
6. `weakly_referenced`，[43]bit，在使用`nonpointer isa`的情况下，是否有被弱引用，决定释放的时候，是否需要到`SideTable`中处理引用问题
7. `deallocating`，[44]bit，在使用`nonpointer isa`的情况下，是否正在释放的标识位
8. `has_sidetable_rc`，[45]bit，在使用`nonpointer isa`的情况下，是否有`SideTable`的引用计数
9. `extra_rc`，[46]~[64]bit，在使用`nonpointer isa`的情况下，少量引用计数量的存放位，计算引用计数时加一，即（extra_rc + 1）。在引用计数超过19位能表达的值时，引用计数会移交`SideTable`记录，其中`RC_ONE`宏，是用来引用计数加一处理的，而`RC_HALF`则是每次引用计数溢出时，移交一半出去的量，剩余的一半依然留在`extra_rc`中。

&emsp;&emsp;`nonpointer isa`这种内存优化方式只在类指针地址在提供的bit能表达的情况下使用，当指针地址超过了提供的bit能表达的数量时，就会切换为正常的指针。这些都是运行时进行的，这体现了`objc`提供出来的运行时特性，具体的实现函数如下：
```ObjC
inline Class 
objc_object::changeIsa(Class newCls)
{
    // This is almost always true but there are 
    // enough edge cases that we can't assert it.
    // assert(newCls->isFuture()  || 
    //        newCls->isInitializing()  ||  newCls->isInitialized());

    ASSERT(!isTaggedPointer()); 

    isa_t oldisa;
    isa_t newisa;

    bool sideTableLocked = false;
    bool transcribeToSideTable = false;

    do {
        transcribeToSideTable = false;
        oldisa = LoadExclusive(&isa.bits);
        if ((oldisa.bits == 0  ||  oldisa.nonpointer)  &&
            !newCls->isFuture()  &&  newCls->canAllocNonpointer())
        {
            // 0 -> nonpointer
            // nonpointer -> nonpointer
#if SUPPORT_INDEXED_ISA
            if (oldisa.bits == 0) newisa.bits = ISA_INDEX_MAGIC_VALUE;
            else newisa = oldisa;
            // isa.magic is part of ISA_MAGIC_VALUE
            // isa.nonpointer is part of ISA_MAGIC_VALUE
            newisa.has_cxx_dtor = newCls->hasCxxDtor();
            ASSERT(newCls->classArrayIndex() > 0);
            newisa.indexcls = (uintptr_t)newCls->classArrayIndex();
#else
            if (oldisa.bits == 0) newisa.bits = ISA_MAGIC_VALUE;
            else newisa = oldisa;
            // isa.magic is part of ISA_MAGIC_VALUE
            // isa.nonpointer is part of ISA_MAGIC_VALUE
            newisa.has_cxx_dtor = newCls->hasCxxDtor();
            newisa.shiftcls = (uintptr_t)newCls >> 3;
#endif
        }
        else if (oldisa.nonpointer) {
            // nonpointer -> raw pointer
            // Need to copy retain count et al to side table.
            // Acquire side table lock before setting isa to 
            // prevent races such as concurrent -release.
            if (!sideTableLocked) sidetable_lock();
            sideTableLocked = true;
            transcribeToSideTable = true;
            newisa.cls = newCls;
        }
        else {
            // raw pointer -> raw pointer
            newisa.cls = newCls;
        }
    } while (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits));

    if (transcribeToSideTable) {
        // Copy oldisa's retain count et al to side table.
        // oldisa.has_assoc: nothing to do
        // oldisa.has_cxx_dtor: nothing to do
        sidetable_moveExtraRC_nolock(oldisa.extra_rc, 
                                     oldisa.deallocating, 
                                     oldisa.weakly_referenced);
    }

    if (sideTableLocked) sidetable_unlock();

    if (oldisa.nonpointer) {
#if SUPPORT_INDEXED_ISA
        return classForIndex(oldisa.indexcls);
#else
        return (Class)((uintptr_t)oldisa.shiftcls << 3);
#endif
    }
    else {
        return oldisa.cls;
    }
}
```

**总结：**`nonpointer isa`的内存优化，本质上，是缩减类指针地址在较低数值的时候在内存读写时的消耗，并不是一个正常的指向内存地址的指针，而是把类指针地址存到到`isa`的部分位内，并且将运行时需要的标识存放于其他位置。仅在类指针地址超过范围后，才转变为真正的指向内存地址的指针，从代码注释中看到`MACH_VM_MAX_ADDRESS 0x1000000000`，占用的是36bit地址位，33位的`nonpointer isa`类地址能处理绝大多数的场景了。这种处理极大地提高了嵌入式设备不需要大量运算的使用场景的运行效率。

## 2.4 `objc_class`

&emsp;&emsp;`struct objc_class`继承自`struct objc_object`，有上述所说`objc_object`的所有特点。

&emsp;&emsp;同时，类型指针别名为`Class`，它就是`Objective-C`中的`类`的数据结构体。

&emsp;&emsp;结构体声明位于`objc-runtime-new.h`文件中，BTW，`objc.h`中的对外暴露的声明，使用的是此处的代码，而非`runtime.h`内的，注意观察宏的值。

&emsp;&emsp;`struct objc_class`与`struct objc_object`的关系是双辅双成的，内部的函数实现，都有使用双方的数据，以这个作为认知前提，可以再理一下`struct objc_class`处理了什么。先看内部的变量：

```ObjC
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

    /*
        ...
    */
}
```

1. `ISA`，来自于`objc_object`的私有变量，通过函数获取`isa`私有变量内的类指针，有2种情况：
   1. 使用`nonpointer isa`的情况，获取时，`objc_object`的`ISA`函数内，会通过`ISA_MASK`提供真正的类地址
   2. 使用正常指针的情况，获取时，`objc_object`的`rawISA`函数内获取`isa`的所有bit作为类的地址
2. `superclass`，父类，也是`objc_class`，用于追溯父类的指针，关于`isa`的指向关系图，早有被广泛使用，如下，
   ![isa]({{site.url}}/../../assets/images/2020-12-01-objective-c-isa.png)
   1. `isa`
      1. 实例对象的`isa`指针指向的是生成它的类，在分析`isa`章节的时候就提及到，使用的是`initInstanceIsa`函数
      2. 而类本身也是`objc_object`，使用的是`initClassIsa`函数，BTW，meta类本身也是`objc_class`，但不论平台是否支持`isa`，使用的都是`rawISA`，即正常的没有内存优化的指针
   2. `superclass`
      1. `Objective-C`中继承关系的实现基础，此变量的目的就是为了给`objc_class`追溯父类，查找父类拥有的成员（如变量、函数，协议）
      2. 当`superclass`为nil时，说明这个类已经是根类了
   3. `isa`与`superclass`
      1. 通过`objc_object`内函数的理解，知道每一个`isa`都是通过`objc_class`（别名`Class`）来初始化的，而因为初始化`isa`的函数实现也只是决定用不用和怎么用优化内存来记录`objc_class`的地址，所以`isa`其实就是个指向生成`isa`的那个`objc_class`的地址的指针，也就是图中虚线的意思
      2. 查看`struct objc_class`，知道继承自`struct objc_object`，那么每一个`Class`也是一个`id`，`Objective-C`中的实例对象的属性和方法并不是由`struct objc_object`记录的，而是生成实例对象的类，所以实例对象的`isa`指向了能初始化实例对象的类，即`struct objc_class`，这样实例对象就能查找自己有什么属性和方法了
      3. 此图的相关逻辑由一个`runtime`函数实现：`objc_allocateClassPair`，位于`objc-runtime-new.mm`文件中，如下，初始化时，开辟2个空间，一个是`Class`自己，一个是`Class(meta)`，接着各自设置`struct objc_class`内的数据，然后互相建立联系，最后添加进类表里。
         1. 在建立联系的过程中，`Class`调用`initClassIsa`函数初始化`isa`指针，参数是`Class(meta)`，所以可以很清晰地知道`Class`的`isa`指向`Class(meta)`
         2. 如果`Class`没有父`Class`，会分开判断，将`Class(meta)`内的`supercls`变量赋值为`Class`，而`Class`的`supercls`赋值为`Nil`，`Class(meta)`的`isa`是自己
         3. 如果`Class`有父`Class`，那么`supercls`就是父`Class`，而`Class(meta)`则是`supercls`的`isa`，`Class(meta)`的`isa`是父`Class`的`Class(meta)`的`Class(meta)`，即`Root Class(meta)`

```ObjC
static Class 
alloc_class_for_subclass(Class supercls, size_t extraBytes)
{
    if (!supercls  ||  !supercls->isAnySwift()) {
        return _calloc_class(sizeof(objc_class) + extraBytes);
    }

    /*
        objc4中对Swift兼容的处理此处忽略
    */
}

// &UnsetLayout is the default ivar layout during class construction
static const uint8_t UnsetLayout = 0;

static void objc_initializeClassPair_internal(Class superclass, const char *name, Class cls, Class meta)
{
    runtimeLock.assertLocked();

    class_ro_t *cls_ro_w, *meta_ro_w;
    class_rw_t *cls_rw_w, *meta_rw_w;
    
    cls_rw_w   = objc::zalloc<class_rw_t>();
    meta_rw_w  = objc::zalloc<class_rw_t>();
    cls_ro_w   = (class_ro_t *)calloc(sizeof(class_ro_t), 1);
    meta_ro_w  = (class_ro_t *)calloc(sizeof(class_ro_t), 1);

    cls->setData(cls_rw_w);
    cls_rw_w->set_ro(cls_ro_w);
    meta->setData(meta_rw_w);
    meta_rw_w->set_ro(meta_ro_w);

    // Set basic info

    cls_rw_w->flags = RW_CONSTRUCTING | RW_COPIED_RO | RW_REALIZED | RW_REALIZING;
    meta_rw_w->flags = RW_CONSTRUCTING | RW_COPIED_RO | RW_REALIZED | RW_REALIZING | RW_META;

    cls_ro_w->flags = 0;
    meta_ro_w->flags = RO_META;
    if (superclass) {
        uint32_t flagsToCopy = RW_FORBIDS_ASSOCIATED_OBJECTS;
        cls_rw_w->flags |= superclass->data()->flags & flagsToCopy;
        cls_ro_w->instanceStart = superclass->unalignedInstanceSize();
        meta_ro_w->instanceStart = superclass->ISA()->unalignedInstanceSize();
        cls->setInstanceSize(cls_ro_w->instanceStart);
        meta->setInstanceSize(meta_ro_w->instanceStart);
    } else {
        cls_ro_w->flags |= RO_ROOT;
        meta_ro_w->flags |= RO_ROOT;
        cls_ro_w->instanceStart = 0;
        meta_ro_w->instanceStart = (uint32_t)sizeof(objc_class);
        cls->setInstanceSize((uint32_t)sizeof(id));  // just an isa
        meta->setInstanceSize(meta_ro_w->instanceStart);
    }

    cls_ro_w->name = strdupIfMutable(name);
    meta_ro_w->name = strdupIfMutable(name);

    cls_ro_w->ivarLayout = &UnsetLayout;
    cls_ro_w->weakIvarLayout = &UnsetLayout;

    meta->chooseClassArrayIndex();
    cls->chooseClassArrayIndex();

    // This absolutely needs to be done before addSubclass
    // as initializeToEmpty() clobbers the FAST_CACHE bits
    cls->cache.initializeToEmpty();
    meta->cache.initializeToEmpty();

#if FAST_CACHE_META
    meta->cache.setBit(FAST_CACHE_META);
#endif
    meta->setInstancesRequireRawIsa();

    // Connect to superclasses and metaclasses
    cls->initClassIsa(meta);

    if (superclass) {
        meta->initClassIsa(superclass->ISA()->ISA());
        cls->superclass = superclass;
        meta->superclass = superclass->ISA();
        addSubclass(superclass, cls);
        addSubclass(superclass->ISA(), meta);
    } else {
        meta->initClassIsa(meta);
        cls->superclass = Nil;
        meta->superclass = cls;
        addRootClass(cls);
        addSubclass(cls, meta);
    }

    addClassTableEntry(cls);
}
/***********************************************************************
* addClassTableEntry
* Add a class to the table of all classes. If addMeta is true,
* automatically adds the metaclass of the class as well.
* Locking: runtimeLock must be held by the caller.
**********************************************************************/
static void
addClassTableEntry(Class cls, bool addMeta = true)
{
    runtimeLock.assertLocked();

    // This class is allowed to be a known class via the shared cache or via
    // data segments, but it is not allowed to be in the dynamic table already.
    auto &set = objc::allocatedClasses.get();

    ASSERT(set.find(cls) == set.end());

    if (!isKnownClass(cls))
        set.insert(cls);
    if (addMeta)
        addClassTableEntry(cls->ISA(), false);
}

Class objc_allocateClassPair(Class superclass, const char *name, 
                        size_t extraBytes)
{
    Class cls, meta;

    // Fail if the class name is in use.
    if (look_up_class(name, NO, NO)) return nil;

    mutex_locker_t lock(runtimeLock);

    // Fail if the class name is in use.
    // Fail if the superclass isn't kosher.
    if (getClassExceptSomeSwift(name)  ||
        !verifySuperclass(superclass, true/*rootOK*/))
    {
        return nil;
    }

    // Allocate new classes.
    cls  = alloc_class_for_subclass(superclass, extraBytes);
    meta = alloc_class_for_subclass(superclass, extraBytes);

    // fixme mangle the name if it looks swift-y?
    objc_initializeClassPair_internal(superclass, name, cls, meta);

    return cls;
}
```

3. `cache`，常用方法缓存，用于优化常用方法的寻找速度，通过`struct cache_t`内函数对`struct bucket_t`进行管理
   1. `CACHE_MASK_STORAGE`宏（定义位于`objc-config.h`中）在64位iPhoneOS中，值为 `CACHE_MASK_STORAGE_HIGH_16`，即高16位掩码
   2. `struct cache_t`，如下，与`nonpointer isa`的优化相似，`_maskAndBuckets`内保存着`bucket_t`的地址、4位用于`msgSend`的0位和16位的高位掩码，其中，16位掩码用于获取`bucket`的容量，同时也是`bucket`数量的最大值，即缓存方法数量的最大值

```ObjC
struct cache_t {
#if CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_OUTLINED
    /*
        ...
    */
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_HIGH_16
    explicit_atomic<uintptr_t> _maskAndBuckets;
    mask_t _mask_unused;
    
    // How much the mask is shifted by.
    static constexpr uintptr_t maskShift = 48;
    
    // Additional bits after the mask which must be zero. msgSend
    // takes advantage of these additional bits to construct the value
    // `mask << 4` from `_maskAndBuckets` in a single instruction.
    static constexpr uintptr_t maskZeroBits = 4;
    
    // The largest mask value we can store.
    static constexpr uintptr_t maxMask = ((uintptr_t)1 << (64 - maskShift)) - 1;
    
    // The mask applied to `_maskAndBuckets` to retrieve the buckets pointer.
    static constexpr uintptr_t bucketsMask = ((uintptr_t)1 << (maskShift - maskZeroBits)) - 1;
    
    // Ensure we have enough bits for the buckets pointer.
    static_assert(bucketsMask >= MACH_VM_MAX_ADDRESS, "Bucket field doesn't have enough bits for arbitrary pointers.");
#elif CACHE_MASK_STORAGE == CACHE_MASK_STORAGE_LOW_4
    /*
        ...
    */
#else
#error Unknown cache mask storage type.
#endif
    
#if __LP64__
    uint16_t _flags;
#endif
    uint16_t _occupied;

public:
    static bucket_t *emptyBuckets();
    
    struct bucket_t *buckets();
    mask_t mask();
    mask_t occupied();
    void incrementOccupied();
    void setBucketsAndMask(struct bucket_t *newBuckets, mask_t newMask);
    void initializeToEmpty();

    unsigned capacity();
    bool isConstantEmptyCache();
    bool canBeFreed();

/*
    忽略一些对 _flag 的处理
*/

    static size_t bytesForCapacity(uint32_t cap);
    static struct bucket_t * endMarker(struct bucket_t *b, uint32_t cap);

    void reallocate(mask_t oldCapacity, mask_t newCapacity, bool freeOld);
    void insert(Class cls, SEL sel, IMP imp, id receiver);

    static void bad_cache(id receiver, SEL sel, Class isa) __attribute__((noreturn, cold));
};
```

   3. 缓存方法时，会调用`insert`函数，如下，
      1. 处理缓存的数量
      2. 根据已缓存容量调整插入后的容量大小，注意到`reallocate`函数，调整缓存空间大小的时候，旧的缓存不会跟随到新的缓存内的，而是会被收集释放(根据参数判断调用`cache_collect_free`函数)，所以每次调整缓存容量（空缓存情况除外）后，都是从0开始重新缓存方法
         1. 已有缓存为0时，不释放空缓存，替换其为初始化的`INIT_CACHE_SIZE`，即（1 << 2）4容量的缓存空间
         2. 已有缓存不超过容量的 3/4 时，不需要处理缓存容量空间
         3. 已有缓存超过容量的 3/4 时，容量扩充2倍，但不超过最大可缓存容量`MAX_CACHE_SIZE`，即（1 << 16）上面提及到的16位掩码能表达的最大值
      3. 通过循环寻找容量范围内的空位插入，如果循环结束还找不到空位插入，调用`bad_cache`，记录崩溃前的日志
   
```ObjC
ALWAYS_INLINE
void cache_t::insert(Class cls, SEL sel, IMP imp, id receiver)
{
#if CONFIG_USE_CACHE_LOCK
    cacheUpdateLock.assertLocked();
#else
    runtimeLock.assertLocked();
#endif

    ASSERT(sel != 0 && cls->isInitialized());

    // Use the cache as-is if it is less than 3/4 full
    mask_t newOccupied = occupied() + 1;
    unsigned oldCapacity = capacity(), capacity = oldCapacity;
    if (slowpath(isConstantEmptyCache())) {
        // Cache is read-only. Replace it.
        if (!capacity) capacity = INIT_CACHE_SIZE;
        reallocate(oldCapacity, capacity, /* freeOld */false);
    }
    else if (fastpath(newOccupied + CACHE_END_MARKER <= capacity / 4 * 3)) {
        // Cache is less than 3/4 full. Use it as-is.
    }
    else {
        capacity = capacity ? capacity * 2 : INIT_CACHE_SIZE;
        if (capacity > MAX_CACHE_SIZE) {
            capacity = MAX_CACHE_SIZE;
        }
        reallocate(oldCapacity, capacity, true);
    }

    bucket_t *b = buckets();
    mask_t m = capacity - 1;
    mask_t begin = cache_hash(sel, m);
    mask_t i = begin;

    // Scan for the first unused slot and insert there.
    // There is guaranteed to be an empty slot because the
    // minimum size is 4 and we resized at 3/4 full.
    do {
        if (fastpath(b[i].sel() == 0)) {
            incrementOccupied();
            b[i].set<Atomic, Encoded>(sel, imp, cls);
            return;
        }
        if (b[i].sel() == sel) {
            // The entry was added to the cache by some other thread
            // before we grabbed the cacheUpdateLock.
            return;
        }
    } while (fastpath((i = cache_next(i, m)) != begin));

    cache_t::bad_cache(receiver, (SEL)sel, cls);
}
```

4. `bits`，`struct class_data_bits_t`类型，ObjC中的类内部内容就存放在这里了，包括变量、属性、方法、遵循的协议

&emsp;&emsp;`struct class_data_bits_t`，如下

1. `struct objc_class`是`struct class_data_bits_t`的友元结构体，可以访问内部的私有函数
2. 内嵌一个`bits`变量，同样是为了优化内存，通过掩码方式存放数据获
   1. [0]bit，Swift的ABI稳定前的标识位
   2. [1]bit，Swift的ABI稳定版本的标识位
   3. [2]bit, 类中有默认`RR`的标识位
   4. [3]~[46]bit，取`class_rw_t`类型的数据指针，共44位

```ObjC
    /*
        ...
    */
    
    // class is a Swift class from the pre-stable Swift ABI
    #define FAST_IS_SWIFT_LEGACY    (1UL<<0)
    // class is a Swift class from the stable Swift ABI
    #define FAST_IS_SWIFT_STABLE    (1UL<<1)
    // class or superclass has default retain/release/autorelease/retainCount/
    //   _tryRetain/_isDeallocating/retainWeakReference/allowsWeakReference
    #define FAST_HAS_DEFAULT_RR     (1UL<<2)
    // data pointer
    #define FAST_DATA_MASK          0x00007ffffffffff8UL

    /*
        ...
    */

    struct class_data_bits_t {
        friend objc_class;

        // Values are the FAST_ flags above.
        uintptr_t bits;
        /*
            ...
        */
        public:

        class_rw_t* data() const {
            return (class_rw_t *)(bits & FAST_DATA_MASK);
        }
        void setData(class_rw_t *newData)
        {
            ASSERT(!data()  ||  (newData->flags & (RW_REALIZING | RW_FUTURE)));
            // Set during realization or construction only. No locking needed.
            // Use a store-release fence because there may be concurrent
            // readers of data and data's contents.
            uintptr_t newBits = (bits & ~FAST_DATA_MASK) | (uintptr_t)newData;
            atomic_thread_fence(memory_order_release);
            bits = newBits;
        }

        // Get the class's ro data, even in the presence of concurrent realization.
        // fixme this isn't really safe without a compiler barrier at least
        // and probably a memory barrier when realizeClass changes the data field
        const class_ro_t *safe_ro() {
            class_rw_t *maybe_rw = data();
            if (maybe_rw->flags & RW_REALIZED) {
                // maybe_rw is rw
                return maybe_rw->ro();
            } else {
                // maybe_rw is actually ro
                return (class_ro_t *)maybe_rw;
            }
        }
        /*
            ...
        */
    };
```

&emsp;&emsp;`struct class_rw_t`深入，如下：

1. `flags`，32bit的变量，用于存放类加载状态的标识，同时也根据加载状态判断以何种数据结构读取数据，其中`RW_REALIZED`即(1<<31)用于判断类是否已经实现
2. `witness`，用于快速判断类是否已经加载完毕
3. `firstSubclass`， 记录第一个子类，在自上而下的类实现的遍历中用到
4. `nextSiblingClass`，记录同父类的兄弟类，同样是在自上而下的类实现的遍历中用到
5. 定义泛型`objc::PointerUnion<const class_ro_t *, class_rw_ext_t *>`为`ro_or_rw_ext_t`类型，`objc::PointerUnion`是个指针联合体（可以理解为同一个放地址空间的内存空间内能存放不同类型但是使用同样父类类型的指针的一个数据结构，目的是方便判断和转换类型），通过判断类的实现状态读取`class_ro_t`内的数据。
   1. 在类实现前，`struct class_data_bits_t`内的`bits`通过掩码获得的指针指向的就是`class_ro_t`，这时，`class_ro_t`同样存在的`flag`的`RW_REALIZED`掩码后值是0
   2. 在类实现后，`struct class_data_bits_t`内的`bits`通过掩码获得的指针指向的是`class_rw_t`，这时，`class_rw_t`的`flag`的`RW_REALIZED`掩码后值是1，读取`class_ro_t`内的数据需要通过`class_rw_t`内的函数`ro`调用
   3. 在编译后，`class_ro_t`内的数据就产生了，所以可以在类实现前就通过`class_ro_t`获得一些关于类的变量、方法、或协议，而在类的实现后，使用`class_rw_ext_t`类型，体现了runtime的特性，在运行时，会通过此类型获得类的的变量、方法、或协议

```ObjC
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint16_t witness;
    /*
        ...
    */
    explicit_atomic<uintptr_t> ro_or_rw_ext;

    Class firstSubclass;
    Class nextSiblingClass;

    private:
    using ro_or_rw_ext_t = objc::PointerUnion<const class_ro_t *, class_rw_ext_t *>;

    const ro_or_rw_ext_t get_ro_or_rwe() const {
        return ro_or_rw_ext_t{ro_or_rw_ext};
    }

    void set_ro_or_rwe(const class_ro_t *ro) {
        ro_or_rw_ext_t{ro}.storeAt(ro_or_rw_ext, memory_order_relaxed);
    }

    void set_ro_or_rwe(class_rw_ext_t *rwe, const class_ro_t *ro) {
        // the release barrier is so that the class_rw_ext_t::ro initialization
        // is visible to lockless readers
        rwe->ro = ro;
        ro_or_rw_ext_t{rwe}.storeAt(ro_or_rw_ext, memory_order_release);
    }

    class_rw_ext_t *extAlloc(const class_ro_t *ro, bool deep = false);

    public:
    
    /*
        ...
    */
}
```

&emsp;&emsp;`struct class_ro_t`深入，如下，内部存放了非常多的数据，这个结构体是真正的存放ObjC中使用到的变量、属性、方法、协议等数据的地方

```ObjC
    struct class_ro_t {
        uint32_t flags;
        uint32_t instanceStart;
        uint32_t instanceSize;
        #ifdef __LP64__
        uint32_t reserved;
        #endif

        const uint8_t * ivarLayout;

        const char * name;
        method_list_t * baseMethodList;
        protocol_list_t * baseProtocols;
        const ivar_list_t * ivars;

        const uint8_t * weakIvarLayout;
        property_list_t *baseProperties;

        /*
            ...
        */
    };
```

1. `flags`，32bit的标识位，记录`struct class_ro_t`的class状态，具体标识意思可以通过位于`objc-runtime-new.h`内的宏查到
2. `instanceStart`，实例对象的起始内存位置，根类位置为0，与父类对象保持相对位置的存放关系，当父类调整了对象大小后，实例对象的起始位置也要跟着改变
3. `instanceSize`，实例对象的大小，子类对象会获得父类对象的大小
4. `reserved`，64位处理器的保留位
5. `ivarLayout`，记录ObjC类中强引用变量的数量（该变量主要有也有弱引用变量的数量，但是在可以忽略的情况下会忽略掉）主要是强引用的变量，在寻找变量的时候，通过对齐地址的位移，获得各个变量的索引再获取变量
6. `name`，类名
7. `baseMethodList`，`entsize_list_tt`迭代器泛型模版类，元素类型为`method_t`，提供通过元素地址获取索引的函数
8. `baseProtocols`，`protocol_list_t`类型的迭代器，相较于`entsize_list_tt`迭代器少了操作符重载和掩码处理，内部真正的协议数据来自`protocol_ref_t`类型（即实际指向`struct protocol_t`，如下）的数组

```ObjC
struct protocol_t : objc_object {
    const char *mangledName;
    struct protocol_list_t *protocols;
    method_list_t *instanceMethods;
    method_list_t *classMethods;
    method_list_t *optionalInstanceMethods;
    method_list_t *optionalClassMethods;
    property_list_t *instanceProperties;
    uint32_t size;   // sizeof(protocol_t)
    uint32_t flags;
    // Fields below this point are not always present on disk.
    const char **_extendedMethodTypes;
    const char *_demangledName;
    property_list_t *_classProperties;
    /*
        ...
    */
};
```

1.  `ivars`，`entsize_list_tt`迭代器泛型模版类，元素类型为`ivar_t`，提供判断类中是否存在`ivar`的函数
2.  `weakIvarLayout`，记录ObjC类中弱饮用变量的数量，与`ivarLayout`对应，主要是若引用的变量，在寻找变量的时候，通过对齐地址的位移，获得各个变量的索引再获取变量
3.  `baseProperties`，`entsize_list_tt`迭代器泛型模版类，元素类型为`property_t`，只有属性名和类型描述

**总结：**`struct objc_class`在继承了`struct objc_object`特性的情况下，增加了对象的关系处理，
1. 通过`isa`与`superclass`联通了实例对象、类、元类，为运行时的消息转发提供了基础
2. 通过`cache`缓存常用方法，优化调用方法的速度
3. 通过`bits`存放变量、属性、方法、遵循的协议及相关的管理函数，在缓存中找不到的方法，就会在这里找

## 2.5 `SideTable`

&emsp;&emsp;`SideTable`用于记录引用计数，通过泛型模版`template<typename T> class StripedMap`包装成`SideTablesMap`，位于`Source/NSObject.mm`，如下，一旦初始化就持续使用不允许释放，通过`SideTables`函数获取C++引用实例使用

```ObjC
static objc::ExplicitInit<StripedMap<SideTable>> SideTablesMap;

static StripedMap<SideTable>& SideTables() {
    return SideTablesMap.get();
}
```

&emsp;&emsp;先看看包装用的`template<typename T> class StripedMap`，如下，将`T`替换成`SideTable`来看待
   1. `StripedMap`在iPhoneOS内只有8份`SideTable`，`objc_object`的引用计数是根据指针地址的位移操作后分配进`array`内的
   2. `array`内的元素是`struct PaddedT`，对应内部的`value`其实就是`SideTable`类型的变量，并且是64位对齐的
   3. `public`部分有操作符重载，用于通过`objc_object`地址获取对象引用计数，另外还有一些加锁操作，是`SideTable`内的自旋锁的外部调用接口

```ObjC
enum { CacheLineSize = 64 };

// StripedMap<T> is a map of void* -> T, sized appropriately 
// for cache-friendly lock striping. 
// For example, this may be used as StripedMap<spinlock_t>
// or as StripedMap<SomeStruct> where SomeStruct stores a spin lock.
template<typename T>
class StripedMap {
#if TARGET_OS_IPHONE && !TARGET_OS_SIMULATOR
    enum { StripeCount = 8 };
#else
    /*
        ...
    */
#endif

    struct PaddedT {
        T value alignas(CacheLineSize);
    };

    PaddedT array[StripeCount];

    static unsigned int indexForPointer(const void *p) {
        uintptr_t addr = reinterpret_cast<uintptr_t>(p);
        return ((addr >> 4) ^ (addr >> 9)) % StripeCount;
    }

 public:
    T& operator[] (const void *p) { 
        return array[indexForPointer(p)].value; 
    }
    const T& operator[] (const void *p) const { 
        return const_cast<StripedMap<T>>(this)[p]; 
    }

    // Shortcuts for StripedMaps of locks.
    /*
        锁操作函数
    */
};
```

&emsp;&emsp;再来看看`SideTable`，如下，
```ObjC
// RefcountMap disguises its pointers because we 
// don't want the table to act as a root for `leaks`.
typedef objc::DenseMap<DisguisedPtr<objc_object>,size_t,RefcountMapValuePurgeable> RefcountMap;

/**
 * The global weak references table. Stores object ids as keys,
 * and weak_entry_t structs as their values.
 */
struct weak_table_t {
    weak_entry_t *weak_entries;
    size_t    num_entries;
    uintptr_t mask;
    uintptr_t max_hash_displacement;
};

struct SideTable {
    spinlock_t slock;
    RefcountMap refcnts;
    weak_table_t weak_table;

    SideTable() {
        memset(&weak_table, 0, sizeof(weak_table));
    }

    ~SideTable() {
        _objc_fatal("Do not delete SideTable.");
    }

    /*
        锁操作
    */
};
```

1. `slock`对应外部调用的自旋锁
2. `refcnts`，引用计数，`RefcountMap`类型，通过泛型模版`DenseMap`创建，模版有5个参数，前2个是key、value，后3个对前2个补充且有默认的泛型模版
   1. `RefcountMap`中的key是`DisguisedPtr<objc_object>`，即伪装了的`struct objc_object`指针，
   2. `RefcountMap`中的value是`size_t`，记录引用计数的多少
3. `weak_table`，弱引用表，`weak_table_t`类型，位于`objc-weak.h`，相关数据结构如下：
   1. `weak_entries`，`weak_entry_t`类型，用于记录弱引用关系，
      1. 被弱引用的对象地址通过`DisguisedPtr<objc_object>`存储，
      2. 引用它的对象通过联合体读取，
         1. 小于4个时，使用内联数组，充分优化内存
         2. 大于4个时，使用哈希表，在iPhoneOS中，最多可记录`2^62`个弱引用

```ObjC
/*
    ...
*/

// The address of a __weak variable.
// These pointers are stored disguised so memory analysis tools
// don't see lots of interior pointers from the weak table into objects.
typedef DisguisedPtr<objc_object *> weak_referrer_t;

#if __LP64__
#define PTR_MINUS_2 62
#else
#define PTR_MINUS_2 30
#endif

/**
 * The internal structure stored in the weak references table. 
 * It maintains and stores
 * a hash set of weak references pointing to an object.
 * If out_of_line_ness != REFERRERS_OUT_OF_LINE then the set
 * is instead a small inline array.
 */
#define WEAK_INLINE_COUNT 4

// out_of_line_ness field overlaps with the low two bits of inline_referrers[1].
// inline_referrers[1] is a DisguisedPtr of a pointer-aligned address.
// The low two bits of a pointer-aligned DisguisedPtr will always be 0b00
// (disguised nil or 0x80..00) or 0b11 (any other address).
// Therefore out_of_line_ness == 0b10 is used to mark the out-of-line state.
#define REFERRERS_OUT_OF_LINE 2

struct weak_entry_t {
    DisguisedPtr<objc_object> referent;
    union {
        struct {
            weak_referrer_t *referrers;
            uintptr_t        out_of_line_ness : 2;
            uintptr_t        num_refs : PTR_MINUS_2;
            uintptr_t        mask;
            uintptr_t        max_hash_displacement;
        };
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };

    bool out_of_line() {
        return (out_of_line_ness == REFERRERS_OUT_OF_LINE);
    }

    weak_entry_t& operator=(const weak_entry_t& other) {
        memcpy(this, &other, sizeof(other));
        return *this;
    }

    weak_entry_t(objc_object *newReferent, objc_object **newReferrer)
        : referent(newReferent)
    {
        inline_referrers[0] = newReferrer;
        for (int i = 1; i < WEAK_INLINE_COUNT; i++) {
            inline_referrers[i] = nil;
        }
    }
};

/**
 * The global weak references table. Stores object ids as keys,
 * and weak_entry_t structs as their values.
 */
struct weak_table_t {
    weak_entry_t *weak_entries;
    size_t    num_entries;
    uintptr_t mask;
    uintptr_t max_hash_displacement;
};

/*
    ...
*/
```

&emsp;&emsp;`weak_table_t`和`weak_entry_t`很容易搞混，它们在增删的情况下略有不同
1. `weak_table_t`，初始化时，容量长度为64，
   2. 扩容情况，容量超过 3/4 时扩容2倍，并且内部的`weak_entry_t`会从新计算哈希插入
   3. 缩减情况，容量超过1024，但在用不足 1/16 时，缩小至 1/8，即最小也不会小于初始化的64大小
2. `weak_entry_t`，初始化时，使用固定容量为4的数组
   1. 扩容情况，大于4时，看是否使用哈希表，如果使用哈希表，则容量超过 3/4 时扩容2倍
   2. 没有缩减情况，但会进行`referer`的置nil操作并数量减1，如果`referer`内为空了，则`weak_table_t`会移除掉这个`weak_entry_t`

**总结：**

&emsp;&emsp;`Struct SideTable`是一个全局的用于记录对象引用计数的结构体，对于内存的管理
1. `Tagged Pointer`的`isa`情况下，不会使用
2. `nonpointer isa`情况下，先记录到`extra_rc`，超过时，把一半的引用计数移到`SideTable`
3. `raw isa`情况下，直接记录到`SideTable`中

&emsp;&emsp;`SideTable`中的数据是根据对象地址处理后分散放到各个`SideTable`表`StripeMap`中的`Array`中的
1. 对于每一个强引用对象，以伪装的对象指针地址为key，引用计数为value进行记录
2. 对于每一个弱引用对象，以弱引用对象的伪装指针地址为key，以数组或者哈希表记录每个引用这个弱引用对象的伪装对象指针

# 3. 宏观

&emsp;&emsp;从`objc_object`到`NSObject`，微入宏观，联系都在这里。

## 3.1 Reference count

&emsp;&emsp;在ObjC中，引用计数是内存管理的一种方式，从微观中的好几个地方，都已经见识它的细节，再来看看它的宏观体现。

1. `isa`中的区别
   1. `Tagged Pointer`的`isa`，没有使用引用计数，有返回值的相关统一处理的代码处都会直接返回`id`本身
   2. `nonpointer isa`，先以`isa`中`extra_rc`记录引用计数，当将要溢出时，标识`has_sidetable_rc`，并让一半的引用计数移交`SideTable`记录
   3. `raw isa`，引用计数全由`SideTable`记录
2. `ivar`中的区别
   1. `class_ro_t`中，`ivarLayout`主要记录强引用变量的内存布局数量，弱引用的变量能忽略就忽略
   2. `class_ro_t`中，`weakIvarLayout`主要记录弱引用变量的内存布局数量，强引用的变量能忽略就忽略
   3. `class_ro_t`中，`ivars`中除了`ivarLayout`和`weakIvarLayout`中记录的布局变量外，都是不使用引用计数的变量
3. 强引用与弱引用的记录方式区别
   1. `SideTable`中的强引用以`RefcountMap`（即`objc::DenseMap<DisguisedPtr<objc_object>,size_t,RefcountMapValuePurgeable>`）类型记录引用计数，散列表，key为伪装对象指针，value为引用计数
   2. `SideTable`中的弱引用以`weak_table_t`（即`struct weak_table_t`）类型记录引用计数，内部维护`weak_entry_t`，散列表，`referent`（`DisguisedPtr<objc_object>`）是被引用的对象实例（理解为value）的伪装对象指针，`referer`(`weak_referrer_t`，也是`DisguisedPtr<objc_object *>`)是弱引用容器指针地址（指向弱引用指针地址的指针），可通过如下代码简单理解，

```ObjC
/*
    假设初始化传递的String参数为不是 Tagged Pointer isa的其他类强引用了的字符串对象实例
    @property 关键字包含 ivar + getter + setter
    那么
    ivar _rawISAString 就是 referer
    参数string 就是 referent
*/
@interface ZObject : NSObject
@property (weak) NSString *rawISAString;
@end

@implement ZObject
- (id)initWithString:(NSString *)string {
    self = [super init];
    if (self) {
        self.rawISAString = string;
    }
    return self;
}
@end
```

## 3.2 `SEL` & `IMP`

&emsp;&emsp;从微观上看一个ObjC类，即`struct objc_class`，方法的本质是`SEL`和`IMP`，它们在`class_ro_t`中被`method_list_t`类型的变量（即列表）管理，列表元素是`struct method_t`，`SEL`与`IMP`（即`MethodListIMP`）就在内部，代码如下：

```ObjC
struct method_t {
    SEL name;
    const char *types;
    MethodListIMP imp;

    struct SortBySELAddress :
        public std::binary_function<const method_t&,
                                    const method_t&, bool>
    {
        bool operator() (const method_t& lhs,
                         const method_t& rhs)
        { return lhs.name < rhs.name; }
    };
};
```

&emsp;&emsp;在宏观上来看，方法在被调用时，会被缓存到`objc_class`的`cache`中。而从方法调用的角度上看，ObjC中的方法调用被叫作消息发送，原因可看`message.h`内的代码：

```ObjC
#include <objc/objc.h>
#include <objc/runtime.h>

/*
    ...
*/


/* Basic Messaging Primitives
 *
 * On some architectures, use objc_msgSend_stret for some struct return types.
 * On some architectures, use objc_msgSend_fpret for some float return types.
 * On some architectures, use objc_msgSend_fp2ret for some float return types.
 *
 * These functions must be cast to an appropriate function pointer type 
 * before being called. 
 */
#if !OBJC_OLD_DISPATCH_PROTOTYPES
/*
    ...
*/
#else
/** 
 * Sends a message with a simple return value to an instance of a class.
 * 
 * @param self A pointer to the instance of the class that is to receive the message.
 * @param op The selector of the method that handles the message.
 * @param ... 
 *   A variable argument list containing the arguments to the method.
 * 
 * @return The return value of the method.
 * 
 * @note When it encounters a method call, the compiler generates a call to one of the
 *  functions \c objc_msgSend, \c objc_msgSend_stret, \c objc_msgSendSuper, or \c objc_msgSendSuper_stret.
 *  Messages sent to an object’s superclass (using the \c super keyword) are sent using \c objc_msgSendSuper; 
 *  other messages are sent using \c objc_msgSend. Methods that have data structures as return values
 *  are sent using \c objc_msgSendSuper_stret and \c objc_msgSend_stret.
 */
OBJC_EXPORT id _Nullable
objc_msgSend(id _Nullable self, SEL _Nonnull op, ...)
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0, 2.0);
/** 
 * Sends a message with a simple return value to the superclass of an instance of a class.
 * 
 * @param super A pointer to an \c objc_super data structure. Pass values identifying the
 *  context the message was sent to, including the instance of the class that is to receive the
 *  message and the superclass at which to start searching for the method implementation.
 * @param op A pointer of type SEL. Pass the selector of the method that will handle the message.
 * @param ...
 *   A variable argument list containing the arguments to the method.
 * 
 * @return The return value of the method identified by \e op.
 * 
 * @see objc_msgSend
 */
OBJC_EXPORT id _Nullable
objc_msgSendSuper(struct objc_super * _Nonnull super, SEL _Nonnull op, ...)
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0, 2.0);
#endif
```

&emsp;&emsp;注释中可以看到`objc_msgSend`函数的参数

1. `id`，消息接收者，通过实例对象的`isa`可找到`objc_class`
2. `SEL`，方法选择器，从`id`中找到的`objc_class`中的`class_ro_t`寻找配对的`SEL`
3. 不定参数，对应消失发送时的参数

&emsp;&emsp;再看`objc_msgSendSuper`函数，与`objc_msgSend`不同的是，首个参数是`struct objc_super * _Nonnull super`，本质上相同，只是多了从实例对象的类中再找父类，但是消息接收的目标还是与`objc_msgSend`相同的实例对象自己。这也是ObjC中`super`关键字的调用中为什么与`self`关键字打印想通实例对象的原因（消息接收者相同）。

```ObjC
#ifndef OBJC_SUPER
#define OBJC_SUPER

/// Specifies the superclass of an instance. 
struct objc_super {
    /// Specifies an instance of a class.
    __unsafe_unretained _Nonnull id receiver;

    /// Specifies the particular superclass of the instance to message. 
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header */
    __unsafe_unretained _Nonnull Class class;
#else
    __unsafe_unretained _Nonnull Class super_class;
#endif
    /* super_class is the first class to search */
};
#endif
```

&emsp;&emsp;深入看`objc_msgSend`的实现，在源码中，实现是汇编语言写的，最好找arm手册对着看指令，这里不纠结具体寄存器的作用，只关注逻辑，`objc-msg-arm64.s`中调用`IMP`比较核心的步骤如下：

1. `_objc_msgSend`
2. `CacheLookup`
3. `TailCallCachedImp`

&emsp;&emsp;配合上`NSObject`中消息的分发机制，成就了消息发送的特性

## 3.3 Property

&emsp;&emsp;在ARC中，`@property`关键字在编译器编译时，会生成`ivar`+`getter`+`setter`，并且在`setter`中处理引用计数（即按照关键字情况调用`objc_retain`和`objc_release`）。

&emsp;&emsp;位于`objc-accesors.mm`中，有对应于各个关键字的实现：

```ObjC
id objc_getProperty(id self, SEL _cmd, ptrdiff_t offset, BOOL atomic) {
    if (offset == 0) {
        return object_getClass(self);
    }

    // Retain release world
    id *slot = (id*) ((char*)self + offset);
    if (!atomic) return *slot;
        
    // Atomic retain release world
    spinlock_t& slotlock = PropertyLocks[slot];
    slotlock.lock();
    id value = objc_retain(*slot);
    slotlock.unlock();
    
    // for performance, we (safely) issue the autorelease OUTSIDE of the spinlock.
    return objc_autoreleaseReturnValue(value);
}


static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy) __attribute__((always_inline));

static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy)
{
    if (offset == 0) {
        object_setClass(self, newValue);
        return;
    }

    id oldValue;
    id *slot = (id*) ((char*)self + offset);

    if (copy) {
        newValue = [newValue copyWithZone:nil];
    } else if (mutableCopy) {
        newValue = [newValue mutableCopyWithZone:nil];
    } else {
        if (*slot == newValue) return;
        newValue = objc_retain(newValue);
    }

    if (!atomic) {
        oldValue = *slot;
        *slot = newValue;
    } else {
        spinlock_t& slotlock = PropertyLocks[slot];
        slotlock.lock();
        oldValue = *slot;
        *slot = newValue;        
        slotlock.unlock();
    }

    objc_release(oldValue);
}

void objc_setProperty(id self, SEL _cmd, ptrdiff_t offset, id newValue, BOOL atomic, signed char shouldCopy) 
{
    bool copy = (shouldCopy && shouldCopy != MUTABLE_COPY);
    bool mutableCopy = (shouldCopy == MUTABLE_COPY);
    reallySetProperty(self, _cmd, newValue, offset, atomic, copy, mutableCopy);
}

void objc_setProperty_atomic(id self, SEL _cmd, id newValue, ptrdiff_t offset)
{
    reallySetProperty(self, _cmd, newValue, offset, true, false, false);
}

void objc_setProperty_nonatomic(id self, SEL _cmd, id newValue, ptrdiff_t offset)
{
    reallySetProperty(self, _cmd, newValue, offset, false, false, false);
}


void objc_setProperty_atomic_copy(id self, SEL _cmd, id newValue, ptrdiff_t offset)
{
    reallySetProperty(self, _cmd, newValue, offset, true, true, false);
}

void objc_setProperty_nonatomic_copy(id self, SEL _cmd, id newValue, ptrdiff_t offset)
{
    reallySetProperty(self, _cmd, newValue, offset, false, true, false);
}
```

1. `atomic`和`nonatomic`，原子性关键字，从`get`和`set`相应函数中发现，`atomic`时，会加锁，相应地会损耗性能
2. `copy`和`mutableCopy`，这些关键字的`get`函数相同，`set`函数根据情况不同传递不同参数给`reallySetProperty`函数进行设置，新实例对象会引用计数加一，旧的实例对象减一
3. `strong`，强引用关键字，调用的是`id objc_autoreleaseReturnValue(id obj)`和`void objc_storeStrong(id *location, id obj)`，`copy`和`mutableCopy`最后都会调用想同的函数，但最终的本质都是`objc_retain`和`objc_release`
4. `weak`，弱引用关键字，调用的是`id objc_loadWeak(id *location)`和`id objc_storeWeak(id *location, id newObj)`，

## 3.4 Extension & Category

&emsp;&emsp;Extension与Category都是ObjC中的特性，为这门语言提供了更多灵活。

&emsp;&emsp;先来看异同：

1. 参与时：Extension在编译时，Category在运行时
2. ivar：Extension可添加ivar，Category只能通过Association Object达到相同效果，这是由`objc_class`中的数据结构实现决定的
3. @implementation：Extension没有自己独立的实现部分，Category有自己独立的实现部分
4. 添加限制：Extension不能添加到不可修改的代码中运行，Category可添加到不可修改的代码中运行

&emsp;&emsp;再看看Category的实现：

```ObjC
struct locstamped_category_t {
    category_t *cat;
    struct header_info *hi;
};

struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties;

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
    
    protocol_list_t *protocolsForMeta(bool isMeta) {
        if (isMeta) return nullptr;
        else return protocols;
    }
};

class category_list : nocopy_t {
    union {
        locstamped_category_t lc;
        struct {
            locstamped_category_t *array;
            // this aliases with locstamped_category_t::hi
            // which is an aliased pointer
            uint32_t is_array :  1;
            uint32_t count    : 31;
            uint32_t size     : 32;
        };
    } _u;

public:
    category_list() : _u{{nullptr, nullptr}} { }
    category_list(locstamped_category_t lc) : _u{{lc}} { }
    category_list(category_list &&other) : category_list() {
        std::swap(_u, other._u);
    }
    ~category_list()
    {
        if (_u.is_array) {
            free(_u.array);
        }
    }

    uint32_t count() const
    {
        if (_u.is_array) return _u.count;
        return _u.lc.cat ? 1 : 0;
    }

    uint32_t arrayByteSize(uint32_t size) const
    {
        return sizeof(locstamped_category_t) * size;
    }

    const locstamped_category_t *array() const
    {
        return _u.is_array ? _u.array : &_u.lc;
    }

    void append(locstamped_category_t lc)
    {
        if (_u.is_array) {
            if (_u.count == _u.size) {
                // Have a typical malloc growth:
                // - size <=  8: grow by 2
                // - size <= 16: grow by 4
                // - size <= 32: grow by 8
                // ... etc
                _u.size += _u.size < 8 ? 2 : 1 << (fls(_u.size) - 2);
                _u.array = (locstamped_category_t *)reallocf(_u.array, arrayByteSize(_u.size));
            }
            _u.array[_u.count++] = lc;
        } else if (_u.lc.cat == NULL) {
            _u.lc = lc;
        } else {
            locstamped_category_t *arr = (locstamped_category_t *)malloc(arrayByteSize(2));
            arr[0] = _u.lc;
            arr[1] = lc;

            _u.array = arr;
            _u.is_array = true;
            _u.count = 2;
            _u.size = 2;
        }
    }

    void erase(category_t *cat)
    {
        if (_u.is_array) {
            for (int i = 0; i < _u.count; i++) {
                if (_u.array[i].cat == cat) {
                    // shift entries to preserve list order
                    memmove(&_u.array[i], &_u.array[i+1], arrayByteSize(_u.count - i - 1));
                    return;
                }
            }
        } else if (_u.lc.cat == cat) {
            _u.lc.cat = NULL;
            _u.lc.hi = NULL;
        }
    }
};
```

1. `category_list`内的联合体`_u`，和`weak_entry_t`相似，量少的时候单独处理，量大的时候使用散列表处理，内部函数也通过字节位判断区分处理
2. `category_t`内记录了一个类可能存在的所有变量列表，这些类型，在微观的发掘中已经遇到过，就不展开了，并且在处理时，会区分是否元类来处理

&emsp;&emsp;Category是在镜像加载的时候处理的，顺序比普通类迟，所以只要有Category重写了原类中的方法，那么Category中的实现就会覆盖原类中的，如果有多个Category重写同一个方法，按照顺序就是最后加载的Category的那个实现为准（这里引申了一个问题，假如二进制重排的order文件内的类顺序调整成与编译顺序不同，那最终的实现以哪个为准？XD）。

## 3.5 Association object

&emsp;&emsp;`Association object`，关联对象，通常在`Category`中用于创建`property`用到，因为`Category`与原类的加载时间不一样，所以原类内存中`ivar`的空间无法为了`category`中的`property`开辟空间，于是为了实现运行时的特性，就有了`Association object`。

&emsp;&emsp;先看看`get`和`set`函数，`ChainedHookFunction`是个泛型模版，`SetAssocHook`类内封装了设置关联对象的函数。

```ObjC
id
objc_getAssociatedObject(id object, const void *key)
{
    return _object_get_associative_reference(object, key);
}

static ChainedHookFunction<objc_hook_setAssociatedObject> SetAssocHook{_base_objc_setAssociatedObject};

void
objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
{
    SetAssocHook.get()(object, key, value, policy);
}

void objc_removeAssociatedObjects(id object) 
{
    if (object && object->hasAssociatedObjects()) {
        _object_remove_assocations(object);
    }
}
```

&emsp;&emsp;再看内部的实现，位于`objc-references.mm`，如下：

```ObjC
#include "objc-private.h"
#include <objc/message.h>
#include <map>
#include "DenseMapExtras.h"

// expanded policy bits.

enum {
    OBJC_ASSOCIATION_SETTER_ASSIGN      = 0,
    OBJC_ASSOCIATION_SETTER_RETAIN      = 1,
    OBJC_ASSOCIATION_SETTER_COPY        = 3,            // NOTE:  both bits are set, so we can simply test 1 bit in releaseValue below.
    OBJC_ASSOCIATION_GETTER_READ        = (0 << 8),
    OBJC_ASSOCIATION_GETTER_RETAIN      = (1 << 8),
    OBJC_ASSOCIATION_GETTER_AUTORELEASE = (2 << 8)
};

spinlock_t AssociationsManagerLock;

namespace objc {

class ObjcAssociation {
    uintptr_t _policy;
    id _value;
public:
    ObjcAssociation(uintptr_t policy, id value) : _policy(policy), _value(value) {}
    ObjcAssociation() : _policy(0), _value(nil) {}
    ObjcAssociation(const ObjcAssociation &other) = default;
    ObjcAssociation &operator=(const ObjcAssociation &other) = default;
    ObjcAssociation(ObjcAssociation &&other) : ObjcAssociation() {
        swap(other);
    }

    inline void swap(ObjcAssociation &other) {
        std::swap(_policy, other._policy);
        std::swap(_value, other._value);
    }

    inline uintptr_t policy() const { return _policy; }
    inline id value() const { return _value; }

    inline void acquireValue() {
        if (_value) {
            switch (_policy & 0xFF) {
            case OBJC_ASSOCIATION_SETTER_RETAIN:
                _value = objc_retain(_value);
                break;
            case OBJC_ASSOCIATION_SETTER_COPY:
                _value = ((id(*)(id, SEL))objc_msgSend)(_value, @selector(copy));
                break;
            }
        }
    }

    inline void releaseHeldValue() {
        if (_value && (_policy & OBJC_ASSOCIATION_SETTER_RETAIN)) {
            objc_release(_value);
        }
    }

    inline void retainReturnedValue() {
        if (_value && (_policy & OBJC_ASSOCIATION_GETTER_RETAIN)) {
            objc_retain(_value);
        }
    }

    inline id autoreleaseReturnedValue() {
        if (slowpath(_value && (_policy & OBJC_ASSOCIATION_GETTER_AUTORELEASE))) {
            return objc_autorelease(_value);
        }
        return _value;
    }
};

typedef DenseMap<const void *, ObjcAssociation> ObjectAssociationMap;
typedef DenseMap<DisguisedPtr<objc_object>, ObjectAssociationMap> AssociationsHashMap;

// class AssociationsManager manages a lock / hash table singleton pair.
// Allocating an instance acquires the lock

class AssociationsManager {
    using Storage = ExplicitInitDenseMap<DisguisedPtr<objc_object>, ObjectAssociationMap>;
    static Storage _mapStorage;

public:
    AssociationsManager()   { AssociationsManagerLock.lock(); }
    ~AssociationsManager()  { AssociationsManagerLock.unlock(); }

    AssociationsHashMap &get() {
        return _mapStorage.get();
    }

    static void init() {
        _mapStorage.init();
    }
};

AssociationsManager::Storage AssociationsManager::_mapStorage;

} // namespace objc

using namespace objc;

void
_objc_associations_init()
{
    AssociationsManager::init();
}

id
_object_get_associative_reference(id object, const void *key)
{
    ObjcAssociation association{};

    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.get());
        AssociationsHashMap::iterator i = associations.find((objc_object *)object);
        if (i != associations.end()) {
            ObjectAssociationMap &refs = i->second;
            ObjectAssociationMap::iterator j = refs.find(key);
            if (j != refs.end()) {
                association = j->second;
                association.retainReturnedValue();
            }
        }
    }

    return association.autoreleaseReturnedValue();
}

void
_object_set_associative_reference(id object, const void *key, id value, uintptr_t policy)
{
    // This code used to work when nil was passed for object and key. Some code
    // probably relies on that to not crash. Check and handle it explicitly.
    // rdar://problem/44094390
    if (!object && !value) return;

    if (object->getIsa()->forbidsAssociatedObjects())
        _objc_fatal("objc_setAssociatedObject called on instance (%p) of class %s which does not allow associated objects", object, object_getClassName(object));

    DisguisedPtr<objc_object> disguised{(objc_object *)object};
    ObjcAssociation association{policy, value};

    // retain the new value (if any) outside the lock.
    association.acquireValue();

    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.get());

        if (value) {
            auto refs_result = associations.try_emplace(disguised, ObjectAssociationMap{});
            if (refs_result.second) {
                /* it's the first association we make */
                object->setHasAssociatedObjects();
            }

            /* establish or replace the association */
            auto &refs = refs_result.first->second;
            auto result = refs.try_emplace(key, std::move(association));
            if (!result.second) {
                association.swap(result.first->second);
            }
        } else {
            auto refs_it = associations.find(disguised);
            if (refs_it != associations.end()) {
                auto &refs = refs_it->second;
                auto it = refs.find(key);
                if (it != refs.end()) {
                    association.swap(it->second);
                    refs.erase(it);
                    if (refs.size() == 0) {
                        associations.erase(refs_it);

                    }
                }
            }
        }
    }

    // release the old value (outside of the lock).
    association.releaseHeldValue();
}

// Unlike setting/getting an associated reference,
// this function is performance sensitive because of
// raw isa objects (such as OS Objects) that can't track
// whether they have associated objects.
void
_object_remove_assocations(id object)
{
    ObjectAssociationMap refs{};

    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.get());
        AssociationsHashMap::iterator i = associations.find((objc_object *)object);
        if (i != associations.end()) {
            refs.swap(i->second);
            associations.erase(i);
        }
    }

    // release everything (outside of the lock).
    for (auto &i: refs) {
        i.second.releaseHeldValue();
    }
}
```

&emsp;&emsp;关联对象内在实现其实是`class AssociationsManager`的函数调用，其中管理内存的也是散列表，并且进行了套娃，数据关系如下：

1. `AssociationsHashMap`，key是`DisguisedPtr<objc_object>`，value是`ObjectAssociationMap`
2. `ObjectAssociationMap`，key是`const void *`，value是`ObjcAssociation`（内有策略与值）
   
&emsp;&emsp;套娃的散列表各有指向，在处理关联对象的时候，也会根据内存策略来判断引用方式，同时对于`isa`，也会标识出是否有关联对象。

## 3.6 Autoreleasepool

&emsp;&emsp;自动释放池是ObjC中处理释放内存的机制，而在每个工程的入口文件`main.m`中，我们也能看到它的存在，即:

```ObjC
@autoreleasepool {}
```

&emsp;&emsp;这是在`ARC`中的自动释放池使用方式，它的本质是成对地调用了下面的函数：

```ObjC
void *
objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}

NEVER_INLINE
void
objc_autoreleasePoolPop(void *ctxt)
{
    AutoreleasePoolPage::pop(ctxt);
}
```

&emsp;&emsp;这样就引申出了`class AutoreleasePoolPage`这个类，这是个按页分离的双向链表，先来看看类中的情况：

```ObjC
/***********************************************************************
   Autorelease pool implementation

   A thread's autorelease pool is a stack of pointers. 
   Each pointer is either an object to release, or POOL_BOUNDARY which is 
     an autorelease pool boundary.
   A pool token is a pointer to the POOL_BOUNDARY for that pool. When 
     the pool is popped, every object hotter than the sentinel is released.
   The stack is divided into a doubly-linked list of pages. Pages are added 
     and deleted as necessary. 
   Thread-local storage points to the hot page, where newly autoreleased 
     objects are stored. 
**********************************************************************/

/*
    ...
*/

class AutoreleasePoolPage : private AutoreleasePoolPageData
{
	friend struct thread_data_t;

public:
	static size_t const SIZE =
#if PROTECT_AUTORELEASEPOOL
		PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
		PAGE_MIN_SIZE;  // size and alignment, power of 2
#endif
    
private:
	static pthread_key_t const key = AUTORELEASE_POOL_KEY;
	static uint8_t const SCRIBBLE = 0xA3;  // 0xA3A3A3A3 after releasing
	static size_t const COUNT = SIZE / sizeof(id);

    // EMPTY_POOL_PLACEHOLDER is stored in TLS when exactly one pool is 
    // pushed and it has never contained any objects. This saves memory 
    // when the top level (i.e. libdispatch) pushes and pops pools but 
    // never uses them.
#   define EMPTY_POOL_PLACEHOLDER ((id*)1)

#   define POOL_BOUNDARY nil

    // SIZE-sizeof(*this) bytes of contents follow

    static void * operator new(size_t size) {
        return malloc_zone_memalign(malloc_default_zone(), SIZE, SIZE);
    }
    static void operator delete(void * p) {
        return free(p);
    }

/*
    ...
*/

}
```

&emsp;&emsp;从注释、变量与初始化函数中，可以先了解到一些自动释放池的设计

1. 自动释放池只与当前线程相关，并且内部会以栈的方式记录需要释放的实例对象指针，并且`struct thread_data_t`是友元结构体
2. 池中的存放实例对象指针前需要先把`POOL_BOUNDARY`，即`nil`入栈，用于做栈中的边界，也称作哨兵(sentinel)
3. 自动释放池的每一页大小为`PAGE_MIN_SIZE`，即 `1 << 12`（定义于`vm_param.h`中），也就是4096字节，在`new`的时候就已经固定开辟了内存空间大小了

&emsp;&emsp;深入查看私有继承的`struct AutoreleasePoolPageData`，如下：

```ObjC
struct magic_t {
	static const uint32_t M0 = 0xA1A1A1A1;
#   define M1 "AUTORELEASE!"
	static const size_t M1_len = 12;
	uint32_t m[4];

	magic_t() {
		ASSERT(M1_len == strlen(M1));
		ASSERT(M1_len == 3 * sizeof(m[1]));

		m[0] = M0;
		strncpy((char *)&m[1], M1, M1_len);
	}

	~magic_t() {
		// Clear magic before deallocation.
		// This prevents some false positives in memory debugging tools.
		// fixme semantically this should be memset_s(), but the
		// compiler doesn't optimize that at all (rdar://44856676).
		volatile uint64_t *p = (volatile uint64_t *)m;
		p[0] = 0; p[1] = 0;
	}

	bool check() const {
		return (m[0] == M0 && 0 == strncmp((char *)&m[1], M1, M1_len));
	}

	bool fastcheck() const {
#if CHECK_AUTORELEASEPOOL
		return check();
#else
		return (m[0] == M0);
#endif
	}

#   undef M1
};

class AutoreleasePoolPage;
struct AutoreleasePoolPageData
{
	magic_t const magic;
	__unsafe_unretained id *next;
	pthread_t const thread;
	AutoreleasePoolPage * const parent;
	AutoreleasePoolPage *child;
	uint32_t const depth;
	uint32_t hiwat;

	AutoreleasePoolPageData(__unsafe_unretained id* _next, pthread_t _thread, AutoreleasePoolPage* _parent, uint32_t _depth, uint32_t _hiwat)
		: magic(), next(_next), thread(_thread),
		  parent(_parent), child(nil),
		  depth(_depth), hiwat(_hiwat)
	{
	}
};
```

&emsp;&emsp;因`class AutoreleasePoolPage`的继承，使它拥有了内部的私有变量，这里以元数据进行代称

1. 经过计算，`class AutoreleasePoolPage`的元数据占用内存空间为 56 Bytes，那么可用于存放自动释放池中的实例对象指针的空间大小就是 「4096 - 56 = 4040」Bytes，即 505 个指向`struct objc_object`的指针
2. 元数据解释
   1. `magic`，`struct magic_t`类型，人为用来标识自动释放池内存页是否正常的结构
   2. `next`，`id`类型，自动释放池中的实例对象存储的栈顶指针
   3. `thread`，`pthread_t`类型，当前线程
   4. `parent`，`AutoreleasePoolPage`类型，双向链表的前一个指向
   5. `child`，`AutoreleasePoolPage`类型，双向链表的后一个指向
   6. `depth`，`uint32_t`类型，当前节点在双向链表中的深度
   7. `hiwat`，`uint32_t`类型，high water，所有自动释放池实际占用了的内存容量

&emsp;&emsp;回归到调用流程，以下为`class AutoreleasePoolPage`中自动释放池会调用到的函数：

```ObjC
class AutoreleasePoolPage : private AutoreleasePoolPageData
{
	
    /*
        前面已经描述的部分，这里忽略
    */

    template<typename Fn>
    void
    busted(Fn log) const
    {
        magic_t right;
        log("autorelease pool page %p corrupted\n"
             "  magic     0x%08x 0x%08x 0x%08x 0x%08x\n"
             "  should be 0x%08x 0x%08x 0x%08x 0x%08x\n"
             "  pthread   %p\n"
             "  should be %p\n", 
             this, 
             magic.m[0], magic.m[1], magic.m[2], magic.m[3], 
             right.m[0], right.m[1], right.m[2], right.m[3], 
             this->thread, objc_thread_self());
    }

    __attribute__((noinline, cold, noreturn))
    void
    busted_die() const
    {
        busted(_objc_fatal);
        __builtin_unreachable();
    }

    inline void
    check(bool die = true) const
    {
        if (!magic.check() || thread != objc_thread_self()) {
            if (die) {
                busted_die();
            } else {
                busted(_objc_inform);
            }
        }
    }

    inline void
    fastcheck() const
    {
#if CHECK_AUTORELEASEPOOL
        check();
#else
        if (! magic.fastcheck()) {
            busted_die();
        }
#endif
    }


    id * begin() {
        return (id *) ((uint8_t *)this+sizeof(*this));
    }

    id * end() {
        return (id *) ((uint8_t *)this+SIZE);
    }

    bool empty() {
        return next == begin();
    }

    bool full() { 
        return next == end();
    }

    bool lessThanHalfFull() {
        return (next - begin() < (end() - begin()) / 2);
    }

    id *add(id obj)
    {
        ASSERT(!full());
        unprotect();
        id *ret = next;  // faster than `return next-1` because of aliasing
        *next++ = obj;
        protect();
        return ret;
    }

    void releaseAll() 
    {
        releaseUntil(begin());
    }

    void releaseUntil(id *stop) 
    {
        // Not recursive: we don't want to blow out the stack 
        // if a thread accumulates a stupendous amount of garbage
        
        while (this->next != stop) {
            // Restart from hotPage() every time, in case -release 
            // autoreleased more objects
            AutoreleasePoolPage *page = hotPage();

            // fixme I think this `while` can be `if`, but I can't prove it
            while (page->empty()) {
                page = page->parent;
                setHotPage(page);
            }

            page->unprotect();
            id obj = *--page->next;
            memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
            page->protect();

            if (obj != POOL_BOUNDARY) {
                objc_release(obj);
            }
        }

        setHotPage(this);

#if DEBUG
        // we expect any children to be completely empty
        for (AutoreleasePoolPage *page = child; page; page = page->child) {
            ASSERT(page->empty());
        }
#endif
    }

    void kill() 
    {
        // Not recursive: we don't want to blow out the stack 
        // if a thread accumulates a stupendous amount of garbage
        AutoreleasePoolPage *page = this;
        while (page->child) page = page->child;

        AutoreleasePoolPage *deathptr;
        do {
            deathptr = page;
            page = page->parent;
            if (page) {
                page->unprotect();
                page->child = nil;
                page->protect();
            }
            delete deathptr;
        } while (deathptr != this);
    }

    static void tls_dealloc(void *p) 
    {
        if (p == (void*)EMPTY_POOL_PLACEHOLDER) {
            // No objects or pool pages to clean up here.
            return;
        }

        // reinstate TLS value while we work
        setHotPage((AutoreleasePoolPage *)p);

        if (AutoreleasePoolPage *page = coldPage()) {
            if (!page->empty()) objc_autoreleasePoolPop(page->begin());  // pop all of the pools
            if (slowpath(DebugMissingPools || DebugPoolAllocation)) {
                // pop() killed the pages already
            } else {
                page->kill();  // free all of the pages
            }
        }
        
        // clear TLS value so TLS destruction doesn't loop
        setHotPage(nil);
    }

    static AutoreleasePoolPage *pageForPointer(const void *p) 
    {
        return pageForPointer((uintptr_t)p);
    }

    static AutoreleasePoolPage *pageForPointer(uintptr_t p) 
    {
        AutoreleasePoolPage *result;
        uintptr_t offset = p % SIZE;

        ASSERT(offset >= sizeof(AutoreleasePoolPage));

        result = (AutoreleasePoolPage *)(p - offset);
        result->fastcheck();

        return result;
    }


    static inline bool haveEmptyPoolPlaceholder()
    {
        id *tls = (id *)tls_get_direct(key);
        return (tls == EMPTY_POOL_PLACEHOLDER);
    }

    static inline id* setEmptyPoolPlaceholder()
    {
        ASSERT(tls_get_direct(key) == nil);
        tls_set_direct(key, (void *)EMPTY_POOL_PLACEHOLDER);
        return EMPTY_POOL_PLACEHOLDER;
    }

    static inline AutoreleasePoolPage *hotPage() 
    {
        AutoreleasePoolPage *result = (AutoreleasePoolPage *)
            tls_get_direct(key);
        if ((id *)result == EMPTY_POOL_PLACEHOLDER) return nil;
        if (result) result->fastcheck();
        return result;
    }

    static inline void setHotPage(AutoreleasePoolPage *page) 
    {
        if (page) page->fastcheck();
        tls_set_direct(key, (void *)page);
    }

    static inline AutoreleasePoolPage *coldPage() 
    {
        AutoreleasePoolPage *result = hotPage();
        if (result) {
            while (result->parent) {
                result = result->parent;
                result->fastcheck();
            }
        }
        return result;
    }


    static inline id *autoreleaseFast(id obj)
    {
        AutoreleasePoolPage *page = hotPage();
        if (page && !page->full()) {
            return page->add(obj);
        } else if (page) {
            return autoreleaseFullPage(obj, page);
        } else {
            return autoreleaseNoPage(obj);
        }
    }

    static __attribute__((noinline))
    id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
    {
        // The hot page is full. 
        // Step to the next non-full page, adding a new page if necessary.
        // Then add the object to that page.
        ASSERT(page == hotPage());
        ASSERT(page->full()  ||  DebugPoolAllocation);

        do {
            if (page->child) page = page->child;
            else page = new AutoreleasePoolPage(page);
        } while (page->full());

        setHotPage(page);
        return page->add(obj);
    }

    static __attribute__((noinline))
    id *autoreleaseNoPage(id obj)
    {
        // "No page" could mean no pool has been pushed
        // or an empty placeholder pool has been pushed and has no contents yet
        ASSERT(!hotPage());

        bool pushExtraBoundary = false;
        if (haveEmptyPoolPlaceholder()) {
            // We are pushing a second pool over the empty placeholder pool
            // or pushing the first object into the empty placeholder pool.
            // Before doing that, push a pool boundary on behalf of the pool 
            // that is currently represented by the empty placeholder.
            pushExtraBoundary = true;
        }
        else if (obj != POOL_BOUNDARY  &&  DebugMissingPools) {
            // We are pushing an object with no pool in place, 
            // and no-pool debugging was requested by environment.
            _objc_inform("MISSING POOLS: (%p) Object %p of class %s "
                         "autoreleased with no pool in place - "
                         "just leaking - break on "
                         "objc_autoreleaseNoPool() to debug", 
                         objc_thread_self(), (void*)obj, object_getClassName(obj));
            objc_autoreleaseNoPool(obj);
            return nil;
        }
        else if (obj == POOL_BOUNDARY  &&  !DebugPoolAllocation) {
            // We are pushing a pool with no pool in place,
            // and alloc-per-pool debugging was not requested.
            // Install and return the empty pool placeholder.
            return setEmptyPoolPlaceholder();
        }

        // We are pushing an object or a non-placeholder'd pool.

        // Install the first page.
        AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
        setHotPage(page);
        
        // Push a boundary on behalf of the previously-placeholder'd pool.
        if (pushExtraBoundary) {
            page->add(POOL_BOUNDARY);
        }
        
        // Push the requested object or pool.
        return page->add(obj);
    }


    static __attribute__((noinline))
    id *autoreleaseNewPage(id obj)
    {
        AutoreleasePoolPage *page = hotPage();
        if (page) return autoreleaseFullPage(obj, page);
        else return autoreleaseNoPage(obj);
    }

public:
    static inline id autorelease(id obj)
    {
        ASSERT(obj);
        ASSERT(!obj->isTaggedPointer());
        id *dest __unused = autoreleaseFast(obj);
        ASSERT(!dest  ||  dest == EMPTY_POOL_PLACEHOLDER  ||  *dest == obj);
        return obj;
    }


    static inline void *push() 
    {
        id *dest;
        if (slowpath(DebugPoolAllocation)) {
            // Each autorelease pool starts on a new pool page.
            dest = autoreleaseNewPage(POOL_BOUNDARY);
        } else {
            dest = autoreleaseFast(POOL_BOUNDARY);
        }
        ASSERT(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
        return dest;
    }

    __attribute__((noinline, cold))
    static void badPop(void *token)
    {
        // Error. For bincompat purposes this is not 
        // fatal in executables built with old SDKs.

        if (DebugPoolAllocation || sdkIsAtLeast(10_12, 10_0, 10_0, 3_0, 2_0)) {
            // OBJC_DEBUG_POOL_ALLOCATION or new SDK. Bad pop is fatal.
            _objc_fatal
                ("Invalid or prematurely-freed autorelease pool %p.", token);
        }

        // Old SDK. Bad pop is warned once.
        static bool complained = false;
        if (!complained) {
            complained = true;
            _objc_inform_now_and_on_crash
                ("Invalid or prematurely-freed autorelease pool %p. "
                 "Set a breakpoint on objc_autoreleasePoolInvalid to debug. "
                 "Proceeding anyway because the app is old "
                 "(SDK version " SDK_FORMAT "). Memory errors are likely.",
                     token, FORMAT_SDK(sdkVersion()));
        }
        objc_autoreleasePoolInvalid(token);
    }

    template<bool allowDebug>
    static void
    popPage(void *token, AutoreleasePoolPage *page, id *stop)
    {
        if (allowDebug && PrintPoolHiwat) printHiwat();

        page->releaseUntil(stop);

        // memory: delete empty children
        if (allowDebug && DebugPoolAllocation  &&  page->empty()) {
            // special case: delete everything during page-per-pool debugging
            AutoreleasePoolPage *parent = page->parent;
            page->kill();
            setHotPage(parent);
        } else if (allowDebug && DebugMissingPools  &&  page->empty()  &&  !page->parent) {
            // special case: delete everything for pop(top)
            // when debugging missing autorelease pools
            page->kill();
            setHotPage(nil);
        } else if (page->child) {
            // hysteresis: keep one empty child if page is more than half full
            if (page->lessThanHalfFull()) {
                page->child->kill();
            }
            else if (page->child->child) {
                page->child->child->kill();
            }
        }
    }

    __attribute__((noinline, cold))
    static void
    popPageDebug(void *token, AutoreleasePoolPage *page, id *stop)
    {
        popPage<true>(token, page, stop);
    }

    static inline void
    pop(void *token)
    {
        AutoreleasePoolPage *page;
        id *stop;
        if (token == (void*)EMPTY_POOL_PLACEHOLDER) {
            // Popping the top-level placeholder pool.
            page = hotPage();
            if (!page) {
                // Pool was never used. Clear the placeholder.
                return setHotPage(nil);
            }
            // Pool was used. Pop its contents normally.
            // Pool pages remain allocated for re-use as usual.
            page = coldPage();
            token = page->begin();
        } else {
            page = pageForPointer(token);
        }

        stop = (id *)token;
        if (*stop != POOL_BOUNDARY) {
            if (stop == page->begin()  &&  !page->parent) {
                // Start of coldest page may correctly not be POOL_BOUNDARY:
                // 1. top-level pool is popped, leaving the cold page in place
                // 2. an object is autoreleased with no pool
            } else {
                // Error. For bincompat purposes this is not 
                // fatal in executables built with old SDKs.
                return badPop(token);
            }
        }

        if (slowpath(PrintPoolHiwat || DebugPoolAllocation || DebugMissingPools)) {
            return popPageDebug(token, page, stop);
        }

        return popPage<false>(token, page, stop);
    }

    static void init()
    {
        int r __unused = pthread_key_init_np(AutoreleasePoolPage::key, 
                                             AutoreleasePoolPage::tls_dealloc);
        ASSERT(r == 0);
    }

    
    /*
        打印函数部分的代码忽略
    */

#undef POOL_BOUNDARY
};
```

&emsp;&emsp;即使忽略了部分，代码部分也还是有点长，这里从入栈到出栈的角度进行描述（`push` -> `pop`）

1. 初始化处理：`static inline void *push()`，就是`void *objc_autoreleasePoolPush(void)`中的实现，在正常的设备运行流程中，调用`static inline id *autoreleaseFast(id obj)`函数，参数是`POOL_BOUNDARY`，然后返回`EMPTY_POOL_PLACEHOLDER`，即自动释放池空占位符
2. 入栈前处理：按照入栈场景有3种情况
   1. 已初始化自动释放池页，且页内栈不满：直接将实例对象入栈
   2. 已初始化自动释放池页，但栈已满的情况下：通过双向链表遍历，找到不满的自动释放池页（如果都是满的，就新建一页），并设置为`hotPage`（当前线程处理中的页，tls，即`Thread Local Storage`），最后将实例对象入栈
   3. 自动释放池页未初始化的情况下（这是最完整的自动释放池页入栈流程），先查当前线程是否有空的内存释放池占位符`EMPTY_POOL_PLACEHOLDER`
      1. 没有且没有打开自动释放池调试环境变量，则会先设置并返回这个内存释放池占位符`EMPTY_POOL_PLACEHOLDER`
      2. 有就代表自动释放池页是空栈，在创建了第一个自动释放池页并设置为`hotPage`后，如果知道这是空栈，那么页中的栈低就需要添加一个哨兵（`sentinel`，即`POOL_BOUNDARY`，也就是`nil`）对象，用于标识栈底，最后将实例对象入栈
3. 入栈时处理：通过`id *add(id obj)`函数入栈，并将栈顶指针`next`指向刚入栈的实例对象
4. 出栈前处理：`static inline void pop(void *token)`，就是`void _objc_autoreleasePoolPop(void *ctxt)`中的实现，函数的参数对应追踪可知，是实例对象在入栈后的栈顶，在出栈函数调用时，需要通过栈顶找到自动释放池页，此处也分3种情况
   1. `token`是自动释放池占位符，即栈是空的
      1. 如果从未有实例对象入栈，那就是空的，置空`hotPage`，不需要更多处理（这种情况通常是花括号里为空的时候）
      2. 如果已有实例对象入栈，但是在自动释放池结束前已经出栈，就准备在`codePage`内出栈，直到`begin`，可以理解为第一个自动释放池内全部出栈，直到空栈（不包括哨兵`POOL_BOUNDARY`），空栈后自动释放池页仍会保留作重用
   2. `token`不是自动释放池占位符，栈内有需要实例对象指针，通过`static AutoreleasePoolPage *pageForPointer(uintptr_t p) `与`token`获得自动释放池页，此处需要注意一个当前版本设计得非常精妙的处理
      1. 在首次调用`push`函数的时候，因为需要初始化自动释放池页，所以走的是`autoreleaseNoPage(POOL_BOUNDARY)`，返回的是内存释放池占位符`EMPTY_POOL_PLACEHOLDER`，即`((id*)1)`
      2. 这个内存释放池占位符`EMPTY_POOL_PLACEHOLDER`指向的地址是没有数据的，所以在`pop`函数内部的`stop = (id *)token;`获得的值与`POOL_BOUNDARY`相同，为`nil`，所以接下来想通的出栈操作不相悖
5. 出栈时处理：通过`template<bool allowDebug> static void popPage(void *token, AutoreleasePoolPage *page, id *stop)`函数同时处理出栈与自动释放池页的删除
   1. 首先用`releaseUntil`循环出栈每一个池子中的实例对象指针，需要注意的是循环内单独判断了哨兵不出栈，并且因为哨兵`POOL_BOUNDARY`的值为`nil`，所以出栈操作是以`SCRIBBLE`，即`0xA3`来赋值，字面意思是涂掉
   2. 然后判断当前自动释放池页的容量，如果少于一半就删除双向链表中的下个节点，否则保留
6. 除了日常写代码时接触到的自动释放池使用，在`Runloop`中也有调用，其释放调用的是`static void tls_dealloc(void *p) `，具体细节在`CF`篇章再深究

# 4. ObjC初始化流程

&emsp;&emsp;对于微观与宏观，再来一次串联，看ObjC的初始化过程中的处理：

```ObjC
/***********************************************************************
* _objc_init
* Bootstrap initialization. Registers our image notifier with dyld.
* Called by libSystem BEFORE library initialization time
**********************************************************************/

void _objc_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
    
    // fixme defer initialization until an objc-using image is found?
    environ_init();
    tls_init();
    static_init();
    runtime_init();
    exception_init();
    cache_init();
    _imp_implementationWithBlock_init();

    _dyld_objc_notify_register(&map_images, load_images, unmap_image);

#if __OBJC2__
    didCallDyldNotifyRegister = true;
#endif
}
```

1. `environ_init()`，初始化运行时环境变量，环境变量在`objc-env.h`中有列出，遍历并根据布尔值处理

```ObjC
// Settings from environment variables
#define OPTION(var, env, help) bool var = false;
#include "objc-env.h"
#undef OPTION

struct option_t {
    bool* var;
    const char *env;
    const char *help;
    size_t envlen;
};

const option_t Settings[] = {
#define OPTION(var, env, help) option_t{&var, #env, help, strlen(#env)}, 
#include "objc-env.h"
#undef OPTION
};
```

2. `tls_init()`，Thread Local Storage的初始化

```ObjC
void tls_init(void)
{
#if SUPPORT_DIRECT_THREAD_KEYS
    pthread_key_init_np(TLS_DIRECT_KEY, &_objc_pthread_destroyspecific);
#else
    _objc_pthread_key = tls_create(&_objc_pthread_destroyspecific);
#endif
}
```

3. `static_init()`，C++静态构造函数的初始化

```ObjC
/***********************************************************************
* static_init
* Run C++ static constructor functions.
* libc calls _objc_init() before dyld would call our static constructors, 
* so we have to do it ourselves.
**********************************************************************/
static void static_init()
{
    size_t count;
    auto inits = getLibobjcInitializers(&_mh_dylib_header, &count);
    for (size_t i = 0; i < count; i++) {
        inits[i]();
    }
}
```

1. `runtime_init()`，运行时初始化，Category和ObjC类的表初始化

```ObjC

class UnattachedCategories : public ExplicitInitDenseMap<Class, category_list> { /* ... */ } 
static UnattachedCategories unattachedCategories;

/***********************************************************************
* allocatedClasses
* A table of all classes (and metaclasses) which have been allocated
* with objc_allocateClassPair.
**********************************************************************/
namespace objc {
static ExplicitInitDenseSet<Class> allocatedClasses;
}

void runtime_init(void)
{
    objc::unattachedCategories.init(32);
    objc::allocatedClasses.init();
}
```

5. `exception_init()`，异常处理初始化，本质是指定异常处理的函数

```ObjC
/***********************************************************************
* _objc_terminate
* Custom std::terminate handler.
*
* The uncaught exception callback is implemented as a std::terminate handler. 
* 1. Check if there's an active exception
* 2. If so, check if it's an Objective-C exception
* 3. If so, call our registered callback with the object.
* 4. Finally, call the previous terminate handler.
**********************************************************************/
static void (*old_terminate)(void) = nil;
static void _objc_terminate(void)
{
    if (PrintExceptions) {
        _objc_inform("EXCEPTIONS: terminating");
    }

    if (! __cxa_current_exception_type()) {
        // No current exception.
        (*old_terminate)();
    }
    else {
        // There is a current exception. Check if it's an objc exception.
        @try {
            __cxa_rethrow();
        } @catch (id e) {
            // It's an objc object. Call Foundation's handler, if any.
            (*uncaught_handler)((id)e);
            (*old_terminate)();
        } @catch (...) {
            // It's not an objc object. Continue to C++ terminate.
            (*old_terminate)();
        }
    }
}
/***********************************************************************
* exception_init
* Initialize libobjc's exception handling system.
* Called by map_images().
**********************************************************************/
void exception_init(void)
{
    old_terminate = std::set_terminate(&_objc_terminate);
}
```

6. `cache_init()`，缓存初始化，iPhone设备中`HAVE_TASK_RESTARTABLE_RANGES`为1

```ObjC
// Define HAVE_TASK_RESTARTABLE_RANGES to enable usage of
// task_restartable_ranges_synchronize()
#if TARGET_OS_SIMULATOR || defined(__i386__) || defined(__arm__) || !TARGET_OS_MAC
#   define HAVE_TASK_RESTARTABLE_RANGES 0
#else
#   define HAVE_TASK_RESTARTABLE_RANGES 1
#endif

void cache_init()
{
#if HAVE_TASK_RESTARTABLE_RANGES
    mach_msg_type_number_t count = 0;
    kern_return_t kr;

    while (objc_restartableRanges[count].location) {
        count++;
    }

    kr = task_restartable_ranges_register(mach_task_self(),
                                          objc_restartableRanges, count);
    if (kr == KERN_SUCCESS) return;
    _objc_fatal("task_restartable_ranges_register failed (result 0x%x: %s)",
                kr, mach_error_string(kr));
#endif // HAVE_TASK_RESTARTABLE_RANGES
}
```


7. `_imp_implementationWithBlock_init()`，在iphoneOS中没处理

```OBJC
/// Initialize the trampoline machinery. Normally this does nothing, as
/// everything is initialized lazily, but for certain processes we eagerly load
/// the trampolines dylib.
void
_imp_implementationWithBlock_init(void)
{
#if TARGET_OS_OSX
    // Eagerly load libobjc-trampolines.dylib in certain processes. Some
    // programs (most notably QtWebEngineProcess used by older versions of
    // embedded Chromium) enable a highly restrictive sandbox profile which
    // blocks access to that dylib. If anything calls
    // imp_implementationWithBlock (as AppKit has started doing) then we'll
    // crash trying to load it. Loading it here sets it up before the sandbox
    // profile is enabled and blocks it.
    //
    // This fixes EA Origin (rdar://problem/50813789)
    // and Steam (rdar://problem/55286131)
    if (__progname &&
        (strcmp(__progname, "QtWebEngineProcess") == 0 ||
         strcmp(__progname, "Steam Helper") == 0)) {
        Trampolines.Initialize();
    }
#endif
}
```

8. `_dyld_objc_notify_register(&map_images, load_images, unmap_image)`，调用`dyld`提供的API，参数是函数指针，用于调用`map_images`、`load_images`和`unmap_image`函数

&emsp;&emsp;这里再对宏观中的几个表做一个入口总览，在首次初始化镜像的时候，初始化流程会先把3个用于管理内存的散列表初始化，调用顺序如下：

1. `void map_images(unsigned count, const char * const paths[], const struct mach_header * const mhdrs[])`
2. `void map_images_nolock(unsigned mhCount, const char * const mhPaths[], const struct mach_header * const mhdrs[])`
3. `void arr_init(void) `

&emsp;&emsp;`void arr_init(void) `函数中有对`AutoreleasePoolPage`、`SideTableMap`和`AssociationsManager`的初始化，它们都是统一用于管理各方面的散列表，首次初始化镜像时初始化构成了管理运行时内存的基础
```ObjC
void arr_init(void) 
{
    AutoreleasePoolPage::init();
    SideTablesMap.init();
    _objc_associations_init();
}
```