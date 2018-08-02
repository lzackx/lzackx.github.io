---
layout: post
title:  "Reverse Defense"
date:   2018-04-10 09:00:00
categories: Reverse
---
**Reverse Defense**

&emsp;&emsp;本文将按阶段整理iOS逆向防御的知识，随着时间推移，内容将不断更新。

&emsp;&emsp;攻防互为矛盾，防得了一时，防不了永远。防御的精髓就是逼疯攻的一方（XD）。

&emsp;&emsp;本文内容以防重签作为抛砖引玉，慢慢深入细节看防护。

### 1. 防重签

&emsp;&emsp;对于`.app`文件防重签的代码，网上有很多介绍且是同一套的代码，如下：

```objc
#define kIdentifier @"XXXXXXXXXX" // 10位的证书ID
- (void)checkCodesign {
    NSString *embeddedPath = [[NSBundle mainBundle] pathForResource:@"embedded" ofType:@"mobileprovision"];
    if ([[NSFileManager defaultManager] fileExistsAtPath:embeddedPath]) {
        NSString *embeddedProvisioning = [NSString stringWithContentsOfFile:embeddedPath encoding:NSASCIIStringEncoding error:nil];
        NSArray *embeddedProvisioningLines = [embeddedProvisioning componentsSeparatedByCharactersInSet:[NSCharacterSet newlineCharacterSet]];
        for (int i = 0; i < [embeddedProvisioningLines count]; i++) {
            if ([[embeddedProvisioningLines objectAtIndex:i] rangeOfString:@"application-identifier"].location != NSNotFound) {
                NSInteger fromPosition = [[embeddedProvisioningLines objectAtIndex:i+1] rangeOfString:@"<string>"].location+8;
                NSInteger toPosition = [[embeddedProvisioningLines objectAtIndex:i+1] rangeOfString:@"</string>"].location;
                NSRange range;
                range.location = fromPosition;
                range.length = toPosition - fromPosition;
                NSString *fullIdentifier = [[embeddedProvisioningLines objectAtIndex:i+1] substringWithRange:range];
                NSArray *identifierComponents = [fullIdentifier componentsSeparatedByString:@"."];
                NSString *appIdentifier = [identifierComponents firstObject];
                if (![appIdentifier isEqual:kIdentifier]) {
                    UIAlertView *alert = [[UIAlertView alloc] initWithTitle:@"Error" message:@"codesign verification failed." delegate:self cancelButtonTitle:@"OK" otherButtonTitles:nil, nil];
                    [alert show];
                }
                break;
            }
        }
    }
}
```

&emsp;&emsp;做防护一定不能不求甚解、生搬硬套，否则，防了跟没防的区别不大。

&emsp;&emsp;就拿上面的防重签代码进行分析，结合App签名打包加壳到`App Store`下载的流程来思考上面这段防重签的代码，在本地编译无论`debug`、`release`（包括`archive`）出来的`.app`文件内，都有`embedded.mobileprovision`文件，可被上面的代码用于验证。但是从`App Store`上下载下来的App解压（与砸壳无关）出来，这个文件都不在存在，代码中`- (BOOL)fileExistsAtPath:(NSString *)path`方法判断就不成立，也就不会进行验证了。在逆向的时候，从`App Store`安装App，砸壳，当需要重新打包运行时，`.app`文件就附加上`embedded.mobileprovision`文件（通过`Provisioning Profile`文件编译而成，开发者账号不一样，这个文件当然也不一样了），运行上设备上时当执行到上面的代码就会验证了。上面的代码写得很漂亮，还会友好提示”codesign verification failed“（XD），不过其实直接复制到`AppDelegate`中调用的话，效果就不怎么好了。

* 首先，因为这一系列的代码是写在一个方法内的，所以是存在Hook了这个方法，让其防重签检验无效的可能。
* 10位证书ID是通过宏来定义的，在预编译以后，就是直接作为参数，在编译成汇编后，它就是一个非常显眼的标识，通过`IDA`看`Mach-O`文件，顺着App的生命周期要找出来并不难，然后通过修改`Mach-O`文件中的数值，改成重签的证书的ID，防重签也就无效了。

