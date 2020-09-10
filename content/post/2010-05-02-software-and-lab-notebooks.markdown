---
author: nelhage
author_email: nelhage@mit.edu
author_login: nelhage
author_url: http://nelhage.com
date: 2010-05-02T23:14:14Z
published: true
status: publish
tags:
- software
- lab notebooks
- engineering
- debugging
title: Software Engineers should keep lab notebooks
url: /2010/05/software-and-lab-notebooks/
wordpress_id: 217
wordpress_url: http://blog.nelhage.com/?p=217
---

<div id="text-1">

<p>
Software engineers, as a rule, suck at writing
things down. Part of this is training &ndash; unlike chemists and
biologists who are trailed to obsessively document everything they do
in their lab notebooks, computer scientists are taught to document the
end results of their work, but aren't, in general, taught to take
notes as they go, and document the steps they take in building a
system. 6.005, MIT's new introductory software engineering class,
attempted to require its students to keep lab notebooks for a few
semesters, and was met with near-universal complaints and ridicule
from the students (“Lab notebooks? For a software engineering class?
What the hell?”). (To be fair, I suspect they did a horrible job of
it, but I'm not sure that students would have been any less confused
at the idea).
</p>
<p>
Part of the reason is probably also the nature of software that makes
it very easy to record certain things as part of our tools, and that
makes experiments cheaply reproducible. Version control lets us
document the process of developing a piece of code, so it feels
superfluous to be taking additional notes on the side. Computers are
mostly deterministic, and cycles are cheap, so why bother meticuously
recording the results of a test run somewhere when you can just run it
again later, any time you want? Computers feel much neater and simpler
than messy bio or chem labs, and software is much simpler than
complicated biology experiments or chemical syntheses, and so no one
feels the need to be nearly as careful.
</p>
<p>
However, I am increasingly of the opinion that most software
engineers' total inability to work in a lab-notebook style, where you
meticulously document your work, is unfortunate and often seriously
detrimental to their work. While it's true that things like commit
logs do a good job of documenting certain processes, here are some
types of situations where I've found working with meticulous notes at
every step can be invaluable:
</p>

<h2>Debugging subtle problems</h2>
Debugging is very much a problem of
gathering data and making and testing hypotheses. For subtle bugs
in large programs, the amount of state you need to keep track of
can rapidly get out of hand. And good luck when a bug is tricky
enough that your debugging gets spread across multiple days, or
even across a lunch break.

<p>
If you've ever found yourself wondering "Wait &ndash; did I see the
bug after I made $CHANGE to my code or test environment?", you
should have been writing more things down.
</p>
<p>
This is especially important for non-deterministic bugs, such as
rare race conditions. If it takes you half a hour on average to
reproduce a bug, and you are experimenting with a dozen different
variables in your test environment that might affect the bug, you
can't afford to forget the results of a single test, or to forget
a single detail of what you did to test. This is the point at
which you should be writing down every single command you type in
any relevant prompts, and every single code change (or, since we
have technology, obsessively saving the output of `history`,
making commits to test branches, and recording the correlation
between them).
</p>
<h2>Profiling and optimization</h2>
This is a process similar in many ways
to debugging, but even more data-driven. When you're done with a
session of optimizing a piece of code or a system, if you can't
show documented evidence of exactly how much faster you've made
it, where that speedup came from, and all the things you tried
and how much they helped, perhaps you should be writing more
things down. And if you (or ideally, even someone else) can't go
back and reproduce the experiments you did, with approximately
the same results, you probably haven't been documenting your work
well enough.

<p>
Even if you're happy with the performance improvements you've
made, you may need to come back and wring even more performance
out of the system, and it'd be nice not to start from scratch. Or
maybe future testing will reveal that or more of your
optimizations was invalid, and you need to go back and consider
alternate options.
</p>
<p>
This is critically important when you're optimizing not just a
piece of code, but some kind of system with lots of configuration
and setup, that you'll later have to duplicate somewhere else,
instead of just checking the result into source control.
</p>
<h2>Understanding a new project's code or documentation</h2>
Whenever I'm
first diving into a large code-base or first playing with a large
new API, I find it invaluable to take notes as I go about what I
look for and where I find it. I'll often need to look up half a
dozen different API calls or pieces of code to understand
something, often too many to keep in my head as I go and dive
through more and more pieces of code or docs.

<p>
And when the documentation is ambiguous, I'll often drop into a
REPL or build test programs to make various calls and understand
what happens. Again, after more than two or three of these, it's
vital that I've been writing down my findings.
</p>
<p>
This is one example where a chronological style documenting
exactly in what order I found things is less critical, but that
detailed notes as I go are still vital.
</p>
<h2>Designing things</h2>
Whenever you're designing something &ndash; be it an
API, a protocol, an interface, some kind of system, or something
else &ndash; it's worth taking notes on the process you took to get to
your final decisions, and the choices you considered and
rejected, and why.

<p>
You'll presumably end up a producing a piece of code or a design
document that indicates what you ended up deciding, but
understanding why you made the decisions you made is often
important to understanding how your system is supposed to work,
and how to best use or extend it in the future. Hopefully, when
you're done, you'd do this writeup in brief somewhere anyways,
but the best way to make sure you don't forget is to take good
notes as your thought process happens.
</p>
<p>
And nothing in software is ever complete. If you have to revise
the design for some reason, because someone points out problems
or new requirements come up, you'll probably want to remember the
other possibilities you came up with &ndash; maybe one of them is now
more right.
</p>

<h2>Give it a try!</h2>

<p>So, if you're a software engineer, I strongly encourage you to try to
get better at writing things down. In a future post, I'll hopefully
write up the techniques I've started using to take notes as I code,
debug, and design, but in the meanwhile, I encourage you to just grab
a text editor or a physical lab notebook, whichever is more
comfortable for you, and start taking more notes on what you're doing.
</p></div>
