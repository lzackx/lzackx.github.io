---
layout: post
title:  "汇编语言"
date:   2018-03-01 09:00:00
categories: Assembly
---
**汇编语言（Assembly Language）**

汇编涉及的知识点较多，由浅入深会分为以下几个部分（此文并非从零开始讲解汇编，适合有一定汇编基础的读者）：

1. 汇编
2. CPU与内存
3. 汇编例子详解（模拟器）
4. 汇编例子详解（真机）
5. 其他

# 1. 汇编

&emsp;&emsp;日常开发中，利用汇编开发的机会并不多，汇编处于高级语言与机器语言之间的位置。

&emsp;&emsp;汇编代码与CPU具体情况密切相关，编译时，每条汇编指令都与机器指令相对应。
汇编通过编译后，成为机器码，机器码因存放在不同的`Segment`而导致被赋予不同的意义。
（电影The Matrix中，那个吹牛的配角说二进制瀑布流看久了就懂意思的，在了解后就会发现这个技能要获得是非常非常难的。）

&emsp;&emsp;汇编格式分为Intel与AT&T两种，以下是他们的区别（来自Wiki）：

|项目								|Intel风格														|AT&T风格
|-----------------------------------|---------------------------------------------------------------|--------
|操作数顺序							|目标操作数在前													|源操作数在前
|寄存器名字							|原样															|加%前缀
|立即数								|原样															|加$前缀
|16进制立即数							|用后缀b与h分别表示二进制与十六进制 对于16进制字母开头的要加前缀0		|加前缀0x
|访问内存长度的表示					|前缀byte ptr, word ptr, dword ptr								|后缀b、w、l表示字节、字、长型
|引用全局或静态变量var的值				|[_var]															|_var
|引用全局或静态变量var的地址			|\_var															|$_var
|引用局部变量							|需要基于栈指针（ESP）
|内存直接寻址							|seg_reg: [base + index * scale + immed32]						|seg_reg: immed32 (base, index, scale)
|寄存器间址							|[reg]															|(%reg)
|寄存器变址寻址						|[reg + _x]														|_x(%reg)
|立即数变址寻址						|[reg + 1]														|1(%reg)
|整数数组寻址							|[eax*4 + array]												|_array (,%eax, 4)

# 2. CPU与内存

&emsp;&emsp;要理解CPU，就要先理解CPU关于总线的知识。
CPU的总线由芯片管脚延伸（管脚少的或许可以直接插，但现代芯片小，管脚多，多采用贴片方式，工艺这里就不展开了），连接到主板，通过主板线路连接并控制相关资源。

&emsp;&emsp;总线大体分三类，即：

* 地址总线
* 数据总线
* 控制总线

&emsp;&emsp;它们相互配合，读写内存，从而达到处理信息的目的。（非嵌入式开发时，关注地址总线的情况比较多）

### 2.1 地址总线

&emsp;&emsp;地址总线：宽度决定CPU的寻址能力，与内存相关，例如：地址总线宽度为64，那么寻址能力就是（2^64）那么多。

*备注：内存中，每个单位为1Byte，即8Bit，寻址时，每个地址都是按Byte算的。（1 word = 2 Byte = 16 Bit）
比较容易理解的例子就是32位时代时，内存最大只能4GB，那是因为32位CPU寻址能力只有4G(2^32)，在这之后就算加上再多的内存，CPU寻址不了，也是没用，而现在64位时代，没有了这种限制。另外，在ARM64中，对栈的操作是16字节对齐的（可以理解为每个地址指向的数据长度为16Byte，即128Bit）。*

### 2.2 数据总线

&emsp;&emsp;数据总线：宽度决定CPU单词数据传送量，体现为传送速度。

*备注：各家CPU数据总线都有可能不同，例如：在地址总线宽度为64的情况下，地址总线为32条时，一条完整的数据要分高低位（HL，大端：高低高低，小端：高高低低，大小端知识这里不展开）传送2次；如果地址总线为64条，那么只需要传送1次就可以。这就体现出速度了。*

### 2.3 控制总线

&emsp;&emsp;控制总线：发送CPU控制信号，宽度由控制信号多少决定。例如：发送读信号时，读取内存当前寻址地址的值，并通过数据总线返回这个值。

### 2.4 寄存器

&emsp;&emsp;CPU通过数据总线获得数据后，会放在寄存器中，等待运算器对其进行处理。寄存器个数和结构根据CPU不同而不同，汇编就是一门可以操作寄存器的语言。

