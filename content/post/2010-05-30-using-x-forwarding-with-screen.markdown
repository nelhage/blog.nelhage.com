---
author: nelhage
author_email: nelhage@mit.edu
author_login: nelhage
author_url: http://nelhage.com
date: 2010-05-30T20:25:52Z
published: true
status: publish
tags: []
title: Using X forwarding with screen by proxying $DISPLAY
url: /2010/05/using-x-forwarding-with-screen/
wordpress_id: 232
wordpress_url: http://blog.nelhage.com/?p=232
---

If you're reading this blog, I probably don't have to explain why I love
[GNU screen][screen]. I can keep a long-running session going on a
server somewhere, and log in and resume my session without losing any
state.

I also love X-forwarding. I love being able to log into a remote
server and work in a shell there, but still pop up graphical windows
(for instance, gitk's) on my local machine when I need to.

Unfortunately, X-forwarding and screen don't totally play nice
together. When I start a screen session, I fix a value of `DISPLAY` in
that session, while I can change it in individual shells, having to
remember to do so whenever I open a new ssh session is
irritating. Ideally, of course, we'd have something like `screen`, but
for X11, so that X sessions could live in a virtual X server on your
machine, which gets forward to a real X server on demand. I hear that
[NX][nx] does something like this, or even just a VNC window.

But in practice, I find I tend to care less about having my X windows
long running. If I'm popping up a `gitk` to look at some commits, I
will probably just close it, and don't care about it being there
tomorrow. Really, I just want a way for `DISPLAY` to magically track
the latest X-forwared `DISPLAY`, so that in any window in my screen
session, I can run `gitk` or `display` or such, and it will magically
pop up in the appropriate X server.

So, last week, I finally wrote a [script][x11-proxy] that lets you do just
that. Instead of futzing with `DISPLAY`, it works by proxying between
two values of `DISPLAY`. You initially open a screen with some dummy
"virtual" DISPLAY that nothing is connected to -- I tend to use `:15`:

    env DISPLAY=:15 screen

Then, whenever you log in to the machine with X forwarding (or log in
locally), you simply run:

    proxy-display :15

`proxy-display` starts listening for connections on display `:15`, and
proxying traffic between there and whatever `$DISPLAY` was when it was
launched. In addition, it makes the appropriate `xauth` incants so
that everything just works. It also looks for any old instances of
itself listening on `:15`, and kills them off, to prevent leaking
processes. Any *connections* to old instances of itself, however, that
had accepted connections and were now proxying data, are left alone --
so existing X windows stay open on whatever display they're on.

So, now you can just add to your dotfiles something along the lines of

    if [ "$DISPLAY" ]; then
        proxy-display :15
    fi

And whenever you `ssh` into your remote machine and resume your screen
sessions, you can run X programs and they'll magically pop up in your
most recent X-forwarded connection.

(The script, available on [github][x11-proxy], is currently a disgusting mess of shell that uses `socat` to do the actual proxying, and mucks around in `/proc/net` to find old instances of itself to kill. But it works wonderfully, so I haven't bothered to clean it up)

[screen]: http://www.gnu.org/software/screen/
[nx]: http://www.nomachine.com/
[x11-proxy]: http://github.com/nelhage/x11-proxy
