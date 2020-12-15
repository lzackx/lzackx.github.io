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
&emsp;&emsp;涉及到的`objc`的设计概念，如下：

1. [Tagged Pointer](https://blog.devtang.com/2014/05/30/understand-tagged-pointer/)

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

- (BOOL)isEqual:(id)object;
@property (readonly) NSUInteger hash;

@property (readonly) Class superclass;
- (Class)class OBJC_SWIFT_UNAVAILABLE("use 'type(of: anObject)' instead");
- (instancetype)self;

- (id)performSelector:(SEL)aSelector;
- (id)performSelector:(SEL)aSelector withObject:(id)object;
- (id)performSelector:(SEL)aSelector withObject:(id)object1 withObject:(id)object2;

- (BOOL)isProxy;

- (BOOL)isKindOfClass:(Class)aClass;
- (BOOL)isMemberOfClass:(Class)aClass;
- (BOOL)conformsToProtocol:(Protocol *)aProtocol;

- (BOOL)respondsToSelector:(SEL)aSelector;

- (instancetype)retain OBJC_ARC_UNAVAILABLE;
- (oneway void)release OBJC_ARC_UNAVAILABLE;
- (instancetype)autorelease OBJC_ARC_UNAVAILABLE;
- (NSUInteger)retainCount OBJC_ARC_UNAVAILABLE;

- (struct _NSZone *)zone OBJC_ARC_UNAVAILABLE;

@property (readonly, copy) NSString *description;
@optional
@property (readonly, copy) NSString *debugDescription;

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

+ (void)load;

+ (void)initialize;
- (instancetype)init
#if NS_ENFORCE_NSOBJECT_DESIGNATED_INITIALIZER
    NS_DESIGNATED_INITIALIZER
#endif
    ;

+ (instancetype)new OBJC_SWIFT_UNAVAILABLE("use object initializers instead");
+ (instancetype)allocWithZone:(struct _NSZone *)zone OBJC_SWIFT_UNAVAILABLE("use object initializers instead");
+ (instancetype)alloc OBJC_SWIFT_UNAVAILABLE("use object initializers instead");
- (void)dealloc OBJC_SWIFT_UNAVAILABLE("use 'deinit' to define a de-initializer");

- (void)finalize OBJC_DEPRECATED("Objective-C garbage collection is no longer supported");

- (id)copy;
- (id)mutableCopy;

+ (id)copyWithZone:(struct _NSZone *)zone OBJC_ARC_UNAVAILABLE;
+ (id)mutableCopyWithZone:(struct _NSZone *)zone OBJC_ARC_UNAVAILABLE;

+ (BOOL)instancesRespondToSelector:(SEL)aSelector;
+ (BOOL)conformsToProtocol:(Protocol *)protocol;
- (IMP)methodForSelector:(SEL)aSelector;
+ (IMP)instanceMethodForSelector:(SEL)aSelector;
- (void)doesNotRecognizeSelector:(SEL)aSelector;

- (id)forwardingTargetForSelector:(SEL)aSelector OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);
- (void)forwardInvocation:(NSInvocation *)anInvocation OBJC_SWIFT_UNAVAILABLE("");
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector OBJC_SWIFT_UNAVAILABLE("");

+ (NSMethodSignature *)instanceMethodSignatureForSelector:(SEL)aSelector OBJC_SWIFT_UNAVAILABLE("");

- (BOOL)allowsWeakReference UNAVAILABLE_ATTRIBUTE;
- (BOOL)retainWeakReference UNAVAILABLE_ATTRIBUTE;

+ (BOOL)isSubclassOfClass:(Class)aClass;

+ (BOOL)resolveClassMethod:(SEL)sel OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);
+ (BOOL)resolveInstanceMethod:(SEL)sel OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);

+ (NSUInteger)hash;
+ (Class)superclass;
+ (Class)class OBJC_SWIFT_UNAVAILABLE("use 'aClass.self' instead");
+ (NSString *)description;
+ (NSString *)debugDescription;

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

+ (void)load {
    if (UseGC) gc_init2();
}

+ (void)initialize {
}

+ (id)self {
    return (id)self;
}

- (id)self {
    return self;
}

+ (Class)class {
    return self;
}

- (Class)class {
    return object_getClass(self);
}

+ (Class)superclass {
    return self->superclass;
}

- (Class)superclass {
    return [self class]->superclass;
}

+ (BOOL)isMemberOfClass:(Class)cls {
    return object_getClass((id)self) == cls;
}

- (BOOL)isMemberOfClass:(Class)cls {
    return [self class] == cls;
}

+ (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = object_getClass((id)self); tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}

- (BOOL)isKindOfClass:(Class)cls {
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}

+ (BOOL)isSubclassOfClass:(Class)cls {
    for (Class tcls = self; tcls; tcls = tcls->superclass) {
        if (tcls == cls) return YES;
    }
    return NO;
}

+ (BOOL)isAncestorOfObject:(NSObject *)obj {
    for (Class tcls = [obj class]; tcls; tcls = tcls->superclass) {
        if (tcls == self) return YES;
    }
    return NO;
}

+ (BOOL)instancesRespondToSelector:(SEL)sel {
    if (!sel) return NO;
    return class_respondsToSelector(self, sel);
}

+ (BOOL)respondsToSelector:(SEL)sel {
    if (!sel) return NO;
    return class_respondsToSelector_inst(object_getClass(self), sel, self);
}

- (BOOL)respondsToSelector:(SEL)sel {
    if (!sel) return NO;
    return class_respondsToSelector_inst([self class], sel, self);
}

+ (BOOL)conformsToProtocol:(Protocol *)protocol {
    if (!protocol) return NO;
    for (Class tcls = self; tcls; tcls = tcls->superclass) {
        if (class_conformsToProtocol(tcls, protocol)) return YES;
    }
    return NO;
}

- (BOOL)conformsToProtocol:(Protocol *)protocol {
    if (!protocol) return NO;
    for (Class tcls = [self class]; tcls; tcls = tcls->superclass) {
        if (class_conformsToProtocol(tcls, protocol)) return YES;
    }
    return NO;
}

