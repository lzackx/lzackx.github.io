---
layout: post
title:  "Objective-C"
date:   2020-12-01 09:00:00
categories: Objective-C
---
**Objective-C**

# 1. 前言

&emsp;&emsp;广泛地，iOS开发者都会以[`Objective-C`](https://github.com/opensource-apple/objc4)是`C++`的超集来作为这门语言的入门描述。

## 1.1. `C++`

&emsp;&emsp;在前言里，对于涉及到的`C++`特性，先描述一下。

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

# 3. 宏观

&emsp;&emsp;从`objc_object`到`NSObject`，联系都在这里。