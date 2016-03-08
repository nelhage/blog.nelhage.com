---
layout: post
title: "Design for Testability"
date: 2016-03-06 09:46:39 -0800
comments: true
draft: true
categories:
---

When designing a new software project, one is often faced with a glut
of choices about how to structure it. What should the core
abstractions be? How should they interact with each other?

In this post, I want to argue for a design heuristic that I've found
to be a useful guide to answering or influencing many of these
questions:

> Optimize your code for testability

Specifically, this means that when you write new code, as you design
it and design its relationships with the rest of the system, ask
yourself this question: "How will I test this code? How will I write
automated tests that verify the correctness of this code, with minimal
irrelevant assumptions about the environment or the rest of the
system?" And if you don't have a good answer to that question,
redesign your abstractions or interfaces until you do.

I've found this heuristic valuable in two different ways, and I'll
discuss both here.

# You get good tests

This one is, perhaps, obvious: If you set out with a goal of having
good tests, you will likely end up with good tests.

However, I think it's worth saying a bit more about it.

First, I want to emphasize how wonderful it is to work in a code base
with good tests. The process of verifying changes is simple: just run
the tests. No need to set up complicated development environments and
poke at the system manually or interactively; just `make test`, and
get, if not a guarantee, high confidence that your code does what you
think it does and hasn't broken anything important.

In large and mature software systems, the hardest part of making
changes is not the change itself, but making the change in a way that
doesn't regress any other important behavior. Good, fast, tests that
can provide basic assurances quickly are therefore an enormous
productivity improvement.

Second, in order to really realize the benefits of good tests, you
need to be able to run tests -- or, at a mininum, the subset of tests
covering your changes -- quickly, in order to keep a fast development
cycle.

In order to preserve this property as a system scales, you need to
have good *unit* tests. There are a lot of religious arguments out
there about the boundaries between "unit", "functional",
"integration", and other sorts of tests; By "unit" tests here I mean
tests that exercise some specific module or component of the code base
with minimal dependencies on or invocation of other code in the
application.

You will always also need some number of integration tests, which
exercise the application end-to-end, in order to shake out the
inevitable subtle interactions betwen components. But relying
primarily on unit tests brings significant advantages:

- Unit tests are fast: By exercising a limited amount of code, they
  run more quickly than tests that must invoke the entire application.
- More importantly, unit tests *scale*: In a codebase with mostly unit
  tests, test speed should scale linearly with application size. With
  more functional end-to-end tests, you risk scaling closer to
  quadratically, as each component requires tests that in turn
  incidentally exercise nearly every other component.
- Because unit tests are associated with clear pieces of code, it's
  easy to run only the tests corresponding to a specific change,
  providing an even faster feedback loop to development.

Good unit tests don't happen by accident, however. It's very hard to
have good unit tests without having good "units": Portions of code
with narrow and well-defined interfaces, which can be tested at those
interfaces.

The best way to have such logical units, which then permit excellent
tests, which in turn permit rapid feedback cycles while maintaining a
mature project, is just to do so directly: Write code with this end in
mind.

# You get better code

The second reason to write code with the tests in mind is that it
actually ends up with better code!

Let's consider at some of the characteristics that easy-to-test code
should have:

## A preference for pure functions over immutable data

Pure functions over immutable data structures are *delightfully* easy
to test: You can just create a table of (example input, expected
output) pairs. They're generally also easy to fuzz with tools like
QuickCheck, since the input is easy to describe.

## Small modules with well-defined interfaces

If a chunk of code has a small, well-defined interface, we can write
black-box tests in terms of that interface, which test the described
interface contracts, without caring excessively about either the
internals of the module, or about the rest of our system.

## A separation of IO and computation

IO is, in general, harder to test than pure code. Testable systems
therefore isolate it from the pure logic of the code, to allow them to
be tested separately.

## Explicit declaration of dependencies

Code that implicitly plucks a database name out of the global
environment is much harder to test than code that accepts a handle as
an argument -- you can call the latter with different handles over the
course of your test, to test against a clean database each time, to
test in multiple threads at once, or whatever else you need.

In general, testable code accepts dependencies as explicit arguments
at some point, instead of assuming their implicit availability.

---

If I look at these characteristics, I find that they're also the
characteristics of *good, well-structured code* in general! By
architecting a system with testing in mind, we generally get a
better-factored system than we would have, otherwise.

Furthermore, by approaching these properties indirectly -- through the
lense of testing -- we make them more concrete. It's easy to debate
indefinitely what the right modules, interfaces and structures are for
a software system; But faced with the concrete frame of "What makes it
easy to adequately test this code?", it's easier to evaluate our
choices and to decide when we've gone far enough.

# Conclusion

There are few things more important for a complex software system than
the ability to make changes with confidence, and high-quality
automated tests are one of the most important tools we have to this
end. Good tests don't happen by accident, or even by brute-force
effort: they happen by design, because the application was written to
enable them.

As you write software, always be asking yourself "How will I test this
software's correctness?" and be willing to design to that goal. In
return, you'll get a system you can be -- and stay -- much more
confident of, and one that's better-structured otherwise.
