---
layout:  post
title:   "Reordering Arguments"
date:    2016-11-07 21:04:16
author:  "jibsen"
tags:    ["C++", Algorithm]
usemath: true
---

When a C program runs, it usually receives any command-line arguments through
the parameters `argc` (argument count) and `argv` (argument vector). It is up
to the program to interpret the contents of this array of strings.

There are many ways to do this, one of the simpler solutions is the [getopt][]
function (in the following, getopt will refer to both getopt and getopt_long).
One extension some getopt implementations offer, is that they reorder the
contents of `argv` as they process it, resulting in an array where all the
options appear first followed by the nonoptions.

This reordering partitions the array. And we want a stable partition, so the
relative order of all the options, and of all the nonoptions is the same.

Last year I released [parg][], a simple parser for `argv` that works similarly
to getopt. It has a separate function, `parg_reorder()`, that performs this
reordering. I used a simple algorithm for this -- allocate a temporary array
the size of `argv`, parse the arguments, and fill in the options from the start
and nonoptions from the end of the new array. Then copy them back, reversing
the nonoptions to their original order in the process. In hindsight, I could
have moved the options into their final place in `argv` right away.

This runs in $$O(n)$$ time, but also requires $$O(n)$$ additional space. The
space requirement could be a problem (for instance on embedded systems), so
let's see if we can do better.

The code examples in the following are pseudo-code, and glance over some
details. Please see the [examples on GitHub][spart] for working C++ code.

We note that if we already partitioned the two halves of an array,
$$L_{1}R_{1}L_{2}R_{2}$$, we can compute the partition of both by swapping
$$R_{1}$$ and $$L_{2}$$. Swapping adjacent blocks in an array is sometimes
called _rotating_, and can be done in linear time, for instance using reversal
(observing that $$LR = (R^{R}L^{R})^{R}$$).

<div>
 <svg class="center-svg" width="80%" viewBox="0 0 380 80" version="1.1" xmlns="http://www.w3.org/2000/svg"  xmlns:xlink="http://www.w3.org/1999/xlink">
  <defs>
   <marker id="arrowhead" orient="auto" markerWidth="4" markerHeight="6" refX="1" refY="3">
    <path d="M0,0 L0.5,3 L0,6 L4,3 z"/>
   </marker>
  </defs>    
  <g style="stroke-linecap:round;stroke-linejoin:round;stroke-width:1;stroke:black;fill:none;">
   <title>Reordering by divide and conquer</title>
   <path d="M10,20 v-5 h180 v5 v-5 h180 v5"/>
   <rect fill="#56b4e9" x="10" y="30" width="100" height="20"/>
   <rect fill="#d55e00" x="110" y="30" width="80" height="20"/>
   <rect fill="#56b4e9" x="190" y="30" width="120" height="20"/>
   <rect fill="#d55e00" x="310" y="30" width="60" height="20"/>
   <path d="M150,65 v-8" marker-end="url(#arrowhead)"/>
   <path d="M150,65 h100 v-8" marker-end="url(#arrowhead)"/>
  </g>
 </svg>
</div>

So we can use [divide and conquer][DandC]. With a recursive function that
splits the array at the middle, this runs in $$O(n \log n)$$ time and requires
$$O(\log n)$$ stack space.

{% highlight cpp %}
stable_partition_recursive(first, n, pred)
{
	if (n == 1) {
		return pred(first) ? first + 1 : first;
	}

	left   = stable_partition_recursive(first, n / 2, pred);
	middle = first + n / 2;
	right  = stable_partition_recursive(middle, n - n / 2, pred);

	rotate(left, middle, right);

	return left + (right - middle);
}
{% endhighlight %}

Much better, but ideally we would like to use only constant extra space.
To achieve this, we can use the same technique as in bottom-up
[merge sort][mergesort]. We first process the array in blocks of size 2, then
again in blocks of size 4 (whose two halves of size 2 are already
partitioned), and so on, until we process the entire array as one block.

{% highlight cpp %}
stable_partition_bottomup(first, n, pred)
{
	for (width = 1; width < n; width *= 2) {
		next = first;

		for (i = 0; i < n; i += 2 * width) {
			limit = n - i < 2 * width ? n - i : 2 * width;

			if (limit > width) {
				middle = next + width;
				left   = std::find_if_not(next, middle, pred);
				next   = middle + width
				right  = std::find_if_not(middle, next, pred);

				rotate(left, middle, right);
			}
		}
	}

	return left + (right - middle);
}
{% endhighlight %}

This runs in $$O(n \log n)$$ time and uses constant extra space.

In a sense, the price we pay to avoid the recursion is that we do not remember
the partition points of the two halves, and need to find them again. This means
we use the predicate $$O(n \log n)$$ times instead of $$O(n)$$.

However, since the two halves are already partitioned, we can use binary search
to find the partition points. This reduces the number of predicate applications
to $$O(n)$$ (since $$\sum_{k=1}^{\infty}\frac{n}{2^k}\log 2^k = 2n$$), which
means we have the same time complexities as the recursive algorithm, but using
only constant extra space.

