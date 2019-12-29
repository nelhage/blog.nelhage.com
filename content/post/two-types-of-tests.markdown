---
title: "Two kinds of testing"
date: 2019-12-24T17:09:55-04:00
slug: two-kinds-of-testing
---

While talking about thinking about tests and testing in software engineering recently, I’ve come to the conclusion that there are (at least) two major ideas and goals that people have when they test or talk about testing. This post aims to outline what I see as these two schools, and explore some reasons engineers coming from these different perspectives can risk talking past each other.


# Two reasons to test
## Testing for correctness

The first school of testing comprises those who see testing as a tool for validating a software artifact against some externally-defined standard of correctness. In the most expansive form of this approach, there might be a full specification of the behavior independent of the code — perhaps in a specification document from a customer, perhaps in an RFC or ISO standard or other standards document. However, I’m also going to encompass more informal and partial senses of “correctness,” including “should not crash on any input” or “does not leak memory” or other properties.

This is, in my experience, often the sense that people first think about when they think about testing. You have some definition of what it means to be correct, and you try to generate as much confidence as you can that the program is correct in that sense.

This approach often considers comparatively static programs and standards for correctness; an archetypal problem might be an implementation of some mathematical algorithm (where the definition is *a priori* and unchanging)


## Testing as a tool for software engineering

