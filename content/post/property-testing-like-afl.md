---
title: "Property Testing Like AFL"
slug: property-testing-like-afl
date: 2017-10-24T09:00:00-07:00
---

In my last [last post](/post/property-testing-is-fuzzing/), I argued
that property-based testing and fuzzing are essentially the same
practice, or at least share a lot of commonality. In this followup
post, I want to explore that idea a bit more: I'll first detour into
some of my frustrations and hesitations around typical property-based
testing tools, and then propose a hypothetical UX to resolve these
concerns, which takes heavy inspiration from modern fuzzing tools,
specifically the [AFL][afl] and [Google's OSS-Fuzz][oss-fuzz].

# Preface: Desiderata for a Test Suite

A large software project, in general, may have multiple test suites,
which are run at different frequencies or different levels of
developer interactivity or involvement, with different goals.

In most projects I've worked on, however, the most common test suite
pattern is a continuous integration suite that runs on every change
(proposed or merged), and a subset of which is run interactively by
developers as they develop to validate their work.

Such a test suite is, in many ways, first and foremost a regression
suite. Developers may write in a test-first "TDD" style or write tests
for compliance with an external spec (implicit or explicit), but once
the code is written, the most important function of the test suite is
to document (in an executable+verifiable way) the critical properties
of the code which must be preserved by future changes. In a large
codebase under active development, this regression testing gives
future developers the ability to make changes with some confidence
that they have not broken everything.

What properties do we want from such a test suite? There are a number,
but I want to highlight two important ones here. Specifically, a test
suite should be **fast** and **reproducible**.

## Speed

Speed of execution is important, because fast tests enable rapid
feedback for developers. In a large codebase, adding a feature or
fixing a bug is often not hard, but adding a feature or fixing a bug
_without breaking any existing behavior_ is the hard part. Since tests
provide your first line of feedback for that property, they often
become a critical rate-limiter on your development feedback
cycle. Slow tests are frustrating to developers, and force them to
resort to manual or ad-hoc testing during development, instead of
relying on the test suite.

## Reproducibility

Test results must also be **reproducible** in order to reliably
provide the desired confidence for developers. By this, I mean that
once a test succeeds, it should reliably continue succeeding until
something meaningful changes, and if a test fails, it should be easy
to reliably reproduce the failure.

Both properties are essential for a CI regression-testing suite to be
viable. Once tests pass and are merged, they must not fail randomly;
otherwise, the compounding effects of random failures makes it
impossible to ever keep the build "green", ruining the feedback value
of CI. On the flip side, if a test fails during development or in a
pre-merge push, it must be easy to reproduce that failure to allow a
developer to debug and fix the issue quickly and with confidence that
the issue has been resolved.

# Where Property-Based Fuzzing Falls Short

Traditional property-based fuzzing fails at both these criteria, which
I suspect is one of the reasons it has gotten less traction than it
might have.

  - Because traditional property-based testing generates random inputs
    at runtime, you generally need a relatively large number of runs
    to get decent test coverage (most systems I've seen default to
    something like 100). While it's pretty easy to execute 100 runs of
    a toy example within human timescales, large systems have a bad
    habit of ending up with test cases that have many expensive
    dependencies, I/O requirements, or other slowdowns, and a 10x
    slowdown vs "executing on 10 hand-chosen examples" is a
    frustrating price to pay.

  - Because property-based testing selects test cases at random, it is
    prone (by design!) to nondeterminism, which spoils the
    above-described reproducibility properties. Authors and adherents
    of property-based test suites often describe this as a feature,
    rightly pointing out that it gives every test run the opportunity
    to discover novel bugs, continually improving your
    coverage. However, this nondeterminism is untenable for a CI suite
    that must run on every commit for every developer; CI's job is to
    prove the absence of (certain) regressions, not absolute
    correctness.

    Many property-based test suites attempt to (optionally) recover
    determinism by supporting a deterministic random-number
    generator. This approach helps a lot, but can still leave you with
    a **brittle** test suite, where small changes to your generation
    strategy or test cases cause cascading changes to the set of
    considered test cases. A developer who adds a branch to a
    data-generation strategy to support their new feature may find
    bizarre cascading failures if doing so changes the examples seen
    by other tests.

# The Workflow I Want

With that motivation as background, I want to lay out the workflow I
believe I want out of property-based testing. Based on my judgment and
experience, I think a design of this shape would effectively bridge
the gap between many of the present tools, and the needs of a
large-scale regression suite. This description is inspired by a
combination of attempting to address the above concerns, and by
techniques already widespread in the fuzzing space, notably the
[afl][afl] and [oss-fuzz][oss-fuzz] tools.

## Hard-coded examples

I want to write tests in a property-based style, by writing functions
that must pass for all inputs of some type. However, I also want to
**commit a list of test examples** to my repository, ideally in a
textual, human-readable format (e.g. as source code literals or JSON)
for ease of review and merging.

When run in a default mode (e.g. `make test` or your CI entrypoint),
the tool should **only** run those examples. By running a fixed,
small, set of examples, we recover the speed and reliably we desired
above.

## Example Generation

So far, what I've described is just a slightly-heavyweight version of
"table-driven testing", and doesn't have the generative nature that
typically defines property-based testing. And while I'm an enormous
proponent of table-driven tests, it seems clear that automatic test
generation has something to offer above and beyond such tests.

To bridge the gap, I desire that the aforementioned lists of inputs
should be, in most cases, automatically generated by the
property-based framework, before being commited. After writing a test,
I might run `proptest generate`, which will begin a traditional
property-based-testing generation-and-minimization process, with the
difference that any failing tests, and a sample of other tests, will
be automatically written out to the "examples" file for review and
potential committing by the developer.

## Coverage-guided generation and feedback

How will `proptest generate` decide which cases to save, and how will
it decide when it has enough tests? Here we take inspiration from AFL,
and use coverage-guided exploration to guide test case generation, and
also to determine when to stop. The generator stops generating tests
cases after some combination of

- A configurable timeout
- When coverage stops improving
- When coverage is sufficiently high in an absolute sense

(Upon completion, it can also print statistics about this exploration
process and the coverage reached, to inform the quality of the test
case and/or generator.)

Once it has stopped (or perhaps concurrently with generation), the
tool will use a coverage-driven minimization process (ala
[`afl-cmin`][afl-cmin]) to find a near-minimal list of test cases that
exercises approximately the same set of coverage as the total
corpus. In addition, any tests that fail are automatically preserved
(perhaps up to uniqueness, as judged by the execution trace and
related criteria).

The end result of such a process should be a small corpus of tests
that execute quickly, while still exercising about as much of the code
under test as possible, and explicitly checking test cases that once
failed, to prevent regressions of known bugs.

## Exploration Mode

Finally, the tool will also have an "unbounded fuzzing" mode, in which
it uses the coverage-guided exploration engine to continually generate
and explore new test cases, reporting any failures. As a developer, I
can run this periodically in spare cycle on a laptop or server
somewhere, or, ideally, in a cloud executor designed for this purpose
(ala [oss-fuzz][oss-fuzz]).

Importantly, since this search process will run as a separate job, it
will discover bugs asynchronously to my main development process, and
not block PRs or break trunk. I can then triage and address these bugs
on whatever timeline I see fit, making my own judgment of severity or
urgency.

Additionally, any failures found in this mode will generate output
that includes a formatted example that I can copy-paste directly into
the committed examples list and commit. This makes local reproduction
easy, since a `make test` will now run this test case and demonstrate
the failure for me, and also ensures that this bug is never regressed,
by directly testing this test case on every future CI run, once I do
land a fix.

## Other Notes

Given such a system, the tool can also implement a "traditional"
property-testing mode, which ignores the hardcoded corpus and runs
each test for a fixed duration or number of runs. We can also mark a
given test as "never autogenerate input", and recover classic
table-driven testing. This flexibility should allow for workflows that
scale from small tests on trivial codebases to very large, complex
test setups, all within a uniform framework.

Similarly, even for tests that already have committed examples, if you
change the code under test, you should be able to re-run the
generator, seeded with the existing examples, to re-start the
exploration process on the new code, and produce a new minimized
high-coverage corpus. Obviously this process shouldn't be necessary --
if the properties haven't changed, the old examples should still be
valid -- but it's important to allow us to evolve our corpus and test
data with an evolving implementation.

# Conclusion

I'm fairly confident I'd be much more willing to adopt property
testing frameworks in my own code if they behaved more like the
(notional) system described above.

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

And in fact, since I first wrote this post, the Hypothesis authors have [launched HypoFuzz](https://hypofuzz.com/) -- in small part inspired by my posts -- to try to better support more diverse workflows with Hypothesis, including ones like the flows discussed in this post. I'm really excited to see what I already thought of as the best-in-breed property-based testing tool getting even more sophisticated.

## Postscript

This post is also available in <a href="http://clipartmag.com/ru-property-testing-like-afl" rel="nofollow">translation into Russian</a>

[afl]: http://lcamtuf.coredump.cx/afl/
[oss-fuzz]: https://github.com/google/oss-fuzz
[afl-cmin]: https://github.com/mirrorer/afl/blob/master/afl-cmin
[hypothesis]: http://hypothesis.works
