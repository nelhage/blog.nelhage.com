---
title: "Property Testing Like AFL"
slug: property-testing-like-afl
date: 2017-10-11T19:15:44-04:00
---

This post is a bit of a followup to my
[last post](post/property-testing-is-fuzzing/), in which I lay out the
developer experience I'd like to have with a property-based testing
tool. In line with the previous post, the UX I describe here is
largely in line with that offered by various tools in the `afl` family
tree, such as [Google's OSS-Fuzz][oss-fuzz].

# Preface: Desiderata for a Unit Test Suite

A large software project, in general, may have multiple test suites,
which are run at different frequencies or different levels of
developer interactivity or involvement. In this post, I'll use the
term "unit test suite" or "unit tests" to describe a suite of tests
that are run on every branch in a continuous integration environment,
and that additionally is likely run interactively (perhaps a subset of
the suite) by developers as they work.

Suchs test suites are, in many ways, first and foremost regression
tests; Their purpose is not to verify absolute correctness against
some spec, but to document (in an executable+verifiable way) which
properties of the code must not be accidentally modified by future
changes. In a large codebase subject to active development, such test
suites are a critical tool for enabling confident development.

What properties do we want from such a test suite? There are a number,
but I want to highlight two important ones here. Specifically, a unit
test suite should be **fast** and **deterministic**.

Speed of execution is important, because fast tests enable rapid
feedback for developers, both locally as they write code (make a
change, re-run that module's tests, get feedback if you broke
something), and in CI, where you run all (or a large swath of) the
tests to get greater cross-program confidence. Slow tests are
frustrating to developers, and force them to resort to manual or
ad-hoc testing during development, instead of leaning on the test
suite.

Similarly, tests must be deterministic; For a CI regression-testing
suite to be viable, "tests pass once before merge" must imply "those
tests will continue to pass until someone does something to break
them". If that inference isn't true, then you'll see CI breakage at
random points in time on random code, which frustrates and slows down
whichever developer happens to hit that failure, and decreases
developer trust in CI as the arbiter of safety.

# Where Property-Based Fuzzing Falls Short

Traditional property-based fuzzing fails at both these criteria, which
I suspect is one of the reasons it has gotten less traction than it
might have.

- Because traditional property-based testing generates random inputs
  at runtime, you need a relatively large number of runs (most systems
  I've seen default to something like 100) to get decent
  coverage. While it's pretty easy to execute 100 runs of a toy
  example within "human" timescales, large systems have a bad habit of
  ending up with test cases that have many expensive dependencies, I/O
  requirements, or other slowdowns, and a 10x slowdown vs "executing
  on 10 hand-chosen examples" may not be acceptable.

- Because property-based testing selects test cases at random, it is
  prone (by design, in many cases!) to nondeterminism. Authors and
  adherents of property-based test suites often describe this as a
  feature, rightly pointing out that it gives every run the
  opportunity to discover novel bugs, continually improving your
  coverage and confidence. However, this nondeterminism is untenable
  for a unit test suite that must run on every commit for every
  developer.

    Many property-based test suites attempt to (optionally) recover
    determinism by supporting a deterministic random-number
    generator. This approach certainly helps, but can still leave you
    with a **brittle** test suite, where small changes to your
    generation strategy or test cases cause cascading changes to the set
    of considered test cases.

# The Workflow I Want

With that motivation out of the way, I want to lay out the workflow I
believe I want out of property-based testing, which I think would
effectively bridge the gap between many of the present tools, and the
needs of a large-scale regression suite. This description is inspired
by a combination of the above concerns and by techniques already
widespread in the fuzzing space, notably the [afl][afl] and
[oss-fuzz][oss-fuzz] tools.

## Hard-coded examples

I want to write tests in a property-based style, by writing functions
that must pass for all inputs of some type. However, I also want to
**commit a list of test examples** to my repository, ideally in a
textual, human-readable format (e.g. as source code literals or JSON)
for ease of review and merging.

When run in a default mode (e.g. `make test` or your CI entrypoint),
the tool should **only** run those examples. By running a fixed,
small, set of examples, we recover the speed and determinism we
desired above.

## Example Generation

So far, what I've described is just a slightly-heavyweight version of
"table-driven testing", and doesn't have the generative nature that
typically defines property-based testing. The next key detail in my
desires is that those lists of inputs should be, in most cases,
automatically generated by the property-based framework. After writing
a test, I might run `proptest generate --test=MyTestCase`, and it will
begin a traditional property-based-testing generation-and-minimization
process, with the difference that any failing tests, and a sample of
other tests, will be automatically written out to the "examples" file
for future runs.

## Coverage-guided generation and feedback

How will `proptest generate` decide which cases to save, and how will
it decide when it has enough tests? It runs my test using AFL-style
coverage-guided exploration, and stops after some combination of

- A configurable max timeout
- When coverage stops improving
- When coverage is sufficiently high

It can also print statistics about this exploration process to inform
the quality of the test case and/or generator.

Once it has stopped (or perhaps concurrently with generation), it will
use a coverage-driven minimization process (ala
[`afl-cmin`][afl-cmin]) to find a near-minimal list of test cases that
exercise the same set of paths as the total corpus.

The end result of such a process should be a small corpus of tests
that execute quickly, while still exercising nearly as much of the
code under test as possible, and explicitly checking test cases that
once failed, to prevent regressions of known bugs.

## Exploration Mode

Finally, the tool will also have an "unbounded fuzzing" mode, in which
it uses the coverage-guided exploration engine to continually generate
and explore new test cases, reporting any failures. As a developer, I
can run this periodically in spare cycle on a laptop or server
somewhere, or, ideally, in a cloud executor designed for this purpose
(ala [oss-fuzz][oss-fuzz]).

Importantly, since this search process will run as a separate job, it
will discover bugs asynchronously to my main development process, and
not block PRs or break trunk, and I can address these bugs on whatever
timeline I see fit, after triaging or otherwise evaluating severity or
urgency.

Additionally, any failures found in this mode will generate output
that includes a formatted example that I can copy-paste directly into
the committed examples list and commit; This makes local reproduction
easy, since a `make test` will now run this test case and demonstrate
the failure for me, and also ensures that this bug is never regressed,
by directly testing this test case on every future CI run, once I do
land a fix.

# Conclusion

I can't speak for others, but I'm pretty confident I'd be much more
willing to adopt property testing frameworks in my own code if they
behaved more like they described above.

I have pretty good confidence in many parts of the above workflow,
because I've largely just described tools that exist in the fuzzing
world, and so this is not completely fabricated from thin air. That
said, I think there are still some open questions about how such
workflows would work as a primary means of testing code; If you see
concerns or have tried something similar, I'd be very curious to hear
experience reports.

Finally, I want to once again include a shout-out to
[Hypothesis][hypothesis]; Although it doesn't implement precisely the
workflows I've described above, it contains essentially all of the
pieces you would need to build them, including a
[persistent example database](http://hypothesis.readthedocs.io/en/latest/database.html)
of failed examples, the ability to
[specify examples manually](http://hypothesis.readthedocs.io/en/latest/details.html#providing-explicit-examples),
and
[coverage-guided exploration](http://hypothesis.readthedocs.io/en/latest/settings.html?highlight=use_coverage#hypothesis.settings.use_coverage).

I don't write much Python these days, but if I did I'd definitely
explore building out a variant of this workflow on top of Hypothesis.


[afl]: http://lcamtuf.coredump.cx/afl/
[oss-fuzz]: https://github.com/google/oss-fuzz
[afl-cmin]: https://github.com/mirrorer/afl/blob/master/afl-cmin
[hypothesis]: http://hypothesis.works