&emsp;&emsp;从这两点非常明显的切入点可以看出，上面这段代码防重签效果非常脆弱，要加强防重签效果要做的事情其实也不多也不难。

&emsp;&emsp;要做到让`IDA`难以找到防重签代码，就要将上面的方法要不进行混淆（简单混淆其实就是利用`.pch`文件（Precompiled Header）通过加密算法算出字符串定义成方法名的宏，通过预编译替换方法名进行混淆），要不打散逻辑混进生命周期中（这种方法效果是非常好的，逆向时找这些逻辑会很痛苦，并且就算找到了也要想怎么去获得需要修改的对象，一旦走到这一步，除非死磕，不然通常逆向时就会改变思路，直接找证书ID来修改。但是代码维护成本变高，一不小心改了，出问题起来也会让人很懵逼，而且其中穿插的逻辑每一步的改动可能都要小心翼翼，毕竟越复杂的代码，越容易出漏洞）。下面是一个混淆方法名的例子：


### 2. 代码混淆

```objc
// SomeClass.h
#import <Foundation/Foundation.h>

@interface SomeClass : NSObject

- (void)someMethodWithParameter:(NSObject *)parameter;

@end
```

```objc
// SomeClass.m
#import "SomeClass.h"

@implementation SomeClass

- (void)someMethodWithParameter:(NSObject *)parameter { }

@end
```

&emsp;&emsp;知道上面的这个类，希望混淆方法（注意：因为文件名不能混淆，所以注意源码文件的对应关系），可以新增一个`.pch`文件，并通过`build setting`中的`prefix header`添加`.pch`文件的路径（注意路径取当前项目的根目录作为基地址，不清楚的可以点`.pch`文件看`File inspector`中`relative to project`下的路径），让这个方法名成为一个宏，在预编译后就会成了混淆后的名字了，后面就算是看`restore symbol`后的`Mach-O`文件，都只能看到混淆后的名字，可以让逆向一脸懵逼。

```objc
// Obfuscation.pch
#ifndef CodeObfuscation_pch
#define CodeObfuscation_pch

// 这里的ea6658e7b280dd39e077fe60016ce863是通过md5对someMethodWithParameter字符串加密生成的
// 当然也可以用其他办法生成加密字符串（甚至用脸滚键盘生成也没所谓）甚至使用混合方式加密
#define someMethodWithParameter		ea6658e7b280dd39e077fe60016ce863

#endif /* Obfuscation */
```

### 3. 关键字符串防护

&emsp;&emsp;另外一点就是别将关键字符串直接当成参数进行传递，也别在关键地方添加容易让人定位的字符串。关键字符串，通常会使用`CCCrypt`函数进行对称加密（也有坑，例如用`lldb`符号断点断住这个函数，就能找到函数内的含加密的Key的参数了），或类似数组和位运算的方式进行保护，例子如下：

```objc
// 字符串加密
// 用脸滚出来的宏随便定义了一个16进制的数
#define DKASDLMNFWEFN  0xabcd
// 一个不显眼（混淆过）的静态函数，用于获取需要保护的字符串，在汇编中，看bl跳转的时候，难看的函数名要跟踪会很难受的
static NSString *vuhwiebasdweoqmc() {
	// 通过宏与被保护的字符串中的字符进行异或运算，防止字符串进入常量区，IDA就不能显示出被保护的字符串信息了
	unsigned char key[] = {
		(DKASDLMNFWEFN ^ 'w'),
		(DKASDLMNFWEFN ^ 'h'),
		(DKASDLMNFWEFN ^ 'a'),
		(DKASDLMNFWEFN ^ 't'),
		(DKASDLMNFWEFN ^ 'e'),
		(DKASDLMNFWEFN ^ 'v'),
		(DKASDLMNFWEFN ^ 'e'),
		(DKASDLMNFWEFN ^ 'r'),
		(DKASDLMNFWEFN ^ '\0'),
	};
	// 通过指针获得数组的地址，对其进行字符串还原（异或位运算做对称加密算法也是很方便）
	unsigned char *p = key;
	*p = (*p ^ DKASDLMNFWEFN);
	while (*p != '\0' ) {
		p++;
		*p = (*p ^ DKASDLMNFWEFN);
	}
	// 返回值就是被保护的字符串，在这里就是@"whatever"了
	return [NSString stringWithUTF8String:(const char *)key];
}

```