+ (NSUInteger)hash {
    return _objc_rootHash(self);
}

- (NSUInteger)hash {
    return _objc_rootHash(self);
}

+ (BOOL)isEqual:(id)obj {
    return obj == (id)self;
}

- (BOOL)isEqual:(id)obj {
    return obj == self;
}


+ (BOOL)isFault {
    return NO;
}

- (BOOL)isFault {
    return NO;
}

+ (BOOL)isProxy {
    return NO;
}

- (BOOL)isProxy {
    return NO;
}


+ (IMP)instanceMethodForSelector:(SEL)sel {
    if (!sel) [self doesNotRecognizeSelector:sel];
    return class_getMethodImplementation(self, sel);
}

+ (IMP)methodForSelector:(SEL)sel {
    if (!sel) [self doesNotRecognizeSelector:sel];
    return object_getMethodImplementation((id)self, sel);
}

- (IMP)methodForSelector:(SEL)sel {
    if (!sel) [self doesNotRecognizeSelector:sel];
    return object_getMethodImplementation(self, sel);
}

+ (BOOL)resolveClassMethod:(SEL)sel {
    return NO;
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    return NO;
}

// Replaced by CF (throws an NSException)
+ (void)doesNotRecognizeSelector:(SEL)sel {
    _objc_fatal("+[%s %s]: unrecognized selector sent to instance %p", 
                class_getName(self), sel_getName(sel), self);
}

// Replaced by CF (throws an NSException)
- (void)doesNotRecognizeSelector:(SEL)sel {
    _objc_fatal("-[%s %s]: unrecognized selector sent to instance %p", 
                object_getClassName(self), sel_getName(sel), self);
}


+ (id)performSelector:(SEL)sel {
    if (!sel) [self doesNotRecognizeSelector:sel];
    return ((id(*)(id, SEL))objc_msgSend)((id)self, sel);
}

+ (id)performSelector:(SEL)sel withObject:(id)obj {
    if (!sel) [self doesNotRecognizeSelector:sel];
    return ((id(*)(id, SEL, id))objc_msgSend)((id)self, sel, obj);
}

+ (id)performSelector:(SEL)sel withObject:(id)obj1 withObject:(id)obj2 {
    if (!sel) [self doesNotRecognizeSelector:sel];
    return ((id(*)(id, SEL, id, id))objc_msgSend)((id)self, sel, obj1, obj2);
}

- (id)performSelector:(SEL)sel {
    if (!sel) [self doesNotRecognizeSelector:sel];
    return ((id(*)(id, SEL))objc_msgSend)(self, sel);
}

- (id)performSelector:(SEL)sel withObject:(id)obj {
    if (!sel) [self doesNotRecognizeSelector:sel];
    return ((id(*)(id, SEL, id))objc_msgSend)(self, sel, obj);
}

- (id)performSelector:(SEL)sel withObject:(id)obj1 withObject:(id)obj2 {
    if (!sel) [self doesNotRecognizeSelector:sel];
    return ((id(*)(id, SEL, id, id))objc_msgSend)(self, sel, obj1, obj2);
}


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


+ (id)new {
    return [callAlloc(self, false/*checkNil*/) init];
}

+ (id)retain {
    return (id)self;
}

// Replaced by ObjectAlloc
- (id)retain {
    return ((id)self)->rootRetain();
}


+ (BOOL)_tryRetain {
    return YES;
}

// Replaced by ObjectAlloc
- (BOOL)_tryRetain {
    return ((id)self)->rootTryRetain();
}

+ (BOOL)_isDeallocating {
    return NO;
}

- (BOOL)_isDeallocating {
    return ((id)self)->rootIsDeallocating();
}

+ (BOOL)allowsWeakReference { 
    return YES; 
}

+ (BOOL)retainWeakReference { 
    return YES; 
}

- (BOOL)allowsWeakReference { 
    return ! [self _isDeallocating]; 
}

- (BOOL)retainWeakReference { 
    return [self _tryRetain]; 
}

+ (oneway void)release {
}

// Replaced by ObjectAlloc
- (oneway void)release {
    ((id)self)->rootRelease();
}

+ (id)autorelease {
    return (id)self;
}

// Replaced by ObjectAlloc
- (id)autorelease {
    return ((id)self)->rootAutorelease();
}

+ (NSUInteger)retainCount {
    return ULONG_MAX;
}

- (NSUInteger)retainCount {
    return ((id)self)->rootRetainCount();
}

+ (id)alloc {
    return _objc_rootAlloc(self);
}

// Replaced by ObjectAlloc
+ (id)allocWithZone:(struct _NSZone *)zone {
    return _objc_rootAllocWithZone(self, (malloc_zone_t *)zone);
}

// Replaced by CF (throws an NSException)
+ (id)init {
    return (id)self;
}

- (id)init {
    return _objc_rootInit(self);
}

// Replaced by CF (throws an NSException)
+ (void)dealloc {
}


// Replaced by NSZombies
- (void)dealloc {
    _objc_rootDealloc(self);
}

// Replaced by CF (throws an NSException)
+ (void)finalize {
}

- (void)finalize {
    _objc_rootFinalize(self);
}

+ (struct _NSZone *)zone {
    return (struct _NSZone *)_objc_rootZone(self);
}

- (struct _NSZone *)zone {
    return (struct _NSZone *)_objc_rootZone(self);
}

+ (id)copy {
    return (id)self;
}

+ (id)copyWithZone:(struct _NSZone *)zone {
    return (id)self;
}

- (id)copy {
    return [(id)self copyWithZone:nil];
}

+ (id)mutableCopy {
    return (id)self;
}

+ (id)mutableCopyWithZone:(struct _NSZone *)zone {
    return (id)self;
}

- (id)mutableCopy {
    return [(id)self mutableCopyWithZone:nil];
}

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

&emsp;&emsp;`objc_object`结构体内的函数，内联部分实现位于`objc-object.h`文件中（引入于`objc-private.h`文件末尾），剩余部分位于`NSObject.mm`文件中。

&emsp;&emsp;先来看`objc-private.h`中的声明，如下，从声明中，可以关注到几点：

