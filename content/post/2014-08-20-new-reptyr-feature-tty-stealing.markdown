---
categories: null
comments: true
date: 2014-08-20T08:41:34Z
published: true
title: 'New reptyr feature: TTY-stealing'
url: /2014/08/20/new-reptyr-feature-tty-stealing/
---

Ever since I wrote [reptyr][reptyr], I've been frustrated by a number
of issues in reptyr that I fundamentally didn't know how to solve
within the reptyr model. Most annoyingly, reptyr fundamentally only
worked on single processes, and [could not attach][issue24] processes
with children, making it useless in a large class of real-world
situations.

## TTY stealing

Recently, I merged an experimental reptyr feature that I call
"tty-stealing", which has the potential to fix all of these issues
(with some other disadvantages, which I'll discuss later).

To try it out, clone and build reptyr from [git master][master], and
attach a process using `reptyr -T PID`.

Unlike in the default mode, `reptyr -T` will attach the entire
terminal session of `PID`, not just the single process. For instance,
if you attach a `less` process and then hit `q`, you'll find yourself
sitting at *the same shell that launched `less`*, not the shell you
launched `reptyr` from. Exit that shell, and you'll be back at your
original session.

One unfortunate known issue: `reptyr -T` cannot attach sessions that
were started directly from an `ssh` session unless it is run as
root. Read on to learn why.

## How it works

In its [default mode][how] of operation, `reptyr` attaches (via
`ptrace(2)`) directly to the target process, and actually swaps out
the terminal file descriptors to point at a new terminal. This mode of
operation explains (mostly) why we can't attach process trees: We
would need to attach to every process individually, and coordinate the
switchover (there are actually even deeper reasons related to the tty
handling, but that's the basic story).

`reptyr -T` tackles the problem from the other end. When you're
running a terminal session on a modern machine, the terminal device
that your shell and commands run on is a "pseudo-terminal", a virtual
terminal device that acts essentially like a two-way pipe, with a
bunch of extra input-handling behavior. When your `bash` writes to
stdout, that data flows over the pty, and is read by the terminal
emulator (which might be `gnome-terminal`, or might be `tmux`, or
might be `ssh`), which then displays it to you in some way.

Instead of switching out terminals in the target process, `reptyr -T`
goes after the *other* end of the `pty`, the so-called
"master". `reptyr -T` attempts to discover the pid of the terminal
emulator, attaches to it via `ptrace(2)`, and finds the fd
corresponding to the master end of the target pty. It then opens a
UNIX socket, and uses [`SCM_RIGHTS`](http://linux.die.net/man/3/cmsg)
to send the file descriptor back to the `reptyr` process. `reptyr`
closes the master fd in the original terminal emulator (opening
`/dev/null` over it, to minimize disruption), and detaches from the
master.

`reptyr` then takes over the role of terminal emulator, reading from
the the master fd and copying output to its own terminal, and copying
input from its terminal back to the master fd. This has the net effect
that you appear to be connected directly to the target session.

Because this process does not touch the terminal session at all, it is
entirely transparent to the process(es) being attached, which and
works in a large number of circumstances where vanilla `reptyr` would
glitch.

## Downsides

Unfortunately, the new mode isn't quite flawless. `ptrace`ing the
terminal emulator instead of the target process brings a few new
problems.

The most notable one is that you need to be able to attach to the
emulator process at all. In the case of an ssh session, that means the
forked `sshd` child for your connection. `sshd` drops privileges to
match the authenticated user (via `setuid(2)` and friends), so the
emulator process *does* match the user ID of your user
account. However, Linux forbids users from `ptrace()`ing a process
that has undergone a UID transition via `setuid`, and so we can't
attach to that process to steal the master end of the pty.

(The reasoning for this restriction confused me at first, but is
perfectly sound: Even though `sshd` has dropped privileges, it might
still contain sensitive information in its heap, such as the machine's
ssh private keys. We can't be confident it's truly unprivileged (and
thus safe to sketch on) until it's called `execve` and cleared out all
interesting state).

## Try it out!

I've tested out `reptyr -T` a fair bit, but it hasn't gotten broad
testing or usage yet. I'd love your reports of what it does or doesn't
work on, for you! Try it, and drop me an email or open an issue!

[reptyr]: /2011/01/reptyr-attach-a-running-process-to-a-new-terminal/
[issue24]: https://github.com/nelhage/reptyr/issues/24
[master]: https://github.com/nelhage/reptyr/tree/master
[how]: /2011/02/changing-ctty/
