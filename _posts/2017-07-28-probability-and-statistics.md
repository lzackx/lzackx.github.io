---
layout: post
title:  "概率与统计（Probability and Statistics）"
date:   2017-07-28 09:00:00
categories: Methodology
---
**概率与统计（Probability and Statistics）**

&emsp;&emsp;概率与统计的篇章比较长，分为以下章节：

#### **随机变量（random variable, r.v.)**

&emsp;&emsp;视觉，在概率中要恰当地调整视觉来观察事件，分为**上帝视觉**和**人类视觉**。

&emsp;&emsp;对于上帝视觉，是可以纵观平行世界的全知状态，概率事件是面积问题。

&emsp;&emsp;对于人类视觉，事件是不确定的，这便是概率。

&emsp;&emsp;随机变量在上帝视觉中，是一个确定的函数；在人类视觉中，其值的出现是概率性的。

#### **概率分布**

&emsp;&emsp;相比起随机变量，概率分布不涉及事件具体发生在哪里，只要知道随机变量，就能算出上帝视觉中能看到的概率在什么条件下是多少这样子的分布了，而这，就是概率分布。概率分布的总和是1。

#### **联合概率、边缘概率、条件概率**

&emsp;&emsp;联合概率：现假设有随机变量X、Y，则联合概率为:

$$P(X,Y)$$

&emsp;&emsp;边缘概率：现假设有随机变量X、Y，则边缘概率为:

$$P(X)$$

或

$$P(Y)$$

&emsp;&emsp;条件概率：现假设有随机变量X、Y，则条件概率为:

$$P(X|Y)$$

或

$$P(Y|X)$$

&emsp;&emsp;联合概率、边缘概率、条件概率的关系如下(Y作为条件)：

$$P(X|Y)=\frac{P(X,Y)}{P(Y)}$$

#### **贝叶斯公式**

&emsp;&emsp;贝叶斯公式：用于求逆问题，公式是通过联合概率、边缘概率、条件概率的关系式逆推出来的到的，如下：

$$P(X=a|Y=b)=\frac{P(X=a,Y=b)}{P(Y=b)}=\frac{P(Y=b|X=a)P(X=a)}{P(Y=b)}$$

&emsp;&emsp;比较简单，但是需要注意的是使用情景，贝叶斯公式是用于求逆问题的，是先知道结果，再求原因。

#### **独立性**

&emsp;&emsp;独立性：现假设有随机变量X、Y互相独立，则会有以下性质：

1

$$P(X=a|Y=b)=P(X=a|Y=\bar{b})$$

2

$$P(X=a|Y=b)=P(X=a)$$

3

$$P(X=a,Y=b):P(X=a,Y=\bar{b})=P(X=\bar{a},Y=b):P(X=\bar{a},Y=\bar{b})$$

4

$$P(X=a,Y=b)=P(X=a)P(Y=b)$$

&emsp;&emsp;结合联合、边缘、条件的关系式，可知独立性第`4`条具有很广的适用性。

&emsp;&emsp;当出现多个随检变量（如X,Y,Z,U,V,W...）的概率分布计算时，条件（`|`）的优先级比联合（`,`）高。

## **离散值的概率分布**

#### **二项分布**

&emsp;&emsp;二项分布：是无序的、基于离散值的、描述发生次数的、随机变量相互独立的一种概率分布。（可与正态分布进行联系，正态分布将在连续值的概率分布章节展开）

&emsp;&emsp;设触发一个按钮发生A事件的概率为`p`，不发生A事件的概率为`q`，求重复触发这个按钮`n`次，事件A发生`k`次的概率分布，定义公式如下：

$$P(X=k)=C_{n}^{k}p^{k}q^{n-k}$$

&emsp;&emsp;顺带补充排列与组合的公式

&emsp;&emsp;排列（有序）公式如下：

$$P_{n}^{k}=\frac{n!}{(n-k)!}$$

&emsp;&emsp;组合（无序）公式如下：

$$C_{n}^{k}=\frac{P_{n}^{k}}{k!}=\frac{n!}{k!(n-k)!}$$