As [Russ Cox puts it](https://research.swtch.com/vgo-eng), riffing off of his colleague [Titus Winters](https://www.youtube.com/watch?v=tISy7EJQPzI&t=8m17s),


> Software engineering is what happens to programming
> when you add time and other programmers.

The second approach to testing takes testing less as a tool to assure correctness, and more as a tool to support large numbers of engineers working on a system over extended periods of time and high rates of change.

Testing is a tool to help those engineers make changes more quickly and with more confidence. This type of testing is almost synonymous with continuous integration practices, of running tests on every change, and requiring a green build to merge to trunk. At the limit in this direction, we find teams that practice continuous deployment with a culture of “If the tests pass, it’s safe to deploy.”

Regardless of whether or not you go that far, the key concept is usually that tests are taken as a tool to help engineers get automated feedback about their changes as they develop and deploy. This feedback can either [replace or augment human communication with code](https://increment.com/testing/testing-as-communication/); instead of having to find a domain expert to ask whether an existing behavior is important, or whether a change is safe, you can run the test suite and get an answer. Even if the test suite isn’t fully trusted, getting fast and automated partial feedback about whether the system still mostly works provides great value to engineers making changes.

As a corollary to all this, this style of testing tends to focus on a system that is subject to a lot of change. Getting good feedback on changes is most valuable when you’re trying to make a lot of them! Moreover, not only is the system implementation changing, but the very definition of desired behavior often is as well, as the product evolves.


# How are these different?
## Novelty

One major distinction, related to the static/changing dichotomy, is whether we’re (primarily) interested in finding novel bugs, or novel test cases which stress the system in new ways. When we’re testing for correctness, we’re almost by definition in pursuit of novel test cases; for any given system under test, we start by checking every test case we know of, and then the question becomes one of how to gain even more confidence in the system.

Any new failure is often considered a success; even if it turns out to be a false alarm (e.g. a bug in the test harness or the specification, not the system under test), we’ve gained knowledge and fixed a bug in the meta-system containing all three of the spec, the system, and the test suite, and thereby (hopefully) increased our overall confidence.

When we’re testing in service of software engineering, we’re often much less interested in true novelty, and much more interested in a weaker property, something like “This change didn’t break anything that was previously known to work.” In fact, a search finding novel bugs can be an explicit anti-goal! If we are trying to make some change, and a test fails because of some unrelated novel bug, this has the effect of creating a yak shave for us: since passing tests are required to merge, we now must fix, or at least triage, this bug, and now additional obstacles have been placed in the way of whatever we were trying to do, unrelated to our previous task. Especially since software engineering, per the above definition, involves (often many) other programmers, this work may additionally be entirely outside of our area of expertise! Testing has now slowed down our process, when its very goal was to speed it up!

This all is not to say that software engineers don’t value correctness and finding novel bugs and test cases; in high-functioning organizations that work is considered important and valuable, but it is a *different kind* of work than what “testing” initially brings to mind. [Separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns) can apply to organizational design and engineering practices, as much as to the code itself!


## Testing changes vs testing the system

When you are testing for correctness, you are very fundamentally testing the entire system under test; you want to know if it correctly implements the specification or follows the properties you’ve outlined.

When you are testing to enable software engineering, the system, writ large, is constantly in flux and changing and evolving, in the hands of many engineers. If you are using testing as a pre-merge blocker in CI, there is a strong sense in which you **don’t** want to test the state of the entire system; instead, you want to answer the much narrower question of “Did **this change** break something?”

If the system is broken but it isn’t your change’s fault, you’d actually (often) like to be able to roll forward with your unrelated change[^0]. This provides resiliency to the development process; it’s undesirable that one team landing an accidental [time bomb](https://en.wikipedia.org/wiki/Time_bomb_(software)) in their system prevent anyone else from testing or merging their changes. This observation also touches on the novelty question raised above, and provides another lens on why unrelated novel failures are actively undesirable in a CI setting.

[^0]: It’s also potentially important to be able to freeze `master`, during incident response perhaps, but having the ability to freeze it is different from forcibly freezing it every time any feature, no matter how ancillary, breaks its test suite.


## Test budget

In a testing-for-correctness environment, in the search of novel bugs and test cases, we can afford to spend a lot of time and/or CPU time on running tests. Such systems commonly employ high-powered techniques, such as coverage-guided randomized searches, symbolic execution, SAT/SMT solvers or otherwise, in the search of finding test cases that exercise new parts of the state space and find new bugs. It’s not uncommon to spend [years of CPU-time on coverage-guided fuzzing](https://github.com/google/oss-fuzz) for large projects.

When we are testing as software engineers, the calculus changes such that both wall-clock time and the dollar cost of running tests becomes critical. A major goal in this environment is to provide feedback to developers about changes, and feedback becomes drastically more valuable the faster it can arrive. As test suites get slower, developers cope by either relying on them less, wasting valuable time waiting for the test results, or both. When the situation gets bad enough, tests once again risk slowing development, instead of speeding it up.

The total cost (to e.g. run the EC2 instances to provide parallel CPU time) also matters past a certain point. Test cost can easily grow quadratically[^1] or worse with the size of the codebase or team, and running tests on every push can become extremely expensive. I have known organizations to run a larger number of EC2 instances running their test suite than they did on serving their production site! Sophisticated CI systems often end up using some kind of selective test execution to only run a subset of tests on every push, even though such systems add complexity and raise the specter of accidentally missing a test.

[^1]: Consider a simplified model where a team writes *N* features in serial. Each feature has one test, and while writing a feature you execute the entire test suite before merging. The total number of tests run becomes a classic triangular sum, resulting in Ω(n²) work. I use here Ω to denote a **lower** bound; in the event that each test exercises more code with each feature, the asymptotics can go even worse!


# Unifying the approaches

Even when we’re doing software engineering and evolving a system rapidly, we still (often, at least) care about correctness! How can we unify these two approaches?

I don’t have a full theory, but I think one important concept is to run both kinds of systems: an online CI system that runs in the development loop, and an asynchronous system which runs more in-depth analyses or searches, updating the system it analyzes at a lower frequency, perhaps daily.

Importantly, in order to close the loop, we should then think of the asynchronous system as finding not just bugs, but *test cases*. Whenever we find an input that causes a spec violation, such as a crash, or even just that exercises novel parts of the state space, we should have easy support for extracting *only* that input (and the corresponding assertions) into our CI suite, to run on every change. This way the offline analyses are not just one-time checks, but the interesting results can become part of the development feedback loop, guaranteeing that we do not regress on them in the future.

I've written some more about this workflow, in different terms, [in a past post](/post/property-testing-like-afl/), but I think the dichotomy explored here helps motivate it better than I have before. In practice, the best system I know of that works this way is Google’s [oss-fuzz](https://github.com/google/oss-fuzz); their [documentation on an ideal integration](https://google.github.io/oss-fuzz/advanced-topics/ideal-integration/) describes exactly this approach in their [section on regression testing.](https://google.github.io/oss-fuzz/advanced-topics/ideal-integration/#regression-testing)
