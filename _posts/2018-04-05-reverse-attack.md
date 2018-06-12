---
layout: post
title:  "Reverse Attack"
date:   2018-04-05 09:00:00
categories: Reverse
---
**Reverse Attack**

&emsp;&emsp;本文将整理关于iOS逆向开发的知识，由于工具多而散，所以随着时间的推移，此处内容将不断更新。

&emsp;&emsp;本文内容如下：

1. 砸壳与重签
2. 逆向开发


# 1. 砸壳与重签

&emsp;&emsp;在iOS的逆向开发中，需要先获得能运行在设备上的[`Mach-O`](https://zh.wikipedia.org/wiki/Mach-O)文件，才能对App进行逆向探索。

&emsp;&emsp;`Mach-O`文件是在`Build`的时候生成并存放于`.app`包内的，对`.app`文件显示包内容就能看到。

&emsp;&emsp;`App Store`上下载下来的文件是`.ipa`文件，这是一个包含`.app`文件和`Apple`验证信息的包，改成`.zip`后解压可看到里面包含的文件（当然，此时`.app`包内的`Mach-O`文件是加密了的）。

&emsp;&emsp;砸壳与重签其实是通过对`Mach-O`文件的解密提取和打包操作。


### 1.1 App加密原理

&emsp;&emsp;首先，要对`.ipa`文件进行一系列操作的前提能了解它的原理机制。

&emsp;&emsp;[iOS App 签名的原理](http://blog.cnbang.net/tech/3386/)，这是一遍很好的介绍签名原理的文章，作者是[JSPatch](https://github.com/bang590/JSPatch)的作者，文章内也有阮一峰大神对非对称加密RSA算法原理文章的链接。这里就不卖弄了，若有相关知识点，提及的时候再关联补充。

### 1.2 砸壳

&emsp;&emsp;在了解App的加密原理后就会发现，这时会遇到个难题：怎么获得解密后的`Mach-O`文件？就算是已经握在手上的设备中，已经存在的`Apple`公布出来的公钥都拿不到！

&emsp;&emsp;所以，破解加密这种操作是做不到的了，那么就只能出绝招了：越狱。ps:现时，只有9.1以下的iOS系统能完美越狱，更新的版本，听说已有团队做到11.2的越狱，但没证实。

&emsp;&emsp;首先，`Mach-O`文件作为可执行文件，它是被`dyld`（存放与/usr/lib/dyld，iOS和macOS都是，代码是开源的，有兴趣可以[看下](https://opensource.apple.com/tarballs/dyld/)）加载到内存中执行的。

&emsp;&emsp;在无法解密的情况下，想要获得已经解密的文件，就是让设备自己解密。

#### 1.2.1 静态砸壳

&emsp;&emsp;静态砸壳，需要越狱设备，但并不需要让App加载到内存，而是调用设备本身的解密算法，将App解密后dump出来。

&emsp;&emsp;[Clutch](https://github.com/KJCracks/Clutch)可以做到静态砸壳，与上面的一条命令完成砸壳的动态砸壳方式相比，有好处，也有坏处，使用方法按照`README.md`走就行。

&emsp;&emsp;好处是`Clutch`静态砸壳可以把从`App Store`上下载下来的`.ipa`中包含的多架构的`Mach-O`文件给dump出来，而动态砸壳方式因为系统只会加载适合当前架构的部分，所以只能dump出相应版本的`Mach-o`文件。

&emsp;&emsp;坏处是`Clutch`对于framework会dump失败，需要手动拉取，某些App在dump的过程中会直接崩溃，另外默认还需要到`/private/var/mobile/Documents/Dumped`目录下把dump好的`.ipa`发回电脑。

#### 1.2.2 动态砸壳

&emsp;&emsp;动态砸壳，就是利用越狱的设备，向进程注入动态库，利用这个动态库，读取已被`dyld`加载到内存里的数据，dump出已经解密了的`Mach-O`文件（说是砸壳，其实有种提取的味道）。

&emsp;&emsp;目前，最舒适方便的砸壳工具是[AloneMonkey](http://www.alonemonkey.com)的[frida-ios-dump](https://github.com/AloneMonkey/frida-ios-dump)了，介绍：[一条命令完成砸壳](http://www.alonemonkey.com/2018/01/30/frida-ios-dump/)，是用[frida](https://www.frida.re/)做到的，搭好环境后一气呵成。

### 1.3 重签

&emsp;&emsp;了解了App加密原理，就可以手动对`.app`重签打包了。

##### 1.3.1 准备

1. 一个已被解密了的`.app`文件
2. 一份允许App安装到真机上的开发者证书，这意味着已经有本地的公私钥了
3. 一份App对应的`Provisioning Profile`，包含`AppID`、`Entitlement`、`DeviceID`等信息

##### 1.3.2 操作

&emsp;&emsp;验证`Mach-O`文件是否已解密可以通过以下命令：

```sh
# cryptid 0 -> 已解密
# cryptid 1 -> 未解密
otool -l [MACH-O_FILE] | grep cryptid
```

&emsp;&emsp;查看自己的开发者证书可以通过以下命令：

```sh
security find-identity -v -p codesigning
```

&emsp;&emsp;将`.app`文件放置于`Payload`目录下（注意这一步是必须的，否则App安装时会报错）。

&emsp;&emsp;从[App开发者账号](https://developer.apple.com/account/ios/certificate/)里创建一个对应`App ID`的`Provisioning Profile`。

&emsp;&emsp;新建一个`Xcode`项目，利用刚创建的`Provisioning Profile`文件进行编译，编译后的`.app`文件内有一`embedded.mobileprovision`文件，检查其内容，保证`App ID`正确，然后将这个文件复制到解密了的`.app`下。

&emsp;&emsp;查看`embedded.mobileprovision`（看`Provisioning Profile`文件也一样）文件中`<key>Entitlements</key>`内的的权限内容，可以通过以下命令：

```sh
security cms -D -i [Profile_Path]
```

&emsp;&emsp;通过一下命令查看原App的权限设置，与自己创建的进行对比修改（为了重签后可以进行调试，可添加`get-task-allow`和`task_for_pid-allow`两个布尔类型的值）。

```sh
codesign -d --entitlements :- [MACH-O_FILE]
# 有ldid也可以用这个
ldid -e [MACH-O_FILE]
```

&emsp;&emsp;复制`<key>Entitlements</key>`内的权限内容到一个新的`.plist`文件中。

&emsp;&emsp;之后就可以通过以下命令进行签名了（注意这里是有坑的，各个App的`.app`文件内存在着不同的内容，有时需要注意`Framework`下的文件进行签名，而有些文件，直接删掉也问题不大）：

```sh
# codesign_identity：开始时查看的自己的开发者证书ID
# entitlements.plist：通过查看权限后生成的权限plist文件
# .app：就是.app目标文件
codesign -fs [codesign_identity] --entitlements=[entitlements.plist] [.app]
```

&emsp;&emsp;签名后，就可以打包成`.ipa`了，可以通过如下命令：

```sh
# 注意这里一定要把Payload目录包含进去，否则安装时会报错
zip -ry [.ipa] [Payload_Path]
```

&emsp;&emsp;然后就可以将此`.ipa`包，通过Xcode安装到真机上了：`Window`->`Devices and Simulators`，拖进去就行了。

&emsp;&emsp;如果签名打包成功，那么此时App已经装到设备上了。

# 2. 逆向开发

### 2.1 非越狱开发

&emsp;&emsp;[MonkeyDev](http://www.alonemonkey.com/2017/07/12/monkeydev-without-jailbreak/)，一个超方便的非越狱逆向开发工具，详细介绍点进去看就是了。

&emsp;&emsp;本节将详解非越狱情况下逆向开发中需要用到的知识。

#### 2.1.1 Xcode的使用

&emsp;&emsp;`Xcode`作为`macOS`上的IDE，包含了很多开发时需要的工具，其中包括逆向开发时，可能会用上的功能。

1. Always Show Disassembly，打开方式：`Debug`->`Debug Workflow`->`Always Show Disassembly`，用于在断点时，将已经编译为机器码的代码反汇编为汇编代码进行显示，用于分析逻辑。
2. View Memory，打开方式：`Debug`->`Debug Workflow`->`View Memory`，快捷键：`⇧⌘M`，用于直接查看内存中的数据，通常配合指针地址或汇编代码中的地址使用。
3. [LLDB](http://lldb.llvm.org/)，断点时触发，下面列出一些逆向开发时比较有用的命令：
```sh
# 读写寄存器值 read/write
register
# 读写当前进程的内存值 read/write
memory
# 读当前进程的内存值，memory read 的快捷方式
x
# 获取可执行文件的依赖库列表，也方便查看ASLR值（第一个就是）
image list
# 获取可执行文件相关信息
image lookup
# 打印当前线程的所有堆栈信息
bt
# 获取当前线程的当前堆栈信息
frame info
# 选择堆栈，后跟堆栈序号
frame select
# 选择前一条堆栈
up
# 选择后一条堆栈
down
# 比Xcode上UI的断点更灵活，详看breakpoint help
breakpoint
```

#### 2.1.2 命令行的使用

##### 2.1.2.1 [otool](https://opensource.apple.com/source/cctools/cctools-895/otool/)

&emsp;&emsp;`otool`（object file displaying tool）是个非常强大的文件信息查看工具，下面列出些逆向开发时比较常用的命令。

```sh
# 查看Mach-O文件的header，-v可以查看比较直观的描述
# 最后的flag中可以发现有PIE (Position Independent Enable)，就是说App启动时会产生ASLR 
# ASLR(Address Space Layout Randomization)，地址空间随机布局，是为了防范对已知地址进行恶意攻击的，一些不完美越狱的机子就是因为这个原因
# ASLR用于App时，会使Mach-O加载到未知的地址，从而导致在App运行中某指令的内存地址与查看Mach-O内的内存地址不匹配的情况
# 但App内的内存地址还是固定的，只是多了一个偏移值，通过lldb打印image list就能查看这个偏移值，加上后就能匹配上了
otool -h -v [[MACH-O_FILE]]
# 查看Mach-O文件的load commands部分，想查看这个Mach-O文件有没有解密，如前所述，用管道筛选查看cryptid就行了
otool -l [MACH-O_FILE]
# 查看Mach-O文件的依赖库
otool -L [MACH-O_FILE]
```

##### 2.1.2.2 [lipo](https://opensource.apple.com/source/cctools/cctools-895/misc/lipo.c.auto.html)

&emsp;&emsp;`lipo`是个对`Mach-O`文件的架构进行操作的命令，通常会在开始分析或进行打包时用到以下一些命令。

```sh
# 查看文件架构
file [FILE]
lipo -info [MACH-O_FILE]
# 分离架构
lipo -thin [architecture, armv7, arm64] [SOURCE_FILE] -output [TARGET_FILE]
# 合拼架构
lipo -create [SOURCE_FILES_DELIMITED_BY_WHITESPACE] -output [TARGET_FILE]
```

##### 2.1.2.3 [class-dump](http://stevenygard.com/projects/class-dump/)

&emsp;&emsp;`class-dump`是个可以把`Mach-O`文件内的头文件导出来的命令，方便通过头文件进行分析和开发。

```sh
class-dump -H [MACH-O_FILE]
```

##### 2.1.2.4 [restore-symbol](https://github.com/tobefuturer/restore-symbol)

&emsp;&emsp;`restore-symbol`是用来恢复符号表的。在断点动态调试时，恢复了符号表的`Mach-O`文件能清楚看到堆栈调用信息，为逆向开发提供极大便利。

```sh
# -j 是用于恢复Block的符号表的，需要将/search_oc_block/ida_search_block.py放到IDA中执行生成的block_symbol.json文件作为参数传入
restore-symbol [SOURCE_FILE] -j [BLOCK_SYMBOL_JSON] -o [TARGET_FILE]
```

##### 2.1.2.5 Hook

**1. Method Swizzling**

&emsp;&emsp;`Method Swizzling`是利用语言的`Runtime`特性实现的，对其陌生的可以看下[Runtime](http://lzackx.com/framework/2018/03/15/runtime.html)。通过前文所述，可以知道编译后的代码（就是`Mach-O`文件内的的部分内容）是通过`dyld`加载到内存中的，在`Class`或`Category`加载到`Runtime`时就会触发`+ (void)load;`类方法，在这时，就可以通过`Method Swizzling`进行类中方法的Hook操作了（包括但不限于这种使用`Method Swizzling`的Hook方式）。

&emsp;&emsp;对于`+ (void)load;`需要注意一些情况，只有新加载的`Class`或`Category`会触发这个方法，而调用顺序如下：

```
1 项目中所有链接了的Framework的初始化调用时
2 项目中所有"+ (void)load;"方法调用时
3 项目中所有C++的静态初始化和C\C++的构造函数调用时
4 项目中所有依赖了项目的其他Framework的初始化调用时
```

&emsp;&emsp;另外，还有细分两个顺序：

```
* Class的"+ (void)load;"会在其父类调用了"+ (void)load;"后再调用
* Category的"+ (void)load;"会在其Class调用"+ (void)load;"后再调用
```

&emsp;&emsp;根据调用顺序可以发现，在`+ (void)load;`类方法中使用`Method Swizzling`进行Hook操作，在被注入代码后“基本”可以Hook住所有方法。

&emsp;&emsp;但对于某些利用`Category`来Hook的方法，有可能因为加载`Category`的顺序不同，或又因为原代码中的方法经过`Method Swizzle`处理过，可能会导致Hook失败。需要具体问题具体分析。

&emsp;&emsp;在用`Method Swizzle`进行Hook的时候，如果只是需要添加代码而不是修改时，通常会在Hook的时候保留原方法的`IMP`，然后在Hook了之后执行它，再添加自己的代码，下面是一个简单的例子：

```objc
//
//  MSHook.h
//  
//
//  Created by lzackx on 2018/4/5.
//  Copyright © 2018年 lzackx. All rights reserved.
//

#import <Foundation/Foundation.h>

@interface MSHook : NSObject

@end
```

```objc
//
//  MSHook.m
//  
//
//  Created by lzackx on 2018/4/5.
//  Copyright © 2018年 lzackx. All rights reserved.
//

#import "MSHook.h"
#import <objc/runtime.h>

@implementation MSHook

static Method originalMethod;
static Method swizzleMethod;
static IMP originalIMP;
static IMP swizzleIMP;

+ (void)load {
    
    originalMethod = class_getInstanceMethod(objc_getClass("AppDelegate"), @selector(application:didFinishLaunchingWithOptions:));
    originalIMP = method_getImplementation(originalMethod);
    swizzleMethod = class_getInstanceMethod(self, @selector(application:didFinishLaunchingWithOptions:));
    swizzleIMP = method_getImplementation(swizzleMethod);
    // 备注：这个函数的执行是不会改变两Method对象的地址的，只是将两个Method内的IMP互相调换了
    method_exchangeImplementations(originalMethod, swizzleMethod);
}

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
   
	// 调用原方法
    BOOL returnValue = ((BOOL(*)(id, SEL, id, id))originalIMP)(self, _cmd, application, launchOptions);
    return returnValue;
}

@end
```

**2. [CaptainHook](https://github.com/rpetrich/CaptainHook)**

&emsp;&emsp;`CaptainHook`就是一个头文件，内有很多方便Hook的宏，原理也是基于`Runtime`的，文件并不大，有兴趣看一下就懂了。

**3. [fishhook](https://github.com/facebook/fishhook)**

&emsp;&emsp;`fishhook`是一个由Facebook提供的C库，只有两个文件，Hook的原理是修改`Mach-O`文件中对应函数的懒加载指针（`__la_symbol_ptr`）与非懒加载指针（`__nl_symbol_ptr`）来达到Hook的目的。

&emsp;&emsp;`fishhook.h`中提供了一个结构体和两个函数：

```c
struct rebinding {
  const char *name;		// 源函数名称
  void *replacement;	// 目标函数指针
  void **replaced;		// 源函数指针的指针（就是用一个指针变量指向了源函数指针，用于保存源函数）
};

// 顾名思义，用目标函数指针重绑源函数符号指向的地址
// 当调用相同的函数名时，由于符号指向的地址被绑定为目标函数指针，所以就会执行目标函数指针指向的函数，而不是源函数指针指向的函数
int rebind_symbols(struct rebinding rebindings[], size_t rebindings_nel);
int rebind_symbols_image(void *header,
                         intptr_t slide,
                         struct rebinding rebindings[],
                         size_t rebindings_nel);
```

&emsp;&emsp;在重绑定符号时，其实是通过源函数名称这个参数查找符号的，当找到匹配的符号时，才会进行重绑定。查找符号的具体原理可以看下Github上的介绍和流程图，这里以某`Mach-O`的`NSLog`函数为例：

* 首先找到`__la_symbol_ptr`中的指针地址，如下图，可以看到地址是`00C1DB80`。

![`__la_symbol_ptr`](http://lzackx.com/images/2018-04-05-reverse-attack-fishhook-lazy-symbol-pointers.png)

* 然后在`Indirect Symbols`中找到对应的符号，可以看到Data为`187`，这是一个对应`Symbol`中的序号。

![Indirect Symbols](http://lzackx.github.io/images/2018-04-05-reverse-attack-fishhook-indirect-symbols.png)

* 在`Symbols`中，可以通过从`Indirect Symbols`中获得的序号找到相应的位置（对于MachOView中`Symbols`表不显示序号的情况，观察下每个符号的地址间隔可以发现他们的地址之间相差一个`0x10`，所以与序号相乘，即十六进制左移一位，这里的例子是`0x1870`，再加上`Symbols`表的初始偏移地址，就可以找到目标函数的符号地址了，当然这只适合于`MachOView`手动查找，其实右上角的搜索功能更方便......，在代码中的实现，其实就是数组通过下标查找的），然后可以发现Data列的值为`2502`，这是对应`String Table`中的偏移地址。如下图所示：

![Symbols](http://lzackx.github.io/images/2018-04-05-reverse-attack-fishhook-symbols.png)

* 最后，在`String Table`中，将初始偏移地址与从`Symbols`表中获得的偏移地址相加，就能找到符号的字符串所在的地址了。如下图所示：

![String Table](http://lzackx.github.io/images/2018-04-05-reverse-attack-fishhook-string-table.png)

&emsp;&emsp;在找到匹配的符号后，替换符号指向的源函数指针地址为目标函数函数指针地址，Hook就生效了。

**备注**：`fishhook`并不是所有C函数都一定能Hook住的，意外情况通常发生在自定义的函数中。当自定义的函数在进行Hook前没有被调用过时，`__la_symbol_ptr`中没有找到与`rebind_symbols`函数中传递参数指定的符号时，`fishhook`会因为找不到对应的符号，所以不会进行指针的替换，进而Hook不住。

##### 2.1.2.6 [yololib](https://github.com/KJCracks/yololib)

&emsp;&emsp;`yololib`是个将动态库注入`Mach-O`文件的命令行工具，原理是把动态库信息写入`Mach-O`文件中，源码是只有200+的`.m`文件，很好理解。注入后，用`MachOView`或`otool`都能看见注入的部分。

```sh
# 注意DYNAMICALLY_LINKED_SHARED_LIBRARY_FILES传递的值，应与重打包后访问的动态库路径匹配，可参考otool中其他动态库的路径
yololib [MACH-O_FILE] [DYNAMICALLY_LINKED_SHARED_LIBRARY_FILES]
```

#### 2.1.3 第三方工具的使用

##### 2.1.3.1 [MachOView](https://sourceforge.net/projects/machoview/)

&emsp;&emsp;`MachOView`是一个用于浏览`Mach-O`文件的工具。可以说是有UI界面的`otool`，在看`Mach-O`内的符号时比较直观。

##### 2.1.3.2 [IDA](https://www.hex-rays.com/products/ida/)

&emsp;&emsp;`IDA`是个很强大的工具，但也比较复杂，甚至有一本书来讲述它的使用，叫《The IDA PRO BOOK》，在逆向中，静态分析时用它比用`Xcode`要有效率得多。


### 2.2 越狱开发

#### 2.2.1 ssh

&emsp;&emsp;越狱后，就有了设备的`root`权限了，在通过`Cydia`安装了`ssh`相关工具后，就可以通过内网连上设备了，连设备跟连普通服务器没区别，甚至可以把设备当成服务器用。

&emsp;&emsp;通常为了方便连接，会将经常连接的电脑与设备设置无密码登陆（生成时注意别填`passphrase`），设置也很简单（如果没有生效，注意文件和目录的权限）。

```sh
# 生成一对公私钥
ssh-keygen
# 将公钥发到设备上
ssh-copy-id -i [PUBLIC_ID_FILE] name@host -p port
# 通过验证无需密码即可
ssh -i [ID_FILE] name@host -p port
```

#### 2.2.3 dyld

&emsp;&emsp;在越狱的情况下，可以通过`ssh`链接到设备上了，这时就可以通过`dyld`动态加载库了。

```sh
DYLD_INSERT_LIBRARIES=[DYNAMICALLY_LINKED_SHARED_LIBRARY_FILES] [MACH-O_FILE]
```

&emsp;&emsp;`dyld`还有很多环境变量，可以在电脑上`man dyld`查看。通过`DYLD_INSERT_LIBRARIES`动态加载的库并不会影响原`Mach-O`，只是让库加载到进程中而已（`dumpdecrypted.dylib`就是通过这种方式进行动态砸壳的）。

&emsp;&emsp;由于`dyld`是开源的，看其代码，可以发现这种方式是可以被禁止的。

#### 2.2.3 [FLEXLoader](https://github.com/qiaoxueshi/FLEXLoader)

&emsp;&emsp;`FLEXLoader`是一个`BigBoss`源上的插件，用于将[FLEX](https://github.com/Flipboard/FLEX)库动态注入到App中，便于动态调试App中的UI。

&emsp;&emsp;`FLEX`当然也可以在非越狱的情况下使用，直接在Hook中调用并注入库就可以了。

#### 2.2.4 [Keychain-Dumper](http://stevenygard.com/projects/class-dump/)

&emsp;&emsp;`Keychain-Dumper`就是一个将钥匙串里的信息（路径：/private/var/Keychains）dump出来的工具。

&emsp;&emsp;使用起来非常简单，将`keychain_dumper`通过`scp`传到越狱设备的`/bin`下，直接执行就可以看到Keychain的信息了。

#### 2.2.5 [cycript](http://www.cycript.org/)

&emsp;&emsp;`cycript`是利用Javascript动态调试App的工具。

&emsp;&emsp;`cycript`在macOS配置使用的时候是有坑的。通常因系统的`ruby`版本过高导致，这时可以机智地进入恢复模式（重启按住⌘R），关掉系统的SIP（System Integrity Protection ），将`/System/Library/Frameworks/Ruby.framework`内的目录复制一份并将版本号全部改成`cycript`要求的版本号就行，后面再打开SIP也不影响。

&emsp;&emsp;越狱条件下使用时，可以通过`Cydia`装上`cycript`，通过`ssh`连接上设备后，通过如下命令直接动态调试App。

&emsp;&emsp;非越狱条件下使用时，需要先给App集成`Cycript.framework`这个库，运行起来后，App中的`Cycript.framework`会通过设定的端口监听，只要与电脑连接在同以局域网内，就能通过安装在电脑上的`cycript`命令连接App，然后进行调试，调试Objective-C的语法可以看官网手册。

```sh
cycript [-c] [-p <pid|name>] [-r <host:port>] [<script> [<arg>...]]
```

&emsp;&emsp;无论越狱和非越狱，都可以通过命令或在运行中通过`import`导入已经写好的脚本，能大大提升效率。不过在越狱情况下，通过指令导入的脚本能比`import`方式导入的脚本更方便，因为命令导入的脚本文件只要改动了，就能生效（毕竟Javascript是解释性语言），而通过`import`导入的，需要退出`cycript`后再进才有效。

#### 2.2.6 [Cydia Substrate](http://www.cydiasubstrate.com/)

&emsp;&emsp;`Cydia Substrate`前身是`MobileSubstrate`，包含3个主要组件：`MobileHooker`、`MobileLoader`、`safe mode`，只能用于越狱设备，主页内的文档超齐全，这里就没必要搬运了。

#### 2.2.7 [theos](https://github.com/theos/theos)

&emsp;&emsp;`theos`是跨平台的不需要使用`Xcode`的用于管理、开发和发布iOS应用的工具，通常越狱设备上的`tweaks`就是用这个来开发的。

&emsp;&emsp;可以按照[Wiki](https://github.com/theos/theos/wiki)进行安装使用，请严谨按照步骤，否则容易踩坑。

&emsp;&emsp;在编译、打包、安装由`theos`生成的项目时，通常用到下面一些命令：

```sh
# 看到Makefile就不难理解了，再看里面的内容，可以发现所有模版的编译、打包都由相应的.mk文件负责
make
make package
# 在安装前，需要先配置好越狱设备的地址，可以通过Wi-Fi，也可以通过经过端口映射了的USB。
# export THEOS_DEVICE_IP=
# export THEOS_DEVICE_PORT=
make install
```

##### 2.2.7.1 NIC(New Instance Creator)

&emsp;&emsp;`theos/bin/nic.pl`是一个Perl脚本，用于创建`theos`项目模版，直接控制台执行就行。

```sh
nic.pl
```

&emsp;&emsp;注意创建过程中，路径避免含有ASCII不能表示的字符，包名请以全小写命名。否则在打包过程中，会遇坑。

&emsp;&emsp;遇坑也别慌，阅读`theos/bin`目录内编译项目和打包项目的脚本就能找到原因了。

##### 2.2.7.2 [Logos](http://iphonedevwiki.net/index.php/Logos)

&emsp;&emsp;`Logos`是`theos`中的一个组件，它有自己的语法，当代码写在`.x`或`.xm`内后，编译时会用`theos/bin/logos.pl`进行预处理，生成`.m`或`.mm`文件，以C/C++的形式调用函数或ObjC的方法（思路跟`CaptainHook`一样，都是为了方便Hook，甚至可以把`Logos`语法看成是另一种类型的宏）。语法用起来更自然，比`CaptainHook`效率高。

&emsp;&emsp;`Logos`在预处理后就是一些C/C++代码了，所以与在越狱和非越狱环境无关，编译成动态库后，可以通过`yololib`注入，也可以通过`dyld`插入。

##### 2.2.7.3 Logify

&emsp;&emsp;`theos/bin/logify.pl`是一个能将`.h`头文件中的内容生成以`Logos`语法实现的`.xm`文件，一句命令就把整个类的方法都hook了，能在开始逆向代码逻辑时，提高观察逻辑调用顺序的效率（不用机械地抄方法来实现还是挺好的）。命令如下：

```sh
logify.pl header.h > source.xm
```

#### 2.2.8 debugserver

&emsp;&emsp;`debugserver`是`LLDB`的一部分，Xcode中断点时就用到了它，在设备上的位置：`/Developer/usr/bin/debugserver`。

&emsp;&emsp;在逆向时，通常会用它在设备端attach进程，对App进行调试。

```sh
#Usage:
debugserver host:port [program-name program-arg1 program-arg2 ...]
debugserver /path/file [program-name program-arg1 program-arg2 ...]
debugserver host:port --attach=<pid>
debugserver /path/file --attach=<pid>
debugserver host:port --attach=<process_name>
debugserver /path/file --attach=<process_name>
```

&emsp;&emsp;在电脑端，通过`lldb`启动`LLDB`后，连接设备端的`debugserver`进行调试：

```sh
process connect connect://host:port
```

&emsp;&emsp;这种调试行为时可以被防御和反防御的，当然也有反反反防御和反反反反防御。。。攻防矛盾内容在之后另外叙述。

##### **备注**

&emsp;&emsp;本文内容多而杂，但总归是逆向相关的归纳总结，后续会以此为骨架逐渐展开细节进行更新。



