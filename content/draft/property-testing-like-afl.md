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
tree, such as [Google's OSS-Fuzz](https://github.com/google/oss-fuzz).

# Preface: Desiderata for a Unit Test Suite

A large software project, in general, may have multiple test suites,
which are run at different frequencies or different levels of
developer interactivity or involvement. In this post, I'll use a "unit
test suite" to describe a suite of tests that are run on every branch
in a continuous integration environment, and that additionally is
likely run interactively (perhaps in part) by developers as they work.

What properties do we want from a unit test suite? There are a number,
but I want to highlight two important ones here. A unit test suite
should be **fast** and **deterministic**.

Speed of execution is important, because fast tests enable rapid
feedback for developers, both locally as they write code (make a
change, re-run that module's tests, get feedback if you broke
something), and in CI, where you (typically) run every test to get
greater cross-program confidence. Slow tests are frustrating to
developers, and force them to resort to manual or ad-hoc testing
during development, instead of just leaning on the test suite.

Similarly, tests must be deterministic; Debugging failing tests must
be easy, and, more importantly, for a CI suite to be viable, "tests
pass once before merge" must imply "those tests will continue to pass
until someone does something to break them". If that inference isn't
true, then you'll see CI breakage at random points in time on random
code, which frustrates and slows down whichever developer happens to
hit that failure, and decreases developer trust in CI as the arbiter
of safety.

# Where Property-Based Fuzzing Fails

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
  prone (by design, in many cases!) to nondeterminism. In many ways,
  this is regarded as a feature, since every test run has a chance at
  finding a novel