1. `isa`是`struct isa_t`类型的私有变量
2. 内存管理相关的函数
3. `sidetable`的用处

```ObjC
struct objc_object {
private:
    isa_t isa;

public:

    // ISA() assumes this is NOT a tagged pointer object
    Class ISA();

    // getIsa() allows this to be a tagged pointer object
    Class getIsa();

    // initIsa() should be used to init the isa of new objects only.
    // If this object already has an isa, use changeIsa() for correctness.
    // initInstanceIsa(): objects with no custom RR/AWZ
    // initClassIsa(): class objects
    // initProtocolIsa(): protocol objects
    // initIsa(): other objects
    void initIsa(Class cls /*indexed=false*/);
    void initClassIsa(Class cls /*indexed=maybe*/);
    void initProtocolIsa(Class cls /*indexed=maybe*/);
    void initInstanceIsa(Class cls, bool hasCxxDtor);

    // changeIsa() should be used to change the isa of existing objects.
    // If this is a new object, use initIsa() for performance.
    Class changeIsa(Class newCls);

    bool hasIndexedIsa();
    bool isTaggedPointer();
    bool isClass();

    // object may have associated objects?
    bool hasAssociatedObjects();
    void setHasAssociatedObjects();

    // object may be weakly referenced?
    bool isWeaklyReferenced();
    void setWeaklyReferenced_nolock();

    // object may have -.cxx_destruct implementation?
    bool hasCxxDtor();

    // Optimized calls to retain/release methods
    id retain();
    void release();
    id autorelease();

    // Implementations of retain/release methods
    id rootRetain();
    bool rootRelease();
    id rootAutorelease();
    bool rootTryRetain();
    bool rootReleaseShouldDealloc();
    uintptr_t rootRetainCount();

    // Implementation of dealloc methods
    bool rootIsDeallocating();
    void clearDeallocating();
    void rootDealloc();

private:
    void initIsa(Class newCls, bool indexed, bool hasCxxDtor);

    // Slow paths for inline control
    id rootAutorelease2();
    bool overrelease_error();

#if SUPPORT_NONPOINTER_ISA
    // Unified retain count manipulation for nonpointer isa
    id rootRetain(bool tryRetain, bool handleOverflow);
    bool rootRelease(bool performDealloc, bool handleUnderflow);
    id rootRetain_overflow(bool tryRetain);
    bool rootRelease_underflow(bool performDealloc);

    void clearDeallocating_slow();

    // Side table retain count overflow for nonpointer isa
    void sidetable_lock();
    void sidetable_unlock();

    void sidetable_moveExtraRC_nolock(size_t extra_rc, bool isDeallocating, bool weaklyReferenced);
    bool sidetable_addExtraRC_nolock(size_t delta_rc);
    size_t sidetable_subExtraRC_nolock(size_t delta_rc);
    size_t sidetable_getExtraRC_nolock();
#endif

    // Side-table-only retain count
    bool sidetable_isDeallocating();
    void sidetable_clearDeallocating();

    bool sidetable_isWeaklyReferenced();
    void sidetable_setWeaklyReferenced_nolock();

    id sidetable_retain();
    id sidetable_retain_slow(SideTable& table);

    uintptr_t sidetable_release(bool performDealloc = true);
    uintptr_t sidetable_release_slow(SideTable& table, bool performDealloc = true);

    bool sidetable_tryRetain();

    uintptr_t sidetable_retainCount();
#if DEBUG
    bool sidetable_present();
#endif
};

// Inlined parts of objc_object's implementation
#include "objc-object.h"

```

&emsp;&emsp;`objc-object.h`中的`objc_object`内联部分函数实现如下：