&emsp;&emsp;二项分布的期望，分为两种情况，即随机变量是否互相独立。当随机变量非互相独立时，需要展开累加计算；若为互相独立，则可以结合期望的性质通过如下计算出二项分布的期望值：

$$
E[X]=E[Z_{1}+...+Z_{n}]\\
=E[Z_{1}]+...+E[Z_{n}]\\
=p+...+p=np
$$

&emsp;&emsp;二项分布的方差，同样分为两种情况，即随机变量是否互相独立。当随机变量非互相独立时，需要展开累加计算；若为互相独立，则可以结合方差的性质通过如下计算出二项分布的方差：

$$
\Rightarrow V[X]=V[Z_{1}]+...+V[Z_{n}],p=1,q=0\\
\Rightarrow V[Z_{t}]=E[(Z_{t}-p)^{2}]\\
=(1-p)^{2}p+(0-p)^{2}q=q^{2}p+p^{2}q\\
=pq(q+p)\\
=pq
$$

#### **期望（Expectation）**

&emsp;&emsp;期望：在上帝视觉中，概率是面积问题的基础上，以概率事件的实质事件值为高，那么期望便是体积。

&emsp;&emsp;设随机变量为X，其发生的概率之和为1，当X=k时，其概率为P(X=k)，则这个随机变量的期望如下：

$$E[X]=\sum_{k}kP(X=k)$$

或

$$E[g(X)]=\sum_{k}g(k)P(X=k)$$

&emsp;&emsp;期望的性质，包括平移、缩放和独立性等几方面。

&emsp;&emsp;平移性质，即随机变量加上一个常数，其期望值也将加上这个常数：

$$E[X+c]=E[X]+C$$

&emsp;&emsp;缩放性质，即随机变量乘以一个常数，其期望值也将乘以这个常数：

$$E[cX]=cE[X]$$

&emsp;&emsp;独立性，随机变量是否独立，会影响期望的计算。

&emsp;&emsp;设随机变量X、Y，通用的随机变量相乘的期望如下：

$$E[XY]=\sum_{a}\sum _{b}(f(X=a)\cdot f(Y=b))P(X=a,Y=b)$$

&emsp;&emsp;当随机变量X与Y相互独立时，由概率相互独立的公式可得：

$$
E[XY]=\sum_{a}\sum _{b}(f(X=a)\cdot f(Y=b))P(X=a,Y=b)\\
=\sum_{a}\sum _{b}(f(X=a)\cdot f(Y=b))P(X=a)P(Y=b)\\
=\sum_{a}f(X=a)P(X=a)\cdot \sum _{b}f(Y=b)P(Y=b)\\
=E[X]\cdot E[Y]
$$

&emsp;&emsp;另外，当随机变量Y，是以随机变量X作条件时，有以下公式（可用于最小二乘法）：

$$E[Y|X=a]=\sum_{b}f(Y=b)P(Y=b|X=a)$$

#### **方差（Variance）**

&emsp;&emsp;方差，是描述期望值的离散程度的一项量值，由于实际值与期望的差的绝对值在计算过程中并步方便，所以在描述变差时，往往使用方差。

&emsp;&emsp;方差的定义公式如下：

$$
E[X]=\mu\\
V[X]=E[(X-\mu)^{2}]\\
=\sum_{a}(a-\mu)^{2}P(X=a)
$$

&emsp;&emsp;方差的性质：

1

$$
V[X]=E[(X-\mu)^2]=0\\
\Rightarrow P(X=\mu)=1
$$

2

$$V[X+c]=E[((x+c)-(\mu+c))^{2}]=E[(x-\mu)^2]=V[X]$$

3

$$
V[cX]=E[(cx-c\mu)^{2}]\\
=E[c^{2}(x-\mu)^{2}]\\
=c^{2}E[(x-\mu)^{2}]\\
=c^{2}V[X]
$$

4