### 4. 防静态分析

&emsp;&emsp;使用`CCCrypt`函数的参数加密了，还要注意防护函数本身，毕竟`lldb`符号断点断住了`CCCrypt`后，参数都是能拿到的，函数要是不防护，那前面做的防护效果也不怎么样了。所以想要做这一层的防护，就要用另外一种方式调用这个函数了（这种做法主要是为了防静态分析）。

&emsp;&emsp;在不直接调用的情况下调用一个函数，这种操作就是C语言的精髓和魅力了，说白了就是指针。

&emsp;&emsp;为了能让一个指针指向一个函数的入口，需要找到函数所在的库和函数的符号，符号是已知条件，而库想找也容易，先给函数打一个符号断点，当符号断点断住了之后`bt`，就能看到库了，例如这里的`CCCrypt`所在的库就是`libcommonCrypto.dylib`，然后通过`image list`查到库的加载地址（用`otool`查也行）。在条件都知道后，就可以进行调用了，例子如下：

```objc
// 因为要用上dlopen，所以要添加对应的头文件
#import <CommonCrypto/CommonCrypto.h>
#import "dlfcn.h"

void someFunction() {

	// 获取库
	void *handler = dlopen("PATH_TO_LIBRARY", RTLD_LAZY); 
	// 获取指向CCCrypt函数入口的指针
	CCCryptorStatus CCCryptPointer(
    CCOperation op,         /* kCCEncrypt, etc. */
    CCAlgorithm alg,        /* kCCAlgorithmAES128, etc. */
    CCOptions options,      /* kCCOptionPKCS7Padding, etc. */
    const void *key,
    size_t keyLength,
    const void *iv,         /* optional initialization vector */
    const void *dataIn,     /* optional per op and alg */
    size_t dataInLength,
    void *dataOut,          /* data RETURNED here */
    size_t dataOutAvailable,
    size_t *dataOutMoved)
    __OSX_AVAILABLE_STARTING(__MAC_10_4, __IPHONE_2_0) = dlsym(handler, "CCCrypt"); // 注意：这里的第二个参数的传值需要使用前面的字符串保护方式进行传值，例子这里就不加了，返回一个CCCrypt指针

    // 良好习惯
    if (!CCCryptPointer) {
    	return nil;
    }

    // 用函数指针调用函数并获得返回值，这里的调用就跟CCCrypt(...)一样
    CCCryptorStatus cryptorStatus = CCCryptPointer(...);
}

```

&emsp;&emsp;通过函数指针调用函数可以去掉`Mach-O`文件中查相应符号的调用（防住了静态分析），可是这种方法还阻止不了`lldb`的符号断点（防不住动态调试），就是说用`CCCrypt`符号断点还是能断住，能获取到参数，结果是依旧没防住。

### 5. 防动态调试

&emsp;&emsp;要防住动态调试，一个思路就是通过`ptrace`函数阻止进程依附。

