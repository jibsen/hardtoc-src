---
layout:  post
title:   "Quoting Command-line Arguments"
date:    2010-09-24 14:22:26
author:  "jibsen"
tags:    [C, Information, Link]
excerpt: "Looking at the two code excerpts, one could wonder if the old
  handling was in fact just a bug introduced by the else part not being in
  braces, and left in for over ten years. Or maybe it was intentional, and the
  change in Visual C++ 8.0 was to support the way two quotes are used as escape
  in other languages."
---
Raymond Chen recently [blogged][ont] about the way CommandLineToArgvW treats
quotes and backslashes. Parsing the command-line into `argv[]` is something I
have had to fight with as well, so besides pointing to Raymond's excellent
post, I wanted to add a few comments of my own here.

We are examining how command-line arguments with spaces and quotes are handled.
Part of the problem comes from the fact that [DOS/Windows uses backslash as
separator in paths][paths]. On systems like Unix, where forward slash is used
instead, using backslash to escape special characters is less of a problem. But
if you ever put a [Windows path][winpath] in a C string literal, you may have
run into [LTS][] -- the situation where a string becomes unreadable due to
escape characters.

Microsoft fixed this in C# with [verbatim string literals][vsl]. C# also
implements a simpler method of escaping a quote inside a quoted string --
doubling it -- which is used in languages like Pascal and BASIC, and is what
Raymond's second hypothetical set of rules suggest.

The compromise we get for parsing command-line arguments in the C runtime
library (and [`CommandLineToArgvW()`][cltoargv]) is [documented on MSDN][MSDN].
What the MSDN documentation does not tell you is that there is a second
mechanism for inserting a literal quote in a quoted string -- or at least there
might be, depending on which version of the C runtime library.

In Visual C++ 2.0 from 1994, the command-line parser handles two quotes inside
a quoted string as an escape. What it does is that it copies the second quote
and switches to unquoted mode. Thus you can insert a literal quote inside a
quoted string by putting three quotes (two to insert the literal quote, and one
more to put you back into quoted mode).

{% highlight c %}
if (*p == DQUOTECHAR) {
    ...
    if (inquote) {
        if (p[1] == DQUOTECHAR)
            p++;    /* Double quote inside quoted string */
        else        /* skip first quote char and copy second */
            copychar = 0;
    } else
        copychar = 0;       /* don't copy quote */

    inquote = !inquote;
    ...
}
{% endhighlight %}

This code is present until Visual C++ 8.0 from 2005, where it is changed so the
two quote escape sequence no longer switches mode. That means that to insert a
literal quote inside a quoted string, you now only need two quotes.

{% highlight c %}
if (*p == DQUOTECHAR) {
    ...
    if (inquote && p[1] == DQUOTECHAR) {
        p++;    /* Double quote inside quoted string */
    } else {    /* skip first quote char and copy second */
        copychar = 0;       /* don't copy quote */
        inquote = !inquote;
    }
	...
}
{% endhighlight %}

This change is probably what is causing some confusion in the comments on
Raymond's post -- the twelve literal quotes in `foo""""""""""""bar` will result
in five quotes with the current parser (opening quote, five pairs, closing
quote), but only four with the older method (opening quote, three triples, and
a double quote).

The implementation of `CommandLineToArgvW()` in current versions of shell32.dll
use the older method. The same goes for the command-line parser used in GCC on
Windows (I checked 4.5.1).

Looking at the two code excerpts, one could wonder if the old handling was in
fact just a bug introduced by the else part not being in braces, and left in
for over ten years. Or maybe it was intentional, and the change in Visual C++
8.0 was to support the way two quotes are used as escape in other languages.

Either way, I find it interesting that this functionality was added (it wasn't
supported in Visual C++ 1.52), and then changed in the C runtime library but
not in the API function, and not mentioned in the documentation of either.

[ont]: http://blogs.msdn.com/b/oldnewthing/archive/2010/09/17/10063629.aspx
[paths]: http://en.wikipedia.org/wiki/Path_%28computing%29
[winpath]: http://msdn.microsoft.com/en-us/library/aa365247.aspx
[LTS]: http://en.wikipedia.org/wiki/Leaning_toothpick_syndrome
[vsl]: http://msdn.microsoft.com/en-us/library/362314fe.aspx
[cltoargv]: http://msdn.microsoft.com/en-us/library/bb776391.aspx
[MSDN]: http://msdn.microsoft.com/en-us/library/a1y7w461.aspx
