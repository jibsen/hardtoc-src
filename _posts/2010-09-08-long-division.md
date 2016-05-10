---
layout:  post
title:   "Long Division"
date:    2010-09-08 06:35:41
author:  "jibsen"
tags:    [C, Compiler, Programming]
excerpt: "The thing is, 32-bit x86 only has instructions for mixed mode 32/64
  bit operations -- with one assembly instruction you can multiply two 32-bit
  integers to get a 64-bit result, or you can divide a 64-bit integer by a
  32-bit as long as the quotient fits in 32 bits. But to divide two 64-bit
  integers, you have to emulate the operation in software."
---
Integer types with at least 64 bits have been a part of the C standard for a
while now (they were added in C99, and were a standard extension in many 32-bit
compilers before that). But have you ever wondered what exactly happens when
you use them?

Consider the following function (substitute `long long` with `__int64` if you
are using an older version of Visual C++):

{% highlight cpp %}
long long div64(long long x, long long y)
{
        return x / y;
}
{% endhighlight %}

let's first have a look at what the VC 64-bit compiler gives us:

{% highlight asm %}
div64:
    mov   r8, rdx
    mov   rax, rcx
    cdq
    idiv  r8
    ret
{% endhighlight %}

Pretty much what you would expect, a little setup and an `idiv` instruction to
perform the division. Now let's try the VC 32-bit compiler:

{% highlight asm %}
div64:
_x$ = 8
_y$ = 16
    mov   eax, _y$[esp]
    mov   ecx, _y$[esp-4]
    mov   edx, _x$[esp]
    push  eax
    mov   eax, _x$[esp]
    push  ecx
    push  edx
    push  eax
    call  __alldiv
    ret
{% endhighlight %}

A little setup and .. a call?

The thing is, 32-bit x86 only has instructions for mixed mode 32/64 bit
operations -- with one assembly instruction you can multiply two 32-bit
integers to get a 64-bit result, or you can divide a 64-bit integer by a 32-bit
as long as the quotient fits in 32 bits. But to divide two 64-bit integers, you
have to emulate the operation in software.

That is where the `_alldiv` function comes in. It is a support function in the
C runtime library that performs a 64-bit divide. In fact there is a whole
family of such functions to do multiply, divide and shifts on 64-bit numbers
(addition and subtraction are inlined since they only require two assembly
instructions).

Other compilers do likewise, for instance GCC calls `__divdi3` to perform
64-bit division.

It is important to be aware of this, because using `long long` instead of `int`
means that arithmetic operations you probably expect to be single instructions,
all of a sudden become library calls who's execution time can depend on the
arguments.

As an example, I timed a [simple implementation][sample] of the [extended
Euclidean algorithm][eea] with 32- and 64-bit integers. Compiling with the VC
32-bit compiler, the 64-bit version was roughly 3 times slower (depending on
the CPU).

[sample]: {{ "/assets/gcdtest.zip" | prepend: site.baseurl | prepend: site.url }}
[eea]: http://en.wikipedia.org/wiki/Extended_Euclidean_algorithm