```ObjC
inline bool
objc_object::isClass()
{
    if (isTaggedPointer()) return false;
    return ISA()->isMetaClass();
}

#if SUPPORT_NONPOINTER_ISA

#   if !SUPPORT_TAGGED_POINTERS
#       error sorry
#   endif


inline Class 
objc_object::ISA() 
{
    assert(!isTaggedPointer()); 
    return (Class)(isa.bits & ISA_MASK);
}


inline bool 
objc_object::hasIndexedIsa()
{
    return isa.indexed;
}

inline Class 
objc_object::getIsa() 
{
    if (isTaggedPointer()) {
        uintptr_t slot = ((uintptr_t)this >> TAG_SLOT_SHIFT) & TAG_SLOT_MASK;
        return objc_tag_classes[slot];
    }
    return ISA();
}


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


inline Class 
objc_object::changeIsa(Class newCls)
{
    // This is almost always rue but there are 
    // enough edge cases that we can't assert it.
    // assert(newCls->isFuture()  || 
    //        newCls->isInitializing()  ||  newCls->isInitialized());

    assert(!isTaggedPointer()); 

    isa_t oldisa;
    isa_t newisa;

    bool sideTableLocked = false;
    bool transcribeToSideTable = false;

    do {
        transcribeToSideTable = false;
        oldisa = LoadExclusive(&isa.bits);
        if ((oldisa.bits == 0  ||  oldisa.indexed)  &&
            !newCls->isFuture()  &&  newCls->canAllocIndexed())
        {
            // 0 -> indexed
            // indexed -> indexed
            if (oldisa.bits == 0) newisa.bits = ISA_MAGIC_VALUE;
            else newisa = oldisa;
            // isa.magic is part of ISA_MAGIC_VALUE
            // isa.indexed is part of ISA_MAGIC_VALUE
            newisa.has_cxx_dtor = newCls->hasCxxDtor();
            newisa.shiftcls = (uintptr_t)newCls >> 3;
        }
        else if (oldisa.indexed) {
            // indexed -> not indexed
            // Need to copy retain count et al to side table.
            // Acquire side table lock before setting isa to 
            // prevent races such as concurrent -release.
            if (!sideTableLocked) sidetable_lock();
            sideTableLocked = true;
            transcribeToSideTable = true;
            newisa.cls = newCls;
        }
        else {
            // not indexed -> not indexed
            newisa.cls = newCls;
        }
    } while (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits));

    if (transcribeToSideTable) {
        // Copy oldisa's retain count et al to side table.
        // oldisa.weakly_referenced: nothing to do
        // oldisa.has_assoc: nothing to do
        // oldisa.has_cxx_dtor: nothing to do
        sidetable_moveExtraRC_nolock(oldisa.extra_rc, 
                                     oldisa.deallocating, 
                                     oldisa.weakly_referenced);
    }

    if (sideTableLocked) sidetable_unlock();

    Class oldCls;
    if (oldisa.indexed) oldCls = (Class)((uintptr_t)oldisa.shiftcls << 3);
    else oldCls = oldisa.cls;
    
    return oldCls;
}


inline bool 
objc_object::isTaggedPointer() 
{
    return ((uintptr_t)this & TAG_MASK);
}


inline bool
objc_object::hasAssociatedObjects()
{
    if (isTaggedPointer()) return true;
    if (isa.indexed) return isa.has_assoc;
    return true;
}


inline void
objc_object::setHasAssociatedObjects()
{
    if (isTaggedPointer()) return;

 retry:
    isa_t oldisa = LoadExclusive(&isa.bits);
    isa_t newisa = oldisa;
    if (!newisa.indexed) return;
    if (newisa.has_assoc) return;
    newisa.has_assoc = true;
    if (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)) goto retry;
}


inline bool
objc_object::isWeaklyReferenced()
{
    assert(!isTaggedPointer());
    if (isa.indexed) return isa.weakly_referenced;
    else return sidetable_isWeaklyReferenced();
}


inline void
objc_object::setWeaklyReferenced_nolock()
{
 retry:
    isa_t oldisa = LoadExclusive(&isa.bits);
    isa_t newisa = oldisa;
    if (!newisa.indexed) return sidetable_setWeaklyReferenced_nolock();
    if (newisa.weakly_referenced) return;
    newisa.weakly_referenced = true;
    if (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)) goto retry;
}


inline bool
objc_object::hasCxxDtor()
{
    assert(!isTaggedPointer());
    if (isa.indexed) return isa.has_cxx_dtor;
    else return isa.cls->hasCxxDtor();
}



inline bool 
objc_object::rootIsDeallocating()
{
    assert(!UseGC);

    if (isTaggedPointer()) return false;
    if (isa.indexed) return isa.deallocating;
    return sidetable_isDeallocating();
}


inline void 
objc_object::clearDeallocating()
{
    if (!isa.indexed) {
        // Slow path for raw pointer isa.
        sidetable_clearDeallocating();
    }
    else if (isa.weakly_referenced  ||  isa.has_sidetable_rc) {
        // Slow path for non-pointer isa with weak refs and/or side table data.
        clearDeallocating_slow();
    }

    assert(!sidetable_present());
}


inline void
objc_object::rootDealloc()
{
    assert(!UseGC);
    if (isTaggedPointer()) return;

    if (isa.indexed  &&  
        !isa.weakly_referenced  &&  
        !isa.has_assoc  &&  
        !isa.has_cxx_dtor  &&  
        !isa.has_sidetable_rc)
    {
        assert(!sidetable_present());
        free(this);
    } 
    else {
        object_dispose((id)this);
    }
}


// Equivalent to calling [this retain], with shortcuts if there is no override
inline id 
objc_object::retain()
{
    // UseGC is allowed here, but requires hasCustomRR.
    assert(!UseGC  ||  ISA()->hasCustomRR());
    assert(!isTaggedPointer());

    if (! ISA()->hasCustomRR()) {
        return rootRetain();
    }

    return ((id(*)(objc_object *, SEL))objc_msgSend)(this, SEL_retain);
}


// Base retain implementation, ignoring overrides.
// This does not check isa.fast_rr; if there is an RR override then 
// it was already called and it chose to call [super retain].
//
// tryRetain=true is the -_tryRetain path.
// handleOverflow=false is the frameless fast path.
// handleOverflow=true is the framed slow path including overflow to side table
// The code is structured this way to prevent duplication.

ALWAYS_INLINE id 
objc_object::rootRetain()
{
    return rootRetain(false, false);
}

ALWAYS_INLINE bool 
objc_object::rootTryRetain()
{
    return rootRetain(true, false) ? true : false;
}

ALWAYS_INLINE id 
objc_object::rootRetain(bool tryRetain, bool handleOverflow)
{
    assert(!UseGC);
    if (isTaggedPointer()) return (id)this;

    bool sideTableLocked = false;
    bool transcribeToSideTable = false;

    isa_t oldisa;
    isa_t newisa;

    do {
        transcribeToSideTable = false;
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;
        if (!newisa.indexed) goto unindexed;
        // don't check newisa.fast_rr; we already called any RR overrides
        if (tryRetain && newisa.deallocating) goto tryfail;
        uintptr_t carry;
        newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc++

        if (carry) {
            // newisa.extra_rc++ overflowed
            if (!handleOverflow) return rootRetain_overflow(tryRetain);
            // Leave half of the retain counts inline and 
            // prepare to copy the other half to the side table.
            if (!tryRetain && !sideTableLocked) sidetable_lock();
            sideTableLocked = true;
            transcribeToSideTable = true;
            newisa.extra_rc = RC_HALF;
            newisa.has_sidetable_rc = true;
        }
    } while (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits));

    if (transcribeToSideTable) {
        // Copy the other half of the retain counts to the side table.
        sidetable_addExtraRC_nolock(RC_HALF);
    }

    if (!tryRetain && sideTableLocked) sidetable_unlock();
    return (id)this;

 tryfail:
    if (!tryRetain && sideTableLocked) sidetable_unlock();
    return nil;

 unindexed:
    if (!tryRetain && sideTableLocked) sidetable_unlock();
    if (tryRetain) return sidetable_tryRetain() ? (id)this : nil;
    else return sidetable_retain();
}


// Equivalent to calling [this release], with shortcuts if there is no override
inline void
objc_object::release()
{
    // UseGC is allowed here, but requires hasCustomRR.
    assert(!UseGC  ||  ISA()->hasCustomRR());
    assert(!isTaggedPointer());

    if (! ISA()->hasCustomRR()) {
        rootRelease();
        return;
    }

    ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_release);
}


// Base release implementation, ignoring overrides.
// Does not call -dealloc.
// Returns true if the object should now be deallocated.
// This does not check isa.fast_rr; if there is an RR override then 
// it was already called and it chose to call [super release].
// 
// handleUnderflow=false is the frameless fast path.
// handleUnderflow=true is the framed slow path including side table borrow
// The code is structured this way to prevent duplication.

ALWAYS_INLINE bool 
objc_object::rootRelease()
{
    return rootRelease(true, false);
}

ALWAYS_INLINE bool 
objc_object::rootReleaseShouldDealloc()
{
    return rootRelease(false, false);
}

ALWAYS_INLINE bool 
objc_object::rootRelease(bool performDealloc, bool handleUnderflow)
{
    assert(!UseGC);
    if (isTaggedPointer()) return false;

    bool sideTableLocked = false;

    isa_t oldisa;
    isa_t newisa;

 retry:
    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;
        if (!newisa.indexed) goto unindexed;
        // don't check newisa.fast_rr; we already called any RR overrides
        uintptr_t carry;
        newisa.bits = subc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc--
        if (carry) goto underflow;
    } while (!StoreReleaseExclusive(&isa.bits, oldisa.bits, newisa.bits));

    if (sideTableLocked) sidetable_unlock();
    return false;

 underflow:
    // newisa.extra_rc-- underflowed: borrow from side table or deallocate

    // abandon newisa to undo the decrement
    newisa = oldisa;

    if (newisa.has_sidetable_rc) {
        if (!handleUnderflow) {
            return rootRelease_underflow(performDealloc);
        }

        // Transfer retain count from side table to inline storage.

        if (!sideTableLocked) {
            sidetable_lock();
            sideTableLocked = true;
            if (!isa.indexed) {
                // Lost a race vs the indexed -> not indexed transition
                // before we got the side table lock. Stop now to avoid 
                // breaking the safety checks in the sidetable ExtraRC code.
                goto unindexed;
            }
        }

        // Try to remove some retain counts from the side table.        
        size_t borrowed = sidetable_subExtraRC_nolock(RC_HALF);

        // To avoid races, has_sidetable_rc must remain set 
        // even if the side table count is now zero.

        if (borrowed > 0) {
            // Side table retain count decreased.
            // Try to add them to the inline count.
            newisa.extra_rc = borrowed - 1;  // redo the original decrement too
            bool stored = StoreExclusive(&isa.bits, oldisa.bits, newisa.bits);
            if (!stored) {
                // Inline update failed. 
                // Try it again right now. This prevents livelock on LL/SC 
                // architectures where the side table access itself may have 
                // dropped the reservation.
                isa_t oldisa2 = LoadExclusive(&isa.bits);
                isa_t newisa2 = oldisa2;
                if (newisa2.indexed) {
                    uintptr_t overflow;
                    newisa2.bits = 
                        addc(newisa2.bits, RC_ONE * (borrowed-1), 0, &overflow);
                    if (!overflow) {
                        stored = StoreReleaseExclusive(&isa.bits, oldisa2.bits, 
                                                       newisa2.bits);
                    }
                }
            }

            if (!stored) {
                // Inline update failed.
                // Put the retains back in the side table.
                sidetable_addExtraRC_nolock(borrowed);
                goto retry;
            }

            // Decrement successful after borrowing from side table.
            // This decrement cannot be the deallocating decrement - the side 
            // table lock and has_sidetable_rc bit ensure that if everyone 
            // else tried to -release while we worked, the last one would block.
            sidetable_unlock();
            return false;
        }
        else {
            // Side table is empty after all. Fall-through to the dealloc path.
        }
    }

    // Really deallocate.

    if (sideTableLocked) sidetable_unlock();

    if (newisa.deallocating) {
        return overrelease_error();
    }
    newisa.deallocating = true;
    if (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)) goto retry;
    __sync_synchronize();
    if (performDealloc) {
        ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc);
    }
    return true;

 unindexed:
    if (sideTableLocked) sidetable_unlock();
    return sidetable_release(performDealloc);
}


// Equivalent to [this autorelease], with shortcuts if there is no override
inline id 
objc_object::autorelease()
{
    // UseGC is allowed here, but requires hasCustomRR.
    assert(!UseGC  ||  ISA()->hasCustomRR());

    if (isTaggedPointer()) return (id)this;
    if (! ISA()->hasCustomRR()) return rootAutorelease();

    return ((id(*)(objc_object *, SEL))objc_msgSend)(this, SEL_autorelease);
}


// Base autorelease implementation, ignoring overrides.
inline id 
objc_object::rootAutorelease()
{
    assert(!UseGC);

    if (isTaggedPointer()) return (id)this;
    if (prepareOptimizedReturn(ReturnAtPlus1)) return (id)this;

    return rootAutorelease2();
}


inline uintptr_t 
objc_object::rootRetainCount()
{
    assert(!UseGC);
    if (isTaggedPointer()) return (uintptr_t)this;

    sidetable_lock();
    isa_t bits = LoadExclusive(&isa.bits);
    if (bits.indexed) {
        uintptr_t rc = 1 + bits.extra_rc;
        if (bits.has_sidetable_rc) {
            rc += sidetable_getExtraRC_nolock();
        }
        sidetable_unlock();
        return rc;
    }

    sidetable_unlock();
    return sidetable_retainCount();
}


// SUPPORT_NONPOINTER_ISA
#else
// not SUPPORT_NONPOINTER_ISA


inline Class 
objc_object::ISA() 
{
    assert(!isTaggedPointer()); 
    return isa.cls;
}


inline bool 
objc_object::hasIndexedIsa()
{
    return false;
}


inline Class 
objc_object::getIsa() 
{
#if SUPPORT_TAGGED_POINTERS
    if (isTaggedPointer()) {
        uintptr_t slot = ((uintptr_t)this >> TAG_SLOT_SHIFT) & TAG_SLOT_MASK;
        return objc_tag_classes[slot];
    }
#endif
    return ISA();
}


inline void 
objc_object::initIsa(Class cls)
{
    assert(!isTaggedPointer()); 
    isa = (uintptr_t)cls; 
}


inline void 
objc_object::initClassIsa(Class cls)
{
    initIsa(cls);
}


inline void 
objc_object::initProtocolIsa(Class cls)
{
    initIsa(cls);
}


inline void 
objc_object::initInstanceIsa(Class cls, bool)
{
    initIsa(cls);
}


inline void 
objc_object::initIsa(Class cls, bool, bool)
{ 
    initIsa(cls);
}


inline Class 
objc_object::changeIsa(Class cls)
{
    // This is almost always rue but there are 
    // enough edge cases that we can't assert it.
    // assert(cls->isFuture()  ||  
    //        cls->isInitializing()  ||  cls->isInitialized());

    assert(!isTaggedPointer()); 
    
    isa_t oldisa, newisa;
    newisa.cls = cls;
    do {
        oldisa = LoadExclusive(&isa.bits);
    } while (!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits));
    
    if (oldisa.cls  &&  oldisa.cls->instancesHaveAssociatedObjects()) {
        cls->setInstancesHaveAssociatedObjects();
    }
    
    return oldisa.cls;
}


inline bool 
objc_object::isTaggedPointer() 
{
#if SUPPORT_TAGGED_POINTERS
    return ((uintptr_t)this & TAG_MASK);
#else
    return false;
#endif
}


inline bool
objc_object::hasAssociatedObjects()
{
    assert(!UseGC);

    return getIsa()->instancesHaveAssociatedObjects();
}


inline void
objc_object::setHasAssociatedObjects()
{
    assert(!UseGC);

    getIsa()->setInstancesHaveAssociatedObjects();
}


inline bool
objc_object::isWeaklyReferenced()
{
    assert(!isTaggedPointer());
    assert(!UseGC);

    return sidetable_isWeaklyReferenced();
}


inline void 
objc_object::setWeaklyReferenced_nolock()
{
    assert(!isTaggedPointer());
    assert(!UseGC);

    sidetable_setWeaklyReferenced_nolock();
}


inline bool
objc_object::hasCxxDtor()
{
    assert(!isTaggedPointer());
    return isa.cls->hasCxxDtor();
}


inline bool 
objc_object::rootIsDeallocating()
{
    assert(!UseGC);

    if (isTaggedPointer()) return false;
    return sidetable_isDeallocating();
}


inline void 
objc_object::clearDeallocating()
{
    sidetable_clearDeallocating();
}


inline void
objc_object::rootDealloc()
{
    if (isTaggedPointer()) return;
    object_dispose((id)this);
}


// Equivalent to calling [this retain], with shortcuts if there is no override
inline id 
objc_object::retain()
{
    // UseGC is allowed here, but requires hasCustomRR.
    assert(!UseGC  ||  ISA()->hasCustomRR());
    assert(!isTaggedPointer());

    if (! ISA()->hasCustomRR()) {
        return sidetable_retain();
    }

    return ((id(*)(objc_object *, SEL))objc_msgSend)(this, SEL_retain);
}


// Base retain implementation, ignoring overrides.
// This does not check isa.fast_rr; if there is an RR override then 
// it was already called and it chose to call [super retain].
inline id 
objc_object::rootRetain()
{
    assert(!UseGC);

    if (isTaggedPointer()) return (id)this;
    return sidetable_retain();
}


// Equivalent to calling [this release], with shortcuts if there is no override
inline void
objc_object::release()
{
    // UseGC is allowed here, but requires hasCustomRR.
    assert(!UseGC  ||  ISA()->hasCustomRR());
    assert(!isTaggedPointer());

    if (! ISA()->hasCustomRR()) {
        sidetable_release();
        return;
    }

    ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_release);
}


// Base release implementation, ignoring overrides.
// Does not call -dealloc.
// Returns true if the object should now be deallocated.
// This does not check isa.fast_rr; if there is an RR override then 
// it was already called and it chose to call [super release].
inline bool 
objc_object::rootRelease()
{
    assert(!UseGC);

    if (isTaggedPointer()) return false;
    return sidetable_release(true);
}

inline bool 
objc_object::rootReleaseShouldDealloc()
{
    if (isTaggedPointer()) return false;
    return sidetable_release(false);
}


// Equivalent to [this autorelease], with shortcuts if there is no override
inline id 
objc_object::autorelease()
{
    // UseGC is allowed here, but requires hasCustomRR.
    assert(!UseGC  ||  ISA()->hasCustomRR());

    if (isTaggedPointer()) return (id)this;
    if (! ISA()->hasCustomRR()) return rootAutorelease();

    return ((id(*)(objc_object *, SEL))objc_msgSend)(this, SEL_autorelease);
}


// Base autorelease implementation, ignoring overrides.
inline id 
objc_object::rootAutorelease()
{
    assert(!UseGC);

    if (isTaggedPointer()) return (id)this;
    if (prepareOptimizedReturn(ReturnAtPlus1)) return (id)this;

    return rootAutorelease2();
}


// Base tryRetain implementation, ignoring overrides.
// This does not check isa.fast_rr; if there is an RR override then 
// it was already called and it chose to call [super _tryRetain].
inline bool 
objc_object::rootTryRetain()
{
    assert(!UseGC);

    if (isTaggedPointer()) return true;
    return sidetable_tryRetain();
}


inline uintptr_t 
objc_object::rootRetainCount()
{
    assert(!UseGC);

    if (isTaggedPointer()) return (uintptr_t)this;
    return sidetable_retainCount();
}


// not SUPPORT_NONPOINTER_ISA
#endif
```

