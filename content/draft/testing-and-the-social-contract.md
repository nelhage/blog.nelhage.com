---
title: "Testing and the Social Contract"
slug: testing-and-the-social-contract
date: 2018-03-18T13:39:01-07:00
---

# On change and blame

In a large software system with any substantial userbase, the hardest
part of making changes is, very often, avoiding regressions. Adding
purely new functionality is usually manageable; Modifying existing
functionality, or refactoring an internal abstraction or data model,
is fraught with the possibility of inadvertently breaking an existing
user or dependent system.

[Hyrum's Law][hyrum] is one statement of this problem; This
only-somewhat-tongue-in-cheek observation states

> With a sufficient number of users of an API,
> it does not matter what you promise in the contract,
> all observable behaviors of your system
> will be depended on by somebody.

[XKCD 1172][xkcd] makes essentially the same observation about
end-user-visible behavior and changes.

Looking beyond the merely technical, enhancing the challenge is the
human tendency to blame any problem on whichever component most
recently changed -- Adam Langley's ["Law of the Internet"][agl].
Anyone making a change to a functioning system thus risks not only
breaking behavior, but being directly blamed for that breakage, no
matter the nature of the breakage.

In terms of an API, it doesn't matter whether a consumer of your API
dependened on unspecified behavior (or even behavior specifically
specified to be subject to change); If you break the consumer because
you changed the behavior, users will be inclined to blame you, and not
the code that used to work.

Hyrum's law suggests that -- in the limit -- *any* change is a
breaking change to some user. And agl's Law of the Internet suggests
that any such change will be perceived as the fault of the actor
making the change.

These compounding effects go a long way to explainining why changing
software systems is hard, especially the deeper in the dependency
stack you go.

This is also a large part of why traditional "sysadmin" roles --
responsible for the ongoing running of someone else's code -- are so
frustrating.

# Building systems for change

If we desire to build software systems capable of weathering the ages,
it must be possible to update and evolve them with time. (Another
option is building systems that are [easy to delete][delete], and
continually be replacing systems in their entirety. I also endorse
this approach, but it's not the topic of this post).

We must, therefore, push back -- wherever possible -- on both halves
of the above observation.

Taking Hyrum's Law as true (and most developers agree it is, at least
in the limit) instructs us that we must aspire to make all specified
behavior apparent in the observable behavior of our code. Input
assumptions must be checked. Anything that is unspecified must be
explicitly randomized, and so on.

#





[agl]: https://www.imperialviolet.org/2016/05/16/agility.html
[hyrum]: http://www.hyrumslaw.com/
[xkcd]: https://xkcd.com/1172/
[delete]: https://programmingisterrible.com/post/139222674273/write-code-that-is-easy-to-delete-not-easy-to
