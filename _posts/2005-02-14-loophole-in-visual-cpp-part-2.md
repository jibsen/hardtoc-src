---
layout:  post
title:   "Loophole in Visual C++, Part 2"
date:    2005-02-14 08:40:46
author:  "jibsen"
tags:    [C, Bug, Compiler]
excerpt: ""
---
Here is a slightly more elaborate example:

{% highlight c %}
#include <stdio.h>
#include <limits.h>

unsigned int ratio(unsigned int x, unsigned int y)
{
        if (x <= UINT_MAX / 100) {
                x *= 100; else y /= 100;
        }

        if (y == 0) {
                y = 1;
        }

        return x / y;
}

int main(void)
{
        unsigned int count;

        for (count = 0x3FFFFFFF; count != 0; ++count) {
                /* Do something */

                /* Show progress */
                printf("\r%u%% done", ratio(count, UINT_MAX));
        }

        return 0;
}
{% endhighlight %}

This program goes through the entire range of the unsigned int type,
performing some action for each. It shows the progress by calling a function
to compute the ratio of <code>count</code> to the maximum possible value.
Again, `count` is incremented in each step, and hence will reach the value
zero at some point.

The program works as expected on the compilers I tried, except for cl.exe
from VC7 and VC71 with the /O2 switch, which stop at 25%. In case you wondered
about the starting point of 0x3fffffff, that's the reason -- no need to watch
your machine chew it's way through all integers up to 25%.

Looking at the code generated for the loop:

{% highlight asm %}
$L873:
        ...
        inc    esi
        add    edi, 100                ; 00000064H
        jne    short $L873
{% endhighlight %}

We see that it fails because the two instructions before the conditional jump
have been reversed. Again it looks like the optimizer fails to recognize the
importance of the increment to the loop.