$$
E[X]=\mu,V[X]=\sigma^{2},Z=X-\mu\\
\Rightarrow E[Z]=0,X=Z+\mu\\
\Rightarrow E[X^{2}]=E[(Z+\mu)^{2}]\\
=E[Z^{2}+\mu^{2}+2\mu Z]\\
=E[Z^{2}]+\mu^{2}+2\mu E[Z]\\
=E[Z^{2}]+\mu^{2}\\
=E[(X-\mu)^{2}]+\mu^{2}\\
=\sigma^{2}+\mu^{2}\\
\Rightarrow V[X]=E[X^{2}]+E[x]^{2}
$$

5

$$
P(X,Y)=P(X)P(Y)\\
\Rightarrow V[X+Y]=E[((X+Y)-(\mu - \nu))^{2}]\\
=E[((X-\mu)+(Y-\nu))^{2}]\\
=E[(X-\mu)^{2}+(Y-\nu)^2+2(X-\mu)(Y-\nu)]\\
=E[(X-\mu)^{2}]+E[(Y-\nu)^{2}]\\
=V[X]+V[Y]
$$

#### **标准差（Standard Deviation）**

&emsp;&emsp;标准差，是由于方差与实际值单位不一致，无法进行比较而进行的一种让方差开方，致使其值能与实际值进行分析的度量。

&emsp;&emsp;标准差的公式如下：

$$\sigma=\sqrt{V[X]}$$

&emsp;&emsp;标准化，是种将`E[X]=0`，`V[X]=1`的转换处理。标准化的转换公式可以如下推导：

$$
W=aX+b(a>0)\\
\Rightarrow E[W]=0=a\mu+b\\
\Rightarrow V[W]=1=a^{2}\sigma^{2}\\
\Rightarrow a=\frac{1}{\sigma},b=-\frac{\mu}{\sigma}
$$

#### **大数定律**

&emsp;&emsp;独立同分布（independent and identically，i.i.d.)，即对于一个事件，无论发生多少次，每次随机变量互相独立且概率分布相同的分布。

&emsp;&emsp;平均值的期望值（符合i.i.d.）：

$$
Z=\frac{X_{1}+...+X_{n}}{n}\\
\Rightarrow E[Z]\\
=E[\frac{X_{1}+...+X_{n}}{n}]\\
=\frac{E[X_{1}+...+X_{n}]}{n}\\
=\frac{E[X_{1}]+...+E[X_{n}]}{n}\\
=\frac{n\mu}{n}\\
=\mu
$$

&emsp;&emsp;平均值的方差（符合i.i.d.）：

$$
Z=\frac{X_{1}+...+X_{n}}{n}\\
\Rightarrow V[Z]\\
=V[\frac{X_{1}+...+X_{n}}{n}]\\
=\frac{V[X_{1}+...+X_{n}]}{n^{2}}\\
=\frac{V[X_{1}]+...+V[X_{n}]}{n^{2}}\\
=\frac{n\sigma^{2}}{n^{2}}\\
=\frac{\sigma^{2}}{n}
$$

&emsp;&emsp;从平均值的期望值与方差公式可以发现，当`n`趋向于无穷时，期望值不变，方差收敛趋向于0。

## **连续值的概率分布**

#### **累积分布函数 & 概率密度函数**

&emsp;&emsp;和离散值不同的是，连续值可以对数值进行无穷细分，即趋向于0而不等于0，所以在计算上，需要使用微分、积分。连续值的概率分布无法做到像离散值那样使用列表显示，需要以公式进行表达，称为累计分布函数，表示的是随机变量X在范围内的概率分布，如下：

$$F_{X}(a)=P(X\leq a)$$

&emsp;&emsp;对累积分布函数进行微分，就能得到概率密度，表示在x加减无穷小的范围内的值的发生概率，如下：

$$f_{X}(x)={F}'_{X}(x)=\frac{\mathrm{d} F_{X}(x)}{\mathrm{d} x}$$

&emsp;&emsp;所以在知道概率密度时，可以反求概率（累积分布函数），即对概率密度函数进行积分计算，如下：

$$P(a\leq X\leq b)=\int_{a}^{b}f_{X}(x)\mathrm{d}x$$

&emsp;&emsp;结合上述表示，应该明白的是，**概率密度不是概率，但两者可通过微分、积分转化**。


#### **均匀分布**

&emsp;&emsp;均匀分布，满足以下两个条件

