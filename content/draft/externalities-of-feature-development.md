---
title: "Software organizations and supporting change"
slug: externalities-of-feature-development
date: 2018-03-18T13:39:01-07:00
---

# On change and blame

In a large software system with any substantial userbase, the hardest
part of making changes very often is avoiding regressions. Adding
purely new functionality is usually manageable; Modifying existing
functionality -- or refactoring an internal abstraction or data model
-- is fraught with the possibility of inadvertently breaking an
existing user or downstream system.

[Hyrum's Law][hyrum] is one statement of this problem; This
only-somewhat-tongue-in-cheek law states

> With a sufficient number of users of an API,
> it does not matter what you promise in the contract,
> all observable behaviors of your system
> will be depended on by somebody.

[XKCD 1172][xkcd] makes essentially the same observation about
end-user-visible behavior and changes.

Looking beyond the merely technical, enhancing the challenge is the
human tendency to blame any problem on whichever component most
recently changed -- what Adam Langley refers to as the
["Law of the Internet"][agl].  Anyone making a change to a functioning
system thus risks not only breaking behavior, but being directly
blamed for that breakage, no matter the nature of the breakage.

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
responsible for the ongoing running and operational environment of
someone else's code -- are so frustrating.

# Building systems for change

If we desire to build and maintain software systems over extended
periods of time, it is extremely valuable to preserve our ability over
time to make changes to any part of the system. In order to preserve
that ability, we need to push back against the above one-two punch
that serves to ossify core components of our system.

The formulation of Hyrum's Law contains within in a hint for how to
cope with it, one that is well-understood by most developers who have
maintained APIs long enough: Strive -- as far as possible -- to
enforce all specified behavior in the observable behavior of the
code. Library authors must explicitly check all assumptions on valid
input, explicitly randomize any values or orders that are left
unspecified, and so. [GREASE][grease] is one of many attempts by Adam
Langley's team to apply this design principle to TLS, in order to
preserve the ability to continue modifying TLS into the future.

However, it's also possible, in certain contexts, to push back against
the Law of the Internet. And, I believe, this is ultimately critical
to maintaining a healthy software-development organization.

# Internalizing the externalities of feature development

Every feature that is written has (presumably) value to
someone. However, it also has costs: it represents a behavior that
must be preserved into the future, and thereby imposes (explicitly or
otherwise) requirements on the system around it (the kinds of
requirements of which Hyrum's Law is made).

If the Law of the Internet is true, these costs represent _negative
externalities_ of feature development: A feature's author bears the
work of writing the feature, but some fraction of the future
maintainence burden for this feature falls on others -- infrastructure
developers, developers of shared libraries or tooling, developers of
future features, operators, and so on. From development onward, any
participant in the sytem bears a tacit responsibility for every
behavior already present in the system.

As any econ 101 student will tell you, negative externalities lead to
distorted incentives, conflicts between actors, and generally
unhappiness all around.

Our problem, therefore, is to _internalize the long-term externalities
of code_. We must find ways to make feature developers responsible for
the correctness of their feature indefinitely into the future, instead
of foisting that work off on others.

In essence, we should aspire to build software-engineering cultures
and organizations where the developer or owner of a feature is
responsible for that feature _even in the ongoing presence of others'
changes to the system_.

# How do we do this?

On the surface, this challenge appears hard or nearly
impossible. Responsibility appears to flow backwards in time: How can
a developer of a feature anticipate and take responsibility for every
future change to the system that might impact their component?

However, while there are [no silver bullets][silver], it's still
possible to make progress on this goal. And, I think, a large number
of developments in software engineering practice can be seen as
efforts to pursue this goal.

Note that most of my thoughts here are in the context of a corporate
development environment, where the pool of developers is relatively
stable and all participants share tooling and infrastructure and
communication channels. I believe that similar principles apply in
open-source world and on the broader internet, but the details of the
practices may differ.

## Testing

Writing tests for features is, among other things, a way of taking on
responsibility for a feature's future as part of developing the
feature. More so than just being about testing a feature for
correctness, tests are safety rails for yourself and for any future
participant in the system, delineating behaviors that must be
preserved.

I've never worked at Google, but I understand Google's
["Beyoncé Rule"][beyonce] ("If you liked it, than you should have put
a test on it") as, in part, an articulation of this principle: If you
write a feature, but don't write tests for it, than you and only you
are responsible if the feature regresses due to changes on the part of
its dependencies or elsewhere in the system.

Testing discipline can also be a form of anti-entropy against Hyrum's
Law. If users depend on unspecified or unintentional features of a
library or system, but also write tests that illustrate those
dependencies, then, at a minimum, developers wishing to make changes
can run those tests themselves and learn about unexpected assumptions,
and react accordingly to minimize disruption.

## DevOps

"devops" has become such a buzzword that it's really hard to figure
out what it means to most people. However, it seems clear to me that a
huge part of the de-siloing of operations and development is about
encouraging development teams to internalize the operational costs of
their system. Instead of building a system and letting someone else
run it -- who is then very constrained in their ability to make
change, lest they break the black box they've been handed -- we
co-locate more of that responsibility.

Furthermore, many specific devops practices -- such as the use of
automation, and good monitoring practices -- are also clearly in line
with a philosophy of a development team internalize a greater share of
responsibility for a feature's indefinite future development.

## Blameless Postmortems

The philosophy of blameless postmortems is, in part, a clear pushback
against zealous application of "whoever made the change take the
blame". Instead of assigning responsibility to whichever actor's
actions most proximally caused an incident, we should step back and
ask how the _system_ could have been more robust and prevented that
incident. In the long run, this means building a system that is
fundamentally more resilient to change; of necessity, a lot of the
work of making particular systems or features more robust and
resilient to change will fall on the owners and developers of that
feature, further helping to internalize the costs of maintenance.



[agl]: https://www.imperialviolet.org/2016/05/16/agility.html
[grease]: https://tools.ietf.org/html/draft-davidben-tls-grease-01
[silver]: https://en.wikipedia.org/wiki/No_Silver_Bullet
[hyrum]: http://www.hyrumslaw.com/
[xkcd]: https://xkcd.com/1172/
[delete]: https://programmingisterrible.com/post/139222674273/write-code-that-is-easy-to-delete-not-easy-to
[beyonce]: https://github.com/cpp-testing/GUnit#testing
[blameless]: https://landing.google.com/sre/book/chapters/postmortem-culture.html