```c
/*
 * Copyright (c) 2000-2005 Apple Computer, Inc. All rights reserved.
 *
 * @APPLE_OSREFERENCE_LICENSE_HEADER_START@
 * 
 * This file contains Original Code and/or Modifications of Original Code
 * as defined in and that are subject to the Apple Public Source License
 * Version 2.0 (the 'License'). You may not use this file except in
 * compliance with the License. The rights granted to you under the License
 * may not be used to create, or enable the creation or redistribution of,
 * unlawful or unlicensed copies of an Apple operating system, or to
 * circumvent, violate, or enable the circumvention or violation of, any
 * terms of an Apple operating system software license agreement.
 * 
 * Please obtain a copy of the License at
 * http://www.opensource.apple.com/apsl/ and read it before using this file.
 * 
 * The Original Code and all software distributed under the License are
 * distributed on an 'AS IS' basis, WITHOUT WARRANTY OF ANY KIND, EITHER
 * EXPRESS OR IMPLIED, AND APPLE HEREBY DISCLAIMS ALL SUCH WARRANTIES,
 * INCLUDING WITHOUT LIMITATION, ANY WARRANTIES OF MERCHANTABILITY,
 * FITNESS FOR A PARTICULAR PURPOSE, QUIET ENJOYMENT OR NON-INFRINGEMENT.
 * Please see the License for the specific language governing rights and
 * limitations under the License.
 * 
 * @APPLE_OSREFERENCE_LICENSE_HEADER_END@
 */
/* Copyright (c) 1995 NeXT Computer, Inc. All Rights Reserved */
/*-
 * Copyright (c) 1984, 1993
 *	The Regents of the University of California.  All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 * 1. Redistributions of source code must retain the above copyright
 *    notice, this list of conditions and the following disclaimer.
 * 2. Redistributions in binary form must reproduce the above copyright
 *    notice, this list of conditions and the following disclaimer in the
 *    documentation and/or other materials provided with the distribution.
 * 3. All advertising materials mentioning features or use of this software
 *    must display the following acknowledgement:
 *	This product includes software developed by the University of
 *	California, Berkeley and its contributors.
 * 4. Neither the name of the University nor the names of its contributors
 *    may be used to endorse or promote products derived from this software
 *    without specific prior written permission.
 *
 * THIS SOFTWARE IS PROVIDED BY THE REGENTS AND CONTRIBUTORS ``AS IS'' AND
 * ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
 * IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
 * ARE DISCLAIMED.  IN NO EVENT SHALL THE REGENTS OR CONTRIBUTORS BE LIABLE
 * FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
 * DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS
 * OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
 * HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
 * LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY
 * OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF
 * SUCH DAMAGE.
 *
 *	@(#)ptrace.h	8.2 (Berkeley) 1/4/94
 */

#ifndef	_SYS_PTRACE_H_
#define	_SYS_PTRACE_H_

#include <sys/appleapiopts.h>
#include <sys/cdefs.h>

enum {
	ePtAttachDeprecated __deprecated_enum_msg("PT_ATTACH is deprecated. See PT_ATTACHEXC") = 10
};


#define	PT_TRACE_ME	0	/* child declares it's being traced */
#define	PT_READ_I	1	/* read word in child's I space */
#define	PT_READ_D	2	/* read word in child's D space */
#define	PT_READ_U	3	/* read word in child's user structure */
#define	PT_WRITE_I	4	/* write word in child's I space */
#define	PT_WRITE_D	5	/* write word in child's D space */
#define	PT_WRITE_U	6	/* write word in child's user structure */
#define	PT_CONTINUE	7	/* continue the child */
#define	PT_KILL		8	/* kill the child process */
#define	PT_STEP		9	/* single step the child */
#define	PT_ATTACH	ePtAttachDeprecated	/* trace some running process */
#define	PT_DETACH	11	/* stop tracing a process */
#define	PT_SIGEXC	12	/* signals as exceptions for current_proc */
#define PT_THUPDATE	13	/* signal for thread# */
#define PT_ATTACHEXC	14	/* attach to running process with signal exception */

#define	PT_FORCEQUOTA	30	/* Enforce quota for root */
#define	PT_DENY_ATTACH	31

#define	PT_FIRSTMACH	32	/* for machine-specific requests */

__BEGIN_DECLS


int	ptrace(int _request, pid_t _pid, caddr_t _addr, int _data);


__END_DECLS

#endif	/* !_SYS_PTRACE_H_ */
```

&emsp;&emsp;`ptrace`函数存放于`user/include/sys/ptrace.h`中，值得注意的是，它是一个用于MAC的头文件而不是iOS，所以为了能调用这个函数，需要手动把这个头文件复制到做防护的项目中，引入并调用。

