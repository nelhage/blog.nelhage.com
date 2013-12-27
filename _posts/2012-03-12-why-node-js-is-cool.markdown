---
layout: post
status: publish
published: true
title: Why node.js is cool (it's not about performance)
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 484
wordpress_url: http://blog.nelhage.com/?p=484
date: 2012-03-12 11:36:35.000000000 +01:00
categories:
- Software Engineering
tags:
- software
- node.js
- networking
- composability
---
For the past N months, it seems like there is no new technology stack
that is either hotter or more controversial than node.js. [node.js is
cancer](http:&#47;&#47;teddziuba.com&#47;2011&#47;10&#47;node-js-is-cancer.html)!
[node.js cures
cancer](http:&#47;&#47;blog.brianbeck.com&#47;post&#47;10967024222&#47;node-js-cures-cancer)!
[node.js is bad ass rock star
tech](http:&#47;&#47;www.youtube.com&#47;watch?v=bzkRVzciAZg)!. I myself have
given node.js a lot of shit, often involving the phrase "explicit
continuation-passing style."

Most of the arguments I've seen seem to center around whether node.js
is "scalable" or high-performance, and the relative merits of
single-threaded event loops versus threading for scaling out, or other
such noise. Or how to best write a Fibonacci server in node.js (wat?).

I am going to completely ignore all of that (and I think you should,
too!), and argue that node.js is in fact on to something really cool,
and is worth using and thinking about, but for a reason that has
absolutely nothing to do with scalability or performance.

The Problem
-----------

node.js is cool because it solves a problem shared by virtually every
mainstream language. That problem is the fact that, as long as
"ordinary" blocking code is the default, it is difficult and unnatural
to write networked code in a way that it can be combined with other
network code, while allowing them to interact.

In most languages&#47;environments -- virtually every other language
people use today -- when you write networked code, you can either make
it fully blocking itself, and implement your own main loop -- which is
almost always easiest -- or you can pick your favorite event loop
library (you probably have half a dozen choices), and write your code
around that. If you do the latter, not only will your code likely be
more awkward than if you chose the blocking main loop approach, but
your only reward for the effort is that your code is combinable with
the small fraction of other code that also chose the same event loop
library.

The upshot of this situation is that if you pick a couple of random
networked libraries written by different people -- let's say, an HTTP
server, a Twitter client, and an IRC client, for example -- and want
to combine them -- maybe you want a Twitter <-> IRC bridge with a
web-based admin panel -- you will end up at best having to write some
awkward glue code, and at worst doing something truly hackish in order
to make them communicate at all.

(In case you're not convinced about the existence or scope of this
problem, you can detour ahead to the optional [example](#example) of
how this problem manifests with typical Python libraries)

Enter node.js
-------------

node.js solves this problem, somewhat paradoxically by reducing the
number of options available to developers. By having a built-in event
loop, and by making that event loop the default way to accomplish
virtually anything at all (e.g. all of the builtin IO functions work
asynchronously on the same event loop), node.js provides a very strong
pressure to write networked code in a certain way. The upshot of this
pressure is that, since essentially every node.js library works this
way, you can pick and choose arbitrary node.js libraries and combine
them in the same program, without even having to think about the fact
that you're doing so.

node.js is cool, then, not for any performance or scalability reasons,
but because **it makes composable networked components the default**.

Concluding Notes
----------------

An interesting point here is that there is not really any fundamental
differences between what you **can** do in Python and node.js. The
Twisted project, for example, is basically an attempt to implement the
node.js ideology in Python (yes, I'm being terribly anachronistic
describing it that way). Twisted, however, has a fairly steep learning
curve, and "feels" unnatural to developers used to writing "normal"
Python, and so relatively few libraries get written for the Twisted
environment, compared to the rest of the Python ecosystem. Twisted
suffers from the fact that the Python language and Python community
are not set up to make Twisted the default way to do things.

The key is that node.js makes it, both technically and socially,
**easier** to write code in this composable way than not to do so. The
built-in event loop and nonblocking primitives make it technically
easy, and the social culture that has grown up around it discourages
libraries that don't work this way, so libraries that attempt to just
block for IO are looked down on and are unlikely to thrive and gain
adoption and development resources.

I'm also not ignoring the downside of the node.js style -- the
potentially convoluted callback-based style, the risk of bringing the
whole world to a halt with an accidental blocking call, the
single-threaded model that makes it hard to effectively exploit
multiple cores. node.js definitely makes you do more work than you
might otherwise have to, in many circumstances. But the key is, in
exchange for this work, you **get** something really cool -- and
something much more valuable, in my opinion, than nebulous performance
gains.

I also don't want to claim that node.js is the only system, or the
only technical approach, that makes this property possible. But
node.js is the most successful such system I know of, and that's worth
at least as much as the technical possibility -- what's the use of
being able to combine third-party libraries, if no one has written
anything worth using in the first place?

And similarly, while the single-threaded callback model may not be the
best possible model, it seems to have hit some sweet spot for finding
a sweet spot in terms of what developers are willing to put up
with. Certainly, people are writing node.js code like mad -- check out
the [npm registry][npm] for a partial list.

[npm]: http:&#47;&#47;search.npmjs.org&#47;


So, node.js is not magic, and it definitely doesn't cure cancer. But
there is something here worth looking at. The next time you need to
glue some unrelated networked services together, give node.js a shot
-- I think you'll like it. And if you're still not convinced, glue a
quick HTTP frontend onto whatever you've created. I promise you'll be
shocked by how easy it is.

<a name="example"><&#47;a>
Postscript - A Python Example
-----------------------------

As an optional addendum, here's a step-by-step discussion of how just
bad the situation is in most other languages.

Let's imagine we want to write a trivial Jabber -> IRC bridge: A bot
that lurks in an IRC channel, and signs on to Jabber, and sends all
the messages it receives on Jabber into the IRC channel. This is the
kind of simple problem that can be described in one sentence, and
sounds like it should take all of 20 lines of code, but actually turns
out to be rather a nuisance.

Python has, by this point, a great library ecosystem, so we happily
start googling, and find the plausible looking [python-irclib][pyirc]
and [xmpppy][xmpppy] libraries. Great. So, what does code in each of
those look like? Well, in python-irclib, we construct a subclass of
`SingleServerIRCBot`, and call the `start` function, which runs the
IRC main loop:

    bot = MyBot(channel, nickname, server, port)
    bot.start()

[pyirc]: http:&#47;&#47;python-irclib.sourceforge.net&#47;
[xmpppy]: http:&#47;&#47;xmpppy.sourceforge.net&#47;

And in xmpppy, we construct an `xmpp.Client` object, and call
`Client.Process` in a loop, with a timeout:

    conn = xmpp.Client(server)
    # connect to the server
    while True:
      conn.Process(1)

Ok, so, we launch a thread for each one, and a few minutes of fumbling
later, we're connected to both Jabber and IRC. So far, so good. We're
using Python's threads, which will inevitably bring us a world of
pain, but I'll ignore that for now, since a better threading
implementation could fix most of the pain.

But now, what do we do when we receive a Jabber message? We want to
send a message out the python-irclib instance, but how do we do that?
python-irclib isn't thread safe, so we can't just call `.send()` from
the Jabber thread. Ok, so we add in a `Queue.Queue`, and have the
Jabber thread push messages onto it.

Now we just need to make the IRC thread fetch messages from this
queue. But how do we do that? The IRC thread is blocked somewhere deep
inside `python-irclib`, waiting for network traffic. How do we wake it
up to read messages from the queue? The easiest way is to switch from
calling `start` to calling `process_once` in a loop with a short
timeout.

This will work, and we'll eventually get something working, but now
we're forced into polling, with all the annoying latency&#47;CPU tradeoffs
that entails, and also half of our code so far has just been spent
gluing these two libraries together.

In node.js, on the other hand, we'd just instantiate client objects
for both protocols, set up some event handlers, and ... well, that's
about it. Because everything's hooked into the same main loop, they'll
Just Work together, and because we're all running single-threaded, we
can mostly just communicate directly between the two libraries without
having to think too hard about race conditions or anything.

The point, of course, is not that this is impossible to write this
program in Python. It is, and I've done it, and it's not **that**
terrible. But any option you take will involve some annoying
tradeoffs, and will involve making lots of irrelevant plumbing
decisions about how to make your pieces play well together. And
compared to all that, node.js feels like a breeze.
