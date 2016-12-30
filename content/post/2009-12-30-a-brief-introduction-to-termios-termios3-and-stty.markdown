---
author: nelhage
author_email: nelhage@mit.edu
author_login: nelhage
author_url: http://nelhage.com
date: 2009-12-30T01:47:17Z
published: true
status: publish
tags:
- termios
- unix
- tty
- apis
title: 'A Brief Introduction to termios: termios(3) and stty'
url: /2009/12/a-brief-introduction-to-termios-termios3-and-stty/
wordpress_id: 27
wordpress_url: http://blog.nelhage.com/?p=27
---

(This is part two of a multi-part introduction to termios and terminal
emulation on UNIX. Read [part 1][part-1] if you're new here)

[part-1]: /2009/12/a-brief-introduction-to-termios

In this entry, we'll look at the interfaces that are used to control
the behavior of the "termios" box sitting between the master and
slave pty. The behaviors I described last time are fine if you have a
completely dumb program talking to the terminal, but if the program
over on the right is using curses (like emacs or vim), or even just
readline (like bash), it will want to disable or customize some of the
behaviors.

The primary programmatic interface to termios is the `struct termios`
and two functions:

       int tcgetattr(int fd, struct termios *termios_p);
       int tcsetattr(int fd, int optional_actions,
                     const struct termios *termios_p);

which retrieve and set the `struct termios` associated with a given terminal device.
They are all documented in `termios(3)` (If you're
unfamiliar with the convention, that means document `termios` in section
`3` of the unix `man` pages -- `man 3 termios` on a command-line will
get it for you).

So what's inside `struct termios`? POSIX
specifies that this structure contains at least the following fields:

           tcflag_t c_iflag;      /* input modes */
           tcflag_t c_oflag;      /* output modes */
           tcflag_t c_cflag;      /* control modes */
           tcflag_t c_lflag;      /* local modes */
           cc_t     c_cc[NCCS];   /* control chars */

Each "flag" field contains a number of flags (implemented as a bitmask) that can be individually enabled or disabled.
`c_iflag` and `c_oflag` contain flags that affect the processing
of input and output, respectively. `c_cflag` we will mostly ignore, as
it contains settings that relate to the control of modems and serial
lines that are mostly irrelevant these days. `c_lflag` is perhaps the most interesting of the flag values. It
contains flags that control the broad-scale behavior of the
`tty`. I'll look at just a few of the interesting bits in each:

### local modes

 * `ICANON` - Perhaps the most important bit in `c_lflag` is the `ICANON`
bit. Enabling it enables "canonical" mode -- also known as
"line editing" mode. When `ICANON` is set, the terminal buffers a line
at a time, and enables line editing. Without `ICANON`, input is made
available to programs immediately (this is also known as "cbreak"
mode).

 * `ECHO` in `c_lflag` controls whether input is immediately re-echoed as
output. It is independent of `ICANON`, although they are often turned on and off together. When `passwd` prompts for your password, your terminal is in canonical mode, but `ECHO` is disabled.

 * `ISIG` in `c_lflag` controls whether `^C` and `^Z` (and friends) generate signals or
not. When unset, they are passed directly through as characters,
without generating signals to the application.

### input and output modes

There are also a few flags in `c_iflag` and `c_oflag` worth mentioning.

 *  `IXON` in `c_iflag` enables the "flow control" mediated by
`^S` and `^Q` (by default). With `IXON`, once `^S` has been received
by the master pty, the slave will not accept any output (`write`s to it will hang) until `^Q` is received by the master pty.

 * `IUTF8` in `c_iflag` is an interesting hack. In canonical mode, backspace needs to
delete the previous character in the input buffer. In non-ASCII encodings, a single character may be several bytes long, but the terminal still only sees a byte stream, and has no explicit information about the encoding or character boundaries on either end.
`IUTF8` tells termios that the input stream is utf-8 encoded, which permits the
correct handling of backspace. If `IUTF8` is unset, and you enter a
multibyte character and then press backspace, only the final byte will
be deleted, leaving you with a corrupt utf-8 stream.

 * `OLCUC` in `c_oflag` "Map[s] lowercase characters to uppercase on
output." Just in CASE YOU NEED YOUR TERMINAL TO LOOK MORE LIKE
SHOUTING.

There are many more flags, controlling such details as newline translation and how character erase works. The full list is documented in `termios(3)`.

### c_cc

Next up is `c_cc`. This field sets the various control characters used to interact with the
terminal. Characters like `^C` and `^Z` and delete that have special meanings to termios are not hard coded anywhere, but rather defined via the `c_cc` array.

`c_cc` is indexed by various constants for the various control characters, and
the value at any index is the character that should have that effect. Some of the notable ones are:

 * `VINTR` -- Generate a `SIGINT` (`^C` by default).

 * `VSUSP` -- Generate a `SIGTSTP` (stop the program) (`^Z` by default).

 * `VERASE` -- Erase the previous character. This tends to be one of
    `^H` and `^?` (ASCII `0x7f`) by default -- if you've ever pressed
    "backspace" and been greeted by a `^H`, your terminal and your
    `struct termios` disagree on the value of `VERASE`.

 * `VEOF` -- End of file. Sends the current line to the program
   without waiting for end-of-line, or, as the first character on the line, causes the next `read` call by the slave to return return end-of-file. (`^D` by default)

 * `VSTOP` and `VSTART` -- `^S` and `^Q` by default, stop and start
   output.

Setting any of these to NULL (0) disables that special control character. Many of the `c_cc` elements are only relevant when certain modes are active -- `VINTR` and `VSUSP`, for instance, only matter if `ISIG` is enabled in `c_lflag`, and `VSTOP` and `VSTART` are ignored unless `IXON` is set.

(A brief note on the representation of control characters -- The characters `^A` through `^Z`, pronounced "Ctrl-FOO", are represented by the bytes with values 1 through 26. So when I say that `c_cc[VINTR]` is equal to `^C` by default,
that's actually just the number `3` -- your terminal took the keypresses and just translated them into the byte `3` on the wire.)

## stty

While `termios(3)` is the standard programmatic interface to control
termios, a much more convenient interface for experimentation is the
`stty` program, which is just a thin wrapper around `tcgetattr` and
`tcsetattr` designed to be usable from shell scripts or directly from the shell.

`stty` gets or sets options on a terminal device. By default, it operates on the one connected to its standard out, but you can pass it an arbitrary device using the `-F` option.

Without aguments, `stty` prints in what way its terminal's settings differ from an internal set of "sane"
defaults. `stty -a` causes it to print the value of every flag in the
`struct termios` in a human-readable format.

You can toggle flags using <tt>stty <i>flag</i></tt> to enable a flag, and <tt>stty
-<i>flag</i></tt> to disable it. So for instance, `stty -isig` will disable
signal generation -- run a program after doing this, and you'll find
yourself unable to `^C` it. In general it uses the same names as the C constants, except in lowercase, but check the man page if in doubt.

`stty` can also change the value of the control characters in
`c_cc`, using <tt>stty <i>symbolic-name</i> <i>character</i></tt>. If you wanted `^G` to be the interrupt character, instead of `^C`, a simple `stty intr ^G` would suffice. You can spell `0` as `undef` to disable a given control character. So, if you hate flow
control and want to totally disable it, you could try `stty -ixon stop
undef` -- disable `IXON`, and then also disable the `VSTOP`
character for good measure. (You might still be foiled by screen or some other layer doing its own flow control, unfortunately).

`stty`'s `-F` option can be great for peeking at what some other program is
doing to its terminal. If you run `tty` in a shell, it will print the
path to that shell's terminal device (usually of the form
<tt>/dev/pts/<i>N</i></tt>, at least under Linux). Now,
from a different shell, you can run <tt>stty -a -F /dev/pts/<i>N</i></tt> to see how
the first shell's terminal is configured.  You can then run programs
in the first shell, and repeat the `stty` incant in shell two to see
what settings are getting set. For example, if I run `stty -F /dev/pts/10` right now (while I have a `bash` talking to a `gnome-terminal` via that pty), I see:

    $ stty -F /dev/pts/10
    speed 38400 baud; line = 0;
    eol = M-^?; eol2 = M-^?; swtch = M-^?; lnext = <undef>; min = 1; time = 0;
    -icrnl iutf8
    -icanon -echo

So we can see that bash/readline has disabled CR→LF translation on input (`icrnl`), disabled canonical mode and echo, but turned on UTF-8 mode (because bash detected a utf-8 locale). Note that if I run `stty` directly in that shell, I see something slightly different:

    $ stty
    speed 38400 baud; line = 0;
    eol = M-^?; eol2 = M-^?; swtch = M-^?;
    iutf8

This is because bash maintains its own set of termios settings (for readline), and saves and restores settings around running programs, so that settings in place while running a program are different from those in place while you're typing at the bash prompt.

In the next post, we'll look at signal generation from `ISIG` and how it interacts with job control in your shell.

## Addendum: `ioctl(2)`

This last section is a brief aside, which has very little to do with termios specifically, so you should feel free to skip it. But read on if you're curious about some of the low-level details of how the APIs work.

If you're familiar with man page conventions, you may have noticed
that the `termios` functions are in man page section 3, which means
that they're provided by system libraries, and are not system
calls. But at the same time, I told you last time that termios is
implemented inside the kernel -- so how are the libraries talking to
the kernel, if not through syscalls?

The answer is a single odd little catch-all system call, known as
`ioctl`. Historically, one of the "big ideas" of UNIX was that
"everything is a file" -- you could communicate with devices just like
you could files, by opening files in `/dev/`. But a file on UNIX is also just a stream of bytes, without any
OS-imposed structure. And for a device, you may often need to send
out-of-band control data -- e.g. to set the baud and parity bit settings on a
serial port. And adding new system calls for every new device type would
be untenable for a number of reasons.

So the answer was one new new system call, `ioctl` (pronounced as any
of "I-O-cuddle", "I-octal" or "I-O-C-T-L"). `ioctl` is prototyped as:

    int ioctl(int fd, int request, ...);

It takes a file descriptor, a numeric "request" code, and an unspecified number of
other arguments. `ioctl` looks up whatever device (or file
system, network protcol, or whatever) is backing that file descriptor, and hands it the "request" and the arguments
to do with as they will.

So any device that needs extra control channels can define some `ioctl` numbers and parameters and document them somewhere, and they become the interface to control that device. So, for instance (at least on Linux), an `ioctl` on a tty device with  a "request" of `TCGETS` (defined in `termios.h`) takes a parameter that is a pointer to a `struct termios`, and copies the in-kernel settings for that tty to the provided struct. So somewhere in libc, `tcgetattr(fd, p)` is just defined to do an `ioctl(fd, TCGETS, p)`. Similar ioctls are defined for `tcsetattr` and all the functions in `termios(3)`. On Linux, at least, the morbidly curious can find out all the gory details in `tty_ioctl(4)`.
