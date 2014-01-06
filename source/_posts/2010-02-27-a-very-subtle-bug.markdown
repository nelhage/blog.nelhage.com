---
layout: post
status: publish
published: true
title: A Very Subtle Bug
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 150
wordpress_url: http://blog.nelhage.com/?p=150
date: 2010-02-27 23:48:47.000000000 +01:00
tags:
- unix
- python
- signals
- tar
- gzip
- pipes
---
[6.033][1], MIT's class on computer systems, has as one of its
catchphrases, "Complex systems fail for complex reasons". As a class
about designing and building complex systems, it's a reminder that
failure modes are subtle and often involve strange interactions
between multiple parts of a system. In my own experience, I've
concluded that they're often wrong. I like to say that complex systems
don't usually fail for complex reasons, but for the [simplest, dumbest
possible reasons][2] -- there are just more available dumb reasons. But sometimes,
complex systems do fail for complex reasons, and tracking them down
does require understanding across many of the different layers of
abstraction we've built up. This is a story of such a bug.

The following code snippet in Python is intended to extract and return
a single file from a tarball. It probably should be using
[`tarfile`][3], but let's ignore that for the moment.

```python
import subprocess
def extractFile(tarball, path):
  p = subprocess.Popen(['tar', '-xzOf', tarball, path],
                       stdout=subprocess.PIPE,
                       stderr=subprocess.PIPE)
  contents, err = p.communicate()
  if p.returncode:
    raise SomethingWentWrong(err)
  return contents
```

This code has a bug. It will often work just fine, but occasionally it
will fail, with 'err' containing a message including `gzip: stdout:
Broken pipe`. If, however you were to write the equivalent code in a
shell script:

```shell
contents="$(tar -xzOf "$tarball" "$path")"
```

you would find that it never fails in this way. So what's going on?

When we launch `tar` with the `-z` option, it creates a `pipe(7)` and
forks off a `gzip` process writing into one end of the pipe. It then
reads the uncompressed tarball from the other end of the pipe, parsing
our `tar` headers, until eventually it's done, and it closes the read
end of the pipe.

Since we've given `tar` a specific path to extract, it doesn't need to
read the entire tarball -- only up until it can find that file. So it
may close the pipe before reading the entire file, and,
correspondingly, before `gzip` is done writing to it. As explained in
`pipe(7)`:

<blockquote>
If all file descriptors referring to the read end of a pipe have been
closed, then a write(2) will cause a SIGPIPE signal to be generated
for the calling process.  If the calling process is ignoring this
signal, then write(2) fails with the error EPIPE.
</blockquote>

Under normal circumstances, `gzip` expects that whoever is downstream
of it may only care about a prefix of the uncompressed stream, and so
it registers a `SIGPIPE` handler which exits cleanly.

Python, however, doesn't want to get `SIGPIPE`s. Instead, Python would
rather just check the return value of every `write` call it makes, and
raise an `IOError` if necessary, so that Python code gets the error in
an appropriately Pythonic way, instead of through an asynchronous
signal. And so, at startup, Python uses `signal(2)` or `sigaction(2)`
to ignore `SIGPIPE` by setting it to `SIG_IGN`.

As explained in `sigaction(2)`:
<blockquote>
A child created via fork(2) inherits a copy of its parent's signal dispositions.   During  an  execve(2),  the  dispositions of handled signals are reset to the default; the dispositions of ignored signals are left unchanged.
</blockquote>

 And so, when started from Python, `gzip` starts up with
`SIGPIPE` ignored. And, for reasons I don't understand, rather than
unconditionally handling `SIGPIPE`, `gzip` first checks whether or
not it's ignored, and only installs a handler if the signal is not
being ignored.

And so, `SIGPIPE` continues to be ignored, which means that `gzip`'s
`write(2)` returns `EPIPE`, which gzip sees is nonzero, calls `perror`
on, and then exits. tar's `wait` then sees `gzip` exit uncleanly,
which causes tar itself to exit uncleanly, which Python then raises as
an exception.

There's an easy workaround, which is to re-enable SIGPIPE in the
`subprocess` child:

```python
p = subprocess.Popen(['tar', '-xzOf', tarball, path],
                     stdout=subprocess.PIPE,
                     stderr=subprocess.PIPE,
                     preexec_fn=lambda:
                      signal.signal(signal.SIGPIPE, signal.SIG_DFL))
```

But who would think of doing that, without first having seen this
horribly subtle chain of bug, and having to track down what went
wrong?

(Here's the [Python bug report][4], and many thanks to cjwatson for
posting his discovery of this class of bug on his [blog][5], which
greatly reduced the amount of time I would have had to spend tracking
this down)

[1]: http://web.mit.edu/6.033/www/
[2]: http://ebroder.net/2010/01/25/complex-systems-and-simple-failures/
[3]: http://docs.python.org/library/tarfile.html
[4]: http://bugs.python.org/issue1652
[5]: http://www.chiark.greenend.org.uk/ucgi/~cjwatson/blosxom/2009-07-02-python-sigpipe.html
