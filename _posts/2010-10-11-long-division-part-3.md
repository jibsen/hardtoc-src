---
layout:  post
title:   "Long Division, Part 3"
date:    2010-10-11 12:49:41
author:  "jibsen"
tags:    [C, CLR, Compiler, Information]
excerpt: "The CLR runtime contains optimizations for certain values (usually
  when both operands fit in 32 bits), but for other cases optimizing the C
  runtime functions appears to provide a direct improvement for the CLR as
  well."
---
I ended [part two][part2] of this series with an open question:

> And of course I can't help but wonder if the [CLR][] is compiled with Visual
> C++, so doing arithmetic on 64-bit numbers in C# and other .NET languages
> ends up at the same runtime functions?

I don't have a lot of experience in debugging the CLR myself, so I asked
[Brian Rasmussen][brian] if he might be interested in taking a look at it. He
was kind enough to take the time to point me in the right direction.

A little digging showed that the CLR does in fact call some of these functions
from the C runtime, but with a twist.

The C# function we looked at was this:

{% highlight csharp %}
[MethodImpl(MethodImplOptions.NoInlining)]
private static Int64 div64(Int64 x, Int64 y) {
    return x / y;
}
{% endhighlight %}

The [JIT compiler][JIT] produced the following 32-bit code:

{% highlight asm %}
    push    ebp
    mov     ebp,esp
    push    dword ptr [ebp+14h]
    push    dword ptr [ebp+10h]
    push    dword ptr [ebp+0Ch]
    push    dword ptr [ebp+8]
    call    clr!JIT_LDiv
    pop     ebp
    ret     10h
{% endhighlight %}

We note the call to `JIT_LDiv`, which is a helper function in clr.dll (the CLR
runtime, formerly mscorwks.dll). Let's have a look at it:

{% highlight cpp %}
INT64 JIT_LDiv(INT64 dividend, INT64 divisor)
{
    ...
    if (Is32BitSigned(divisor))
    {
        if ((INT32)divisor == 0)
        {
            ehKind = kDivideByZeroException;
            goto ThrowExcep;
        }

        if ((INT32)divisor == -1)
        {
            if ((UINT64) dividend == UI64(0x8000000000000000))
            {
                ehKind = kOverflowException;
                goto ThrowExcep;
            }
            return -dividend;
        }

        // Check for -ive or +ive numbers in -2**31 to 2**31
        if (Is32BitSigned(dividend))
            return((INT32)dividend / (INT32)divisor);
    }

    // For all other combinations fallback to int64 div.
    return(dividend / divisor);
    ...
}
{% endhighlight %}

This excerpt is from the [Shared Source CLI][SSCLI], but the same code is
present in clr.dll from .NET Framework 4.0.

The division in the return statement at line 28 generates a call to `_alldiv`
from the C runtime library.

Now, the twist is that this helper function, besides checking for division by
zero and overflow, also optimizes division of two values that both fit in 32
bits (the division with casts at line 24).

This is one of the optimizations I used in WCRT as well, so I was interested in
seeing how the CLR code performed. I removed the exception handling along with
associated checks and ended up with this function, which I tested using the
timing application from part 2:

{% highlight cpp %}
__int64 JIT_LDiv(__int64 dividend, __int64 divisor)
{
    if (Is32BitSigned(divisor))
    {
        if ((int)divisor == -1)
        {
            return -dividend;
        }

        // Check for -ive or +ive numbers in -2**31 to 2**31
        if (Is32BitSigned(dividend))
            return((int)dividend / (int)divisor);
    }

    // For all other combinations fallback to __int64 div.
    return(dividend / divisor);
}
{% endhighlight %}

Here is a summary of the results from my Athlon 64:

    Function          WCRT        VC       Diff
    -------------------------------------------
    (CLR runtime function)
    HH JIT_LDiv       2125      4222     +49.7%
    HL JIT_LDiv       4104      5128     +20.0%
    LL JIT_LDiv       1312      1312      +0.0%

    (C runtime function)
    HH _alldiv        1709      3907     +56.2%
    HL _alldiv        3260      4309     +24.3%
    LL _alldiv        1204      2161     +44.2%

    HH = 64-bit op 64-bit
    HL = 64-bit op 32-bit
    LL = 32-bit op 32-bit

In the WCRT column, `JIT_LDiv` is compiled with `_alldiv` from WCRT, and in the
VC column with `_alldiv` from the VC CRT. So the first three rows compare the
speed of the CLR helper function when using those as fallback.

We can see that there is a noticeable improvement on HH and HL, even though the
check for two values that both fit in 32 bits is done twice (once in
`JIT_LDiv`, and once in the WCRT version of `_alldiv`).

The next three rows compare the C runtime function `_alldiv` from WCRT and the
VC CRT. These results are the same as in part 2.

The special handling of two values that both fit in 32 bits is a big
improvement. On LL, `JIT_LDiv` is almost as fast as the WCRT version of
`_alldiv`. For HL and HH, `JIT_LDiv` is slightly slower than the regular VC
`_alldiv` due to the extra checks.

So, in conclusion, the CLR runtime is compiled with Visual C++, and when code
runs under 32-bit, the CLR runtime will fall back to the 64-bit arithmetic
functions from the C runtime library.

The CLR runtime contains optimizations for certain values (usually when both
operands fit in 32 bits), but for other cases optimizing the C runtime
functions appears to provide a direct improvement for the CLR as well.

[part2]: {% post_url 2010-09-20-long-division-part-2 %}
[CLR]: http://en.wikipedia.org/wiki/Common_Language_Runtime
[brian]: http://kodehoved.dk/
[JIT]: http://en.wikipedia.org/wiki/Just-in-time_compilation
[SSCLI]: http://en.wikipedia.org/wiki/Shared_Source_Common_Language_Infrastructure
