---
layout:  post
title:   "System Up Time"
date:    2009-07-07 12:52:02
author:  "jibsen"
tags:    [Information, Programming]
---
A while ago I set out to write a little tool that would show the time a system
had been running since the last reboot. It seemed like something that should be
fairly easy to do, but as it turns out, it isn't entirely straightforward.

The first thing that came to my mind was the [`GetTickCount()`][GetTickCount]
function. It returns the number of milliseconds since the system was started,
which fits nicely. There is one problem of course, the value returned is a
`DWORD`, and that limits the time it can handle to about 50 days.

Microsoft realized this as well, and added [`GetTickCount64()`][GetTickCount64],
which returns a 64-bit value instead. Unfortunately it only works on Vista+.

Looking more closely at the documentation for `GetTickCount()` we see:

> To obtain the time elapsed since the computer was started, retrieve the
> System Up Time counter in the performance data in the registry key
> `HKEY_PERFORMANCE_DATA`. The value returned is an 8-byte value. For more
> information, see [Performance Counters][perfcount].

The performance counters are a nice idea. Basically they provide a homogeneous
interface to a multitude of counters that give information about how well the
operating system or an application, service, or driver is performing.

The recommended way to access the data is through the [PDH interface][PDH]. You
access the counters by specifying a [counter path][counterpath]; a string that
describes the computer, object, counter, and instance you are interested in.

Looking through the list of [counters by objects][counters] for the `System`
object, we find the `System Up Time` counter, which is exactly what we need.

So I wrote a test application to open a query, add the counter
`"\System\System Up Time"`, collect the data, and display it. It failed.
Apparently my computer did not have a `System Up Time` counter.

Reading over the documentation for the PDH interface again did not help, but
after some searching I ended up at [this help article][helparticle]:

> Performance Data Helper (PDH) APIs use object and counter names that are in
> the localized language. Therefore, applications that use PDH APIs should
> always use the localized string for the object or counter name specification.

Since I was running a Danish version of Windows, that explained the problem!

Following the steps outlined there, I found that the `System` object has index
2, and the `System Up Time` counter has index 674. With these indices in hand,
you can then call the [`PdhLookupPerfNameByIndex()`][lpnbi] function to get the
localized names. Using the localized path
`"\System\Computerperiode uden afbrydelser"` gave the desired result.

The choice to make the paths use localized names makes it somewhat more
involved to use these functions. Also, this should have been described much
more clearly in the PDH documentation, since it is quite possible for a
developer using English Windows to read over the documentation like I did, and
use a hardcoded path for a counter. This will work nicely while testing, and
then fail if someone with a localized Windows uses it.

As an example, let's take a look at the [PsInfo][] tool. It is written by [Mark
Russinovich][MarkR], one of the people behind [Sysinternals.com][sysinternals],
a site that specializes in advanced system utilities and technical information.
He is also a coauthor of the [Windows Internals][wininternals] book, describing
the inner workings of Windows operation systems.

Running PsInfo on my system I get:

<pre>
 PsInfo v1.75 - Local and remote system information viewer
 Copyright (C) 2001-2007 Mark Russinovich
 Sysinternals - www.sysinternals.com

 System information for \\removed:
 Uptime:                    Error reading uptime
 ...
</pre>

Could it be? let's have a little peek inside `PsInfo.exe`:

{% highlight asm %}
perform_query:
    push    esi
    lea     ecx, [esp+834h+szCounterPath]
    push    offset aSUT ; "\\\\%s\\System\\System Up Time"
    push    ecx
    call    sprintf?
    mov     ecx, [esp+83Ch+hQuery]
    add     esp, 0Ch
    lea     edx, [esp+830h+phCounter]
    push    edx
    push    0
    lea     eax, [esp+838h+szCounterPath]
    push    eax
    push    ecx
    call    PdhAddCounterW
{% endhighlight %}

Indeed, a hardcoded path to the counter using the English names.

Microsoft must have realized it could be a problem as well, because I found the
function [`PdhAddEnglishCounter()`][aec], added in Vista, which made me smile.

[GetTickCount]: http://msdn.microsoft.com/en-us/library/ms724408%28VS.85%29.aspx
[GetTickCount64]: http://msdn.microsoft.com/en-us/library/ms724411%28VS.85%29.aspx
[perfcount]: http://msdn.microsoft.com/en-us/library/aa373083(VS.85).aspx
[PDH]: http://msdn.microsoft.com/en-us/library/aa373214%28VS.85%29.aspx
[counterpath]: http://msdn.microsoft.com/en-us/library/aa373193%28VS.85%29.aspx
[counters]: http://technet.microsoft.com/en-us/library/cc737309%28WS.10%29.aspx
[helparticle]: http://support.microsoft.com/?scid=kb%3Ben-us%3B287159&x=11&y=9
[lpnbi]: http://msdn.microsoft.com/en-us/library/aa372648%28VS.85%29.aspx
[PsInfo]: http://technet.microsoft.com/en-us/sysinternals/bb897550.aspx
[MarkR]: http://blogs.technet.com/markrussinovich/about.aspx
[sysinternals]: http://technet.microsoft.com/en-us/sysinternals/
[wininternals]: http://technet.microsoft.com/en-us/sysinternals/bb963901.aspx
[aec]: http://msdn.microsoft.com/en-us/library/aa372536%28VS.85%29.aspx