&emsp;&emsp;可以放心的是，这个头文件不是私有API，不会影响上架审核。其实注意到这个头文件中引入的2个头文件就知道它们本身就是iOS可用的系统头文件。

&emsp;&emsp;利用`ptrace`可以防止进程依附，基于`debugserver`的`lldb`直接对做了防护的App进行依附调试是会报错的，App也会闪退。进行防护时的调用如下：

```c
/* parameter
	_request: 头文件中列出的一些宏对应的值，防止进程依附就是PT_DENY_ATTACH，即31
	_pid: 进程id，0表示自身进程
	_addr: ptrace可以访问第二个参数即指定进程内的寄存器和内存，根据第一个参数做操作，如读写内存，这个参数就是指定地址
	_data: 当ptrace对指定进程的指定地址进行操作时的数据
 */
ptrace(PT_DENY_ATTACH, 0, 0, 0);
```

&emsp;&emsp;调用了`ptrace`函数就能做到最基本的反调试防护了，但是这样并不足够，因为还有反反调试这样的操作存在。。。

&emsp;&emsp;反反调试，顾名思义，也就是不让`ptrace`生效，达到调试目的的操作。实现原理在之前的文章也说过，就是`fishhook`，毫无疑问`ptrace`是个C语言函数，在`Mach-O`文件中有它对应的符号，利用`fishhook`勾住`ptrace`替换成空操作，就能继续进行调试了。

&emsp;&emsp;既然有了反反调试，要是还想防护要怎么做？那就反反反调试呗。。。

&emsp;&emsp;反反反调试有3种方法

* 将`ptrace`函数的调用放到调试顺序最前面，当`ptrace`成为最先执行的代码之一时，后面进行的`fishhook`已经无所谓了。关于代码的加载执行顺序，可以查看之前的文章中提及的代码加载执行顺序，这里的结论就是，若想做到最早调用自己的防护代码，就把防护代码打包成动态库Framework，放在项目依赖的最前面，同样的，这样的动态库最好有点耦合的功能代码，毕竟`Mach-O`文件中，是可以看到动态库的，不影响功能的话，直接删掉防护代码这种事情不要希望逆向开发时不去试。（实际上，QQ就是一个类似这样做法的App，QQ的`Mach-O`文件非常的小，大概只有一些涉及生命周期的的调用，它的主逻辑全都放在了其中一个`Framework`中）

* `sysctl`函数在头文件`usr/include/sys/sysctl.h`内，头文件里定义了很多宏，这些宏是用来拼成控制码的，本文内容为防护，这里就以查询是否有进程依附这个信息来判断App是否正在被进程依附，例子如下：

首先来看一下这个函数：
```c
/*
	这个函数就看声明真的很莫名其妙，让人一脸懵逼的。
	return value: int，整型类型。0，代表操作成功；1，代表操作失败；
	parameter 1: (int *)，整型指针，指向存放控制码的地址，用于传递指定操作的信息
	parameter 2: u_int，无符号整型类型，用于传递第一个参数的大小
	parameter 3: (void *)，指针类型，用于传递操作信息的旧值，对于获取信息时的情况来说，它就是指向查询信息的指针
	parameter 4: (size_t *)，旧值的大小
	parameter 5: (void *), 指针类型，与parameter 3相似，但传递的是新值
	parameter 6: (size_t *), 新值的大小
 */
int	sysctl(int *, u_int, void *, size_t *, void *, size_t);
// 像这么优秀的函数还有两个，如下所示，他们是对sysctl的简化，直接用宏定义好的字符串映射控制码，方便调用
int	sysctlbyname(const char *, void *, size_t *, void *, size_t);
int	sysctlnametomib(const char *, int *, size_t *);
```

