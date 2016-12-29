---
author: nelhage
author_email: nelhage@mit.edu
author_login: nelhage
author_url: http://nelhage.com
date: 2008-09-16T12:08:12Z
published: true
status: publish
tags: []
title: autocutsel
url: /2008/09/16/autocutsel/
wordpress_id: 12
wordpress_url: http://blog.nelhage.com/?p=12
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

[autocutsel]: http://www.nongnu.org/autocutsel/
