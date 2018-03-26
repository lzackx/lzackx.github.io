---
layout: post
title:  "内存管理及其相关"
date:   2018-03-06 09:00:00
categories: Memory
---
**内存管理（Memory Management）及其相关**

本文内容包括如下内容：

1. 内存管理策略 & 实用性说明
2. 自动释放池(Auto Release Pool)
3. categories与extension
4. 属性（Property）
5. 块(Block)
6. 强引用(strong)与弱引用(weak)

* 平台：iOS
* 语言：Objective-C
* 源码地址：[源码](https://github.com/lzackx/Zone/tree/master/iOS/MemoryManagement)

# 1. 内存管理策略 & 实用性说明

&emsp;&emsp;目前内存管理提供了2中方式，即：

* MRC(Manual Reference Counting)
* ARC(Automatic Reference Counting)

&emsp;&emsp;官方推荐使用ARC，因为可以不用理会底层实现，但其实ARC就是在MRC的基础上，在编译时，加插了一些内存管理的方法在其中，从而提升开发者开发效率。

### 1.1 内存管理策略

&emsp;&emsp;内存管理模型是建立在对象拥有权的基础上的。意思就是一个对象可能会有很多个拥有者，而只要这个对象有拥有者，这个对象就会一直保留，不会释放。当没有拥有者保留时，系统会自动销毁这个对象。

&emsp;&emsp;具体遵循以下的基础规则：

&emsp;&emsp;**谁创建，谁保留**，使用类似带有`alloc`、`new`、`copy`、`mutableCopy`这些方法创建的对象的对象引用计数加一。（*`new`是C++中的操作符，不但分配内存，还调用构造函数；`alloc`会调用C中的`malloc`函数，仅分配内存，所以可以发现，通常`alloc`后，还要向初始化方法发送信息。*）


&emsp;&emsp;**`retain`可使接收者有效保留对象的拥有权**（即引用计数加一），`retain`可以在下面两个情况中使用：

* 在访问器方法或初始化方法的实现中，目的是让属性保留对象的拥有权。
* 为了防止对象在被调用时无效。

&emsp;&emsp;**当不再需要使用对象时，要释放掉拥有权（即减去引用计数）**，在代码中，体现为给对象发送`release`或`autorelease`信息。

&emsp;&emsp;**谁保留，谁释放**，拥有权的释放操作只能让保留者来做才能保证对象在使用的生命周期中正常使用。

### 1.2 实用性说明

&emsp;&emsp;通过以上的叙述，下面列出比较有代表性的代码，加强认知（该项目代码以MRC的形式运行，以方便显式表达ARC中编译器加插的方法，若需要单独需要某文件以MRC形式编译，可以在`Compiler Flags`中加上`-fno-objc-arc`）：

```
//
//  Rule.h
//  Policy
//
//  Created by lzackx on 2018/3/06.
//  Copyright © 2018年 lzackx. All rights reserved.
//

#import <Foundation/Foundation.h>

@interface Rule : NSObject

@property (retain) NSString *example;

- (instancetype)initWithExample:(NSString *)example;

@end
```

```
//
//  Rule.m
//  Policy
//
//  Created by lzackx on 2018/3/06.
//  Copyright © 2018年 lzackx. All rights reserved.
//

#import "Rule.h"

@interface Rule ()

@end

@implementation Rule

// 这里显式声明@property会做的操作
@synthesize example = _example;
- (NSString *)example {
    
    // 因为返回的是是synthesize对象，所以不需要retain或release
    return _example;
}
- (void)setExample:(NSString *)example {
    
    /*
     显然，要让synthesize对象保留参数对象，
     那么就要先拔参数传递过来的对象保留拥有权，并把旧的对象释放掉
     */
    [example retain];
    [_example release];
    _example = example;
}

- (instancetype)initWithExample:(NSString *)example {
    self = [super init];
    if (self) {
        // 在init方法里，不要使用访问器方法。
        _example = [NSString string];
    }
    return self;
}

- (void)dealloc {
    
    // 在dealloc方法里，不要使用访问器方法。
    [_example release];
    [super dealloc];
}

- (NSString *)rule {
    
    /*
     [NSString string]是工程方法，依照规则，它没有保留对象的拥有权，所以没问题。
     */
    NSString *r = [NSString string];
    
    /*
     [[NSString alloc] init]
     创建了一个对象，依照规则，它保留对象的拥有权。
     在MRC中，堆中创建了空间后，没有释放，就会出现内存泄漏。
     
    NSString *r = [[NSString alloc] init];
     */
    /*
     [[[NSString alloc] init] autorelease]
     创建了一个对象后，依照规则，它保留对象的拥有权，
     而在后面使用了autorelease，那么在使用完毕后，堆中开辟的内存就会释放，不会出现内存泄漏问题。
     这里要注意的是，用autorelease而不是release是因为这里需要作为返回值，
     而autorelease能允许方法在返回对象前，不释放对象。
     
    NSString *r = [[[NSString alloc] init] autorelease];
     */
    
    /*
     这里要注意的是error对象，它并不是由[NSString alloc]初始化的，它的初始化在方法内部，
     所以这个error对象并不需要在这里释放。
     
    NSError *error;
    NSString *r = [[[NSString alloc] initWithContentsOfFile:@"" encoding:NSUTF8StringEncoding error:&error] autorelease];
     */
    
    return r;
}

@end
```

&emsp;&emsp;以上代码基本体现了内存管理策略的底层实现，另外要提及以下的就是集合类型（如数组、字典、Set），它们内部会另外保留数据的拥有权，在创建时，并不需要开发者手动释放。但也正因为这样，如果创建了一个局部变量，并让集合类型保留它，那么这个局部变量在不需要的时候，还是要释放掉的。

# 2. 自动释放池

&emsp;&emsp;自动释放池，即`@autoreleasepool`，提供一种防止对象立即释放拥有权的机制，通常并不需要使用，但某些情况下使用，效果拔群。

&emsp;&emsp;`@autoreleasepool`是可以嵌套使用的，但日常开发中，很少遇见，但其实就是为了兼容在其内调用的一些方法又再调用的一次这种情况。

&emsp;&emsp;一个自动释放池块，会在块的最后向对象发送释放消息，有三种情况，推荐使用自动释放池：

* 非UI操作的代码中：其实UI操作也有使用自动释放池，但是UI框架中已经实现好了，不需要开发者自己实现。
* 在循环中创建过多的临时对象：如果在一个方法中的一个循环里，循环初始化多个临时对象（这些临时对象正常情况下是在方法结束后才会释放内存），一旦循环次数过多，这些对象一直占着内存，最后，在内存占用超过一定数量后，就会crash。但如果使用自动释放池块，就能有效避免内存峰值，防止这种crash。（此前，曾试过在一次读取文件内容的循环中遇到过这个坑，内存疯狂上升，且一直没释放内存，但又不是内存泄漏，读取小文件时，读取完毕后内存又会降下来，差点在坑中爬不上来。）
* 产生子线程时：记得谁创建，谁保留，谁释放就明白，子线程需要自己维护自己的堆栈，所以在最后，还是得用上自动释放池。（另外，自动释放池是基于`Cocoa`的，如果直接使用POSIX的线程API，那就没法使用自动释放池了，得自己管理好。）

# 3. categories与extension

&emsp;&emsp;categories与extension都为类提供了一种可以额外添加功能的方法，两者是相似的，但细节方面又大大的不同。代码如下：

```
//
//  ViewController.h
//  CategoryAndExtension
//
//  Created by lzackx on 2018/3/06.
//  Copyright © 2018年 lzackx. All rights reserved.
//

#import <UIKit/UIKit.h>

@interface ViewController : UIViewController


@end
```

```
//
//  ViewControllerPrivate.h
//  CategoryAndExtension
//
//  Created by lzackx on 2018/3/06.
//  Copyright © 2018年 lzackx. All rights reserved.
//

#ifndef ViewControllerPrivate_h
#define ViewControllerPrivate_h

// Extension声明
@interface ViewController ()

@property NSString *extensionVariable;

@end

#endif /* ViewControllerPrivate_h */
```

```
//
//  ViewController.m
//  CategoryAndExtension
//
//  Created by lzackx on 2018/3/06.
//  Copyright © 2018年 lzackx. All rights reserved.
//

#import "ViewController.h"
#import "ViewControllerPrivate.h"
#import "UIViewController+Category.h"

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 调用extension中添加的私有属性
    self.extensionVariable = @"Extension";
    NSLog(@"%@", self.extensionVariable);
    
    // 调用category中添加的静态方法
    [UIViewController method];
}

@end
```

```
//
//  UIViewController+Category.h
//  CategoryAndExtension
//
//  Created by lzackx on 2018/3/06.
//  Copyright © 2018年 lzackx. All rights reserved.
//

#import <UIKit/UIKit.h>

// Category声明
@interface UIViewController (Category)

+ (void)method;

@end
```

```
//
//  UIViewController+Category.m
//  CategoryAndExtension
//
//  Created by lzackx on 2018/3/06.
//  Copyright © 2018年 lzackx. All rights reserved.
//

#import "UIViewController+Category.h"

// Category实现
@implementation UIViewController (Category)

+ (void)method {
    
    NSLog(@"Category");
}

@end
```

### 3.1 categories

&emsp;&emsp;categories能在不改变原类实现的情况下，添加额外的功能，但正常情况下，并不能添加存储变量（正常情况下不能，但利用运行时的特性，是可以的），categories与原类分离，通常代码文件名以`原类名+category名`这种形式保存，且内部实现的方法最好加上category的前缀作为与原类方法的区分。方法实现与原类相似，但声明格式略有区别，声明格式如下：

```
@interface ClassName (CategoryName)

@end
```

### 3.2 extension

&emsp;&emsp;extension通常用来扩展原类的一些功能，有时也用于限制方法的调用，它能添加存储变量，但与原类是一体的，即在编译时，是与原类一同编译的，所以在编译后，想扩展功能，使用extension就不合适了。方法实现与原类相似，但声明格式也有不同，如下：

```
@interface ClassName ()

@end
```

&emsp;&emsp;这时，假如需要添加一个私有属性，那么就可以声明在extension的声明内，并在原类中正常使用。而声明也可以放在私有的头文件中，例如`ClassNamePrivate.h`。这样，私有属性就不能显式地表示出来，就达到了隐藏的目的。（但其实通过反射是能知道私有属性的声明的，再基于KVC的特性，就可以使用KVC来调用私有属性了，emmm，如果被拒，不关我事）

# 4. 属性（Property）

&emsp;&emsp;这里贴下通过`clang`编译OC代码为C++的命令，方便查看具体实现。

```
xcrun -sdk iphonesimulator clang -rewrite-objc *.m
```

&emsp;&emsp;属性的知识点比较零散，示例代码如下：

```
//
//  Property.h
//  Property
//
//  Created by lzackx on 2018/3/06.
//  Copyright © 2018年 lzackx. All rights reserved.
//

#import <Foundation/Foundation.h>

@interface Property : NSObject

/* 默认的property属性为readwrite、assign、atomic，
 其中assign适用于基础数据类型和C数据类型，意为简单赋值
 */
@property (atomic, readwrite, assign) int i;

@property NSString *string;

/*
 与assign不同的是，strong适用于继承NSObject的对象，引用计数会增1，即对实例对象保留所有权。
 若被定为assign的话，当实例对象引用计数为0时，属性就会变成悬垂指针，指向的数据未知。
 */
@property (atomic, readwrite, strong) NSNumber *number;

/*
 weak与strong和assign都不一样，weak不增加引用计数，也不持有对象，
 在实例对象的引用计数为0时，指针会被自动设置为nil。
 */
@property (atomic, readwrite, weak) NSNumber *weakNumber;


// 对于BOOL属性，默认的getter就是添加is前缀
@property (readonly, getter=isRb) BOOL rb;

- (instancetype)init

@end
```

```
//
//  Property.m
//  Property
//
//  Created by lzackx on 2018/3/06.
//  Copyright © 2018年 lzackx. All rights reserved.
//

#import "Property.h"

@implementation Property


/*
 以string属性为例，以下是默认的属性会生成的部分。
 1. 默认情况下，属性会生成一个带下划线的实例变量。
 2. 默认rewrite的属性，会生成getter和setter访问器方法，而点语法（.）其实就是调用getter和setter。
 3. 当显示地为属性实现了访问器方法时，编译器会认为需要自己控制属性，所以就不会自动生成实例变量，所以在这种情况下，如果需要一个实例变量的话，需要使用@synthesize再请求生成。
 */
@synthesize string = _string;
- (NSString *)string {
    return _string;
}
- (void)setString:(NSString *)string {
    _string = string;
}

- (instancetype)init {
    /*
     开发中通常会使用这种方式进行初始化，目的是为了避免因KVC机制导致的副作用。
     例如属性的Setter方法。
     */
    self = [super init];
    if (self) {
        _i = 0;
        _string = @"";
        _number = @"";
        _rb = NO;
    }
    return self;
}

@synthesize rb = _rb;
- (BOOL)isRb {
    return _rb;
}

@end
```

&emsp;&emsp;除了直接在代码中的注释的说明外，还有些需要注意的地方。

&emsp;&emsp;`atomic`是原子的意思，当属性为此时，对属性的操作都是原子操作，但注意，原子操作跟线程安全是两个概念！在不同的线程中对同一个实例对象操作此属性只能保证获取或赋值时，不会被打断，但并不保证值是你想要的。

# 5. 块(Block)

&emsp;&emsp;关于块（block），如下所示：

```
/*
  block可通过类型定义来简化使用，如下
  typedef void (^BlockType)(void);
  实现时，如下
  BlockType block = ^{
        ...
    };
*/

// 返回类型  block名    参数   block定义
	void (^blockName)(void) = ^{
	        NSLog(@"I am block");
	};
// block的调用
	blockName();
```

&emsp;&emsp;声明和实现block比较简单，需要注意的知识点时块对局部变量的捕获。

&emsp;&emsp;使用`__block`的局部变量可以为block提供共享存储的能力，通过以下代码进行解释：

```
int i = 0;

void (^block0)(void) = ^{
    NSLog(@"i is: %i", i);
};

i = 1;

// 调用后控制台的结果是0
block0();

/*
 __block这里起到共享存储的作用。
 当使用__block声明变量时，意味着变量是存在于在作用域范围之内声明的块共享的存储空间。
 所以当块中取以__block声明的变量时，获取的位置跟作用范围内的一致。
 同样的情况但不以__block声明时，因为不共享，自然block中就是定义时捕获的值了。
 */
__block int j = 0;
void (^block1)(void) = ^{
    NSLog(@"j is: %i", j);
};

j = 1;

// 调用后控制台的结果是1
block1();
```

通过`clang`编译成C++文件后可以看到如下代码：

```
static void _I_ViewController_testBlock(ViewController * self, SEL _cmd) {
    int i = 0;
    // i在block的实现中使用的是值传递
    void (*block0)(void) = ((void (*)())&__ViewController__testBlock_block_impl_0((void *)__ViewController__testBlock_block_func_0, &__ViewController__testBlock_block_desc_0_DATA, i));

    i = 1;


    ((void (*)(__block_impl *))((__block_impl *)block0)->FuncPtr)((__block_impl *)block0);






    // j在使用__block声明后，block的实现是传递了地址&j
    __attribute__((__blocks__(byref))) __Block_byref_j_0 j = {(void*)0,(__Block_byref_j_0 *)&j, 0, sizeof(__Block_byref_j_0), 0};
    void (*block1)(void) = ((void (*)())&__ViewController__testBlock_block_impl_1((void *)__ViewController__testBlock_block_func_1, &__ViewController__testBlock_block_desc_1_DATA, (__Block_byref_j_0 *)&j, 570425344));

    (j.__forwarding->j) = 1;


    ((void (*)(__block_impl *))((__block_impl *)block1)->FuncPtr)((__block_impl *)block1);
}
```

&emsp;&emsp;当然块的作用还有很多，例如充当参数做回掉，又例如充当返回值。

# 6. 强引用(strong)与弱引用(weak)

&emsp;&emsp;强引用和弱引用在日常开发中，非常常见，也是要时刻注意的，通常，会在如下等一些地方发现到：

* `delegate`对象
* `block`内外

### 6.1 `delegate`对象

&emsp;&emsp;当使用自定义类需要使用`delegate`模式时，通常把属性声明为`weak`，这时，属性不保留实例对象的拥有权，引用计数不变。目的自然是实例对象不希望自己应该被释放的时候，因为其他不相关的类拥有自己所有权，导致引用计数不为0，所以释放不掉，最后导致内存泄漏。例子可参考`UITableViewController`或`UICollectionViewController`。

### 6.2 `block`内外

&emsp;&emsp;在`block`中，一不注意，就很容易出现循环引用的情况。示例代码及解释如下：

```
//
//  ViewController.m
//  Reference
//
//  Created by lzackx on 2018/3/06.
//  Copyright © 2018年 lzackx. All rights reserved.
//

#import "ViewController.h"

@interface ViewController ()

@property NSString *blockProperty;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    
    self.blockProperty = @"Block Property";
    
    // 情况1 block会捕获self，于是出现循环引用，导致内存泄漏
//    void (^block)(void) = ^{
//        NSLog(@"%@", self.blockProperty);
//    };
    
    // 情况2 不会出现循环引用，因为block中并没有让self的引用计数加一
//    __weak ViewController *weakSelf = self;
//    void (^block)(void) = ^{
//        NSLog(@"%@", weakSelf.blockProperty);
//    };
    
    // 情况3 不会出现循环引用，并且能保证block中代码运行时，确保实例变量不为nil
    __weak ViewController *weakSelf = self;
    void (^block)(void) = ^{
        __strong ViewController *strongSelf = weakSelf;
        NSLog(@"%@", strongSelf.blockProperty);
    };
    
    block();
}

@end
```



