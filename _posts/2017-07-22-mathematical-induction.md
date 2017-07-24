---
layout: post
title:  "数学归纳法（Mathematical Induction）"
date:   2017-07-22 09:00:00
categories: Methodology
---
**数学归纳法（Mathematical Induction**

&emsp;&emsp;数学归纳法，简单来说就是通过2个步骤，证明断言，而在编程中，数学归纳法可以在循环中得到体现。

&emsp;&emsp;数学归纳法步骤如下：

1. 证明F(0)成立
2. 假设F(n)成立，则对于符合n条件的任一k，都能使F(k)成立，则可以通常使用`左边`=`右边`这种方式证明F(k+1)

&emsp;&emsp;可以把高斯求和作为数学归纳法的最佳实践，即：

1 + 2 + 3 + ... + n = (1 + n) * n / 2

&emsp;&emsp;证明很简单，此处不再计算。