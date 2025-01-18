---
author: nelhage
author_email: nelhage@mit.edu
author_login: nelhage
author_url: http://nelhage.com
date: 2010-01-11T01:42:52Z
status: publish
tags:
- termios
- unix
- tty
- signals
- shell
title: 'A Brief Introduction to termios: Signaling and Job Control'
url: /2010/01/a-brief-introduction-to-termios-signaling-and-job-control/
wordpress_id: 60
wordpress_url: http://blog.nelhage.com/?p=60
---

(This is part three of a multi-part introduction to termios and
terminal emulation on UNIX. Read [part 1][part-1] or [part 2][part-2]
if you're new here)

[part-1]: /2009/12/a-brief-introduction-to-termios
[part-2]: /2009/12/a-brief-introduction-to-termios-termios3-and-stty/

For my final entry on termios, I will be looking at job control in the
shell (i.e. backgrounding and foreground jobs) and the very closely
related topic of signal generation by termios, in response to `INTR`
and friends.

## Sessions and Process Groups

For the purposes of termios, processes are organized into two
hierarchical groups, **process groups** and **sessions**. Every
process belongs to exactly one process group and one session, and each
process group is contained entirely within a session.

Process groups and sessions are both named by the process ID of the
process to create the group. This process is known as the **process
group leader** or **session leader**. A process creates a new session
using `setsid(2)`, or a new process group using `setpgid(2)`.

On Linux, you can inspect the process group and session of a process
using the `stat` field in `/proc/$PID`. The first several fields in
that file are:

        pid (name) state ppid pgid sid …

or, in more words:

        [process id] ([name]) [state] [parent process id] [process group id] [session id] …

### Sessions

Sessions are the fundamental group of terminal management. Every
session may have an associated **controlling terminal**, which is
treated specially. A process may open and talk to any number of
terminals, but the special behaviors related to access control and job
control only apply to a process's controlling terminal. Each terminal
may be the controlling terminal of at most one session. It follows
that calling `setsid(2)` to create a new session causes a process to
lose its previous controlling terminal. Acquiring a controlling
terminal is OS-specific, but can usually be accomplished by opening a
terminal device without the `O_NOCTTY` flag, while not already having
a controlling terminal.

Generally, all your processes within a single login session, or within
a single instance of your terminal emulator, are within the same
session, with the `pty` allocated by your terminal emulator or `ssh`
or whatever as their controlling terminal.

### Process Groups

Process groups are the unit of control for signal generation by a
terminal. A terminal never sends a signal to a specific process, but
always to all processes within a process group.

Access to a terminal is also mediated in terms of process groups. In
addition to having an associated session, every terminal has exactly
one **foreground process group**. Every other process in that
terminal's session is a **background process group**.

The foreground process group is awarded special access to its
controlling terminal. It is allowed unrestricted access to read from
and write to the terminal, as well as to call various control
functions, such as `tcsetattr` on it.

In addition, if any signal-generating character is read by a terminal,
it generates the appropriate signal to the foreground process group.

Background process groups are restricted in their access to their
controlling terminal. If any process in a background process group
attempts to read from its controlling terminal, it will result in
`SIGTTIN` being sent to its process group. Background processes may
write to their controlling terminal, unless `TOSTOP` is set in
`c_lflag`, in which case doing so will generate `SIGTTOU` to its
process group. Calling terminal control functions such as `tcsetattr`
is treated like a write operation with `TOSTOP` set (i.e. `SIGTTOU` is
sent unless the process is blocking or ignoring it).

The foreground process group for a terminal may be set by the
`tcsetpgrp(3)` function, which may be called by any process in the
terminal's session, but is treated in the same way as `tcsetattr` in
the previous paragraph.

## Job control

We've now got most of what we need to understand job control in your
shell.

When processing each command line, the shell uses `setpgid` to place
all of the programs executed by the line into the same process group,
and then calls `tcsetpgrp` to make that job the foreground job, and
does a `waitpid` to wait on that process.

Thus, when you run a shell pipeline (`foo | bar | grep baz`), all the
programs in the pipeline are in the same process group, and in the
foreground, which is why a `^C` kills all of them.

When you `^Z` a jobs, all the processes in the process group are
stopped, and the shell's `waitpid` returns, informing it of the status
change. The shell restores itself to the foreground process group, and
marks the job as backgrounded.

When you use `bg` to background a stopped job, the shell just uses
`killpg(2)` to `SIGCONT` the group. If a job in the background tries
to read from the terminal, the `SIGTTIN` stops it and the shell's
`wait` detects the state change and adjusts the job's state
appropriately.

If you launch a job in the background, the shell simply doesn't
`tcsetpgrp` it into the foreground, nor `wait` on it.

# In conclusion

That's probably all I want to say about termios. I could talk more about terminal emulation, ncurses, `$TERM` and friends, but it's less interesting to me -- I think I'm a kernel hacker at heart, and that stuff is just userspace programs talking to each other at this point. I hope you found this series interesting and/or informative, and I'm always happy to answer questions.
