---
date: 2017-06-11T13:42:01-07:00
slug: null
title: Two Perspectives on the End-to-End Principle
---

Back when I was an undergraduate, as part of a class called
["Computer Systems Engineering"][6.033], we read numerous classic
papers of systems design. I enjoyed and learned a great deal from many
of these papers, but one that paper that has stuck with me in
particular was Saltzer et al's
["End-to-End Arguments in Systems Design"][end-to-end]. The paper is a
very general tract on systems design -- it does explore several
examples of concrete systems or applications, but it ultimately
expounds upon the end-to-end principle as a perspective or design
heuristic that can apply to virtually any system design.

I recommend reading the paper -- it's short and very readable -- but
in brief, it argues that many functions of a system are better
addressed at the "ends" of the system, instead of at each lower-level
interface boundary. As a concrete example, the paper argues that a
file transfer system is better served by assuring correctness
end-to-end by way of strong checksums and retries as needed, than by
insisting on perfect lossless transfer out of each of its underlying
network layers.

The longer I work professionally in software and design and work with
systems, the more I find myself reflecting on the end-to-end argument
and its nuance and implications for sytems design. I want to here
commit to writing two different perspectives I find particularly
interesting.

To understand either perspective, we'll need to first reflect on what
we mean by a "system" and "system design." Merriam-Webster
[defines][mw] a "system" a "a regularly interacting or interdependent
group of items forming a unified whole". The essential character of a
system, then, is that it is a unit that decomposes into some number of
sub-components, which can be understood or considered substantially in
isolation of each other. Systems design, then, is the process of
performing this decomposition: of deciding how to break a desired
system into individual units that can be built up into a functioning
whole.


[mw]: https://www.merriam-webster.com/dictionary/system

## The Depressing Take: You Can't Compose Correctness

One property we might hope for in systems design is that we can, in
some sense, arrive at a correct system merely by properly composing
correct subsystems. This hope is sometimes phrased in terms of
desiring "LEGO blocks" for software design: If we can just design the
right, robust, fundamental building blocks of software design, we'll
be able to snap them together in arbitrary ways and design complex
systems at low cost and effort.

The end-to-end argument challenges this optimism directly. No matter
the sophistication of the underlying building blocks, it argues, we'll
always have to define and enforce the essential correctness properties
of our system at the topmost end-to-end layer of design. We can't
trivially derive correctness from the correctness of our subsystems:
we must always consider it as an end-to-end property.

The end-to-end argument thus also speaks to a lack of scale-invariance
in abstraction and systems design. Systems that own the "ends" are
subject to the end-to-end principle and responsible for enforcing
correctness. Conversely, systems that exist only as subsystems or
components of a larger whole may not have such responsibility, and can
delegate certain guarantees to the end-to-end system outside of their
scope.

This scale-dependence, therefore, implies a certain lack of
"fundamentally correct" abstractions in the world. If we want to
design a component for a certain class of functionality, the
appropriate design constraints and guarantees depend on whether we're
building an end-to-end system or merely a component of a larger
one. And, if we're designing a component, which guarantees we must
provide depends on which properties the top-level system is prepared
to enforce itself.

In this regard, I think this take on the end-to-end argument is a
close peer to the Law of Leaky Abstractions, which states that all
sufficiently-complex abstractions are to some degree leaky. Both
principles express a certain inevitability for "out-of-band"
interactions between higher levels of the abstraction stack and the
implementation details of the underlying system.

This is the perspective that struck me most strongly when I first read
the paper in college. I found it fairly disheartening at the time; As
a young enthusiastic aspiring systems designer, I wanted to believe in
the existence of some set of idealized platonic abstractions for
fundamental capabilities, which, if we could just extract them from
the raw ether of software stuff, would seamlessly and forever raise
the level of abstraction at which software designers work. The
end-to-end principle is a statement that the world is messy, and that
while there may exist "good-enough" abstractions to be
nearly-universal, we'll forever be carrying complexity with us to
higher and higher levels of the systems design stack.

## The Optimistic Take: The TCB Perspective

This second perspective is one that I've appreciated more and more as
I've done actual systems design, and I think is closer to the wisdom
the paper's authors hoped to impart.

Correctness is hard. Anyone who's worked for long in software is well
familiar with the reality that it's dramatically easier to write
software that works most of the time than software that works all of
the time. This truism is only more true in large systems, where we
must interact with external components that themselves may have
uncertain reliability properties.

The end-to-end argument encourages us to accept and embrace this
reality. Instead of demanding absolute correctness from every part of
our system, we can choose some essential correctness properties (e.g.:
messages are copied unmodified from point A to point B; every
transaction appears exactly once in our ledger) and to **locate**
those properties within a subset of our system (at the "ends"). Once
we've done this design step, we can demand correctness from these
end-to-end components, and treat the rest of the system as more of an
optimization problem.

I call this the "TCB" perspective because I liken it to the notion of
a "trusted computing base" in operating systems theory. We seek to
reduce the portions of the system for which bugs directly threaten the
integrity of the entire system; having done so, we can then develop
the rest of the system safe in the confidence that errors may manifest
as performance problems or even as detectable faults in the overall
system, but that it should be impossible to silently cause
catastrophic failure.

# Conclusion

Systems design is intricate, and full of tradeoffs, tools, tricks, and
heuristics for how to assemble complex systems out of components. I
really love the end-to-end argument, and this paper about it, because
I think it's one of our best "top-down" tools for systems design. Many
techniques and principles focus on "building up" -- taking components,
composing them, architecting up from the implementation primitives
available to us. But at the end of the day, our goal is to solve a
concrete problem with an overall system, and the end-to-end argument
is a powerful tool to relate this overall goal to the decomposition of
the system into components and modules.

This perspective, of working top-down and relating end-to-end function
to component design, can be frustrating, as it means we may need to
visit every systems design problem substantially anew; components that
look similar between two problems may have importantly different
functional requirements depending on the ultimate goal. At the same
time, however, it gives us a powerful tool and structure for doing
this decomposition and thinking about systems design.


[6.033]: http://web.mit.edu/6.033/www/
[end-to-end]: http://web.mit.edu/Saltzer/www/publications/endtoend/endtoend.pdf
[tcb]: https://en.wikipedia.org/wiki/Trusted_computing_base
