---
layout: post
status: publish
published: true
title: todo.pl ratmenu
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 7
wordpress_url: http://nelhage.scripts.mit.edu/madeofbugs/?p=7
date: 2008-02-19 23:46:00.000000000 +01:00
categories:
- Uncategorized
tags:
- hiveminder
- ratmenu
- todo.pl
---
[broder][broder] has been hacking on some better quicksilver
integration for [Hiveminder][hm] using todo.pl.

I don't use a mac, but I don't see why linux users shouldn't get fun
toys to. So I hacked up the following two-liner that uses
[todo.pl][todopl]</a> and ratmenu to pop up a list of tasks, and mark
one as completed:

    #!/bin/sh
    todo.pl | perl -ne 'push @a,$2,"todo.pl done $1" if /^#([\w]+) (.+)$/;' \
                   -e 'END{exec("ratmenu",@a)}'

I dropped it into my `~/bin` and bound it to `C-t x` in my window
manager ([XMonad][xmonad]). I love it already.

[broder]: http://ebroder.net
[hm]: http://hiveminder.com
[todopl]: http://hiveminder.com/tools
[xmonad]: http://xmonad.org
