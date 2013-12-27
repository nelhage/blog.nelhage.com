---
layout: post
status: publish
published: true
title: autocutsel
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 12
wordpress_url: http://blog.nelhage.com/?p=12
date: 2008-09-16 12:08:12.000000000 +02:00
categories:
- Uncategorized
- linux
tags: []
---
As most of you probably know, X has several different mechanisms for
copy-paste, used by different applications in different ways. I know
some people who use them deliberately, juggling two pieces of text in
different clipboards at once, but for me, it's always just been
annoying. When I copy something, be it by Gnome `C-c`, emacs `C-w`, or
selecting it in an xterm, I then want to be able to paste it again, no
matter what mechanism I use.

I've long thought it should be trivial to write a daemon that
synchronizes the clipboards, and it turns out that indeed someone's
done so: [Autocutsel][autocutsel]. And now, it turns out there are in
fact at least three clipboards, but by running it twice, syncing
between two pairs, I've no longer had the issue of pasting from the
wrong clipboard and having to remember *how* I copied that URL to give
to someone. My `.xsession` incant is simply:

    autocutsel -fork
    autocutsel -selection PRIMARY -fork

[autocutsel]: http:&#47;&#47;www.nongnu.org&#47;autocutsel&#47;
