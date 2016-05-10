---
layout:  post
title:   "Loophole in Visual C++, Part 1"
date:    2005-02-06 11:58:30
author:  "jibsen"
tags:    [C, Bug, Compiler]
excerpt: ""
---
Lets start this post by recalling what the gosp^H^H^H^Hstandard has to say
about unsigned arithmetic:

> A computation involving unsigned operands can never overflow, because a
> result that cannot be represented by the resulting unsigned integer type is
> reduced modulo the number that is one greater than the largest value that
> can be represented by the resulting unsigned integer type.

This is from the C89 draft (3.1.2.5p5), statements to the same effect are
present in the C99 standard (6.2.5p9) and the C++ standards.

Now consider the following program:

{% highlight c %}
#include <stdio.h>

int main(void)
{
        unsigned int count = 0;

        do {
                printf("%u\n", count);
                count += 1;
        } while (count != 0);

        return 0;
}
{% endhighlight %}

Since `count` starts at zero and is incremented each time through the loop,
the standard tells us it will wrap to zero when it reaches a result that
cannot be represented by an unsigned int, making the program terminate.
Compiling the program with various compilers gives the expected stream of
increasing numbers.

However, if you compile it with cl.exe from Visual C++ using the /O2 switch
(maximize speed) you get a somewhat surprising result; a single zero and the
program exits. This goes for VC6, VC7 and VC71.

If you initialize `count` to one instead, the program works fine. So it looks
like the optimizer fails to recognize the addition as changing the value of
`count`, and thus optimizes away the loop.

I have not tested the various VC8 betas, so if you have any of them installed,
feel free to try it out and post your results (just remember to compile from
the command-line using cl.exe and /O2).
