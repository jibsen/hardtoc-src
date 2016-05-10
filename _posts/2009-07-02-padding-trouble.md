---
layout:  post
title:   "Padding Trouble"
date:    2009-07-02 13:49:38
author:  "jibsen"
tags:    [Asm, Bug, C, Compiler]
excerpt: "When building the aPLib compression library, I use Visual C++ to
  generate assembly listings, which I then perform some changes on with a Perl
  script, before assembling the object files. While working on the recently
  released 64-bit version, I ran into a problem -- the debug build of the
  library worked fine, but the release build did not."
---
When Intel [expanded the 8086 architecture to 32-bit][Intel386] in 1985, they
extended the 16-bit registers present to 32-bit registers. `ax` became `eax`,
but it was still possible to use the low 16 bits of `eax` as `ax` just like
before. Their choice was that performing operations on the low 16 bits did not
change the high 16 bits of the register.

AMD [expanded the 32-bit architecture to 64-bit][X86-64] in 2003. This was
again a superset of the original, making it backwards compatible. They extended
the 32-bit registers to 64-bit, and `eax` became `rax`. Again it was possible
to to perform operations on the low 32 bits, but doing so clears the high 32
bits of the register.

> Operations that output to a 32-bit subregister are automatically
> zero-extended to the entire 64-bit register. Operations that output to 8-bit
> or 16-bit subregisters are not zero-extended (this is compatible x86
> behavior). [[source]][zero-extend]

Now both choices work as far as backwards compatibility goes, and as long as we
as programmers are aware of what happens, neither is a problem.

When building the [aPLib compression library][aPLib], I use Visual C++ to
generate assembly listings, which I then perform some changes on with a Perl
script, before assembling the object files. While working on the recently
released 64-bit version, I ran into a problem -- the debug build of the library
worked fine, but the release build did not.

Bugs like this are often caused by some improper memory usage, so I spent a day
trying to track down the problem without much luck. Somehow the contents of a
register was corrupted.

Looking through the code in [HIEW][] I finally found the cause; a seemingly
random instruction that wrote to the 32-bit part of a register, thereby
clearing the high 32 bits. Then it dawned on me.

Visual C++ emits padding macros into assembly listings to align code and
improve performance. These macros, `npad`, are defined in a file called
`listing.inc` which resides in the Visual C++ include folder. But there is no
64-bit version of this file!

Let's have a look then:

{% highlight asm %}
;; LISTING.INC

;; non destructive nops
npad macro size
if size eq 1
  nop
else
 if size eq 2
   mov edi, edi
 else
   ...
{% endhighlight %}

And there we have it. An instruction like `mov edi, edi` is safe to use as
padding in 32-bit code, because moving the register to itself has no effect.
But if you insert it in 64-bit code, it all of a sudden has an effect -- the
high 32 bits of `rdi` are cleared.

I have reported the problem to Microsoft and they say it will be addressed in a
future release.

[Intel386]: http://en.wikipedia.org/wiki/Intel_386
[X86-64]: http://en.wikipedia.org/wiki/X86-64
[zero-extend]: http://msdn.microsoft.com/en-us/library/cc267760.aspx
[aPLib]: http://www.ibsensoftware.com/products_aPLib.html
[HIEW]: http://www.hiew.ru/
