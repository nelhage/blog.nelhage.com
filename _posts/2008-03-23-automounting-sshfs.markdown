---
layout: post
status: publish
published: true
title: Automounting sshfs
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 9
wordpress_url: http://nelhage.scripts.mit.edu/madeofbugs/?p=9
date: 2008-03-23 18:54:00.000000000 +01:00
categories:
- Uncategorized
- linux
tags:
- linux
- afuse
- sshfs
- ssh
---
For some time now, many of us around MIT have noticed just how awesome
[sshfs][sshfs] is. It gives a totally lightweight way to access the
remote filesystem of any machine you have ssh to, without requiring
_any_ extra setup on the host. I've been running for at least a year
now with my `&#47;data` RAID on my server sshfs-mounted on my laptop, and
it works totally great.

Recently, I came across two awesome things that make sshfs even
neater. The first is the `ServerAliveInterval` ssh configuration
option. I (and many others) had noticed that if you changed IP
addresses (which happens all the time with our laptops), sshfs will
just kinda hang there, and so will anything that tries to access
anything in the ssfs-mounted filesystem. `sshfs` has a `-o reconnect`
option that makes it automatically reconnect the underlying ssh if it
dies, but it doesn't solve the problem of the ssh hanging forever. The
solution, it turns out, is the `ServerAliveInterval` config
option. Just add

    Host *
    ServerAliveInterval 15

to `.ssh&#47;config`, and ssh will send in-protocol keepalives every 15
seconds if the connection is idle, and die if it doesn't receive
anything back. Combine this with `-o reconnect`, and everything Just
Works when you change IPs

The second cool thing is [afuse][afuse], the FUSE automounter. It lets
you set up an automounter for just about anything you can think of,
using another FUSE filesystem itself. I simply run it as

    afuse -o mount_template='sshfs -o reconnect %r:&#47; %m' -o unmount_template='fusermount -u -z %m' &#47;ssh

from my `.xsession`, and I have a `&#47;ssh` automounter!  Combined with
the wonders of kerberos and public keys, so I never have to type a
password, and I can get easy remote access to just about every machine
I care about!

(Note that I did have to chown `&#47;ssh` to me in order for me to be able
to run `afuse` as me, which is necessary for `sshfs` to access my
kerberos tickets and ssh keys. This is fine for my laptop, but
obviously wouldn't work for a dialup or other multi-user machine.)

[afuse]: http:&#47;&#47;afuse.sourceforge.net&#47;
[sshfs]: http:&#47;&#47;fuse.sourceforge.net&#47;sshfs.html
