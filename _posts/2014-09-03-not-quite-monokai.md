---
layout:  post
title:   "Not Quite Monokai"
date:    2014-09-03 07:24:34
author:  "jibsen"
tags:    [Monokai, sRGB, TextMate, Themes]
---
[TextMate](http://macromates.com/) is a popular text editor for OS X. Since the
first release 10 years ago, a lot of people have contributed color themes, many
of which have been ported to other editors.

![Will the real Monokai please stand up?]({{ "/assets/realmonokai.png" | prepend: site.baseurl | prepend: site.url }})

If you are using a Windows or Linux text editor with such a conversion, you
might be looking at something slightly different than what the designer
intended.

This quote from the [Wikipedia article on RGB][RGB] sums up the problem:

> RGB is a device-dependent color model: different devices detect or reproduce
> a given RGB value differently, since the color elements (such as phosphors or
> dyes) and their response to the individual R, G, and B levels vary from
> manufacturer to manufacturer, or even in the same device over time. Thus an
> RGB value does not define the same color across devices without some kind of
> color management.

So, an RGB color like `#272822` does not necessarily look the same on all
computers, at least not without [color management][colorman]. When we combine a
color model like RGB with a [color space][colorspace], we get device
independent colors, which can be used to produce consistent results on
calibrated equipment.

A simple way to think of this is that an RGB color contains the relative level
of the red component, but does not specify what _red_ means. An RGB color space
describes what exactly red, green, blue, and white are.

The color spaces used on Windows and Mac OS computers more or less reflect the
hardware each platform was using at the time. This had the advantage that
existing media matched the default profiles.

HP and Microsoft created the [sRGB][] color space, which was later adopted as
the default color space for the web, and used in many consumer products.

On the Apple side, Adobe created [Apple RGB][AppleRGB] for use in their
software. With Mac OS X, Apple introduced a number of profiles, and Generic RGB
was the default assumed for untagged media up to OS X 10.7 Lion, 2011.

The color differences between sRGB and Generic RGB are relatively subtle, the
biggest difference is [gamma correction][gamma], which used a gamma value of
1.8 on Apple computers, while Windows uses 2.2.

This means that colors from OS X Generic RGB will look darker on Windows, and
colors from Windows sRGB will look light and washed out on OS X. This has long
been a known issue, and today it is much less of a problem due to color
management, browser support for images tagged with color profiles, and Apple
changing the default gamma from [1.8 to 2.2][gammachange] in OS X 10.6 Snow
Leopard, 2009.

So how does this all relate to TextMate themes?

TextMate was first released in 2004, and a number of popular color themes were
developed for it on Mac OS X, with the Generic RGB default color profile and a
gamma of 1.8.

These themes have since been ported to other editors on Windows and Linux (some
even support the same color theme file format that TextMate uses).

The colors are stored as RGB hex codes, like `#272822`, which means we will not
necessarily get the same color on Windows as on Mac OS. If the RGB color values
are used directly on Windows, the themes will look darker. In fact, even OS X
after the gamma change may render the colors darker outside TextMate.

The same problem has also occurred the other way around, where for instance the
[Solarized][] palette was developed in sRGB, and using the RGB values directly
in a TextMate theme would result in [lighter colors][tmsolar].

As an example, let us look at the [Monokai][] color theme by Wimer Hazenberg.
It was made in 2006 for TextMate, so we assume it was designed under the
default Generic RGB profile, with a gamma of 1.8.

Monokai has been ported to many editors, including Atom, Notepad++, and Sublime
Text. In these three editors (and presumably many others), the RGB values from
the original TextMate theme are used directly, which means they all appear
darker on Windows.

In 2013 TextMate [added a key][tmsrgb] to the theme files that allows choosing
sRGB color space ([without it][tmnosrgb], colors should be treated as Generic
RGB). This is great for new themes, but you cannot simply put a sRGB tag on old
themes without converting the colors into sRGB.

Converting from one RGB color space to another involves removing the source
gamma, doing a linear transformation, and applying the destination gamma. If
you are interested in more details on color spaces and the math involved in
converting between them, [Lindbloom][] and [Pascale][] (PDF) are both good
resources.

I wrote a [Python script](https://github.com/jibsen/tmcolorconv) to convert the
RGB values in a tmTheme file from Generic RGB to sRGB. Here is a screenshot
comparing Monokai original and sRGB in Sublime Text 3 on Windows:

![Monokai original and sRGB]({{ "/assets/monokai_compare.png" | prepend: site.baseurl | prepend: site.url }})

You may object that the screenshot on the Monokai blog matches the dark
appearance of the theme on Windows, but this is because the image is a GIF file
without embedded color profile, and thus suffers the same fate as the theme,
when its RGB values are assumed to be sRGB by your browser.

[RGB]: https://en.wikipedia.org/wiki/RGB_color_model
[colorman]: https://en.wikipedia.org/wiki/Color_management
[colorspace]: https://en.wikipedia.org/wiki/Color_space
[sRGB]: https://en.wikipedia.org/wiki/SRGB
[AppleRGB]: https://developer.apple.com/library/mac/qa/qa1430/_index.html
[gamma]: https://en.wikipedia.org/wiki/Gamma_correction
[gammachange]: http://support.apple.com/kb/ht3712
[Solarized]: http://ethanschoonover.com/solarized
[tmsolar]: https://github.com/deplorableword/textmate-solarized/issues/33
[Monokai]: http://www.monokai.nl/blog/2006/07/15/textmate-color-theme/
[tmsrgb]: https://github.com/textmate/textmate/commit/d70ccc7c
[tmnosrgb]: https://github.com/aziz/tmTheme-Editor/issues/15
[Lindbloom]: http://www.brucelindbloom.com/
[Pascale]: http://www.babelcolor.com/download/A%20review%20of%20RGB%20color%20spaces.pdf
