---
title: "Software organizations and supporting change"
slug: externalities-of-feature-development
date: 2018-03-18T13:39:01-07:00
---

# On change and blame

In a large software system with any substantial userbase, the hardest
part of making changes tends to be avoiding regressions. Adding purely
new functionality is usually manageable, but modifying existing
functionality -- or refactoring an internal abstraction -- is fraught
with the possibility of inadvertently breaking an existing user or
downstream system.

The difficulty is exacerbated by users' (both human and computer)
tendency to depend on every observable behavior of a system that they
use. One notable formulation of this tendency is [Hyrum's Law][hyrum],
which observes:

> With a sufficient number of users of an API,
> it does not matter what you promise in the contract,
> all observable behaviors of your system
> will be depended on by somebody.

In a very different medium, [XKCD 1172][xkcd] makes essentially the
same observation.

Beyond the merely technical, enhancing the challenge is the human
tendency to blame any problem on whichever component most recently
changed. Google's Adam Langley refers to as the
["Law of the Internet"][agl], but it can apply in systems of any
variety, including those much smaller than the public internet. No
matter what any specification document, contract, SLA, or unwritten
norm says, someone who makes a change to a functioning system that
results in breaking something tends to, as a default, unconditionally
take the blame.

In narrow programming terms, if you expose a public API, and a
downstream developer depends on an undocumented behavior, and you
later change that behavior, you tend to get blamed, instead of the
downstream developer who was -- in some sense -- technically in the
wrong.

This observation, of course, then chains with Hyrum's law, which
suggests that, in the limit, some user will depend on *any* observable
behavior in your API, specified or not. Thus, in a sufficiently large
system, any observable change at all will break some user -- and the
developer who made the change will take the blame.

The net result of these conjoined observations is to ossify software
systems over time; As they grow more users, and especially as they
grow more distinct downstream dependencies, they become harder and
harder to change. As they become harder to change, developers change
them less and less frequently, and downstream users become even more
accustomed to them never changing, and the cycle of ever-increasing
brittleness intensifies.

# Building systems for change

If we desire to build and maintain software systems over extended
periods of time, this ossification process is dangerous and costly. If
we can't change the core-most components of our systems, we risk
instead having to build increasingly complex workarounds as new needs
arise, or maintaining numerous parallel systems with subtly differnet
behavior, increasing the cost of development and change. If we want to
avoid this fate, we need to figure out how to resist this vicious
cycle.

The main strategy for coping with Hyrum's law are hinted at by the
very formulation of the law, and are well-known by most developers who
maintain APIs long enough: One must strive -- as far as possible -- to
make the "observable behavior" of the system identical with "what you
promise in the contract"; All assumptions on incoming data must be
strictly checked, and behavior on outputs must be either specified, or
explicitly randomized. Developers of standards on the public internet
have become experts at these techniques: As two examples,
[GREASE][grease] applies explicit randomization to TLS, in order to
shake out and prevent brittle clients, and [HTML 5][html5] fully
specifies parser behavior on *any* input string, not just valid ones,
making it harder to rely on undocumented behavior [^html].

[^html]: Note, of course, that this latter strategy brings "observable behavior" and "specified behavior" closer together, but at the cost of making change *harder*. For a standard where the goal is interoperability, and where evolution happens at other layers (e.g. the DOM), this is a good tradeoff, but it doesn't solve all problems.

On the public internet, these strategies may be our best option for
enabling change and encouraging interoperability. However, in other
contexts -- especially inside smaller, more closed organizations -- I
believe we can also usefully push back against the "Law of the
Internet", and encourage the development of living systems capable of
absorbing ongoing change through an additional set of
tools. Furthermore, I believe that organizations that manage do so
gain an incredible advantage in the long run.

# Internalizing the externalities of feature development

Every feature that is written has (hopefully) value to
someone. However, it also has costs: it represents a behavior that
users will come to rely on and thus must be preserved into the future,
and it imposes (explicitly or otherwise) requirements on the system
around it (of which Hyrum's Law is made).

In most sytems, many of these costs represent _negative externalities_
of feature development: A feature's author bears the work of writing
the feature, but some substantial fraction of the future maintainence
burden for this feature falls on others -- infrastructure developers,
developers of shared libraries or tooling, developers of future
features, operators, and so on. From development onward, any
participant in the sytem bears a tacit responsibility for every
behavior already present in the system.

As any econ 101 student will tell you, negative externalities lead to
distorted incentives, conflicts between actors, and generally
unhappiness all around.

As we saw earlier, in software systems, these externalities show up
most acutely (although not exclusively) when other participants in the
system attempt to make changes, and thus contribute to the ongoing
brittleness and ossification of systems depended on by features.

Our problem, therefore, is to _internalize the long-term externalities
of feature code_. To encourage systems capable of change, we should
find ways to make feature developers more responsible for the
correctness of their feature indefinitely into the future, instead of
forever foisting that work off onto downstream maintainers.

In essence, we should aspire to build software engineering cultures
and organizations where the developer or owner of a feature is
responsible for that feature _even in the ongoing presence of others'
changes to the system_.

# How do we do this?

On the surface, this challenge appears hard or
impossible. Responsibility appears to flow backwards in time: How can
a developer of a feature anticipate and take responsibility for every
future change to the system that might impact their component?

This task is hard, and there are [no silver bullets][silver], but it's
still possible to make progress on this goal. And, I think, a large
number of developments in software engineering practice can be
usefully understood as efforts in pursuit of this goal.

## Ownership



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
[html5]: https://developer.mozilla.org/en-US/docs/Web/Guide/HTML/HTML5
[silver]: https://en.wikipedia.org/wiki/No_Silver_Bullet
[hyrum]: http://www.hyrumslaw.com/
[xkcd]: https://xkcd.com/1172/
[delete]: https://programmingisterrible.com/post/139222674273/write-code-that-is-easy-to-delete-not-easy-to
[beyonce]: https://github.com/cpp-testing/GUnit#testing
[blameless]: https://landing.google.com/sre/book/chapters/postmortem-culture.html
