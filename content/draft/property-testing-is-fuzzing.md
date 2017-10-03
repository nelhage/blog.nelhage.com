---
title: "Property-Based Testing Is Fuzzing"
slug: property-testing-is-fuzzing
date: 2017-10-03T09:00:00-07:00
---

["Property-based testing"][proptesting] refers to the idea of writing
statements that **should** be true of your code ("properties"), and
then using automated tooling to generate test inputs (typically,
randomly-generated inputs of an appropriate type), and observe whether
the properties hold for that input. If an input violates a property,
you've demonstrated a bug, as well as a convenient example that
demonstrates it.

A classic example of property-based testing is testing a `sort`
function:

{{<highlight python>}}
@given(st.lists(st.integers()))
def test_sort(s):
  out = list(sorted(s))
  assert set(out) == set(s)
  assert all(x<=y for x,y in zip(out, out[1:]))
{{< / highlight >}}

This test asserts that, given a list of integers, sorting the list
- preserves the set of elements
- produces a sorted output

The test framework will then automatically execute it over some set of
input lists, and report if any counter-examples are found.


["Fuzzing"][fuzzing] is a much older practice, and generally refers to
passing randomly-generated data of some variety (often a purely-random
bytestream, but potentially chosen in some intelligent way) to a
program in the hopes of finding an input that causes a crash (and
thus, also, demonstrating a bug).

In recent years, pioneered largely by [AFL][afl], the practice of
coverage-guided fuzzing has used a form of code
instrumentation/coverage to explore inputs more likely to exercise
interesting behavior; This technique has proven to be incredibly
effective for a large variety of fuzz targets.

Historically, fuzzing and property-based testing have been regarded as
fairly separate practices. Property-based testing originated primarily
with [Haskell's QuickCheck][quickcheck], and so tends to be associated
with richly-typed languages, formal specifications, and related
fields. Fuzzing, on the other hand, was usually practiced against C or
C++ binaries, typically with a security bent -- aiming at finding
exploitable memory corruption bugs.

In this post, however, I want to present an argument that fuzzing and
property-based testing are essentially the same practice, at least at
a certain level of abstraction. I'm hopeful that the recognition of
this similarity can help practitioners of each improve their tools and
workflows.

# Property-Based Testing is Fuzzing

If we back up one level of abstraction or so, property-based testing
and fuzzing appear very similar. In both cases, we have:

- A system under test

  The traditional granularity of a property-based test is a function,
  and for a fuzzer is a binary[^libfuzzer], but both are just different
  implementation of "some arbitary computation"

- A property we want to ensure

  Traditionally, property-based testing has us write a property as
  explicit code, while fuzzers only test for the property "does not
  crash". However, by the simple expedient of `assert(property())` we
  can turn any property into an assertion about not-crashing, and
  [people have used this technique][fuzz-bn] to find surprisingly
  subtle behavioral bugs.

- A strategy for finding inputs that might violate the property

  Quickcheck, and many derivative property-based test suites, have
  used type-driven generation, while fuzzers have used random
  bytestreams, hand-coded generators, or
  [random mutations of known-good inputs][5linefuzzer]. However,
  ultimately all of these approaches are just strategies to
  automatically generate inputs that are hoped to trigger violations
  of the asserted properties of the system under test.

There are numerous differences in how they're used in practice and in
the tooling. But it seems clear to me that there's also a deep
similarity, and they are less fundamentally-different practices than
they may appear.

# Why should we care?

Both fuzzing and property-based testing have a rich history of
development, varied ecosystems of tools, and communities of users and
fans. However, in my experience, they relatively rarely overlap, and
there's not -- by and large -- an enormous amount of cross-pollination
between the ecosystems. I think that's a mistake, and the tools should
be converging more than they have been to date. In a future post, I
hope to go into a bit more detail about some of the specific
techniques I'd love to see making their way into property-based
testing tools.

# Postscript: Hypothesis

[Hypothesis][hypothesis] is an open-source property-based testing
tool, primarily implemented for Python. It is, in my opinion, worlds
ahead of any other tool I'm aware of in many ways. Relevantly for this
post, however, is that its author is way ahead of me on recognizing
the fundamental similarily between fuzzing and property-based testing,
and has
[written about it](http://hypothesis.works/articles/what-is-property-based-testing/),
as well as adopted many ideas from the fuzzing world into the tool.

If you maintain a Python codebase, you should probably be using
Hypothesis. If you don't, you should probably understand Hypothesis,
so you can crib its best ideas for yourself.


[proptesting]: http://blog.jessitron.com/2013/04/property-based-testing-what-is-it.html
[fuzzing]: https://en.wikipedia.org/wiki/Fuzzing
[quickcheck]: http://www.cse.chalmers.se/~rjmh/QuickCheck/manual.html
[5linefuzzer]: http://flatlinesecurity.com/posts/charlie-miller-five-line-fuzzer/
[fuzz-bn]: https://blog.fuzzing-project.org/31-Fuzzing-Math-miscalculations-in-OpenSSLs-BN_mod_exp-CVE-2015-3193.html
[hypothesis]: http://hypothesis.works
[afl]: http://lcamtuf.coredump.cx/afl/

[^libfuzzer]: Although more recent tools, such as [libfuzzer](https://llvm.org/docs/LibFuzzer.html), have also supported writing fuzz targets with the granularity of "a function"
