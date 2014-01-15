---
layout: post
tags: [C, FreeBSD]
---
I just pushed a [new project](https://github.com/p120ph37/libfb-bsd) up to
GitHub. It's the beginning of a console
[framebuffer](http://en.wikipedia.org/wiki/Framebuffer) graphics library for
FreeBSD.  While Linux has [SVGALib](http://www.svgalib.org/), and BSD used to
have
[VGL](http://manpages.unixforum.co.uk/man-pages/unix/freebsd-6.2/3/vgl-man-page.html),
there doesn't seem to be anything current for this purpose, and the
documentation for the kernel
[framebuffer ioctls](http://fxr.watson.org/fxr/source/sys/fbio.h)
is minimal (and even these seem mostly to just be
[thunks](http://en.wikipedia.org/wiki/Thunk_(compatibility_mapping)) to the old
[interrupt 10h](http://www.ctyme.com/intr/int-10.htm) functions). I need this
for a certain BSD-based side project, and since
[no](http://forums.freebsd.org/showthread.php?t=23163)
[one](http://lists.freebsd.org/pipermail/freebsd-arch/2004-February/001752.html)
[else](http://forums.freebsd.org/showthread.php?t=7384)
[seems](http://people.freebsd.org/~jmg/ppmdisp.c) to have it working at the
moment I had to do it myself.