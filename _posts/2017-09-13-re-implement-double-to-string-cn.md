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
.NET Core 2.0发布后，整个框架的性能提升了不少。但是仍然有一些基础数据类型的性能不尽入人意，例如`double.ToString()`这个常用方法的性能。一开始.NET Core团队发现这个问题，是因为`double.ToString()`在Linux下的性能比在Windows下[慢7倍](https://github.com/dotnet/coreclr/issues/10390). 后来为了提升性能，.NET Core团队利用Core RT的代码重写了造成性能瓶颈的 [_ecvt](https://github.com/dotnet/coreclr/blob/62849aabbad290a81349cc2a642f3bb240677c7f/src/pal/src/cruntime/misctls.cpp#L156) 函数，但性能仍然比在Windows下[慢了3倍](https://github.com/dotnet/coreclr/issues/10651)。

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
namespace roundtrip
{
    class Program
    {
        static void Main(string[] args)
        {
            double d = 1.11;

            string normal = d.ToString();
            string roundTrippable = d.ToString("G17");

            Console.WriteLine(normal);
            Console.WriteLine(roundTrippable);
        }
    }
}
```

输出如下:

```sh
1.11
1.1100000000000001
```

> 注意: 如果你查看老的msdn文档，会发现微软建议使用ToString("R")来保证round-trip. 但实际上使用"R"是有bug的，目前已经改为使用"G17". 如果你想知道上下文，可以查看[这个PR](https://github.com/dotnet/coreclr/pull/13140)和[这个issue](https://github.com/dotnet/docs/issues/2919).


# 重写double.ToString()的核心算法
有了前面的理论基础，我们可以着手开始重写`double.ToString()`了。但是，并不是简单地实现[这篇论文](https://www.cs.indiana.edu/~dyb/pubs/FP-Printing-PLDI96.pdf)就能够满足需求，还需要将这个算法完美地嵌入到.NET Core的代码中。

## dragon4算法简介
Robert的这篇论文的算法有个别名，叫做`dragon4`. 别名的来历也很有趣，有兴趣的朋友可以去查一下。

算法的核心思想如下:

1. 为了保持精度，需要将double用整形来表示。可以想到，double可以用分子和分母两个整数表示。这样一来，在转换过程中，所有的计算都是整数算数运算，都不会损失精度。
2. 使用分子和分母两个整数来表示double带来了另一个问题。由于double由64位二进制数字表示，那么分子或分母都有可能会超过64位整数（为什么:)?)。即便用`unsigned long long`也无法表示分子和分母。所以只能摈弃内置的数据类型，自己实现BigInteger数据类型，用于表示分子和分母。 
3. 转换后的字符串实际上存有2个信息，一个是具体数字是什么，一个是小数点的位置。
4. 数字可以用BigInteger的除法计算得来，但是应该保留多少位数字呢？论文中有2种实现，一种是free format, 意思是"一直输出数字，直到能够唯一表示这个double为止" （这是一个比较复杂的判断，这篇文章不详述）。另一种是fixed format, 意思是你要告诉算法，你期望输出多少位数字。.NET Core目前使用的是fixed format方式，所以我们只需要实现fixed format方式就可以了。
5. 小数点位置的判断

## 准备工作: 实现BigInteger

C++并没有内置的BigInteger实现，为了实现BigInteger, 我们可以使用一个`unsigned int`的数组来表示数字。例如

```cpp
static const UINT32 BIGSIZE = 35;
UINT32 m_blocks[BIGSIZE];
```

BigInteger可以表示为:

```
m_blocks[0] << 32 * 0 + m_blocks[1] << 32 * 1 + ...
```

具体的BigInteger实现可以看[bignum.h](https://github.com/dotnet/coreclr/blob/master/src/classlibnative/bcltype/bignum.h)和[bignum.cpp](https://github.com/dotnet/coreclr/blob/master/src/classlibnative/bcltype/bignum.cpp)。

## dragon4算法实现概述

首先我们需要排除特殊数字。比如NaN, 0, Infinite (具体的概念不在这里阐述，可以查看IEEE对double的定义).

对于NaN和Infinite, .NET Core原本的代码已经进行过特殊处理，即[把存放数字的数组清空](https://github.com/dotnet/coreclr/blob/master/src/classlibnative/bcltype/number.cpp#L427):

```cpp
void DoubleToNumber(double value, int precision, NUMBER* number)
{
    WRAPPER_NO_CONTRACT
    _ASSERTE(number != NULL);

    number->precision = precision;
    if (((FPDOUBLE*)&value)->exp == 0x7FF) {
        number->scale = (((FPDOUBLE*)&value)->mantLo || ((FPDOUBLE*)&value)->mantHi) ? SCALE_NAN: SCALE_INF;
        number->sign = ((FPDOUBLE*)&value)->sign;
        number->digits[0] = 0;
    }
    else {
        DoubleToNumberWorker(value, precision, &number->scale, &number->sign, number->digits);
    }
}
```

对于0, 可以在进入核心算法之前[过滤掉](https://github.com/dotnet/coreclr/blob/master/src/classlibnative/bcltype/number.cpp#L169):

```cpp
// Shortcut for zero.
if (value == 0.0)
{
    *dec = 0;
    *sign = 0;

    // Instead of zeroing digits, we just make it as an empty string due to performance reason.
    *digits = 0;

    return;
} 
```
这里为了性能用了一个技巧，原本根据论文，digits应该按照要求的精度填充等长的0，但实际情况下这么做浪费了填充内存的时间，直接把digits赋值为空数组既不影响后续使用，也节约了时间。虽然NaN和Infinite也把digits设置为了空数组, 但是通过exponent和mantissa可以轻易区分0, NaN和Infinite.

接下来进入算法的主步骤:

首先我们需要按照论文把fraction和biased exponent计算出来:

```cpp
 // Step 1:
    // Extract meta data from the input double value.
    //
    // Refer to IEEE double precision floating point format.
    UINT64 f = 0;
    int e = 0;
    UINT32 mantissaHighBitIdx = 0;
    if (((FPDOUBLE*)&value)->exp != 0)
    {
        // For normalized value, according to https://en.wikipedia.org/wiki/Double-precision_floating-point_format
        // value = 1.fraction * 2^(exp - 1023) 
        //       = (1 + mantissa / 2^52) * 2^(exp - 1023) 
        //       = (2^52 + mantissa) * 2^(exp - 1023 - 52)
        //
        // So f = (2^52 + mantissa), e = exp - 1075; 
        f = ((UINT64)(((FPDOUBLE*)&value)->mantHi) << 32) | ((FPDOUBLE*)&value)->mantLo + ((UINT64)1 << 52);
        e = ((FPDOUBLE*)&value)->exp - 1075;
        mantissaHighBitIdx = 52;
    }
    else
    {
        // For denormalized value, according to https://en.wikipedia.org/wiki/Double-precision_floating-point_format
        // value = 0.fraction * 2^(1 - 1023)
        //       = (mantissa / 2^52) * 2^(-1022)
        //       = mantissa * 2^(-1022 - 52)
        //       = mantissa * 2^(-1074)
        // So f = mantissa, e = -1074
        f = ((UINT64)(((FPDOUBLE*)&value)->mantHi) << 32) | ((FPDOUBLE*)&value)->mantLo;
        e = -1074;
        mantissaHighBitIdx = BigNum::LogBase2(f);
    }
```

接下来估算小数点的位置

```cpp
 // Step 2:
    // Estimate k. We'll verify it and fix any error later.
    //
    // This is an improvement of the estimation in the original paper.
    // Inspired by http://www.ryanjuckett.com/programming/printing-floating-point-numbers/
    //
    // LOG10V2 = 0.30102999566398119521373889472449
    // DRIFT_FACTOR = 0.69 = 1 - log10V2 - epsilon (a small number account for drift of floating point multiplication)
    int k = (int)(ceil(double((int)mantissaHighBitIdx + e) * LOG10V2 - DRIFT_FACTOR));

    // Step 3:
    // Store the input double value in BigNum format.
    //
    // To keep the precision, we represent the double value as r/s.
    // We have several optimization based on following table in the paper.
    //
    //     ----------------------------------------------------------------------------------------------------------
    //     |               e >= 0                   |                         e < 0                                 |
    //     ----------------------------------------------------------------------------------------------------------
    //     |  f != b^(P - 1)  |  f = b^(P - 1)      | e = min exp or f != b^(P - 1) | e > min exp and f = b^(P - 1) |
    // --------------------------------------------------------------------------------------------------------------
    // | r |  f * b^e * 2     |  f * b^(e + 1) * 2  |          f * 2                |            f * b * 2          |
    // --------------------------------------------------------------------------------------------------------------
    // | s |        2         |        b * 2        |          b^(-e) * 2           |            b^(-e + 1) * 2     |
    // --------------------------------------------------------------------------------------------------------------  
    //
    // Note, we do not need m+ and m- because we only support fixed format input here.
    // m+ and m- are used for free format input, which need to determine the exact range of values 
    // that would round to value when input so that we can generate the shortest correct digits.
    //
    // In our case, we just output digits until reaching the expected precision. 
    BigNum r(f);
    BigNum s;
    if (e >= 0)
    {
        // When f != b^(P - 1):
        // r = f * b^e * 2
        // s = 2
        // value = r / s = f * b^e * 2 / 2 = f * b^e / 1
        //
        // When f = b^(P - 1):
        // r = f * b^(e + 1) * 2
        // s = b * 2
        // value = r / s =  f * b^(e + 1) * 2 / b * 2 = f * b^e / 1
        //
        // Therefore, we can simply say that when e >= 0:
        // r = f * b^e = f * 2^e
        // s = 1

        r.ShiftLeft(e);
        s.SetUInt64(1);
    }
    else
    {
        // When e = min exp or f != b^(P - 1):
        // r = f * 2
        // s = b^(-e) * 2
        // value = r / s = f * 2 / b^(-e) * 2 = f / b^(-e)
        //
        // When e > min exp and f = b^(P - 1):
        // r = f * b * 2
        // s = b^(-e + 1) * 2
        // value = r / s =  f * b * 2 / b^(-e + 1) * 2 = f / b^(-e)
        //
        // Therefore, we can simply say that when e < 0:
        // r = f
        // s = b^(-e) = 2^(-e)

        BigNum::ShiftLeft(1, -e, s);
    }

    // According to the paper, we should use k >= 0 instead of k > 0 here.
    // However, if k = 0, both r and s won't be changed, we don't need to do any operation.
    //
    // Following are the Scheme code from the paper:
    // --------------------------------------------------------------------------------
    // (if (>= est 0)
    // (fixup r (∗ s (exptt B est)) m+ m− est B low-ok? high-ok? )
    // (let ([scale (exptt B (− est))])
    // (fixup (∗ r scale) s (∗ m+ scale) (∗ m− scale) est B low-ok? high-ok? ))))
    // --------------------------------------------------------------------------------
    //
    // If est is 0, (∗ s (exptt B est)) = s, (∗ r scale) = (* r (exptt B (− est)))) = r.
    //
    // So we just skip when k = 0.
    
    if (k > 0)
    {
        BigNum poweredValue;
        BigNum::Pow10(k, poweredValue);
        s.Multiply(poweredValue);
    }
    else if (k < 0)
    {
        BigNum poweredValue;
        BigNum::Pow10(-k, poweredValue);
        r.Multiply(poweredValue);
    }

    if (BigNum::Compare(r, s) >= 0)
    {
        // The estimation was incorrect. Fix the error by increasing 1.
        k += 1;
    }
    else
    {
        r.Multiply10();
    }

    *dec = k - 1;
```

接下来算出除最后一位的所有数字，如果数字不足以填满需要的位数，会在后面的代码中补0.

```cpp
// Step 4:
// Calculate digits.
//
// Output digits until reaching the last but one precision or the numerator becomes zero.
int digitsNum = 0;
int currentDigit = 0;
while (true)
{
    currentDigit = BigNum::HeuristicDivide(&r, s);
    if (r.IsZero() || digitsNum + 1 == count)
    {
        break;
    }

    digits[digitsNum] = L'0' + currentDigit;
    ++digitsNum;

    r.Multiply10();
}
```

最后一位数字涉及到应该向上近似还是向下近似的问题，策略是"向最接近的数字近似"。

```cpp
// Step 5:
// Set the last digit.
//
// We round to the closest digit by comparing value with 0.5:
//  compare( value, 0.5 )
//  = compare( r / s, 0.5 )
//  = compare( r, 0.5 * s)
//  = compare(2 * r, s)
//  = compare(r << 1, s)
r.ShiftLeft(1);
int compareResult = BigNum::Compare(r, s);
bool isRoundDown = compareResult < 0;

// We are in the middle, round towards the even digit (i.e. IEEE rouding rules)
if (compareResult == 0)
{
    isRoundDown = (currentDigit & 1) == 0;
}

if (isRoundDown)
{
    digits[digitsNum] = L'0' + currentDigit;
    ++digitsNum;
}
else
{
    wchar_t* pCurDigit = digits + digitsNum;

    // Rounding up for 9 is special.
    if (currentDigit == 9)
    {
        // find the first non-nine prior digit
        while (true)
        {
            // If we are at the first digit
            if (pCurDigit == digits)
            {
                // Output 1 at the next highest exponent
                *pCurDigit = L'1';
                ++digitsNum;
                *dec += 1;
                break;
            }

            --pCurDigit;
            --digitsNum;
            if (*pCurDigit != L'9')
            {
                // increment the digit
                *pCurDigit += 1;
                ++digitsNum;
                break;
            }
        }
    }
    else
    {
        // It's simple if the digit is not 9.
        *pCurDigit = L'0' + currentDigit + 1;
        ++digitsNum;
    }
}
```

如果数字的个数不足以填满要求的精度，则用0来补充。
```cpp
 while (digitsNum < count)
{
    digits[digitsNum] = L'0';
    ++digitsNum;
}

digits[count] = 0;
```

最后保存符号位:

```cpp
*sign = ((FPDOUBLE*)&value)->sign;
```

# 优化算法

在上述算法实现过程中，有些地方没有完全遵照论文的设计，而是做了一些优化。特别感谢[这篇文章](http://www.ryanjuckett.com/programming/printing-floating-point-numbers/), 里面涉及的启发式除法提升了寻找小数点位置的效率，而`logTable`大大提升了log的计算效率。

## 利用启发式除法提升效率



## 利用logTable提升效率



# 总结