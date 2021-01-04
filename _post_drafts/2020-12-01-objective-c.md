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
   ![isa]({{site.url}}/../../images/2020-12-01-objective-c-isa.png)
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
   1. `struct class_data_bits_t`，如下
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
   2. `struct class_rw_t`深入，如下：
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
   1. `struct class_ro_t`深入，如下，内部存放了非常多的数据，这个结构体是真正的存放ObjC中使用到的变量、属性、方法、协议等数据的地方
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

&emsp;&emsp;

# 3. 宏观

&emsp;&emsp;从`objc_object`到`NSObject`，联系都在这里。

## 3.1 Reference count

## 3.2 `SEL` & `IMP`

## 3.3 Property

## 3.4 Category

## 3.5 Association

## 3.6 Autoreleasepool