&emsp;&emsp;通用寄存器有什么，网上一搜一大把，这里就不贴出来了，具体跑起来时怎么样的，在下面会详细讲解。

&emsp;&emsp;这里讲解一下比较关键的几个寄存器，如下：

* CS:IP: 代码段寄存器（Code Segment)及（Instruction Pointer）指令指针寄存器，当IP指向某一地址时，此地址内的值即被CPU认为是代码，并通过CPU解析读取并运算。*（CS:IP无法被直接赋值，只能通过类似`jmp`、`call`等指令来控制。
在寻址能力比较低的时候，为了能获得更强的寻址能力，只用了CS:IP这种形式的寻址方式，现在在寻址能力足够的情况下，可以直接通过IP指向具体地址，减少了与CS的组合。）*
* DS：数据段寄存器（Data Segment），当某个地址赋值给DS时，此地址加上的值即被CPU认为是数据，可通过增加偏移值进行寻址获取值。
* SS:SP：堆栈段寄存器（Stack Segment）及栈顶指针（Stack Pointer)，用于物理意义上的入栈出栈（FILO），SP为栈顶指针。*（当SS:SP被指定时，堆栈大小即被指定，SP通过`push`进行入栈，`pop`进行出栈，需要注意栈顶越界问题。）*

&emsp;&emsp;在ARM64中，因各方面的进步，在寻址和计算方面都得到升级，`PC`寄存器的功能与上述的`CS:IP`类似，它指向将被执行的指令地址。现在比较强大的CPU也都基本拥有一级缓存与二级缓存，更加强大的甚至有三级缓存，用于把内存中的指令缓存到处理器中，在处理器需要时以更快的速度处理。另外，想要读懂ARM64中的汇编，`PC`、`SP`、`FP`、`LR`这几个寄存器要结合观察（以A11处理器为例，通过Xcode查看的汇编中，`FP`、`LR`并不能直观看到，因为它们是`x29`、`x30`通用寄存器，但Xcode的`Variable View`是能看到它们的，同样地，也能看到浮点寄存器和向量寄存器等寄存器。）

&emsp;&emsp;另外，还有一个寄存器`cpsr`（current program status register，当前程序状态寄存器），是一个32位按位标记状态的寄存器，用于存储条件码。其中`adds`、`subs`等指令会影响此寄存器，在分析App的汇编代码时，需要关注最后四位，即28、29、30、31的位数据，分别对应如下：

* 28 Bit，V，溢出标志。溢出时为1，否则为0。
* 29 Bit，C，进借位标志。加法产生进位时，C=1，反之C=0；减法产生借位时，C=0，反之C=1。
* 30 Bit，Z，零标志。执行运算后，结果为0时，Z=1，反之Z=0。
* 31 Bit，N，负数标志。执行运算后，结果为负时，N=1，反之N=0。

# 3. 汇编例子详解（模拟器）

&emsp;&emsp;以上作为讲解的预备知识差不多了，下面将大篇幅根据实际例子进行讲解。（为便于理解，建议分屏跟踪查看）