{% highlight cpp %}
stable_partition_bsearch(first, n, pred)
{
	for (width = 1; width < n; width *= 2) {
		next = first;

		for (i = 0; i < n; i += 2 * width) {
			limit = n - i < 2 * width ? n - i : 2 * width;

			if (limit > width) {
				middle = next + width;
				left   = std::partition_point(next, middle, pred);
				next   = middle + width
				right  = std::partition_point(middle, next, pred);

				rotate(left, middle, right);
			}
		}
	}

	return left + (right - middle);
}
{% endhighlight %}

If the array is almost partitioned, these algorithms still go through every
step. We can do something similar to natural merge sort, and repeatedly
look for the next place where a rotate is needed, stopping if we find none.
We must be careful though, if we simply look for $$RL$$ areas and rotate them,
we get quadratic time on alternating patterns. Instead we can look for
$$L_{1}R_{1}L_{2}R_{2}$$ and rotate the middle two, just like above.

{% highlight cpp %}
stable_partition_natural(first, last, pred)
{
	do {
		next = first;
		change = false;

		do {
			left   = std::find_if_not(next, last, pred);
			middle = std::find_if(left, last, pred);
			right  = std::find_if_not(middle, last, pred);
			next   = std::find_if(right, last, pred);

			if (left != middle && middle != right) {
				rotate(left, middle, right);
				change = true;
			}
		} while (next != last);
	} while (change);
}
{% endhighlight %}

At best this runs in $$O(n)$$, while the worst-case complexity is the same as
the bottom-up version. Since the widths are no longer fixed size, we have to
perform a linear search for the partition points, so the number of predicate
applications is back to $$O(n \log n)$$. Also, we have to use the predicate
to find the middle and next starting point, so we may use more predicate
applications than the bottom-up version.

There is an algorithm that can perform stable partitioning in $$O(n)$$ using
constant extra space ([PDF][katajainen]), but it is somewhat more involved,
and not practical given the constraints of the specific task.

Let us return to the problem of reordering arguments. One issue here is that we
cannot tell if any given element is an option, an option argument, or a
nonoption, without parsing the entire array up to that point (i.e. we cannot
assume random access).

This is because any given element could be preceded by an option that uses it
as its option argument (looking at the previous element is not enough, since
that might be an option argument instead of an option). Or there could be a
`--` somewhere, which means the remaining elements are nonoptions (unless that
`--` is in fact the option argument of a preceding option).

This makes the natural algorithm a good fit, since it applies the predicate
linearly from left to right in each loop.

So how does this all compare to getopt implementations? The two I checked
perform the reordering in each call by scanning over any nonoptions from the
current position, and then rotating them to the end of the array. This
effectively builds the array of nonoptions at the end, while moving down the
part that has not yet been processed.

<div>
 <svg class="center-svg" width="80%" viewBox="0 0 380 80" version="1.1" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">
  <defs>
   <marker id="arrowhead" orient="auto" markerWidth="4" markerHeight="6" refX="1" refY="3">
    <path d="M0,0 L0.5,3 L0,6 L4,3 z"/>
   </marker>
  </defs>    
  <g style="stroke-linecap:round;stroke-linejoin:round;stroke-width:1;stroke:black;fill:none;">
   <title>Reordering by moving to end</title>
   <path d="M115,25 v-10 h240 v8" marker-end="url(#arrowhead)"/>
   <rect fill="#56b4e9" x="10" y="30" width="90" height="20"/>
   <rect fill="#d55e00" x="100" y="30" width="30" height="20"/>
   <rect fill="#f0e442" x="130" y="30" width="240" height="20"/>
   <path d="M255,55 v10 h-138" marker-end="url(#arrowhead)"/>
  </g>
 </svg>
</div>

This requires constant extra space, but takes worst-case $$O(n^{2})$$ time.
While this could theoretically be used in an algorithmic complexity attack,
most systems limit the size of the command-line arguments in some way. As an
example of the worst-case behavior, the following line runs `ls` with 200,000
nonoptions (redirecting the error messages for missing file), and takes about
0.3 seconds (Fedora 24 running in a VM):

{% highlight bash %}
time ls `perl -e "print 'a a ' x 100000"` 2>/dev/null
{% endhighlight %}

Whereas this line runs `ls` with the same number of arguments, but alternating
nonoptions and options (the `-a` option enables showing files that start with
a period, and has no effect in this case), and takes 11 seconds:

{% highlight bash %}
time ls `perl -e "print 'a -a ' x 100000"` 2>/dev/null
{% endhighlight %}

[parg]: https://github.com/jibsen/parg
[getopt]: https://en.wikipedia.org/wiki/Getopt
[DandC]: https://en.m.wikipedia.org/wiki/Divide_and_conquer_algorithms
[mergesort]: https://en.m.wikipedia.org/wiki/Merge_sort
[spart]: https://github.com/jibsen/spart-example
[katajainen]: http://www.diku.dk/~jyrki/Paper/KP1992bJ.pdf
