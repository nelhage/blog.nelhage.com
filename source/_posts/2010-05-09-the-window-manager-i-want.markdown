---
layout: post
status: publish
published: true
title: The Window Manager I Want
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 220
wordpress_url: http://blog.nelhage.com/?p=220
date: 2010-05-09 17:08:47.000000000 +02:00
tags:
- linux
- emacs
- ui
- xmonad
- window manager
- ratpoison
---
<div id="outline-container-1" class="outline-2">
<div id="text-1">

<p>Since I first discovered <a href="http://www.nongnu.org/ratpoison/">ratpoison</a> in 2005 or so, I've basically
exclusively used tiling window managers, going through, over the
years, <a href="http://www.nongnu.org/stumpwm/">StumpWM</a>, <a href="http://en.wikipedia.org/wiki/Ion_(window_manager)">Ion 3</a>, and finally <a href="http://xmonad.org/">XMonad</a>. They've all had various
strengths and weaknesses, but I've never been totally happy with any
of them. This blog entry is a writeup of what I want to see as a
window manager. It's possible that some day I'll get annoyed enough to
write it, but maybe this post will inspire someone else to (Not
likely, but I can hope).
</p>

</div>

<div id="outline-container-1.1" class="outline-3">
<h3 id="sec-1.1">Layout </h3>
<div id="text-1.1">


<p>
At any given moment, the screen<sup><a class="footref" name="fnr.1" href="#fn.1">1</a></sup>
is divided into one or more <b>panes</b>. These panes always tile the
screen. Each pane may or may not currently be displaying a window. If
it is not, then the desktop will be displayed in that pane.
</p>
<p>
The primitive operations you can perform on the pane include, but are
not necessarily limited to:
</p>
<dl>
<dt>Split</dt><dd>
Splits the focused pane into two equally-sized panes,
either horizontally or vertically. One of the child panes
will display whichever window the previous pane displayed,
the other is empty.
</dd>
<dt>Resize</dt><dd>
Change the relative size of two child windows that were
split from the same parent.
</dd>
<dt>Kill Pane</dt><dd>
Destroy the current pane. If it is displaying a window,
that window is not sent a close message. The pane's
sibling window expands to consume its space.
</dd>
<dt>Next/Previous pane</dt><dd>
Focus the next or previous pane. Panes are
ordered based on position on the screen, regardless of how they
were split to arrive at the current layout.
</dd>
<dt>Goto Pane</dt><dd>
Panes are numbered, based on their location on
screen. <code>Mod-N</code> focuses the Nth pane.

</dd>
</dl>
</div>

</div>

<div id="outline-container-1.2" class="outline-3">
<h3 id="sec-1.2">Selecting windows </h3>
<div id="text-1.2">


<p>
There is a global list of all windows. Windows are ordered arbitrarily
in this list (maybe based on the order they were opened?). Every
window has a name, which is initially set to the <code>WM_NAME</code> property
set by the window's client.
</p>
<p>
The following operations are available to manipulate windows:
</p>
<dl>
<dt>Next/Previous Window</dt><dd>
Replace the window in the current panel with
the next or previous window in the list.
</dd>
<dt>Rename</dt><dd>
Prompts for a new name for the current window. If the user
enters an empty string, the window reverts to the default
behavior of using the <code>WM_NAME</code>.
</dd>
<dt>Goto Window</dt><dd>
Prompts for the name of a window to switch to. This
prompt matches on substrings or even sub-sequences of
the window name, displaying the result of the
selection as you type. After typing some characters,
you can scroll through the list to select an entry,
instead of completing typing.
</dd>
<dt>Kill</dt><dd>
Sends a close message to the window in the focused panel.

</dd>
</dl>
</div>

</div>

<div id="outline-container-1.3" class="outline-3">
<h3 id="sec-1.3">Desktops </h3>
<div id="text-1.3">


<p>
It seems likely I will want multiple desktops. Each desktop will have
its own pane layout. However, the window list is still global, not
per-desktop. <code>Goto Window</code> will always operate on the global window
list. Selecting a window that is visible through a pane somewhere else
causes that pane to become empty.
</p>
<p>
Alternately, there is no concept of multiple desktops. However, there
is the ability to save and restore layouts, meaning both the layout of
the panels on the screen, and the list of which window is in each
pane. The primary difference here is that a window can be associated
with multiple panes in different saved layouts, and gets resized/moved
as necessary as you switch panes. I suspect I like this model better.
</p>
</div>

</div>

<div id="outline-container-1.4" class="outline-3">
<h3 id="sec-1.4">Postscript </h3>
<div id="text-1.4">


<p>
If this sounds familiar to you, it probably should. What I've
described is essentially identical to how emacs manages buffers and
windows (analogous to X windows and panes in the above, respectively),
with <a href="http://www.emacswiki.org/emacs/window-number.el">window-number.el</a> and either <a href="http://www.emacswiki.org/emacs/IswitchBuffers">iswitchb</a> or <a href="http://www.emacswiki.org/emacs/InteractivelyDoThings">ido</a>. I manage hundreds
of buffers in emacs this way, and complicated screen layouts, whenever
I'm doing any hacking, and I love it.
</p>
<p>
I would in fact be tempted to write my window manager into emacs
itself, except for the annoying fact that emacs is very much
single-threaded. It's already annoying enough when network drops and a
hung network filesystem takes down my emacs waiting for a timeout; It
would be utterly unacceptable if that took down my entire window
manager, too.
</p>

<p>
I'd alternately be tempted to try to make this an XMonad plugin, but I
think that it's sufficiently different from XMonad's data model that
the impedance mismatch would suck.
</p>

</div>
</div>
</div>
<div id="footnotes">
<h2 class="footnotes">Footnotes: </h2>
<div id="text-footnotes">
<p class="footnote"><sup><a class="footnum" name="fn.1" href="#fnr.1">1</a></sup> I'm only going to consider the
single-monitor case for now, but the generalization should be easy.
</p>
</div>
</div>
