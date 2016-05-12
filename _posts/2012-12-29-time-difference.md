---
layout:  post
title:   "Time Difference"
date:    2012-12-29 19:09:04
author:  "jibsen"
tags:    [C, Programming, Time]
---
This post is based on a discussion about <a href="http://www.donationcoder.com/forum/index.php?topic=33116.0">Progress Bars of Life</a>, where I was foolish enough to claim that printing a text string representing the difference between two times could not be that hard in C. It is not hard, but turned out not to be entirely trivial either.

![Pocket Watch]({{ "/assets/stockvault-pocket-watch100366-300x225.jpg" | prepend: site.baseurl | prepend: site.url }}){:.center-image}

The problem we will consider is; given two <a href="http://en.cppreference.com/w/c/chrono/tm"><code>tm</code> structs</a>, compute the difference in time between them, in such a way that we can easily format a string that gives a textual representation of it. We want years, months, days, hours, minutes, seconds.

The first idea you might get is to use <a href="http://en.cppreference.com/w/c/chrono/difftime"><code>difftime()</code></a> to get the difference in seconds between the two times, and then compute the quantities we want by simple arithmetic. So,

{% highlight c %}
start_time = mktime(&start);
end_time = mktime(&end);
secdiff = difftime(end_time, start_time);

years   = secdiff / SEC_IN_YEAR;
months  = (secdiff % SEC_IN_YEAR) / SEC_IN_MONTH;
days    = (secdiff % SEC_IN_MONTH) / SEC_IN_DAY;
hours   = (secdiff % SEC_IN_DAY) / SEC_IN_HOUR;
minutes = (secdiff % SEC_IN_HOUR) / SEC_IN_MINUTE;
seconds = (secdiff % SEC_IN_MINUTE);
{% endhighlight %}

You often see something like this in timing code -- it works great for showing elapsed time in seconds, minutes, even hours. Do you see any problems with this approach?
<!--more-->
How many seconds are there in a month? That depends on which month of course. And even worse, it also depends on which year, due to <a href="http://en.wikipedia.org/wiki/Leap_year">leap years</a>.

For instance, the difference between Jan 31st and Mar 1st is sometimes 29 days, sometimes 30 days, but always 1 month 1 day. The difference between Jul 2nd and Aug 1st is 30 days, but not a month.

So once the difference in time exceeds hours, we somehow need to use the additional information about where this period of time starts and ends, in order to get answers that make sense to humans. Luckily this information is already available in the <code>tm</code> structs.

If we accept the convention that the time between the 1st of a month and the 1st of the following is a "month" regardless of how many days are between, then one approach is to use elementary school subtraction on the <code>tm</code> structs. We subtract one field at a time, starting at the lowest, borrowing from the next one if required.

For instance, we get the difference in seconds by subtracting the <code>tm_sec</code> fields. If the result is negative, we have to borrow a minute.

{% highlight c %}
/* Difference in the seconds */
seconds = end.tm_sec - start.tm_sec;

/* If negative, we have to borrow a minute */
if (seconds < 0) {
        seconds = 60 + seconds;
        min_borrow = 1;
}
{% endhighlight %}

But how do we borrow a month? Since we already know the number of days in the end month (<code>end.tm_mday</code>), and all months in between are full calendar months, we only need to figure out how many days are in the start month.

{% highlight c %}
/* Returns 1 if y is a leap year, 0 otherwise */
static int leap(int y)
{
        return (y % 400 == 0 || (y % 4 == 0 && y % 100 != 0)) ? 1 : 0;
}

const int md[] = { 31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31 };

/* ... */

/* Difference in the days of month */
days = end.tm_mday - start.tm_mday - day_borrow;

/* If negative, we have to borrow a month */
if (days < 0) {
        int start_mon_days;

        /* Get number of days in start month */
        start_mon_days = md[start.tm_mon];

        /* If february, correct for leap year */
        if (start.tm_mon == 1) {
                start_mon_days += leap(1900 + start.tm_year);
        }

        days = start_mon_days + days;
        mon_borrow = 1;
}
{% endhighlight %}

So, if we borrow a month, then the difference in the days of the month is the number of days left in the start month, plus the number of days in the end month, minus any borrow from the calculation of hours of the day. This is the same as the total number of days in the start month plus the negative difference we already computed.

This method gives the desired results on examples like the ones given above -- months are what we perceive as months on the calendar.

There is one more detail we may have to consider -- how many hours are there in a day? Usually 24, but due to <a href="http://en.wikipedia.org/wiki/Daylight_saving_time">daylight saving time</a> (summer time) it might be 23 or 25.

As an example, consider the difference between 01:45 and 03:15, which is 1 hour 30 minutes in most cases. But the night DST starts, it will be only 30 minutes (from 01:45 STD to 03:15 DST). Even worse, the night DST ends, it will first be 2 hours 30 minutes (from 01:45 DST to 03:15 STD), then 1 hour 30 minutes (from 01:45 STD to 03:15 STD) as the clock runs through the extra hour.

Much like the relationship between months and days, I think it makes sense to interpret the difference between the same time on consecutive dates as a "day", no matter if it takes 23, 24, or 25 actual hours. Once the difference goes below 1 day however, it is more logical to use the actual difference in clock time.

So we would like the difference between 12:00 STD the day before DST starts, and 12:30 DST the day DST starts to be 1 day 30 minutes even though it only takes 23 hours 30 minutes. And we want the difference between 12:30 DST the day before DST ends, and 12:00 STD the day DST ends to be 24 hours 30 minutes but not a day.

Handling of DST is implementation defined in C, so to keep things simple, we
can let [`mktime()`](http://en.cppreference.com/w/c/chrono/mktime) handle this
for us. We just have to adjust the hours of the day in case the difference is
below one day.

{% highlight c %}
/* If difference is below one calendar day and there was a DST difference
   we adjust hour difference to clock time */
if (years + months + days == 0
 && hours != (secdiff % SEC_IN_DAY) / SEC_IN_HOUR) {
        int oldhours = hours;

        hours = (secdiff % SEC_IN_DAY) / SEC_IN_HOUR;

        /* Handle the special case where DST increased hours past 24 */
        if (oldhours - hours > 11) {
                hours += 24;
        }
}
{% endhighlight %}

In many cases you can safely ignore these details, and I am sure there are
better ways to handle it -- but I hope this has given some indication of how
it may not be quite as simple as it appears. The code I wrote while playing
around with this problem is [available here](https://bitbucket.org/jibsen/tmdiff).
