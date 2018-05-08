---
title: "Software organizations and enabling change"
slug: software-evolution
date: 2018-03-18T13:39:01-07:00
---

# On change and blame

In a large software system with any appreciable userbase, the hardest
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

With a very different lens, [XKCD 1172][xkcd] makes essentially the
same observation.

Beyond the merely technical, enhancing the challenge is the human
tendency to blame any problem on whichever component most recently
changed. Google's Adam Langley refers to as the
["Law of the Internet"][agl], but it can apply in systems of any
variety, including those much smaller than the public internet. No
matter what any specification document, contract, SLA, or unwritten
norm says, an who makes a change to a functioning system that breaks a
flow someone else depend on tends to, as a strong default, take the
blame.

In narrow software engineering terms, if you expose a public API, and
a downstream developer depends on undocumented behavior, and you later
change that behavior, you tend to get blamed, instead of the
downstream developer who was -- in some sense -- technically in the
wrong.

This tendency combines in an unfortunate way with Hyrum's Law. Hyrum's
Law suggests that, in the limit, some user will depend on *any*
observable behavior in your API, specified or not. Thus, in a
sufficiently large system, any observable change at all will break
some user -- and the developer who made the change will take the
blame.

The net result of these conjoined observations is to ossify software
systems over time; As they grow more users, and especially as they
grow more distinct downstream dependencies, they become harder and
harder to change. As they become harder to change, developers change
them less and less frequently, and downstream users become even more
accustomed to them never changing, and the cycle of ever-increasing
brittleness intensifies.

# Building systems for change

If we desire to build and maintain software systems to withstand the
test of time, this ossification process is dangerous and costly. If
components -- past some age -- are effectively frozen in time, we can
never directly fix past mistakes, or retool systems for new
challenges; We're forced instead to craft layers of workarounds, or
build entirely new parallel systems, thereby increasing total
complexity and the cost of development. If we want to avoid this fate,
we need to figure out how to resist this vicious cycle.

The main strategy for coping with Hyrum's law is hinted at by the very
formulation of the law, and is well-known by most developers who
maintain APIs long enough: One must strive to make the "observable
behavior" of the system as identical as possible to "what you promise
in the contract". For example, any assumptions on incoming data must
be strictly checked, and all behavior on outputs must be either
specified or explicitly randomized. Developers of standards on the
public internet have become experts at these techniques:
[GREASE][grease] applies explicit randomization to TLS in order to
shake out and prevent brittle clients, and [HTML 5][html5] fully
specifies parser behavior on *any* input string, not just valid ones,
drastically reducing the space of undocumented behavior [^html].

[^html]: Note, of course, that this latter strategy brings "observable behavior" and "specified behavior" closer together, but at the cost of making change *harder*. For a standard where the goal is interoperability, and where evolution happens at other layers (e.g. the DOM), this is a good tradeoff, but it doesn't solve all problems.

On the public internet, these strategies may be our best option for
enabling change and encouraging interoperability. However, in more
controlled contexts, such as within single software-development
organizations -- even fairly large ones -- I believe there's scope for
building a culture and practices that support change and evolution
through other tools, largely through pushing back against the mindset
of the "Law of the Internet".

Furthermore, I believe that organizations that manage do so gain an
incredible advantage in the long run, since they will be able to
manage technical debt and evolve their infrastructure and systems to
face new needs and challenges with much greater agility than those
that don't.

# Internalizing the externalities of feature development

Every feature that is written has (hopefully) value to
someone. However, it also has costs: it represents a behavior that
users will rely on and which must be preserved into the future, and it
imposes (explicitly or otherwise) requirements on the system around it
on which it depends.

These costs represent _negative externalities_ of feature development:
A feature's author (individual or team) bears the work of writing the
feature, but some substantial fraction of the future maintainence
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

Part of our problem, therefore, is to _internalize the long-term
externalities of feature code_. To encourage systems capable of
change, we should find ways to make feature developers more
responsible for the correctness of their feature indefinitely into the
future, instead of forever foisting that work off onto downstream
maintainers.

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

