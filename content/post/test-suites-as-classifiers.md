---
title: "Test suites as classifiers"
slug: test-suites-as-classifiers
date: 2020-03-01T15:34:00-05:00
---
Suppose we have some codebase we’re considering applying some patch to, and which has a robust and maintained test suite.

Considering the patch, we may ask, *is this patch acceptable* to apply and deploy. By this we mean to ask if the patch breaks any important functionality, violates any key properties or invariants of the codebase, or would otherwise cause some unacceptable risk or harm. In principle, we can divide all patches into “acceptable” or “unacceptable” relative to some project-specific notion of what we’re willing to allow.

In practice, it’s not always easy to determine for sure if a patch is acceptable. We may not even have a rigorously-defined definition of acceptability, and even if we did, it may be challenging or impossible to fully reason about the new code’s interactions. What we can do, however, is run the test suite with the patch applied, which will return a verdict: either “PASS” or “FAIL”.

With this perspective, we can see the test suite’s output as a **prediction** about the acceptability of this patch, and the test suite itself as a **classifier** over changes: It is a computation, hopefully reasonably-efficient, which accepts as input a change, runs some analysis (the tests themselves), and attempts to classify the change relative to our (conceptual) acceptability criterion.

Like most classifiers, the test suite does not promise perfect accuracy, but aims to be as good as possible within some constraints of implementation effort and cost to evaluate (e.g. test runtime).

Like all classifiers, a test suite admits two kinds of possible errors[^terminology]:

- A **false alarm**, when the tests return “FAIL,” even though we would like to regard the change as “acceptable”
- A **missed alarm**, when the tests return “PASS,” even though we would like to regard the chance as “unacceptable”

[^terminology]: These are traditionally referred to “Type I” vs ”Type II” errors, or “false positives” vs “false negatives.” I find either set of terms very prone to confusion and parity errors, so I’m trying to use terminology I find less ambiguous.


## The costs of errors

It’s worth being explicit that the costs of the two kinds of errors are different and asymmetrical. A missed alarm potentially means that a bug escapes into the product, with varying costs depending on the product and the bug. This cost could be as small as a minor nuisance to the tool’s own developer, who discovers it by hand fifteen minutes later, and as large as a major production incident with downtime or data loss.

The costs of a false alarm are often smaller in the short term, but also pernicious. False alarms slow down the development process by requiring a developer to evaluate the failure and fix the test and/or code. They also discourage developers from making certain kinds of changes (which they’ve learned mean losing days fighting the test suite), and lead to general mistrust in the testing system. False alarms can, in the long run, contribute to the ossification of a code base as a terrifying legacy system that no one is comfortable making significant changes of any kind to.

False alarms can also contribute to [alert fatigue](https://en.wikipedia.org/wiki/Alarm_fatigue) on the part of developers. If developers are used to fixing flaky tests every time they make a change, there’s an increased risk that they disregard a true failure as “just another bad test” and work around by “fixing” the test instead of fixing the bugs. Overly brittle tests can therefore sometimes lead to similar outcomes as missing tests, where the where you technically have tests but they can’t be relied upon because developers have learned to ignore them!

Most testing advice you hear tends to focus on reducing missed alarms; coverage goals, for instance, are predominantly an attempt to eliminate potential areas for missed alarms. However, when I talk to working software engineers about their experiences and frustrations with testing, I find that some huge fraction of the complaints I hear are about brittle test suites prone to constant false alarms. Virtually every engineer, it seems, has a horror story of a test suite that broke for bad reasons every time they made *any* change, or of the [snapshot tests](https://blog.nelhage.com/post/record-replay-in-sorbet/) that just have to be blindly re-recorded on every pull request, or of that one test file that just has to be updated every time you add a feature.

I think the framework sketched here, of thinking of tests as a classifier whose goal is to correctly *predict* the safety of a patch, is a productive model for reasoning about these kinds of complaints, and creating test suites that add the most value we can.


## Minimizing false alarms

How do we go about minimizing the rate of false alarms in a test suite, or when writing tests? I think a productive place to start is just to bring this mindset to evaluating tests you write or review. When code-reviewing a test, ask yourself these questions:

- What kinds of true failures will this test catch? What kinds of changes can I imagine being made to the system that would break some property we care about, and which would be caught by this test?
- What kinds of false alarms is this test prone to? What assumptions does this test make that are incidental to the behavior it’s testing? What kinds of changes can I imagine wanting to make that will break this test but will be considered acceptable?

Balance the answers against each other; if you can’t come up with many answers to the first set of questions, and come up with many for the second, consider whether this test is worth adding, or should be rewritten or even deleted. Apply this thinking retroactively, too; encourage and celebrate improvements to existing brittle tests, or even deleting them if they’re causing more harm than good.

Because the cost of shipping bugs is often so visible, there’s commonly organizational pressure to add tests, including hyper-specific tests for that one bug we saw that one time; try to counterbalance that pressure to some extent by acknowledging the costs of brittle tests on productivity and on the team’s ability to ship features. Weigh the costs of the two kinds of failures as you do: a brittle test that tests for behaviors also well-covered elsewhere in the codebase can probably just be removed, while a brittle test that’s the only safeguard against some specific catastrophic failure might merit investment in a better test or other safeguard, first.


# Measuring error

If we’re thinking of tests as classifiers, one follow-up question is whether we can try to measure the rates of true and false alarms. As mentioned, it’s hard or impossible to know for sure the ground truth of “acceptability,” but this doesn’t necessarily stop us from trying to gather data here. There are some easy ideas, such as the observation that any change that we are forced to revert is probably an instance of a missed alarm, and I suspect that at scale there may be others.

With access to a full history of git pushes and CI runs (and, potentially, even local test runs, to observe state before a developer even pushes), we can imagine looking at changes to test and non-test files, and trying to measure, for each test, the frequency with which a test failure is fixed by changing the code, or by changing a test file. This isn’t a perfect measure and has some challenges, but it seems like it should, at a minimum, shed light on tests that are outliers for false alarms. I’d love to hear from anyone who’s tried something like this or has any other creative ideas for how we can try to quantify our test suites through this lens!