* 平台：iOS
* 语言：Objective-C
* 源码地址：[源码](https://github.com/lzackx/Zone/tree/master/iOS/AssemblyLanguage)
* 运行环境：模拟器（即x86_64上，若运行环境为真机，即arm64，可以发现汇编格式会改变为Intel风格）

&emsp;&emsp;*下面是ViewController.m文件*
```
#import "ViewController.h"


@interface ViewController ()
@end

@implementation ViewController

// Xcode 9想要这样写C函数不报错，记得在Build SettingS中改下C Language Dialect
int function(int a, int b, int c) {
    
    return a + b + c;
}

- (int)methodWithA:(int)a b:(int)b c:(int)c {
    
    return a + b + c;
}

- (int)methodWithX:(int)x y:(int)y z:(int)z {
    
    function(x, y, z);
    return x + y + z;
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    int parameter1 = 1;
    int parameter2 = 2;
    int parameter3 = 3;
    
    function(parameter1, parameter2, parameter3);
    
    [self methodWithA:parameter1 b:parameter2 c:parameter3];
    
    [self methodWithX:parameter1 y:parameter2 z:parameter3];
    
}

@end
```

&emsp;&emsp;以下汇编皆为在`Debug`的配置下运行。`Release`时，编译器会对代码进行优化，优化等级可在`Build Setting`中设置。

&emsp;&emsp;*下面是viewDidLoad在模拟器中运行时的汇编代码, 为便于讲解，加上了行号*

```
 1 AssemblyLanguage`-[ViewController viewDidLoad]:
 2     0x10867e660 <+0>:   pushq  %rbp
 3     0x10867e661 <+1>:   movq   %rsp, %rbp
 4     0x10867e664 <+4>:   subq   $0x50, %rsp
 5     0x10867e668 <+8>:   leaq   -0x20(%rbp), %rax
 6     0x10867e66c <+12>:  movq   %rdi, -0x8(%rbp)
 7     0x10867e670 <+16>:  movq   %rsi, -0x10(%rbp)
 8     0x10867e674 <+20>:  movq   -0x8(%rbp), %rsi
 9     0x10867e678 <+24>:  movq   %rsi, -0x20(%rbp)
10     0x10867e67c <+28>:  movq   0x263d(%rip), %rsi        ; (void *)0x0000000108680cd0: ViewController
11     0x10867e683 <+35>:  movq   %rsi, -0x18(%rbp)
12     0x10867e687 <+39>:  movq   0x260a(%rip), %rsi        ; "viewDidLoad"
13     0x10867e68e <+46>:  movq   %rax, %rdi
14     0x10867e691 <+49>:  callq  0x10867ea22               ; symbol stub for: objc_msgSendSuper2
15     0x10867e696 <+54>:  movl   $0x1, -0x24(%rbp)
16     0x10867e69d <+61>:  movl   $0x2, -0x28(%rbp)
17     0x10867e6a4 <+68>:  movl   $0x3, -0x2c(%rbp)
18     0x10867e6ab <+75>:  movl   -0x24(%rbp), %edi
19     0x10867e6ae <+78>:  movl   -0x28(%rbp), %esi
20     0x10867e6b1 <+81>:  movl   -0x2c(%rbp), %edx
21     0x10867e6b4 <+84>:  callq  0x10867e5d0               ; function at ViewController.m:17
22     0x10867e6b9 <+89>:  movq   -0x8(%rbp), %rcx
23     0x10867e6bd <+93>:  movl   -0x24(%rbp), %edx
24     0x10867e6c0 <+96>:  movl   -0x28(%rbp), %esi
25     0x10867e6c3 <+99>:  movl   -0x2c(%rbp), %r8d
26     0x10867e6c7 <+103>: movq   0x25d2(%rip), %r9         ; "methodWithA:b:c:"
27     0x10867e6ce <+110>: movq   %rcx, %rdi
28     0x10867e6d1 <+113>: movl   %esi, -0x30(%rbp)
29     0x10867e6d4 <+116>: movq   %r9, %rsi
30     0x10867e6d7 <+119>: movl   -0x30(%rbp), %ecx
31     0x10867e6da <+122>: movl   %eax, -0x34(%rbp)
32     0x10867e6dd <+125>: callq  0x10867ea1c               ; symbol stub for: objc_msgSend
33     0x10867e6e2 <+130>: movq   -0x8(%rbp), %rsi
34     0x10867e6e6 <+134>: movl   -0x24(%rbp), %edx
35     0x10867e6e9 <+137>: movl   -0x28(%rbp), %ecx
36     0x10867e6ec <+140>: movl   -0x2c(%rbp), %r8d
37     0x10867e6f0 <+144>: movq   0x25b1(%rip), %rdi        ; "methodWithX:y:z:"
38     0x10867e6f7 <+151>: movq   %rdi, -0x40(%rbp)
39     0x10867e6fb <+155>: movq   %rsi, %rdi
40     0x10867e6fe <+158>: movq   -0x40(%rbp), %rsi
41     0x10867e702 <+162>: movl   %eax, -0x44(%rbp)
42     0x10867e705 <+165>: callq  0x10867ea1c               ; symbol stub for: objc_msgSend
43     0x10867e70a <+170>: movl   %eax, -0x48(%rbp)
44     0x10867e70d <+173>: addq   $0x50, %rsp
45     0x10867e711 <+177>: popq   %rbp
46     0x10867e712 <+178>: retq
```

&emsp;&emsp;*下面是`int function(int a, int b, int c)`函数的汇编代码, 为便于讲解，加上了行号*

```
 1 AssemblyLanguage`function:
 2     0x10867e5d0 <+0>:  pushq  %rbp
 3     0x10867e5d1 <+1>:  movq   %rsp, %rbp
 4     0x10867e5d4 <+4>:  movl   %edi, -0x4(%rbp)
 5     0x10867e5d7 <+7>:  movl   %esi, -0x8(%rbp)
 6     0x10867e5da <+10>: movl   %edx, -0xc(%rbp)
 7     0x10867e5dd <+13>: movl   -0x4(%rbp), %edx
 8     0x10867e5e0 <+16>: addl   -0x8(%rbp), %edx
 9     0x10867e5e3 <+19>: addl   -0xc(%rbp), %edx
10     0x10867e5e6 <+22>: movl   %edx, %eax
11     0x10867e5e8 <+24>: popq   %rbp
12     0x10867e5e9 <+25>: retq
```

&emsp;&emsp;*下面是`- (int)methodWithA:(int)a b:(int)b c:(int)c`方法的汇编代码, 为便于讲解，加上了行号*

```
 1 AssemblyLanguage`-[ViewController methodWithA:b:c:]:
 2     0x10867e5f0 <+0>:  pushq  %rbp
 3     0x10867e5f1 <+1>:  movq   %rsp, %rbp
 4     0x10867e5f4 <+4>:  movq   %rdi, -0x8(%rbp)
 5     0x10867e5f8 <+8>:  movq   %rsi, -0x10(%rbp)
 6     0x10867e5fc <+12>: movl   %edx, -0x14(%rbp)
 7     0x10867e5ff <+15>: movl   %ecx, -0x18(%rbp)
 8     0x10867e602 <+18>: movl   %r8d, -0x1c(%rbp)
 9     0x10867e606 <+22>: movl   -0x14(%rbp), %ecx
10     0x10867e609 <+25>: addl   -0x18(%rbp), %ecx
11     0x10867e60c <+28>: addl   -0x1c(%rbp), %ecx
12     0x10867e60f <+31>: movl   %ecx, %eax
13     0x10867e611 <+33>: popq   %rbp
14     0x10867e612 <+34>: retq
```

&emsp;&emsp;*下面是`- (int)methodWithX:(int)x y:(int)y z:(int)z`方法的汇编代码, 为便于讲解，加上了行号*

```
 1 AssemblyLanguage`-[ViewController methodWithX:y:z:]:
 2     0x10867e620 <+0>:  pushq  %rbp
 3     0x10867e621 <+1>:  movq   %rsp, %rbp
 4     0x10867e624 <+4>:  subq   $0x20, %rsp
 5     0x10867e628 <+8>:  movq   %rdi, -0x8(%rbp)
 6     0x10867e62c <+12>: movq   %rsi, -0x10(%rbp)
 7     0x10867e630 <+16>: movl   %edx, -0x14(%rbp)
 8     0x10867e633 <+19>: movl   %ecx, -0x18(%rbp)
 9     0x10867e636 <+22>: movl   %r8d, -0x1c(%rbp)
10     0x10867e63a <+26>: movl   -0x14(%rbp), %edi
11     0x10867e63d <+29>: movl   -0x18(%rbp), %esi
12     0x10867e640 <+32>: movl   -0x1c(%rbp), %edx
13     0x10867e643 <+35>: callq  0x10867e5d0               ; function at ViewController.m:17
14     0x10867e648 <+40>: movl   -0x14(%rbp), %ecx
15     0x10867e64b <+43>: addl   -0x18(%rbp), %ecx
16     0x10867e64e <+46>: addl   -0x1c(%rbp), %ecx
17     0x10867e651 <+49>: movl   %eax, -0x20(%rbp)
18     0x10867e654 <+52>: movl   %ecx, %eax
19     0x10867e656 <+54>: addq   $0x20, %rsp
20     0x10867e65a <+58>: popq   %rbp
21     0x10867e65b <+59>: retq  
```

### 3.1 变量

##### 3.1.1 局部变量

&emsp;&emsp;这里以`viewDidLoad`方法第3、4行的指令为例，即：

```
movq   %rsp, %rbp
subq   $0x50, %rsp
```

&emsp;&emsp;这里在进入方法后，编译器知道方法内部有局部变量，所以在把Objective-C方法编译成汇编时，会开辟存放局部变量的栈空间。

&emsp;&emsp;从这句指令可以发现，它的本质就是把`sp`（栈顶指针）地址减0x50，那么就等于开辟了0x50那么大的栈空间，并且通过第15～17行代码，即如下指令，把局部变量1、2、3赋值到对应空间的内存中。
这里或许会奇怪，`bp`又是什么鬼？`bp`其实就是基数指针，因为`SS:SP`之间的差值具有表达栈大小的意义，所以在需要读写栈中内存时，需要这个叫`bp`的指针。

```
movl   $0x1, -0x24(%rbp)
movl   $0x2, -0x28(%rbp)
movl   $0x3, -0x2c(%rbp)
```

##### 3.1.2 参数

&emsp;&emsp;以`viewDidLoad`方法第18～20行的指令为例，即：

```
movl   -0x24(%rbp), %edi
movl   -0x28(%rbp), %esi
movl   -0x2c(%rbp), %edx
```

&emsp;&emsp;因为在第15～17行中，已经把局部变量赋值到栈内存中，所以通过一样的地址，可以把它们都分别取出来并赋值到其他的通用寄存器中。

&emsp;&emsp;这里需要注意的是，并不一定所有情况都是直接赋值到通用寄存器中的。当参数数量很多，多于用于存储参数的寄存器数量时，编译器无法把所有参数放到寄存器中，于是参数会被入栈，当需要用时再出栈。

**备注：当参数以入栈的形式传递时，栈中的顺序将变成如下所示的形式，在`bp`保护现场后，`bp`寄存器加出来的获取的变量必定是局部变量，而减出来获得的变量必定是参数**

![CallStack](https://github.com/lzackx/lzackx.github.io/blob/master/images/2018-03-01-assembly-language-call-stack.png?raw=true)

##### 3.1.3 返回值

&emsp;&emsp;返回值在汇编中，是比较不起眼的，这里以`function`中第10行中的汇编指令为例，即：

```
movl   %edx, %eax
```

&emsp;&emsp;可以看到，这里的`ax`内的值，便是返回值，`ax`是一个通用寄存器，CPU能在任何时候访问，所以当函数或方法执行完毕后，在重新给这个寄存器赋值前，如果需要获取该值，编译器会直接读取这个寄存器的值。

### 3.2 函数或方法

&emsp;&emsp;Objective-C与其他语言都不同的一点是，其他语言要执行一个函数或方法的时候，叫做"调用"，而在Objective-C中，会被叫作“发送信息”。
在这里可以稍微看到，每一个`callq`后面都被注释了`objc_msgSend`或`objc_msgSendSuper2`，这就是它的本质。鉴于本文主题并非这个，所以就不展开了。

&emsp;&emsp;现在以`viewDidLoad`方法内第21行的汇编代码为例，它是调用`function`的一段指令，如下：

```
callq  0x10867e5d0
```

&emsp;&emsp;这是一条`call`指令，它会把下一条指令的地址（在这里就是第22行的地址0x10867e6b9）入栈，然后再把`ip`指向`0x10867e5d0`这个地址，并执行下去。

&emsp;&emsp;`0x10867e5d0`这个地址无疑就是`function`函数的入口地址，在该函数调用完毕后，可以看到`function`的最后一行，即如下指令。

```
retq
```

&emsp;&emsp;该指令就是把一开始入栈了的0x10867e6b9指令（即进入该函数前的下一条指令的地址）出栈，并让`ip`指向它，使其跳出`function`函数，并继续执行剩下的指令。

### 3.3 栈平衡

&emsp;&emsp;栈平衡产生的原因，在于`call`指令之后，会进行入栈操作，`sp`在不断的入栈出栈后，指向不同的地址，当所有指令执行完毕后，若`sp`最后`pop`出来的不是`call`前的下一条指令，多出来的栈空间在重复这样操作后，就会出现因内存泄漏导致的内存溢出。

##### 3.3.1 外平栈

&emsp;&emsp;在`call`外部入栈后，在`call`的`ret`后，需要把`sp`回到入栈前的地址。

##### 3.3.2 内平栈

&emsp;&emsp;外平栈的做法不适合放在函数内部，因为`ret`前进行`sp`地址操作，会让`ret`取不到`call`前入栈的下一条指令的地址。但可以让`ret`后加地址偏移值来达到这个效果。

### 3.4 现场保护

&emsp;&emsp;可以看到基本每个函数或方法，在开始与结束时，都会有相同的指令，如下：

```
pushq  %rbp
movq   %rsp, %rbp
...
popq   %rbp
```

&emsp;&emsp;这几句指令，是为了保护`bp`在`call`前的值，防止在当前函数或方法调用其他函数或方法后`bp`值丢失导致无法返回前一个函数或方法并继续执行下去的保护操作。

&emsp;&emsp;同样的，不止`bp`如此，其他通用寄存器也需要进行入栈出栈的保护操作。


# 4. 汇编例子详解（真机）

### 4.1 汇编示例

&emsp;&emsp;上述同样的代码，运行在真机中时的汇编如下：

&emsp;&emsp;*下面是viewDidLoad在模拟器中运行时的汇编代码, 为便于讲解，加上了行号*

```
 1 AssemblyLanguage`-[ViewController viewDidLoad]:
 2     0x1043ae700 <+0>:   sub    sp, sp, #0x50             ; =0x50
 3     0x1043ae704 <+4>:   stp    x29, x30, [sp, #0x40]
 4     0x1043ae708 <+8>:   add    x29, sp, #0x40            ; =0x40
 5     0x1043ae70c <+12>:  add    x8, sp, #0x20             ; =0x20
 6     0x1043ae710 <+16>:  adrp   x9, 2
 7     0x1043ae714 <+20>:  add    x9, x9, #0xc98            ; =0xc98
 8     0x1043ae718 <+24>:  adrp   x10, 2
 9     0x1043ae71c <+28>:  add    x10, x10, #0xcc0          ; =0xcc0
10     0x1043ae720 <+32>:  stur   x0, [x29, #-0x8]
11     0x1043ae724 <+36>:  stur   x1, [x29, #-0x10]
12     0x1043ae728 <+40>:  ldur   x0, [x29, #-0x8]
13     0x1043ae72c <+44>:  str    x0, [sp, #0x20]
14     0x1043ae730 <+48>:  ldr    x10, [x10]
15     0x1043ae734 <+52>:  str    x10, [sp, #0x28]
16     0x1043ae738 <+56>:  ldr    x1, [x9]
17     0x1043ae73c <+60>:  mov    x0, x8
18     0x1043ae740 <+64>:  bl     0x1043aeb6c               ; symbol stub for: objc_msgSendSuper2
19     0x1043ae744 <+68>:  orr    w11, wzr, #0x3
20     0x1043ae748 <+72>:  orr    w12, wzr, #0x2
21     0x1043ae74c <+76>:  orr    w13, wzr, #0x1
22     0x1043ae750 <+80>:  str    w13, [sp, #0x1c]
23     0x1043ae754 <+84>:  str    w12, [sp, #0x18]
24     0x1043ae758 <+88>:  str    w11, [sp, #0x14]
25     0x1043ae75c <+92>:  ldr    w0, [sp, #0x1c]
26     0x1043ae760 <+96>:  ldr    w1, [sp, #0x18]
27     0x1043ae764 <+100>: ldr    w2, [sp, #0x14]
28     0x1043ae768 <+104>: bl     0x1043ae648               ; function at ViewController.m:17
29     0x1043ae76c <+108>: adrp   x8, 2
30     0x1043ae770 <+112>: add    x8, x8, #0xca0            ; =0xca0
31     0x1043ae774 <+116>: ldur   x9, [x29, #-0x8]
32     0x1043ae778 <+120>: ldr    w2, [sp, #0x1c]
33     0x1043ae77c <+124>: ldr    w3, [sp, #0x18]
34     0x1043ae780 <+128>: ldr    w4, [sp, #0x14]
35     0x1043ae784 <+132>: ldr    x1, [x8]
36     0x1043ae788 <+136>: str    w0, [sp, #0x10]
37     0x1043ae78c <+140>: mov    x0, x9
38     0x1043ae790 <+144>: bl     0x1043aeb60               ; symbol stub for: objc_msgSend
39     0x1043ae794 <+148>: adrp   x8, 2
40     0x1043ae798 <+152>: add    x8, x8, #0xca8            ; =0xca8
41     0x1043ae79c <+156>: ldur   x9, [x29, #-0x8]
42     0x1043ae7a0 <+160>: ldr    w2, [sp, #0x1c]
43     0x1043ae7a4 <+164>: ldr    w3, [sp, #0x18]
44     0x1043ae7a8 <+168>: ldr    w4, [sp, #0x14]
45     0x1043ae7ac <+172>: ldr    x1, [x8]
46     0x1043ae7b0 <+176>: str    w0, [sp, #0xc]
47     0x1043ae7b4 <+180>: mov    x0, x9
48     0x1043ae7b8 <+184>: bl     0x1043aeb60               ; symbol stub for: objc_msgSend
49     0x1043ae7bc <+188>: str    w0, [sp, #0x8]
50     0x1043ae7c0 <+192>: ldp    x29, x30, [sp, #0x40]
51     0x1043ae7c4 <+196>: add    sp, sp, #0x50             ; =0x50
52     0x1043ae7c8 <+200>: ret    
```

&emsp;&emsp;*下面是`int function(int a, int b, int c)`函数的汇编代码, 为便于讲解，加上了行号*

```
 1 AssemblyLanguage`function:
 2     0x1043ae648 <+0>:  sub    sp, sp, #0x10             ; =0x10
 3     0x1043ae64c <+4>:  str    w0, [sp, #0xc]
 4     0x1043ae650 <+8>:  str    w1, [sp, #0x8]
 5     0x1043ae654 <+12>: str    w2, [sp, #0x4]
 6     0x1043ae658 <+16>: ldr    w0, [sp, #0xc]
 7     0x1043ae65c <+20>: ldr    w1, [sp, #0x8]
 8     0x1043ae660 <+24>: add    w0, w0, w1
 9     0x1043ae664 <+28>: ldr    w1, [sp, #0x4]
10     0x1043ae668 <+32>: add    w0, w0, w1
11     0x1043ae66c <+36>: add    sp, sp, #0x10             ; =0x10
12     0x1043ae670 <+40>: ret  
```

&emsp;&emsp;*下面是`- (int)methodWithA:(int)a b:(int)b c:(int)c`方法的汇编代码, 为便于讲解，加上了行号*

```
 1 AssemblyLanguage`-[ViewController methodWithA:b:c:]:
 2     0x1043ae674 <+0>:  sub    sp, sp, #0x20             ; =0x20
 3     0x1043ae678 <+4>:  str    x0, [sp, #0x18]
 4     0x1043ae67c <+8>:  str    x1, [sp, #0x10]
 5     0x1043ae680 <+12>: str    w2, [sp, #0xc]
 6     0x1043ae684 <+16>: str    w3, [sp, #0x8]
 7     0x1043ae688 <+20>: str    w4, [sp, #0x4]
 8     0x1043ae68c <+24>: ldr    w2, [sp, #0xc]
 9     0x1043ae690 <+28>: ldr    w3, [sp, #0x8]
10     0x1043ae694 <+32>: add    w2, w2, w3
11     0x1043ae698 <+36>: ldr    w3, [sp, #0x4]
12     0x1043ae69c <+40>: add    w0, w2, w3
13     0x1043ae6a0 <+44>: add    sp, sp, #0x20             ; =0x20
14     0x1043ae6a4 <+48>: ret
```

&emsp;&emsp;*下面是`- (int)methodWithX:(int)x y:(int)y z:(int)z`方法的汇编代码, 为便于讲解，加上了行号*

```
 1 AssemblyLanguage`-[ViewController methodWithX:y:z:]:
 2     0x1043ae6a8 <+0>:  sub    sp, sp, #0x30             ; =0x30
 3     0x1043ae6ac <+4>:  stp    x29, x30, [sp, #0x20]
 4     0x1043ae6b0 <+8>:  add    x29, sp, #0x20            ; =0x20
 5     0x1043ae6b4 <+12>: stur   x0, [x29, #-0x8]
 6     0x1043ae6b8 <+16>: str    x1, [sp, #0x10]
 7     0x1043ae6bc <+20>: str    w2, [sp, #0xc]
 8     0x1043ae6c0 <+24>: str    w3, [sp, #0x8]
 9     0x1043ae6c4 <+28>: str    w4, [sp, #0x4]
10     0x1043ae6c8 <+32>: ldr    w0, [sp, #0xc]
11     0x1043ae6cc <+36>: ldr    w1, [sp, #0x8]
12     0x1043ae6d0 <+40>: ldr    w2, [sp, #0x4]
13     0x1043ae6d4 <+44>: bl     0x1043ae648               ; function at ViewController.m:17
14     0x1043ae6d8 <+48>: ldr    w1, [sp, #0xc]
15     0x1043ae6dc <+52>: ldr    w2, [sp, #0x8]
16     0x1043ae6e0 <+56>: add    w1, w1, w2
17     0x1043ae6e4 <+60>: ldr    w2, [sp, #0x4]
18     0x1043ae6e8 <+64>: add    w1, w1, w2
19     0x1043ae6ec <+68>: str    w0, [sp]
20     0x1043ae6f0 <+72>: mov    x0, x1
21     0x1043ae6f4 <+76>: ldp    x29, x30, [sp, #0x20]
22     0x1043ae6f8 <+80>: add    sp, sp, #0x30             ; =0x30
23     0x1043ae6fc <+84>: ret
```

### 4.2 与模拟器中运行的区别

&emsp;&emsp;显然，真机(ARM64)与模拟器(x86_64)的汇编区别很大，如下：

* 汇编格式不一样了。
* ARM64中能改变`pc`寄存器的地址的指令是`b`或`bl`等指令。
* ARM64中使用`fp`作用与`bp`相同，即：为`sp`记录栈低。
* 链接寄存器`lr`保存了`bl`下一条指令的地址（其实是`pc`执行时，是先把地址加了后再执行当前指令的），当`ret`执行后`pc`指向`lr`指向的地址，从而达到平栈目的。
* 函数的参数是存放在`x0`到`x7`(`w0`到`w7`)这8个寄存器里面的。如果超过8个参数，就会入栈。
* 函数的返回值是放在`x0`寄存器里面的.

*（备注：在A11处理器中，`fp`是`x29`、`lr`是`x30`，汇编代码中没有显式显示寄存器名。利用参数和返回值的存放规律，结合`Runtime`中函数的调用方式，可以从`x0`中获得`self`、从`x1`中获得`_cmd`，并在return后通过`x0`获得返回值）*

# 5. 其他

&emsp;&emsp;这里将补充一些能帮助阅读汇编的小Tips：

* Xcode菜单中`Debug`下`Debug Workflow`的`Always Show Disassembly`可让断点触发时查看经过反汇编的汇编指令。
* 断点时，`register`和`memory`可帮助查看寄存器和内存。
* 每一条汇编指令执行时，`ip`都会先指向下一条指令的地址，再执行。这一点很重要！在通过偏移地址查内存值时，找下一条汇编指令地址加偏移地址得出的地址，才是指令操作的实际地址！
* 在通过汇编代码进行静态分析时，遇见`adrp`时，需要连同下一句一起看，同时对于`adrp`指令，从`pc`指向的地址上，进行低12位的0掩码与操作，然后加上左移了12位的页数（一页大小为4KB，所以是12位，2^12，0x1000），即为寻址的目标地址，[ARM架构手册](https://static.docs.arm.com/ddi0487/ca/DDI0487C_a_armv8_arm.pdf)中的描述如下：

```
Literal variant
ADRP <Xd>, <label>

Decode for this encoding
integer d = UInt(Rd); 
bits(64) imm;
imm = SignExtend(immhi:immlo:Zeros(12), 64);

Assembler symbols
<Xd> Is the 64-bit name of the general-purpose destination register, encoded in the "Rd" field.
<label> Is the program label whose 4KB page address is to be calculated. Its offset from the page address of this instruction, in the range +/-4GB, is encoded as "immhi:immlo" times 4096.

Operation
bits(64) base = PC[]; 
base<11:0> = Zeros(12); 
X[d] = base + imm;
```

* 同一份代码多次运行查看断点后，可以发现，每条指令的后几位都是一样的，那就是偏移地址，而基地值就是前面不同的几位，这是因为现在`ip`的位数足够多，不需要像以前那样需要定义`cs:ip`才能定位到实际地址。
* 默认状态下，`Release`和`Debug`模式下编译出来的的汇编略有不同，那是因为`Release`下编译时，会被编译器优化，一些比较容易得出的值，会直接得出使用，减少指令数，运行起来就稍微比`Debug`情况下的要快点了。可以在`Build Settings`的`Optimization Level`设置优化等级。
* 在汇编代码中要判断控制流的代码，需要注意`b`系列的汇编指令。另外，switch的本质其实是对if的优化，当case值多，但它们之间的差别不大时，编译器会在内存中建立一个表，通过直接寻址跳到对应的case值中执行指令，否则与if的差别不大。
* 一个多线程运行中的程序，寄存器的数据保护由操作系统负责，App的汇编代码中只能发现对内存的保护。
* Linux系统中，终端输入`uname -m`就可以对查找适合该CPU的汇编风格了。









