---
layout: post
title:  "What Else Could Go Wrong?"
date:   2005-02-27 13:04:20
tags:   [cpp]
---
[JrDebugLogger][] is a very nice debug logging library. Much of it's
functionality is implemented through macros to allow it to be selectively left
out when compiling. Along the way the author has had some interesting problems
to solve, and this post is about one of them.

Assume we use the following macro:

{% highlight cpp %}
#define DEBUGOUT if (debug_on) debug_stream
{% endhighlight %}

to allow us to perform debug logging with a stream-like interface. We can then
do:

{% highlight cpp %}
DEBUGOUT << "hello";
{% endhighlight %}

which expands to:

{% highlight cpp %}
if (debug_on) debug_stream << "hello";
{% endhighlight %}

Now if the compiler knows that `debug_on` is false, it can leave out all code
related to the debug logging, since it knows it will never be called. If it
does not know the value at compile time, the resulting code will contain a
very fast check around the call, allowing debug logging to be turned on and
off dynamically with little performance overhead.

There is, however, an insidious bug lurking in the corner, waiting to jump at
the user. Can you spot the problem? Think about it for a minute or two before
reading on.

Consider this use:

{% highlight cpp %}
if (i > limit)
    DEBUGOUT << "i too big";
else
    do_computation(i);
{% endhighlight %}

it expands to:

{% highlight cpp %}
if (i > limit)
    if (debug_on) debug_stream << "i too big";
else
    do_computation(i);
{% endhighlight %}

This is valid C++, and compiled without warnings on the three compilers I
tried. But who does that else belong to?

Let's see what the standard says:

> An else is associated with the lexically nearest preceding if that is
> allowed by the syntax.

This is from the C99 standard (6.8.4.1p3), which was the most clear, however
statements to the same effect are present in the C++ standards.

So the above is equivalent to:

{% highlight cpp %}
if (i > limit)
{
    if (debug_on)
        debug_stream << "i too big";
    else
        do_computation(i);
}
{% endhighlight %}

which was of course not the intention.

So how can we solve this without giving up the nice properties of the if? The
simple solution is to give the if in the macro its own else:

{% highlight cpp %}
#define DEBUGOUT if (!debug_on) ; else debug_stream
{% endhighlight %}

We now get the expansion:

{% highlight cpp %}
if (i > limit)
    if (!debug_on) ; else debug_stream << "i too big";
else
    do_computation(i);
{% endhighlight %}

and the compiler will correctly associate the users else with his if. So it is
equivalent to:

{% highlight cpp %}
if (i > limit)
{
    if (!debug_on)
        ;
    else
        debug_stream << "i too big";
} else {
    do_computation(i);
}
{% endhighlight %}

Thanks to Jesse for the nice topic.

[JrDebugLogger]: http://jrdebug.sourceforge.net/
