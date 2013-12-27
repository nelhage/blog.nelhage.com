---
layout: post
status: publish
published: true
title: Some Android reverse-engineering tools
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 436
wordpress_url: http://blog.nelhage.com/?p=436
date: 2010-12-27 16:26:13.000000000 +01:00
categories:
- Low-level hacking
tags:
- security
- android
- dalvik
- dedexer
- reverse-engineering
---
I've spent a lot of time this last week staring at decompiled Dalvik
assembly. In the process, I created a couple of useful tools that I
figure are worth sharing.

I've been using [dedexer][dedexer] instead of [baksmali][baksmali],
honestly mainly because the former's output has fewer blank lines and
so is more readable on my netbook's screen. Thus, these tools are
designed to work with the output of dedexer, but the formats are
simple enough that they should be easily portable to smali, if that's
your tool of choice (And it does look like a better tool overall, from
what I can see).

`ddx.el`
--------

I'm an emacs junkie, and I can't stand it when I have to work with a
file that doesn't have an emacs mode. So, a day into staring at
un-highlighted `.ddx` files in `fundamental-mode`, I broke down and
threw together [`ddx-mode`][ddx.el]. It's fairly minimal, but it
provides functional syntax highlighting, and a little support for
navigating between labels. One cute feature I threw in is that, if you
move the point over a label, any other instances of that label get
highlighted, which I found useful in keeping track of all the "lXXXXX"
labels dedexer generates.

[caption id="attachment_438" align="aligncenter" width="499" caption="An example file (from k9mail) highlighted using ddx-mode"]<a href="http://blog.nelhage.com/wp-content/uploads/2010/12/ddx-e1293480505629.png"><img src="http://blog.nelhage.com/wp-content/uploads/2010/12/ddx-e1293480505629.png" alt="" title="ddx-mode" width="499" height="471" class="size-full wp-image-438" /></a>[/caption]

`ddx2dot`
---------

Dalvik assembly is, on the whole pretty easy to read, but occasionally
you stumble on huge methods that clearly originated from multiple
nested loops and some horrible chained if statements. And what you'd
really like is to be able to see the structure of the code, as much as
the details of the instructions.

To that end, I threw together a Python script that "parses" `.ddx`
files, and renders them to a control-flow graph using [dot][dot]. As
an example, the [`parseToken`][parseToken] method from the IMAP parser
in the [k9mail][k9] application for Android looks like the following,
when disassembled and rendered to a CFG:

[caption id="attachment_442" align="aligncenter" width="300" caption="A CFG for k9mail\'s <tt>ImapResponseParser.parseToken</tt> method"]<a href="http://blog.nelhage.com/wp-content/uploads/2010/12/parseToken.png"><img src="http://blog.nelhage.com/wp-content/uploads/2010/12/parseToken-300x143.png" alt="" title="parseToken" width="300" height="143" class="size-medium wp-image-442" /></a>[/caption]

I use the term "parses" because it's really just a pile of regexes, `line.split()` and `line.startswith("...")`, but it gets the job done, so I hope it might be of use to someone else. The biggest missing feature is that it doesn't parse `catch` directives, so those just end up floating out to the side as unattached blocks.

You'll also notice the rounded "return" blocks -- either `javac` or `dx` merges all exits from a function to go through the same `return` block, but I found that preserving that feature in the CFG produces a lot of clutter and makes it hard to read, so I lift every edge that would go to that common block to go to a separate block.

Github
------

Both tools live in my "reverse-android" [repository][git-repo] on
github, and are released under the MIT license. Please feel free to do
whatever you want with them, although I'd appreciate it if you let me
know if you make any improvements or find them useful.

[dedexer]: http://dedexer.sourceforge.net/
[baksmali]: http://code.google.com/p/smali/
[ddx.el]: https://github.com/nelhage/reverse-android/blob/master/ddx.el
[k9]: http://code.google.com/p/k9mail/
[parseToken]: http://code.google.com/p/k9mail/source/browse/k9mail/trunk/src/com/fsck/k9/mail/store/ImapResponseParser.java?r=2996#119
[git-repo]: https://github.com/nelhage/reverse-android
[dot]: http://www.graphviz.org/
