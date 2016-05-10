---
layout:  post
title:   "Move to Parent"
date:    2010-10-01 22:04:56
author:  "jibsen"
tags:    [Information, Programming]
excerpt: "Moving folders is one of those tasks that appear trivial. But as
  always, the devil is in the details."
---
This post is inspired by a discussion regarding a [coding snack][snack] done by
lanux128 on [DonationCoder][dcc]. A user requested a tool to add a context menu
entry that would copy selected files and folders to the parent folder.

Moving folders is one of those tasks that appear trivial. But as always, the
devil is in the details.

To [keep it simple][KISS], we will assume regular files and folders in a
filesystem where paths are unique. The problem we will look at is: Given
absolute paths Src and Dst (where Dst is not equal to, or a subfolder of Src),
move the contents of Src to Dst.

The first solution that comes to mind is to move files and folders recursively.
In pseudocode it would be something along the lines of:

{% highlight cpp %}
MoveSimple(Src, Dst)
{
    // Loop through files/folders in Src folder
    for each Name in Src {
        if Name is a folder {
            // Create folder in Dst
            mkdir Dst\Name

            // Recursively move contents of subfolder
            MoveSimple(Src\Name, Dst\Name)

            // Remove subfolder, which is now empty
            rmdir Src\Name
        }
        else {
            // Move file to Dst
            move Src\Name Dst\Name
        }
    }
}
{% endhighlight %}

This works -- well, almost.

Consider the following folder structure (please forgive the tree output, it was
an easy way to get a consistent representation):

    \---parent
        \---foo
            |   log.txt  ; contains "first"
            |
            \---foo
                    log.txt  ; contains "second"

We are in \parent\foo and want to move the contents (the subfolder foo and the
file log.txt) to \parent. The correct result would be:

    \---parent
        |   log.txt  ; contains "first"
        |
        \---foo
                log.txt  ; contains "second"

The problem is that if we call MoveSimple(\parent\foo, \parent), and we process
the subfolder foo first, then we will overwrite \parent\foo\log.txt before it
gets moved out to \parent.

You might suggest that we handle files first, and then recurse on subfolders.
But that won't work either:

<pre>
\---parent
    \---foo
        +---foo
        |   \---zeb
        |           log.txt
        |
        \---zeb
                log.txt
</pre>

If we recurse into foo before zeb, we will overwrite \parent\foo\zeb\log.txt
before it gets moved out.

By now it should be clear that the culprit is the foo subfolder. With that, we
end up writing to parts of the folder structure we are in the process of
moving.

More generally, we may run into problems if any destination path touches the
source tree.

The sledge hammer fix is to make sure there is no way we can overwrite anything
in the source folder, by first moving it to someplace safe, and then to where
we want it:

{% highlight cpp %}
MoveSafeButSlow(Src, Dst)
{
    Tmp = random unique folder name

    // Create temporary folder in Dst
    mkdir Dst\Tmp

    // Move contents of Src to Tmp
    MoveSimple(Src, Dst\Tmp)

    // Remove Src which is now empty
    rmdir Src

    // Move contents of Tmp to Dst
    MoveSimple(Dst\Tmp, Dst)

    // Remove Tmp which is now empty
    rmdir Dst\Tmp
}
{% endhighlight %}

This works, but now we are moving every file and folder twice. Besides the
performance impact, it would also make it hard to interrupt the operation and
handle errors -- the user could end up with files lost in a temporary
directory.

Let's go back to MoveSimple and try to figure out what exactly went wrong. It
worked fine as long as we did not write to parts of the source tree that we had
not moved yet.

The only way we can end up writing into the source tree, is if Src contains a
special subfolder with a relative path that exactly matches the path from Dst
to Src. Such a path would be unique.

Moving that special folder is only a problem if there is still something else
left in Src that has not yet been moved. If we moved everything else, there is
nothing it can overwrite.

So if we check if we run into such a special folder, and make sure we move
everything else before it, it should work:

{% highlight cpp %}
MoveSafeHelper(Src, Dst, OrgSrc)
{
    Special = 0

    // Loop through files/folders in Src folder
    for each Name in Src {
        if Name is a folder {
            // Check if we are about to overlap original Src
            if Dst\Name == OrgSrc {
                // Remember Name, but skip for now
                Special = Name
            }
            else {
                // Create folder in Dst
                mkdir Dst\Name

                // Recursively move contents of subfolder
                Tmp = MoveSafeHelper(Src\Name, Dst\Name, OrgSrc)

                // If we found a point of overlap inside Name,
                // set Special to the relative path to it
                if Tmp != 0 {
                    Special = Name\Tmp
                }
                else {
                    // Otherwise remove subfolder which is now empty
                    rmdir Src\Name
                }
            }
        }
        else {
            // Move file to Dst
            move Src\Name Dst\Name
        }
    }

    return Special
}

MoveSafe(Src, Dst)
{
    Special = MoveSafeHelper(Src, Dst, Src)

    // If the call returned a relative path to a folder that touches Src,
    // then we have moved everything else in Src now, and can safely
    // move that folder
    if Special != 0 {
        MoveSafe(Src\Special, Dst\Special)
    }
}
{% endhighlight %}

I wrote a simple Perl script that implements MoveSafe, and it works for all the
test cases in this post, but of course standard disclaimers apply.

There are other ways to implement safely moving a folder, depending on what
features you desire. For instance you can make it safe to move a folder to a
child folder, which this function does not handle. The aim with this particular
method was to get (roughly) the same speed as the simple recursive method.

At this point I started wondering if I was overcomplicating things -- if this
was in fact all rather trivial. So I set out to check how some [popular file
managers][pfm] would fare.

The five I tried were: [Directory Opus][DO] 9.5.5.0, [Total Commander][TC]
7.55a, [xplorer&sup2;][x2] 1.8.0.13, [XYplorer][XY] 9.50, and [Windows
Explorer][winexp] (WinXP SP3).

I made two test cases, which are similar to the examples above, but contain a
few more files to make sure sorting order does not help any application get it
right by chance.

I used a script to generate the test cases to make sure they were identical,
and used any internal move command where possible. I answered yes to any
dialogs that asked if I wanted to overwrite anything.

The first test case is this (as usual we are in \parent\foo and want to move
the contents to \parent):

    \---parent
        \---foo
            |   a.txt ; "first"
            |   z.txt ; "first"
            |
            \---foo
                |   a.txt ; "second"
                |   z.txt ; "second"
                |
                \---foo
                        a.txt ; "third"
                        z.txt ; "third"

And the second test case is:

    \---parent
        \---foo
            +---ccc
            |       a.txt
            |       x.txt
            |
            +---foo
            |   +---ccc
            |   |       b.txt
            |   |       y.txt
            |   |
            |   \---zzz
            |           b.txt
            |           y.txt
            |
            \---zzz
                    a.txt
                    x.txt

None of the five file managers tested handled these two test correctly. I
realize this special condition is not very common, but I still found it
interesting that none of them, not even Windows Explorer, appear to be able to
move any folder to a parent folder.

[snack]: http://www.donationcoder.com/Forums/bb/index.php?topic=24095.0
[dcc]: http://www.donationcoder.com/
[KISS]: http://en.wikipedia.org/wiki/KISS_principle
[pfm]: http://www.donationcoder.com/Forums/bb/index.php?topic=9958.0
[DO]: http://www.gpsoft.com.au/
[TC]: http://www.ghisler.com/
[x2]: http://www.zabkat.com/
[XY]: http://www.xyplorer.com/
[winexp]: http://en.wikipedia.org/wiki/Windows_Explorer
