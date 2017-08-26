---
layout: post
title: .NET中值类型比较的问题以及修复
date: 2017-08-24 12:50
author: mazong1123
comments: true
categories: [Uncategorized]
published: true
---

.NET中值类型(`ValueType`)比较时隐藏着一个底层的bug, 这个bug已经有接近十年未被修复。从 .NET Framework 到 .NET Core一直存在。StackOverflow上早在2010年就有人提出了这个问题:  [10 years old bug](https://stackoverflow.com/questions/2508945/can-anyone-explain-this-strange-behavior-with-signed-floats-in-c/2509152).

随着[我的PR](https://github.com/dotnet/coreclr/pull/13164)被合并到 .NET Core主干，这个bug终于被修复了。预计在 .NET Core 2.1.0发布时会包含这个bug fix.

接下来分享一下这个bug到底是什么，以及我是如何修复的。值得注意的是，因为涉及到native代码，如果你没有C++基础的话可能看代码会比较吃力，但是算法和原理还是很好理解的。

# 存在了10年的神奇BUG

从表象上看，这个bug可以分为2个问题（虽然本质上是同一个问题）:

- 当`struct`中有浮点数类型(double/float)且字节对齐时，.NET Framework在某些特殊情况下无法正确比较两个该类型的实例。

- 当用户重写(override)了`ValueType`的`Equals`方法时，.NET Framework无法正确比较两个该类型的实例。

关于第一个问题，可以看
[Stackoverflow的例子](https://stackoverflow.com/questions/2508945/can-anyone-explain-this-strange-behavior-with-signed-floats-in-c/2509152)

```csharp
class Program
{
    // first version of structure
    public struct D1
    {
        public double d;
        public int f;
    }

    // during some changes in code then we got D2 from D1
    // Field f type became double while it was int before
    public struct D2 
    {
        public double d;
        public double f;
    }

    static void Main(string[] args)
    {
        // Scenario with the first version
        D1 a = new D1();
        D1 b = new D1();
        a.f = b.f = 1;
        a.d = 0.0;
        b.d = -0.0;
        bool r1 = a.Equals(b); // gives true, all is ok

        // The same scenario with the new one
        D2 c = new D2();
        D2 d = new D2();
        c.f = d.f = 1;
        c.d = 0.0;
        d.d = -0.0;
        bool r2 = c.Equals(d); // false! this is not the expected result        
    }
}

```

很明显我们期望`0.0`和`-0.0`在逻辑上是相等的（虽然它们的二进制内容是不同的，可以看[IEEE double precision floating point format](https://en.wikipedia.org/wiki/Double_precision)). 但实际上在上述例子的`D2`实例比较时， .NET Framework 认为`0.0`和`-0.0`不等，所以得出了错误的结果。

# 为什么?

当比较两个值类型实例时，.NET Framework会尽量避免使用反射机制（为了提升性能）。当满足以下条件时，它会直接比较两个实例在内存中的二进制内容:

- 值类型不包含任何指针。也就是说`struct`中不能有`class`类型的字段。

- 值类型`tightly packed`. 可以理解为struct字节对齐。比如在上面的例子中，`D1`结构体不是`tightly packed`, 因为它的字段含有2种类型，`double`和`int`两种数据类型的大小是不同的; 但`D2`的2个字段都是`double`, 所以`D2`是`tightly packed`.

以上逻辑相关的[代码](https://github.com/dotnet/coreclr/blob/76b7d89f6e32ff51ae4809827163864c40a6a76f/src/vm/comutilnative.cpp#L2628)如下:

```cpp
FCIMPL1(FC_BOOL_RET, ValueTypeHelper::CanCompareBits, Object* obj)
{
    FCALL_CONTRACT;

    _ASSERTE(obj != NULL);
    MethodTable* mt = obj->GetMethodTable();
    FC_RETURN_BOOL(!mt->ContainsPointers() && !mt->IsNotTightlyPacked());
}
FCIMPLEND
```

可以看到，`D1`不满足使用二进制比较(bit comparison)的条件。所以 .NET Framework会使用反射一个一个地比较它的字段，得到正确的结果。`D2`满足了二进制比较的条件，速度是提升了不少，但却得到了错误的结果。

看来问题出在使用二进制比较上。为什么它会出错？其实从原理上看不难发现，`0.0`和`-0.0`虽然逻辑上应该相等，但实际上用于表示它们的二进制内容是不同的。当 .NET Framework 使用bit comparison快速比较2个`struct`实例时，会认为`0.0`和`-0.0`不同，所以得出了错误的结果。

而使用反射进行比较的话，实际上是调用了`Double`的`==`或`Equals`方法，显然 .NET Framework 会保证`==`和`Equals`能正确处理`0.0`和`-0.0` (你可以试着直接比较2个`double`值，一个为`0.0`, 一个为`-0.0`)，所以能得出正确的结果。

总体上来说，我们不应该在有浮点数存在的时候，使用bit comparison. 但是现在.NET的实现并没有这个判断，所以导致了我们选择了错误的比较方式。

# 重写`Equals`引起的bug

类似地，如果我们重写了`ValueType`的`Equals`方法， 同时 .NET Framework 使用bit comparison进行判断，则可能得出错误答案。因为`Equals`并不会被调用。可以看以下例子:

```csharp
using System;

struct NeverEquals
{
    byte _a;

    public override bool Equals(object obj)
    {
        return false;
    }

    public override int GetHashCode()
    {
        return 0;
    }
}

struct MyStruct
{
    NeverEquals _x;
    NeverEquals _y;
}

class Program
{
    static void Main(string[] args)
    {
        MyStruct i = new MyStruct();
        MyStruct j = new MyStruct();

        Console.WriteLine(i.Equals(j));
    }
}
```

我们期待结果为`false`, 因为我们重写了`NeverEquals`的`Equals`方法，使得它永远返回`fasle`. 但实际上运行结果却为`true`. 因为 .NET Framework 使用了bit comparison, 并不会调用`Equals`.

# Fix it

可以看出，整个问题都在于我们没有在bit comparison和普通的比较方式之间做出正确的选择。很明显，只依赖于现在的判断条件(`ContainsPointers`和`IsNotTightlyPacked`)是不够的。如果要使用bit comparison, 我们需要增加两个额外的条件:

- 在整个类型树中没有浮点数类型的字段。就是说如果类型继承了很多层，任意一层有浮点数类型的字段我们都不能使用bit comparison.

- 在整个类型树中没有重写(override)的`Equals`.就是说如果类型继承了很多层，任意一层重写了`Equals`我们都不能使用bit comparison.

显然这是个深度优先搜索的算法。但是为了提高效率，需要做适当的缓存，使得只有第一次使用该类型进行比较时才需要做搜索。另外，还可以利用一些边界条件进行剪枝，[核心算法实现](https://github.com/dotnet/coreclr/blob/master/src/vm/comutilnative.cpp#L2661)如下:

```cpp
static BOOL CanCompareBitsOrUseFastGetHashCode(MethodTable* mt)
{
    CONTRACTL
    {
        THROWS;
        GC_TRIGGERS;
        MODE_COOPERATIVE;
    } CONTRACTL_END;

    _ASSERTE(mt != NULL);

    if (mt->HasCheckedCanCompareBitsOrUseFastGetHashCode())
    {
        return mt->CanCompareBitsOrUseFastGetHashCode();
    }

    if (mt->ContainsPointers()
        || mt->IsNotTightlyPacked())
    {
        mt->SetHasCheckedCanCompareBitsOrUseFastGetHashCode();
        return FALSE;
    }

    MethodTable* valueTypeMT = MscorlibBinder::GetClass(CLASS__VALUE_TYPE);
    WORD slotEquals = MscorlibBinder::GetMethod(METHOD__VALUE_TYPE__EQUALS)->GetSlot();
    WORD slotGetHashCode = MscorlibBinder::GetMethod(METHOD__VALUE_TYPE__GET_HASH_CODE)->GetSlot();

    // Check the input type.
    if (HasOverriddenMethod(mt, valueTypeMT, slotEquals)
        || HasOverriddenMethod(mt, valueTypeMT, slotGetHashCode))
    {
        mt->SetHasCheckedCanCompareBitsOrUseFastGetHashCode();

        // If overridden Equals or GetHashCode found, stop searching further.
        return FALSE;
    }

    BOOL canCompareBitsOrUseFastGetHashCode = TRUE;

    // The type itself did not override Equals or GetHashCode, go for its fields.
    ApproxFieldDescIterator iter = ApproxFieldDescIterator(mt, ApproxFieldDescIterator::INSTANCE_FIELDS);
    for (FieldDesc* pField = iter.Next(); pField != NULL; pField = iter.Next())
    {
        if (pField->GetFieldType() == ELEMENT_TYPE_VALUETYPE)
        {
            // Check current field type.
            MethodTable* fieldMethodTable = pField->GetApproxFieldTypeHandleThrowing().GetMethodTable();
            if (!CanCompareBitsOrUseFastGetHashCode(fieldMethodTable))
            {
                canCompareBitsOrUseFastGetHashCode = FALSE;
                break;
            }
        }
        else if (pField->GetFieldType() == ELEMENT_TYPE_R8
                || pField->GetFieldType() == ELEMENT_TYPE_R4)
        {
            // We have double/single field, cannot compare in fast path.
            canCompareBitsOrUseFastGetHashCode = FALSE;
            break;
        }
    }

    // We've gone through all instance fields. It's time to cache the result.
    // Note SetCanCompareBitsOrUseFastGetHashCode(BOOL) ensures the checked flag
    // and canCompare flag being set atomically to avoid race.
    mt->SetCanCompareBitsOrUseFastGetHashCode(canCompareBitsOrUseFastGetHashCode);

    return canCompareBitsOrUseFastGetHashCode;
}
```

# What's Next?

- 根据和[Jan Kotas](https://github.com/jkotas)的[讨论](https://github.com/dotnet/coreclr/pull/13164#issuecomment-322919479), 我会在[CoreFX](https://github.com/dotnet/corefx)中添加对这次修改的测试用例.

- 虽然这个bug被修复了，但是性能还是受到了影响，特别是用户第一次使用值类型进行比较的时候。 [Jan Kotas](https://github.com/jkotas) [指出了另一个更高效的方法，从编译器的角度去解决这个问题](https://github.com/dotnet/coreclr/pull/13164#discussion_r133353129) :
> The most performant way to write these FCalls is to have the code that executes in steady state in the main method, and have all one time initialization in the helper method.

这是第一篇中文博客，希望对 .NET Core在国内的推广有一点点贡献:)