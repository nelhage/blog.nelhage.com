---
author: nelhage
author_email: nelhage@mit.edu
author_login: nelhage
author_url: http://nelhage.com
date: 2011-01-21T21:56:01Z
published: true
status: publish
tags:
- linux
- termios
- ptrace
- screenify
- reptyr
title: 'reptyr: Attach a running process to a new terminal'
url: /2011/01/reptyr-attach-a-running-process-to-a-new-terminal/
wordpress_id: 450
wordpress_url: http://blog.nelhage.com/?p=450
---

Over the last week, I've written a nifty tool that I call
[reptyr][github]. reptyr is a utility for taking an existing running
program and attaching it to a new terminal. Started a long-running
process over ssh, but have to leave and don't want to interrupt it?
Just start a screen, use reptyr to grab it, and then kill the ssh
session and head on home.

You can [grab the source][github], or read on for some more details.

There's a shell script called [screenify][screenify] that's been going
around the internet for nigh on 10 years now that is supposed to use
gdb to accomplish the same thing. There's also a project called
[retty][retty] that tries to do the same thing, in C using `ptrace()`
directly.

The difference between those programs and reptyr is that reptyr works
much, much, better.

If you attach a `less` using screenify or retty, it will still take
input from the old terminal. If you attach an ncurses program, and
resize the window, the program probably won't resize correctly. `^C`
and `^Z` will still be processed on the old terminal -- typing them in
the new terminal won't do anything useful.

reptyr fixes all of these problems and more, and is the only such tool
I know of that does so. I've never seen a program that doesn't behave
noticeably incorrectly after attaching with retty or screenify,
whereas with reptyr most programs I have tried work flawlessly.

How does it work?
-----------------

`reptyr` works in the same basic way as `screenify` and `retty` -- it
attaches to the target process using the `ptrace` API, opens the new
terminal, and `dup2`s it over the old file descriptors. It also copies
the termios settings from the old terminal to the new terminal.

The main thing that reptyr does that no one else does is that it
actually changes the controlling terminal of the process you are
attaching. This is the detail that makes many things Just Work,
including `^C` and `^Z` and window resizing.

Switching the target's controlling terminal is not easy and involves a
fair bit of trickery with `ptrace` and Linux's terminal APIs. I will
probably do another blog post some time about the dirty details of how
I make this work, but for now you can check out
[attach.c](https://github.com/nelhage/reptyr/blob/master/attach.c) if
you really want to know.

reptyr still has a number of limitations -- it doesn't generally work,
for example, if the target process has any children. I know how to fix
most of these problems, though, so expect it to get better with
time. Please let me know if you find it useful!

Appendix
--------

(Edited to add:) Nothing is really new. A commenter on reddit pointed out that [injcode][injcode]
and [neercs][neercs] both accomplish the same thing, even using the same trick
to change the CTTY. Ah well, I had run writing it anyways, and apparently I
wasn't the only one who didn't know about the existing alternatives. `neercs` is a full screen replacement, though, and I think that reptyr should be more robust than `injcode` -- I use a different techique for `ptrace`-hijacking, for example -- and so hopefully this tool still has a niche as a more robust standalone utility. Certainly, judging from the amount of enthusiasm I've seen for this tool, this still isn't a problem that is solved to the average user's satisfaction.

[github]: http://github.com/nelhage/reptyr
[screenify]: http://tomaw.net/tmp/screenify
[retty]: http://pasky.or.cz/~pasky/dev/retty/
[injcode]: http://blog.habets.pp.se/2009/03/Moving-a-process-to-another-terminal
[neercs]: http://caca.zoy.org/wiki/neercs
