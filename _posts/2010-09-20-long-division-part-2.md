---
layout:  post
title:   "Long Division, Part 2"
date:    2010-09-20 07:24:06
author:  "jibsen"
tags:    [C, Compiler, Information]
excerpt: "To my surprise, I found the GCD test I wrote for Long Division ran
  faster when compiled with WCRT."
---
In [part one][part1] I talked about the support functions in the C standard
libraries of various x86 32-bit compilers that perform arithmetic operations
when you use 64-bit integers in your code.

While updating [WCRT][] to work with the latest Visual C++ compilers, I was
writing my own implementations of these functions, and naturally I tested them
against the versions supplied in the VC CRT to verify they worked.

To my surprise, I found the GCD test I wrote for [Long Division][part1] ran
faster when compiled with WCRT.

    C:\gcdtest>gcdtest_gcc-4.5.1.exe
    32-bit GCD: 314
    64-bit GCD: 851

    C:\gcdtest>gcdtest_vc2010.exe
    32-bit GCD: 300
    64-bit GCD: 835

    C:\gcdtest>gcdtest_vc2010-wcrt.exe
    32-bit GCD: 300
    64-bit GCD: 656

This naturally piqued my curiosity.

It turns out that the 64-bit arithmetic functions in VC have remained unchanged
since Visual C++ 4.2 from 1996. The compiler and optimizer have evolved over
the years, adapting to new processors, and improving the speed of your code.
But if you happen to divide two 64-bit integers you are relying on code that is
14 years old.

This was of course more than enough incentive to spend a little time trying to
improve my implementations to see if I could beat the VC CRT.

Here are parts of the result from running [this test][sample] on an Athlon 64:

    Function          WCRT        VC       Diff
    -------------------------------------------
    HH _alldiv        1709      3907     +56.2%
    HL _alldiv        3260      4309     +24.3%
    LL _alldiv        1204      2161     +44.2%
    ...
    HH _allmul         293       337     +13.0%
    HL _allmul         631       739     +14.6%
    LL _allmul         279       291      +4.1%
    ...
    <32 _allshl        657       745     +11.8%
    <64 _allshl        568       566      -0.3%
    <96 _allshl        654       537     -21.7%
    ...

As you can see, the difference ranges from a few percent to over 50% in the
most favorable test.

The only operations that are slower in general are shifts of more than 64 bits.
The shift functions do not contain a lot of code to work with in the first
place, but I did choose to sacrifice some speed in shifts above 64 bits to get
a small improvement in shifts below 32 bits, because they seem more likely to
occur in actual code.

Now of course these improvements do come with some reservations. For one, the
CRT functions have to perform well across all processors from 386 up to the
cutting edge, while I only wrote mine with Intel Core and AMD Athlon in mind.
Also, the test functions I use do not necessarily reflect the average use, so
while some of the optimizations I did were favorable to the tests used, they
may not be for other cases.

That being said, I think this does show that it is possible to get a
non-trivial improvement by optimizing these functions. And if critical parts of
your code deal with 64-bit numbers it is certainly worth investigating
alternatives.

And of course I can't help but wonder if the [CLR][] is compiled with Visual
C++, so doing arithmetic on 64-bit numbers in C# and other .NET languages ends
up at the same runtime functions?

[part1]: {% post_url 2010-09-08-long-division %}
[WCRT]: http://www.ibsensoftware.com/download.html
[sample]: {{ "/assets/comptime.zip" | prepend: site.baseurl | prepend: site.url }}
[CLR]: http://en.wikipedia.org/wiki/Common_Language_Runtime
