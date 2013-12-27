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
[todo.pl][todopl]<&#47;a> and ratmenu to pop up a list of tasks, and mark
one as completed:

    #!&#47;bin&#47;sh
    todo.pl | perl -ne 'push @a,$2,"todo.pl done $1" if &#47;^#([\w]+) (.+)$&#47;;' \
                   -e 'END{exec("ratmenu",@a)}'

I dropped it into my `~&#47;bin` and bound it to `C-t x` in my window
manager ([XMonad][xmonad]). I love it already.

[broder]: http:&#47;&#47;ebroder.net
[hm]: http:&#47;&#47;hiveminder.com
[todopl]: http:&#47;&#47;hiveminder.com&#47;tools
[xmonad]: http:&#47;&#47;xmonad.org
