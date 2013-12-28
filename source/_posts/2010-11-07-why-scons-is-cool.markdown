---
layout: post
status: publish
published: true
title: Why scons is cool
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 394
wordpress_url: http://blog.nelhage.com/?p=394
date: 2010-11-07 18:00:38.000000000 +01:00
tags:
- unix
- C
- scons
- make
- build
---
I've recently started playing with `scons` a little for some small
personal projects. It's not perfect, but I've rapidly come to the
conclusion that it's a probably far better choice than `make` in many
cases. The main exceptions would be cases where you need to integrate
into legacy build systems, or if asking or expecting developers to
have `scons` installed is unreasonable for some reason.

The main reason that `scons` is cool to me, and the thing that makes
it fundamentally different from `make`, is the introduction of actual
scoping.

`make` has a single global scope. This is one of the main reasons that
people write recursive Makefiles; By giving you one file per
directory, you get one scope per directory, which makes it possible to
have per-directory pattern rules, variables, and all that other stuff,
without driving yourself insane.

`make`'s awful syntax, confusing varieties of variable,
whitespace-sensitivity, and all the other things that people love to
bitch about are annoying, but to my mind, the single scope that makes
recursive Makefiles the dominant (and, really, the only scalable)
paradigm is the one thing that really sucks.


`scons` solves this by baking various kinds of scoping into the
tool. `scons` lets you include sub-build-scripts (typically named
`SConscript`, by convention). Those scripts run in their own namespace
and can establish their own variables, rules, etc., but the end result
is then merged back into the global rule list (handling sub-directory
paths intelligently), so that the scheduler can work globally, instead
of having to recurse.

Furthermore, because of this explicit scoping, you can pass variables,
including targets, between build files, letting you explicitly set up
cross-directory dependencies or share `CFLAGS` or other variables,
making it easy for different directories to share exactly as much or
as little configuration as you want.

In addition, `scons` has the concept of "build environments", which
are objects that include build rules, variables, and so on. By
reifying what `make` just represents as the global environment into
objects, it makes it much easier to scope and program things. For
example, if you have a set of targets that should be built using the
global default rules, except with debugging enabled, you can do:

    myenv = env.Clone()
    myenv.Append(CFLAGS = ['-g'])
    myenv.Program(...)

By making it (optionally) explicit which sets of rules and variables
are being used in each place, it becomes much easier to share multiple
kinds of targets and rule sets in a single file, without necessitating
lots of sub-files just for scoping, like `make` tends to lead to.

`scons` is cool for a bunch of reasons. It eliminates most of the
stupid little annoyances you've probably had with `make`. But, in my
mind, this is the thing that makes it cool. They've added sane scoping
to the build tool, so that you can construct non-recursive build
systems without going insane.

I'll definitely be considering `scons` for any new projects I write going
forward. I hate `make`, and this definitely feels like a path forward.