&emsp;&emsp;`NSObject.mm`中的`objc_object`剩余部分函数实现如下：

```ObjC
#if SUPPORT_NONPOINTER_ISA

NEVER_INLINE id 
objc_object::rootRetain_overflow(bool tryRetain)
{
    return rootRetain(tryRetain, true);
}


NEVER_INLINE bool 
objc_object::rootRelease_underflow(bool performDealloc)
{
    return rootRelease(performDealloc, true);
}


// Slow path of clearDeallocating() 
// for objects with indexed isa
// that were ever weakly referenced 
// or whose retain count ever overflowed to the side table.
NEVER_INLINE void
objc_object::clearDeallocating_slow()
{
    assert(isa.indexed  &&  (isa.weakly_referenced || isa.has_sidetable_rc));

    SideTable& table = SideTables()[this];
    table.lock();
    if (isa.weakly_referenced) {
        weak_clear_no_lock(&table.weak_table, (id)this);
    }
    if (isa.has_sidetable_rc) {
        table.refcnts.erase(this);
    }
    table.unlock();
}

#endif

__attribute__((noinline,used))
id 
objc_object::rootAutorelease2()
{
    assert(!isTaggedPointer());
    return AutoreleasePoolPage::autorelease((id)this);
}


BREAKPOINT_FUNCTION(
    void objc_overrelease_during_dealloc_error(void)
);


NEVER_INLINE
bool 
objc_object::overrelease_error()
{
    _objc_inform_now_and_on_crash("%s object %p overreleased while already deallocating; break on objc_overrelease_during_dealloc_error to debug", object_getClassName((id)this), this);
    objc_overrelease_during_dealloc_error();
    return false;  // allow rootRelease() to tail-call this
}


/***********************************************************************
* Retain count operations for side table.
**********************************************************************/


#if DEBUG
// Used to assert that an object is not present in the side table.
bool
objc_object::sidetable_present()
{
    bool result = false;
    SideTable& table = SideTables()[this];

    table.lock();

    RefcountMap::iterator it = table.refcnts.find(this);
    if (it != table.refcnts.end()) result = true;

    if (weak_is_registered_no_lock(&table.weak_table, (id)this)) result = true;

    table.unlock();

    return result;
}
#endif

#if SUPPORT_NONPOINTER_ISA

void 
objc_object::sidetable_lock()
{
    SideTable& table = SideTables()[this];
    table.lock();
}

void 
objc_object::sidetable_unlock()
{
    SideTable& table = SideTables()[this];
    table.unlock();
}


// Move the entire retain count to the side table, 
// as well as isDeallocating and weaklyReferenced.
void 
objc_object::sidetable_moveExtraRC_nolock(size_t extra_rc, 
                                          bool isDeallocating, 
                                          bool weaklyReferenced)
{
    assert(!isa.indexed);        // should already be changed to not-indexed
    SideTable& table = SideTables()[this];

    size_t& refcntStorage = table.refcnts[this];
    size_t oldRefcnt = refcntStorage;
    // not deallocating - that was in the isa
    assert((oldRefcnt & SIDE_TABLE_DEALLOCATING) == 0);  
    assert((oldRefcnt & SIDE_TABLE_WEAKLY_REFERENCED) == 0);  

    uintptr_t carry;
    size_t refcnt = addc(oldRefcnt, extra_rc << SIDE_TABLE_RC_SHIFT, 0, &carry);
    if (carry) refcnt = SIDE_TABLE_RC_PINNED;
    if (isDeallocating) refcnt |= SIDE_TABLE_DEALLOCATING;
    if (weaklyReferenced) refcnt |= SIDE_TABLE_WEAKLY_REFERENCED;

    refcntStorage = refcnt;
}


// Move some retain counts to the side table from the isa field.
// Returns true if the object is now pinned.
bool 
objc_object::sidetable_addExtraRC_nolock(size_t delta_rc)
{
    assert(isa.indexed);
    SideTable& table = SideTables()[this];

    size_t& refcntStorage = table.refcnts[this];
    size_t oldRefcnt = refcntStorage;
    // isa-side bits should not be set here
    assert((oldRefcnt & SIDE_TABLE_DEALLOCATING) == 0);
    assert((oldRefcnt & SIDE_TABLE_WEAKLY_REFERENCED) == 0);

    if (oldRefcnt & SIDE_TABLE_RC_PINNED) return true;

    uintptr_t carry;
    size_t newRefcnt = 
        addc(oldRefcnt, delta_rc << SIDE_TABLE_RC_SHIFT, 0, &carry);
    if (carry) {
        refcntStorage =
            SIDE_TABLE_RC_PINNED | (oldRefcnt & SIDE_TABLE_FLAG_MASK);
        return true;
    }
    else {
        refcntStorage = newRefcnt;
        return false;
    }
}


// Move some retain counts from the side table to the isa field.
// Returns the actual count subtracted, which may be less than the request.
size_t 
objc_object::sidetable_subExtraRC_nolock(size_t delta_rc)
{
    assert(isa.indexed);
    SideTable& table = SideTables()[this];

    RefcountMap::iterator it = table.refcnts.find(this);
    if (it == table.refcnts.end()  ||  it->second == 0) {
        // Side table retain count is zero. Can't borrow.
        return 0;
    }
    size_t oldRefcnt = it->second;

    // isa-side bits should not be set here
    assert((oldRefcnt & SIDE_TABLE_DEALLOCATING) == 0);
    assert((oldRefcnt & SIDE_TABLE_WEAKLY_REFERENCED) == 0);

    size_t newRefcnt = oldRefcnt - (delta_rc << SIDE_TABLE_RC_SHIFT);
    assert(oldRefcnt > newRefcnt);  // shouldn't underflow
    it->second = newRefcnt;
    return delta_rc;
}


size_t 
objc_object::sidetable_getExtraRC_nolock()
{
    assert(isa.indexed);
    SideTable& table = SideTables()[this];
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it == table.refcnts.end()) return 0;
    else return it->second >> SIDE_TABLE_RC_SHIFT;
}


// SUPPORT_NONPOINTER_ISA
#endif


__attribute__((used,noinline,nothrow))
id
objc_object::sidetable_retain_slow(SideTable& table)
{
#if SUPPORT_NONPOINTER_ISA
    assert(!isa.indexed);
#endif

    table.lock();
    size_t& refcntStorage = table.refcnts[this];
    if (! (refcntStorage & SIDE_TABLE_RC_PINNED)) {
        refcntStorage += SIDE_TABLE_RC_ONE;
    }
    table.unlock();

    return (id)this;
}


id
objc_object::sidetable_retain()
{
#if SUPPORT_NONPOINTER_ISA
    assert(!isa.indexed);
#endif
    SideTable& table = SideTables()[this];

    if (table.trylock()) {
        size_t& refcntStorage = table.refcnts[this];
        if (! (refcntStorage & SIDE_TABLE_RC_PINNED)) {
            refcntStorage += SIDE_TABLE_RC_ONE;
        }
        table.unlock();
        return (id)this;
    }
    return sidetable_retain_slow(table);
}


bool
objc_object::sidetable_tryRetain()
{
#if SUPPORT_NONPOINTER_ISA
    assert(!isa.indexed);
#endif
    SideTable& table = SideTables()[this];

    // NO SPINLOCK HERE
    // _objc_rootTryRetain() is called exclusively by _objc_loadWeak(), 
    // which already acquired the lock on our behalf.

    // fixme can't do this efficiently with os_lock_handoff_s
    // if (table.slock == 0) {
    //     _objc_fatal("Do not call -_tryRetain.");
    // }

    bool result = true;
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it == table.refcnts.end()) {
        table.refcnts[this] = SIDE_TABLE_RC_ONE;
    } else if (it->second & SIDE_TABLE_DEALLOCATING) {
        result = false;
    } else if (! (it->second & SIDE_TABLE_RC_PINNED)) {
        it->second += SIDE_TABLE_RC_ONE;
    }
    
    return result;
}


uintptr_t
objc_object::sidetable_retainCount()
{
    SideTable& table = SideTables()[this];

    size_t refcnt_result = 1;
    
    table.lock();
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it != table.refcnts.end()) {
        // this is valid for SIDE_TABLE_RC_PINNED too
        refcnt_result += it->second >> SIDE_TABLE_RC_SHIFT;
    }
    table.unlock();
    return refcnt_result;
}


bool 
objc_object::sidetable_isDeallocating()
{
    SideTable& table = SideTables()[this];

    // NO SPINLOCK HERE
    // _objc_rootIsDeallocating() is called exclusively by _objc_storeWeak(), 
    // which already acquired the lock on our behalf.


    // fixme can't do this efficiently with os_lock_handoff_s
    // if (table.slock == 0) {
    //     _objc_fatal("Do not call -_isDeallocating.");
    // }

    RefcountMap::iterator it = table.refcnts.find(this);
    return (it != table.refcnts.end()) && (it->second & SIDE_TABLE_DEALLOCATING);
}


bool 
objc_object::sidetable_isWeaklyReferenced()
{
    bool result = false;

    SideTable& table = SideTables()[this];
    table.lock();

    RefcountMap::iterator it = table.refcnts.find(this);
    if (it != table.refcnts.end()) {
        result = it->second & SIDE_TABLE_WEAKLY_REFERENCED;
    }

    table.unlock();

    return result;
}


void 
objc_object::sidetable_setWeaklyReferenced_nolock()
{
#if SUPPORT_NONPOINTER_ISA
    assert(!isa.indexed);
#endif

    SideTable& table = SideTables()[this];

    table.refcnts[this] |= SIDE_TABLE_WEAKLY_REFERENCED;
}


// rdar://20206767
// return uintptr_t instead of bool so that the various raw-isa 
// -release paths all return zero in eax
__attribute__((used,noinline,nothrow))
uintptr_t
objc_object::sidetable_release_slow(SideTable& table, bool performDealloc)
{
#if SUPPORT_NONPOINTER_ISA
    assert(!isa.indexed);
#endif
    bool do_dealloc = false;

    table.lock();
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it == table.refcnts.end()) {
        do_dealloc = true;
        table.refcnts[this] = SIDE_TABLE_DEALLOCATING;
    } else if (it->second < SIDE_TABLE_DEALLOCATING) {
        // SIDE_TABLE_WEAKLY_REFERENCED may be set. Don't change it.
        do_dealloc = true;
        it->second |= SIDE_TABLE_DEALLOCATING;
    } else if (! (it->second & SIDE_TABLE_RC_PINNED)) {
        it->second -= SIDE_TABLE_RC_ONE;
    }
    table.unlock();
    if (do_dealloc  &&  performDealloc) {
        ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc);
    }
    return do_dealloc;
}


// rdar://20206767 
// return uintptr_t instead of bool so that the various raw-isa 
// -release paths all return zero in eax
uintptr_t 
objc_object::sidetable_release(bool performDealloc)
{
#if SUPPORT_NONPOINTER_ISA
    assert(!isa.indexed);
#endif
    SideTable& table = SideTables()[this];

    bool do_dealloc = false;

    if (table.trylock()) {
        RefcountMap::iterator it = table.refcnts.find(this);
        if (it == table.refcnts.end()) {
            do_dealloc = true;
            table.refcnts[this] = SIDE_TABLE_DEALLOCATING;
        } else if (it->second < SIDE_TABLE_DEALLOCATING) {
            // SIDE_TABLE_WEAKLY_REFERENCED may be set. Don't change it.
            do_dealloc = true;
            it->second |= SIDE_TABLE_DEALLOCATING;
        } else if (! (it->second & SIDE_TABLE_RC_PINNED)) {
            it->second -= SIDE_TABLE_RC_ONE;
        }
        table.unlock();
        if (do_dealloc  &&  performDealloc) {
            ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc);
        }
        return do_dealloc;
    }

    return sidetable_release_slow(table, performDealloc);
}


void 
objc_object::sidetable_clearDeallocating()
{
    SideTable& table = SideTables()[this];

    // clear any weak table items
    // clear extra retain count and deallocating bit
    // (fixme warn or abort if extra retain count == 0 ?)
    table.lock();
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it != table.refcnts.end()) {
        if (it->second & SIDE_TABLE_WEAKLY_REFERENCED) {
            weak_clear_no_lock(&table.weak_table, (id)this);
        }
        table.refcnts.erase(it);
    }
    table.unlock();
}
```

# 3. 宏观

&emsp;&emsp;从`objc_object`到`NSObject`，联系都在这里。