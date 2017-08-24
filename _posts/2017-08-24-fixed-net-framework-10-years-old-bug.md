---
layout: post
title: Fixed .NET Framework 10 years old bug
date: 2017-08-24 12:50
author: mazong1123
comments: true
categories: [Uncategorized]
published: true
---

Since [my PR](https://github.com/dotnet/coreclr/pull/13164) has been merged into the master branch of .NET core, this [10 years old bug](https://stackoverflow.com/questions/2508945/can-anyone-explain-this-strange-behavior-with-signed-floats-in-c/2509152) from .NET framework to .NET Core finally gets fixed. I'd expect it be included in 2.1.0 release.

# What's the bug?

The bug can be splitted to 2 kind of problems (although they're fundmentally the same bug).

The first example is from [stackoverflow](https://stackoverflow.com/questions/2508945/can-anyone-explain-this-strange-behavior-with-signed-floats-in-c/2509152)

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

Apparently, we expect `0.0` and `-0.0` should be logically the same value (although the binary representations are different. See [IEEE double precision floating point format](https://en.wikipedia.org/wiki/Double_precision)). 

# Why?

When comparing two `ValueType` objects, .NET framework tries to avoid use reflection if possible: It tries to compare the bit representations of the two `ValueType` objects if:

- The type does not contain any pointer. E.G. cannot contain class type field.
- The type is tightly packed. That means the fields should be aligned. For example, in the previouse example, `D1` is not tightly packed because `double` and `int` have different size which is not aligned. By contrast, `D2` is tightly packed.

Related [source code](https://github.com/dotnet/coreclr/blob/76b7d89f6e32ff51ae4809827163864c40a6a76f/src/vm/comutilnative.cpp#L2628):

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

As you can see, `D1` does not meet the requirements of using bit comparison. So it goes to reflection which produces correct result. `D2` goes to the fast path (bit comparison) which produces the wrong result.

So why fast path will produce wrong result? Recall `0.0` and `-0.0` have different bit representations, bit comparison will consider `0.0` and `-0.0` are different values. That's the reason why fast path produces wrong result in this case.

Why the slow path (using reflection) will work as expected? Let's think about how the comparison works. It simply needs to compare all fields of the `ValueType` in a **Normal** way, e.g. call `==` or `Equals` of `Double`. Obviously, the framework can handle `0.0` and `-0.0` correctly in this way becasue if you compare raw `double` values with `0.0` and `-0.0` it will produce correct result.

Generally speaking, the problem is we did a wrong judgement when choosing fast/slow path.

# Another bug introduced by Overridden Equals

Similarly, if we have a user defined struct which overrode `Equals`, the framework may choose fast path which will produce wrong result. See following example:

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

The expected result is `false` because we overrode the `Equals` in `NeverEquals` struct. However, the actual result is `true` because it goes to a fast path which ends up a bit comparison - it won't call the overridden `Equals` method!

# Fix it

The problem is how to correclty choose the path. Apparently, relies on `ContainsPointers` and `IsNotTightlyPacked` is not enough to make a right decision. We should involve two additional conditions:

- No floating point number fields in the type hierachy tree.

- No overridden `Equals` in the type hierachy tree.

Obviousely, we need a depth-first-search through the whole type hierachy tree in the worst case. That may slow down the comparing process but make a right decision always. To reduce the performance degrade, we can cache the DFS result. The [key implementation](https://github.com/dotnet/coreclr/blob/master/src/vm/comutilnative.cpp#L2661) is as following:

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

- As [discussed](https://github.com/dotnet/coreclr/pull/13164#issuecomment-322919479) with [Jan Kotas](https://github.com/jkotas), I'll add tests for this bug fix in CoreFX project.

- Although the bug has been fixed, the performance is degraded when user compares `ValueTypes` at first time. [Jan Kotas](https://github.com/jkotas) [pointed a new way to improve the performance] which looks promising:(https://github.com/dotnet/coreclr/pull/13164#discussion_r133353129):
> The most performant way to write these FCalls is to have the code that executes in steady state in the main method, and have all one time initialization in the helper method.