* 区间内任意值的概率密度（出现的概率）恒定
* 不会出现区间范围之外的值

$$
f_{X}(x)=\left\{\begin{matrix}
\frac{1}{\beta - \alpha }(\alpha \leq x\leq \beta )\\
0(x\leq \alpha ,x\geq  \beta )
\end{matrix}\right.
$$

#### **变量变换**

&emsp;&emsp;变量变化，即是当随机变量X被另一个随机变量Y等价变换后概率密度函数的变换。如下所示：

$$
X\rightarrow f_{X},Y\rightarrow f_{Y},Y=aX+b\\
\Rightarrow X=\frac{Y-b}{a}\\
\Rightarrow {X}'=\frac{1}{a}\\
\Rightarrow f_{Y}(y)=\left | \frac{1}{a} \right |\cdot f_{X}(\frac{y-b}{a})\\
=\left | \frac{f_{X}(x)}{a} \right |\\
=\left | \frac{f_{X}(\frac{y-b}{a})}{a} \right |
$$

此时，在获得概率密度函数后，可通过积分计算累积概率函数，如下：

$$
P(y_{1}\leq y\leq y_{2})=\int_{y_{1}}^{y_{2}}f_{X}(x)\mathrm{d}x\\
=\int_{y_{1}}^{y_{2}}f_{X}(\frac{y-b}{a})\cdot \frac{\mathrm{d} x}{\mathrm{d} y}\mathrm{d}y
$$

#### **联合概率、边缘概率、条件概率**

&emsp;&emsp;联合概率、边缘概率、条件概率在连续值中的性质与离散值中的性质相似，就是概率中的**级数运算**转换成概率密度中的**积分运算**。性质如下：

1

$$f_{X}(a)=\int_{-\infty }^{\infty }f_{X,Y}(a,y)\mathrm{d}y$$

2

$$
f_{Y|X}(b|a)=\frac{f_{X,Y}}{f_{X}(a)}\\
f_{X,Y}(a,b)=f_{Y|X}(b|a)f_{X}(a)
$$

3

$$f_{X|Y}(a|b)=\frac{f_{Y|X}(b|a)f_{X}(a)}{\int_{-\infty }^{\infty }f_{Y|X}(b|x)f_{X}(x)\mathrm{d}x}$$

4

$$
f_{Y|X}(b|a)=f_{Y}(b)\\
f_{X,Y}(a,b)=f_{X}(a)f_{Y}(b)
$$

#### **期望、方差、标准差**

&emsp;&emsp;期望、方差、标准差在连续值中的性质与离散值中的性质相似，就是概率中的**级数运算**转换成概率密度中的**积分运算**。性质如下：

1

$$E[X]=\int_{-\infty }^{\infty }xf_{X}(x)\mathrm{d}x$$

2

$$E[g(X)]=\int_{-\infty }^{\infty }g(x)f_{X}(x)\mathrm{d}x$$

3

$$E[h(X,Y)]=\int_{-\infty }^{\infty }\int_{-\infty }^{\infty }h(x,y)f_{X,Y}(x,y)\mathrm{d}x\mathrm{d}x$$

4

$$E[aX+b]=aE[X]+b$$

5

$$V[X]=E[(X-\mu)^{2}],\mu=E[X]$$

6

$$V[aX+b]=a^{2}V[X]$$

7

$$\sigma_{X}=\sqrt{V[X]}$$

8

$$\sigma_{aX+b}=\left | a \right |\sigma_{X}$$

9

$$E[Y|X=a]=\int_{-\infty }^{\infty }yf_{Y}(y|X=a)\mathrm{d}y$$

10

$$V[Y|X=a]=E[(Y-\mu(a))^{2}|X=a]$$

#### **正态分布 & 中心极限定理**

&emsp;&emsp;正态分布（Gauss分布，高斯分布），标准正态分布概率密度函数如下：

$$f(z)=\frac{1}{\sqrt{2\pi}}e^{-\frac{z^{2}}{2}}$$

&emsp;&emsp;中心极限定理，认为正态分布随处可发现，是由误差的叠加引起的。





