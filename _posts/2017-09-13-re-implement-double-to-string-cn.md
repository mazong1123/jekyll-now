---
layout: post
title: 重新实现.NET Core的 double.ToString()
date: 2017-09-13 13:02
author: mazong1123
comments: true
categories: [Uncategorized]
published: false
---

# 目前C#中double.ToString()的问题
.NET Core 2.0发布后，整个框架的性能提升了不少。但是仍然有一些基础数据类型的性能不尽入人意，例如`double.ToString()`这个常用方法的性能。一开始.NET Core团队发现这个问题，是因为`double.ToString()`在Linux下的性能比在Windows下[慢7倍](https://github.com/dotnet/coreclr/issues/10390). 后来为了提升性能，.NET Core团队利用Core RT的代码重写了造成性能瓶颈的[_ecvt](https://github.com/dotnet/coreclr/blob/62849aabbad290a81349cc2a642f3bb240677c7f/src/pal/src/cruntime/misctls.cpp#L156)函数，但性能仍然比在Windows下[慢了3倍](https://github.com/dotnet/coreclr/issues/10651)。

您可以分别在Linux和Windows下运行以下示例代码，观察结果 (注，需要安装[BenchMarkDoNet](https://github.com/dotnet/BenchmarkDotNet)):

```csharp
[Benchmark]
[InlineData("fr")]
[InlineData("da")]
[InlineData("ja")]
[InlineData("")]
public void ToString(string culturestring)
{
    double number = 104234.343;
    CultureInfo cultureInfo = new CultureInfo(culturestring);
    foreach (var iteration in Benchmark.Iterations)
        using (iteration.StartMeasurement())
        {
            for (int i = 0; i < innerIterations; i++)
            {
                number.ToString(cultureInfo); number.ToString(cultureInfo); number.ToString(cultureInfo);
                number.ToString(cultureInfo); number.ToString(cultureInfo); number.ToString(cultureInfo);
                number.ToString(cultureInfo); number.ToString(cultureInfo); number.ToString(cultureInfo);
            }
        }
}
```

为什么会造成这个问题呢？首先，对于32bit Windows, `double.ToString()`的实现是用纯汇编直接写的，并不会去调用_ecvt. 代码看[这里](https://github.com/dotnet/coreclr/blob/32f0f9721afb584b4a14d69135bea7ddc129f755/src/vm/i386/fptext.asm#L78)。

其次，虽然64bit的Windows和Linux的`double.ToString()`是同一份代码，但是`_ecvt`的实现依赖于一个C函数[`snprintf`](https://github.com/dotnet/coreclr/blob/62849aabbad290a81349cc2a642f3bb240677c7f/src/pal/src/cruntime/misctls.cpp#L195). 显然Windows下的`snprintf`做了更多优化，性能超过Linux, 才造成了`double.ToString()`在Windows和Linux平台的性能差距。

实际上无论Windows还是Linux, 其性能都非常低。现在的大致流程是:

- 将double通过snprintf转换成字符串。
- 对转换后的字符串做字符串操作，组装成我们期待的格式。

`snprintf`已经很慢了，还需要对其结果做字符串操作，使得性能进一步降低。

能否不依赖`snprintf`, 直接将double转换成我们期待的格式呢？其实早在90年代就已经有了[论文](https://www.cs.indiana.edu/~dyb/pubs/FP-Printing-PLDI96.pdf)，我就是以这个[论文](https://www.cs.indiana.edu/~dyb/pubs/FP-Printing-PLDI96.pdf)为基础重写的`double.ToString()`. 

为了实现论文中的算法，我们需要一些基础知识。

# double数据类型在内存中是如何存储的

众所周知，浮点数是无法精确地被二进制表示的。但是原因是什么呢？ IEEE提出了一个浮点数的设计规范，完整的内容可以参考[Wiki](https://en.wikipedia.org/wiki/Double-precision_floating-point_format).

## IEEE Double-precision Floating Point存储结构
double数据类型在内存中占用64位。这64位分为三个部分，分别是sign(符号, 1bit), exponent(指数, 11bit), fraction(因子, 52bit). 如下图所示:

![double-mem.png](images/double-mem.png)

- 第63位为符号位，表示正负, 以sign表示。
- 62~52位表示指数，以e表示。具体的含义可以看后面的解释。
- 51~0位表示double的具体数值，以f表示。具体的含义可以看后面的解释。

利用这3个概念，就可以表示一个double数值了 (下面的公式指数中为什么要减去1023会在后面讲到):

> (-1)^sign(1.f) * 2^(e - 1023)

或者将f展开，wiki上的图看起来更清晰:

![double-f1.svg](images/double-f1.svg)

进一步展开可得:

![double-f2.svg](images/double-f2.svg)

为什么指数需要减去1023呢？在IEEE的规范中，`e - 1023`叫做[biased exponent](https://en.wikipedia.org/wiki/Exponent_bias). biased exponent的具体概念不会在这篇文章中讲解，目前您只需要记住double的指数不是直接使用e, 而是使用biased exponent (即e - 1023)就行了。

有了以上概念，大家可以试着算一下下面的二进制表示的double数值是多少:

> 0100000000110111000000000000000000000000000000000000000000000000

>0011111111100000000000000000000000000000000000000000000000000000

附上C++查看内存数据的代码:

```cpp
#include <iostream>
#include <bitset>

int main()
{
	double d = 0.5; // 修改成您想查看的数字
	unsigned long long i = *((unsigned long long*)&d);
	std::bitset<64> b(i);
	
	std::cout << b << std::endl;

	return 0;
}
```

## double的精度问题
由于计算机只能用二进制表示double, 所以有一些限制。例如0.5和0.25是很容易表示的（可以试着写一下这两个数的内存数据）. 但是0.26就无法精确地表示了。原因也很直观，0.26落在2^-1和2^-2之间，无法用二进制表示。

针对这种情况，IEEE有一系列的规定来约束使用什么样的二进制内容来表现这种无法精确用二进制表示的浮点数。这个规则不会在这篇文章中具体描述，可以阅读IEEE的相关资料获取信息。

## 什么是round-trip
如果从字面上理解就是"回路"的意思。这是IEEE规定的浮点数转换成字符串的一个约束条件，即如果将一个double转换成字符串，再将这个字符串转换成double, 转换后的double和原来的double二进制内容保持一致，则说明这个字符串转换满足round-trip. 上文已经说过，有些double数据是无法用二进制精确表示的，那么转换成可以阅读的字符串后再转换回来时，有可能就无法再还原原始的二进制内容了。

IEEE 指出，如果一个double要满足round-trip的条件，至少要有17位数字。具体的原因可以参考[这篇文章](http://www.exploringbinary.com/number-of-digits-required-for-round-trip-conversions/)。所以为了要让一个double数字满足round-trip, 在将double转换成字符串时至少要有17位！想办法将其精度提升到17位！看以下的C#例子就可以比较直观地理解round-trip:

```csharp

```

如果一个double数字在转换成字符串后再转换回来，转换后的double数字和转换前的double数字相同，则说这个double数字满足round-trip.

# 重写double.ToString()的核心算法

## dragon4算法简介

## 准备工作: 实现BigInteger

## dragon4算法实现概述

# 优化算法

## 利用启发式除法提升效率

## 利用logTable提升效率

# 总结