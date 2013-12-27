---
layout: post
status: publish
published: true
title: ! 'Wordpress tricks: Disabling editing shortcuts'
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 250
wordpress_url: http://blog.nelhage.com/?p=250
date: 2010-06-13 20:07:00.000000000 +02:00
categories:
- Uncategorized
tags:
- javascript
- emacs
- wordpress
- greasemonkey
- keyboard
---
One of the major reasons I can't stand webapps is because I'm a
serious emacs junkie, and I can't edit text in anything that doesn't
have decent emacs keybindings.

Fortunately, on Linux, at least, GTK provides basic emacs keybindings
if you add

    gtk-key-theme-name = "Emacs"

to your `.gtkrc-2.0`. However, some webapps think that they deserve
total control over your keys, and grab key combinations for a WYSIWYG
editor of some sort. And so whenever I try to edit a post in Wordpress
(most of them are written in emacs and then copied over), I find
myself trying to go backwards a word, and inserting random
<code>&lt;strong&gt;</code> tags all over my post (Because `M-b` is
bound to make text bold, by Wordpress's editor). I finally got annoyed
enough to do some source-diving, and discovered that Wordpress's
editor constructs keyboard shortcuts using the HTML <a
href="http://www.w3.org/TR/html5/editing.html#dfnReturnLink-0">accesskey</a>
attribute. This is easy enough to manipulate from Javascript, so I
went and wrote up a quick Greasemonkey user script. The bulk of it is
a simple XPath:

        var buttons = document.evaluate('//input[@type="button"][@accesskey]', poststuff);
        var button;

You can <a href="http://nelhage.com/files/wp-keys.user.js">install the
script</a> off of nelhage.com.

Let me know if you find this useful, or if anyone figures out a
general way to disable (sets of) keyboard shortcuts for websites,
without relying on knowing the specific tricks that a website uses.