再来看下如何查询进程被依附：
```c
// 以数组的方式填写控制码
int name[4];
name[0] = CTL_KERN;			// 内核查看
name[1] = KERN_PROC;		// 查询进程
name[2] = KERN_PROC_PID;	// 传递的参数是进程的id，即PID
name[3] = getpid();			// PID，通过调用getpid()来获得当前进程的id
struct kinfo_proc info;				// 接受进程查询结果信息的结构体，这个结构体的定义也在同一个头文件中，查看也方便
size_t info_size = sizeof(info);	// 结构体的大小
// 调用sysctl时，应注意的就是传参，第3个参数是获得信息的变量，传指针，函数内部赋值后，就能通过这个指针获得想要的信息，c语言很多函数都是这么传递信息的而不是通过返回值。
int error = sysctl(name, sizeof(name)/sizeof(*name), &info, &info_size, 0, 0);
assert(error == 0);			// 0,代表操作成功;1,代表操作失败;
// 最后可以通过位与操作判断当前进程是否依附了进程
// 查看kinfo_proc结构体中的extern_proc结构体（位与usr/include/sys/proc.h中）的p_flag值，与P_TRACED宏位与运算可知道进程有没有被依附
// P_TRACED宏的介绍是：#define	P_TRACED	0x00000800	/* Debugged process being traced */
// 位与运算时结果为0，表示没有被依附；结果非0，就是被进程依附了
(info.kp_proc.p_flag & P_TRACED) == 0
```

**备注：`sysctl`函数与`ptrace`函数一样，是可以被`fishhook`的，所以会同样有`ptrace`函数的的尴尬情况，但不用担心，下面一种方法就是一个大杀器，利用内联汇编实现上面做的防护，使`fishhook`失效，但防动态调试却依然有效！**

* 要做动态调试的防护，基于C语言函数可以通过`fishhook`勾住这种情况，除了将防护代码做成`Framework`并放到开始加载的首批执行代码中外，还可以使用内联汇编的方式调用防护函数。关于系统的一些函数，可以在`usr/include/sys/syscall.h`内找到，内有定义了几百个的系统函数的宏，对应的是这些函数的序号。在知道了这些函数后，结合前面关于汇编语言提及到的知识，就可以直接用内联汇编来执行这些函数了（注意：内里的宏可以看到是被设置了为私有API的，所以不能使用里面定义的宏，而是直接使用宏对应的序号，至于用宏会不会出审核问题，本人没试过，有不怕碰壁的同学可以试试然后告诉一下呗，XD）。例子如下：

```objc
+ (void)load {
	// 这里的内联汇编执行的就是ptrace函数，对应26，它有4个参数，第一个参数是宏PT_DENY_ATTACH，即31，后3个参数上面提及过，都是0。调用效果同ptrace(PT_DENY_ATTACH, 0, 0, 0)
	asm(
	    "mov X0, #31\n"
	    "mov X1, #0\n"
	    "mov X2, #0\n"
	    "mov X3, #0\n"
	    "mov w16, #26\n"
	    "svc #0x80"
	);
	// 以下调用的内联汇编调用的是exit函数，对应1，只有一个参数，它的调用效果同exit(0)
	asm(
        "mov X0, #0\n"
        "mov w16, #1\n"
        "svc #0x80"
    );
}
```

### 总结：

&emsp;&emsp;现在从新回到开始处提及的防重签代码，会发现这一段代码可以深入进行防护的地方不少，并且在检测到是重签时，可以直接用内联汇编调用`exit`函数闪退，而不是乖巧地提示“codesign verification failed.”（XD）。

&emsp;&emsp;App的防护思路在本文中大概都提及了，全是细节，目的就是逼疯逆向开发人，消磨他们的意志，但世上不存在破不开的盾。毕竟连最新的闭源iOS也会被JB（即便是非完美的）, 一个App真要死磕进行逆向，一行一行进行反汇编，终究是能攻破的，所以在防护这方面还存在另一层的做法，那就获知正在逆向的谁，然后进行限制，甚至是封号（微信就是这样子，一个不小心检测到逆向行为，先进行警告，再进行限制，最后进行封号）。

&emsp;&emsp;这里提及到的防重签思路方法基本是通用的。重点是：灵活多变难寻找，生搬硬套死翘翘。
