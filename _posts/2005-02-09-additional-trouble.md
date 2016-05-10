---
layout:  post
title:   "Additional Trouble"
date:    2005-02-09 11:01:13
author:  "jibsen"
tags:    [C]
---
2 plus 2 is 4, but does that generalize?

What is your immediate reaction to this little program?

{% highlight c %}
#include <stdio.h>

int main(void)
{
        if (20000 + 20000 == 40000) {
                printf("HardToC");
        }

        return 0;
}
{% endhighlight %}

If it was something along the lines of 'depends' then you're either a raider
of the standard, or you've just been around C/C++ for too long like me.

The type of an unsuffixed decimal integer constant is the first type from a
list in which its value can be represented:

|C89|-|int, long int, unsigned long int|
|C99|-|int, long int, long long int|
|C++|-|int, long int|

Now, the problem with the little program above is that if the int type is
16-bit, then 20000 + 20000 results in an overflow because the maximum value
of a 16-bit int is 32767. We are guaranteed that computations involving
unsigned operands cannot overflow, but there is no such guarantee for signed
operands. So the addition may leave us in the land of undefined behaviour.

I compiled the above example with three DOS 16-bit compilers; Borland, Open
Watcom and Digital Mars. None of the programs gave any output when run.
Borland warned about the overflow, Open Watcom warned at -w2, Digital Mars
did not warn.

What happens is that in the x86 two's complement representation, 20000 + 20000
overflows and becomes -25536, which is not equal to 40000.

Writing portable, standard compliant C/C++ is not always easy .. and it can
be Hard to C the problems.
