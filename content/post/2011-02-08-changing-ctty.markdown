---
author: nelhage
author_email: nelhage@mit.edu
author_login: nelhage
author_url: http://nelhage.com
date: 2011-02-08T23:06:50Z
status: publish
tags:
- linux
- termios
- unix
- tty
- reptyr
title: 'reptyr: Changing a process''s controlling terminal'
url: /2011/02/changing-ctty/
wordpress_id: 461
wordpress_url: http://blog.nelhage.com/?p=461
---

[reptyr][reptyr] ([announced][announce] recently on this blog) takes a
process that is currently running in one terminal, and transplants it
to a new terminal. `reptyr` comes from a proud family of similar
hacks, and works in the same basic way: We use [`ptrace(2)`][ptrace]
to attach to a target process and force it to execute code of our own
choosing, in order to open the new terminal, and `dup2(2)` it over
stdout and stderr.

The main special feature of `reptyr` is that it actually changes the
controlling terminal of the target process. The "controlling terminal"
is a concept maintained by UNIX operating systems that is independent
of a process's file descriptors. The controlling terminal governs
details like where `^C` gets delivered, and how applications are
notified of changes in window size.

Processes are grouped into two levels of hierarchical groups:
sessions, and process groups. Each group is named by an ID, which is
the PID of the initial **leader** (either "session leader" or "process
group leader"). Even if the leader exits, that number is still the ID
for the group. Sessions are used for terminal management -- Every
process in a session has the same controlling terminal, and each
terminal belongs to at most one session. Process groups are a
sub-division within sessions, and are used primarily for job control
within the shell. For a more in-depth explanation, see [part
3][termios] of my earlier series on termios.

If you check out `tty_ioctl(4)`, you'll find that Linux has an
`ioctl`, `TIOCSCTTY`, that can be used to set the controlling terminal
of a process, and you could be forgiven for thinking that all we need
is to make the target call that ioctl, and we're done.

However, if we read closer, we find that it has several
restrictions. In particular:

> The calling process must be a session leader and not have a
> controlling terminal already.  If this terminal is already the
> controlling terminal of a different session group then the ioctl fails
> with EPERM […]

In the typical case, where I'm trying to attach a (say) `mutt` that
you spawned from your shell, `mutt` won't be a session leader -- your
shell will be the session leader, and `mutt` will be the process group
leader for a process group containing only itself.

So, we need to make the target a session leader. Conveniently, there's
a system call for that: `setsid(2)`.

However, reading that man page, we find a new caveat: `setsid(2)`
fails with `EPERM` if

> The process group ID of any process equals the PID of the calling
> process.  Thus, in particular, setsid() fails if the calling process
> is already a process group leader.

The shell creates a new process group for every job you launch, and so
our target `mutt` will be a process group leader, and unable to
`setsid()`. The usual solution for programs that want to setsid is to
`fork()`, so that the child is still in the parent's session and
process group, and then `setsid()` in the child. However, `fork()`ing
our `mutt` and killing off the parent seems potentially disruptive, so
let's see if we can avoid that.

So, we're going to need to change `mutt`'s process group ID, so that
there are no processes with process group IDs equal to its
PID. Following some trusty *`SEE ALSO`* links, we get to
`setpgid(2)`. There's a bunch of text in that man page, but the key
bit is:

> If setpgid() is used to move a process from one process group to
> another, both process groups must be part of the same session (see
> setsid(2) and credentials(7)).  In this case, the pgid specifies an
> existing process group to be joined and the session ID of that group
> must match the session ID of the joining process.

We need to find a process group in the same session as `mutt` to move
our `mutt` into, and then we'll be able to `setsid`. We could try to
find one -- the shell is a plausible candidate, for instance -- but
there's an alternate, more direct route: Create one.

While we have `mutt` captured with `ptrace`, we can make it `fork(2)`
a dummy child, and start tracing that child, too. We'll make the child
`setpgid` to make it into its own process group, and then get `mutt`
to `setpgid` itself into the child's process group. `mutt` can then
`setsid`, moving into a new session, and now, as a session leader, we
can finally `ioctl(TIOCSCTTY)` on the new terminal, and we win.

It turns out I didn't invent this technique -- [injcode][injcode] and
[neercs][neercs] work the same way. But I did discover it
independently of them, and it was a fun little hunt through unix
arcana.

[reptyr]: https://github.com/nelhage/reptyr
[announce]: http://blog.nelhage.com/2011/01/reptyr-attach-a-running-process-to-a-new-terminal/
[ptrace]: http://linux.die.net/man/2/ptrace
[retty]: http://pasky.or.cz/~pasky/dev/retty/
[termios]: http://blog.nelhage.com/2010/01/a-brief-introduction-to-termios-signaling-and-job-control/
[injcode]: http://blog.habets.pp.se/2009/03/Moving-a-process-to-another-terminal
[neercs]: http://caca.zoy.org/wiki/neercs