The first component is to build a culture of ownership over features
that extends throughout their entire lifecycle. It's not uncommon
that, once a feature is "done", or "stable", the development team
stops feeling responsibility for it; over time, engineers leave the
team or company, teams are reorganized, priorities shift, expertise is
lost, and eventually no one really understands how the feature ever
worked or how it was implemented, and the feature becomes pure
externality; Users still rely on it, so it must be maintained, but the
work of maintenance falls entirely on downstream teams, often with
"operations" somewhere in their name.

A culture of ownership means that a team takes responsibility for
their code and systems indefinitely into the future. They take
responsibility for how it is behaving in production. They take
responsibility for reviewing necessary changes as time moves on. They
train new hires on the feature so they are equipped to answer
questions or do necessary maintenance. They take responsibility for,
or partner in, necessary upgrades to move the code to new versions of
internal services or libraries where necessary. If team structures
shift, they make sure some team ends up with ownership, and agrees to
continue taking on all of these duties.

This need not mean the team is an island; Taking ownership for
behavior in production or upgrades doesn't mean you need to do 100% of
the work; but you need to partner with those that do, support them
with your expertise and context, and ultimately take responsibility
for the fate of the features you own.

## Testing

Writing tests for features is, among other things, a way of taking on
responsibility for a feature's future. More so than ensuring
correctness at the time of development or release, the purpose of
software testing provides safety rails for yourself and for any future
participant in the system, documenting which behaviors must be
preserved into the future as the system evolves.

I've never worked at Google, but I understand Google's
["Beyoncé Rule"][beyonce] ("If you liked it, than you should have put
a test on it") as an articulation of this principle: If you write a
feature, but don't write tests for it, than you and only you are
responsible if the feature regresses due to changes on the part of its
dependencies or elsewhere in the system.

Testing discipline can also provide a form of anti-entropy against
Hyrum's Law. If users depend on unspecified or unintentional features
of a library or system, but also write tests that illustrate those
dependencies, then, at a minimum, developers wishing to make changes
can run those tests themselves and learn about unexpected assumptions,
and react accordingly to minimize disruption.

## DevOps

"devops" has become such a buzzword that it's really hard to figure
out what it means to most people. However, it seems clear to me that a
huge part of the de-siloing of operations and development is about
encouraging the kind of culture of ownership I talked about above, and
encouraging development teams to internalize the operational costs of
their system. Instead of building a system and letting someone else
run it -- who is then very constrained in their ability to make
change, lest they break the black box they've been handed -- we
co-locate more of that responsibility.

Aside from the broad philosophy, many of the concrete practices
associated with "devops" clearly share the philosophy of allowing
development teams to internalize the costs of a future's indefinite
future maintenance:

 - Automated monitoring and observability practices let the developers
   define the "health" metrics of an application, making its health
   clear and consumable by those outside the team without a detailed
   understanding of its implementation.
 - Automation allows the developers to encode behaviors and practices
   around their system once, which can then be carried out, again,
   without requiring specific knowledge of their system by those
   running the infrastructure.

## Blameless Postmortems

The philosophy of blameless postmortems includes, very claerly,
pushback against zealous application of "whoever made the change take
the blame". In an archetypal "blameless postmortem", one or more
participants in the system likely took actions that, in some way,
contributed to an incident. Instead of blaming those participants for
neglience of human error, however, we step back and examine the larger
system and system dynamics, and ask how could have better-informed
those participants, provided better systematic safety mechanisms, or
otherwise prevented the incident more holistically.

In the long run, this means building a system that is fundamentally
more resilient to change; of necessity, a lot of the work of making
particular systems or features more robust and resilient to change
will fall on the owners and developers of that feature, further
helping to internalize the costs of maintenance.

# Conclusion



[agl]: https://www.imperialviolet.org/2016/05/16/agility.html
[grease]: https://tools.ietf.org/html/draft-davidben-tls-grease-01
[html5]: https://developer.mozilla.org/en-US/docs/Web/Guide/HTML/HTML5
[silver]: https://en.wikipedia.org/wiki/No_Silver_Bullet
[hyrum]: http://www.hyrumslaw.com/
[xkcd]: https://xkcd.com/1172/
[delete]: https://programmingisterrible.com/post/139222674273/write-code-that-is-easy-to-delete-not-easy-to
[beyonce]: https://github.com/cpp-testing/GUnit#testing
[blameless]: https://landing.google.com/sre/book/chapters/postmortem-culture.html
