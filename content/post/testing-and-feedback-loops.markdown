---
title: "Testing and feedback loops"
slug: "testing-and-feedback-loops"
date: 2020-01-19T12:00:00-08:00
---
# Testing and feedback loops
This post tries to set out one mental model I have for thinking about testing and the purpose testing serves in software engineering, and to explore some of the suggestions of this model.

As mentioned in [an earlier post](https://blog.nelhage.com/post/two-kinds-of-testing/), I think a lot about working in long-lived software projects that are undergoing a lot of development and change. The goal when working on these projects is not just to produce a useful artifact at one time, but to maintain and evolve the project over time, optimizing for some combination of the present usefulness of the software, and our ability to continue to evolve and improve it into the future.

Whenever we’re analyzing a system that evolves over time, a key question to understanding its behavior is to consider what [feedback loops](https://en.wikipedia.org/wiki/Feedback) are in effect governing its behavior. I think of testing and testing practices — when deployed well — as some the most important feedback loops maintaining the health of a software project as it evolves over time. Good testing practices help ensure that as our software evolves it tends to get and remain more correct over time, instead of randomly wandering around behavior-space accruing new features, new bugs, and re-gaining old bugs.


## Testing as a ratchet

The key practice that makes this model work is reliably adding regression tests whenever we find bugs. Consider a system where the following properties hold:

- The desired behavior of the system, or at least of a meaningful subset of the system, is relatively constant over time.
- The system has a test suite, and we enforce via CI and other practices that these tests pass on every change.
- Every time we find and fix a bug in the system, we add a regression test demonstrating the bug and ensuring that it doesn’t re-occur.

In such a system, these testing practices will ensure that as time passes and development happens, we are constantly growing the number of “interesting” edge cases which are tested, and the number of bugs which we have (with some confidence) ensured cannot be re-introduced. The system will thus tend towards behaving correctly in an ever-larger fraction of cases, and we will we also have confidence in this fact, since the test suite is continually checking it.

It’s worth being clear that this is a rough mental model, and not any kind of rigorous proof; we can imagine this construction going wrong in a lot of ways. For one, the set of test cases really only generalizes if the system’s behavior is “smooth” in some way, and we can test independent features mostly-independently. We can imagine systems where, for whatever reason, a test case only gives us confidence about that particular case, and not about some broader category of similar cases. Additionally, the first property above is important; experimental projects where we are continually redefining the very goal to be solved don’t benefit from testing in nearly the same way.

However, I find it a helpful model to think about, and it helps motivate at least some of my interest in testing. I also find this models rings particularly true when I compare well-tested systems I’ve worked on to the less-well-tested ones, where it often feels like every attempted bug fix causes a regression in some other part of the system.

### A prototypical example
My prototypical example for a system that can exhibit this feedback loop is a parser for some complex or underspecified file format.

Let’s suppose we are writing a new [Markdown](https://en.wikipedia.org/wiki/Markdown) implementation for some reason. Markdown is a vague and poorly-specified format, with lots of conflicting interpretations in how to implement it. If we try to write an implementation, we’re almost guaranteed to miss some edge cases or make some decisions that turn out not to work well for some use cases in the wild.

However, this problem is very amenable to testing. We can create a test suite that lets us drop in example `.md` files, and annotate the expected behavior, perhaps as output HTML or a formatted Markdown AST. Given this test harness, we can start with an initial implementation, and then try it out on real pieces of markdown text in the wild, or wait for bug reports from early users. Whenever we find some input where our behavior is buggy (or even defensible but undesirable), we can add it to the test corpus and iterate on the implementation until it passes along with all of our existing tests. If we do this every time we find a buggy case, we gain confidence that *over time* our implementation will grow to handle every edge case we care about.

## Implications of this model

### Find stable sub-problems
In my experience, when you find a problem or a sub-problem that has this desirable feedback loop, that module tends to become one of those desirable pieces of code that you mostly don’t have to think about too often. You may have to update it to handle new behaviors or fix the occasional bug, but they rarely become [haunted forests](https://john-millikin.com/sre-school/no-haunted-forests) that engineers quail to touch.

Because these components are so desirable, it’s worth giving thought in your architecture to sub-components where you can encourage this desirable behavior, even if you can’t for the application as a whole for any reason. Are there internal components that can be factored out to have (relatively) stable interfaces and good test coverage? Even if the overall feature set is changing and evolving, are there some core features that are stable over time you can try to lock down in this way?

I also find this feedback loop really valuable when writing [fakes](https://blog.nelhage.com/2016/12/how-i-test/#write-lots-of-fakes) for other components or external services to test against. There is always a risk with a fake that it does not accurate reflect the behavior of the faked system, and thereby gives you false confidence in the system under test.

However, the fakes *themselves* are amenable to this sort of feedback loop. Write tests for your fakes (or use your normal test, documenting the behavior in the fake that they rely on). Every time you discover a relevant discrepancy between the fake and the production behavior, update the fake and the tests in turn. Over time, your fake will converge on accurately covering all of the edge cases you care about, and becoming an ever-more-powerful tool for future tests.

### Invest in regression testing
This feedback loop is critically dependent on consistently adding regression tests when we fix bugs. If we don’t do so, we have little protection against continually introducing and fixing the same (or very similar) bugs, and we have little hope of achieving ever-growing correctness.

This suggests to me that, when designing a test suite and testing practices, we often want to invest in all the *future* tests that we will write, as much as the tests we’re writing now. In my experience, we can encourage this behavior in at least two ways:

- We can make it an explicit part of our engineering and code-review culture. Make it clear that reviewing the tests is a key part of reviewing a pull request, and that reviewers are also expected to think about which tests a change might benefit from, and to review tests that were added for coverage and appropriateness. In some projects, I’ve found it more valuable to review the added tests when reviewing a pull request than to review the change to the application code. If the demonstrated behaviors are desirable and improvements, I can worry less about the implementation details, since I know very concretely what impact they have, at least in the tested instances. I can also have confidence that we can always change the implementation in the future, with the tests guiding us as to which behaviors to preserve.
- Support tooling and test-suite design to make adding tests very straightforward and low-cost. I really love [table-driven testing](https://github.com/golang/go/wiki/TableDrivenTests), [record/replay tests](https://blog.nelhage.com/post/record-replay-in-sorbet/) like I described for Sorbet, and [“zoo” tests](http://design-a-testing-minilanguage-or-zoo) for this reason. It’s also really valuable to be able to take bugs encountered in production and translate them into test cases in a straightforward manner. In Sorbet, using bare `.rb` files as input meant we could often add reproducers from bug reports to the test corpus directly as-is, which is about as easy as you can get. Consider investing in tooling to let you near-directly copy production traces into your test suite, be they HTTP request logs, a data file in S3, or a user bug report. If production data is sensitive or too large to copy directly, consider building automated [minimization](https://embed.cs.utah.edu/creduce/) or redaction tools to make life easier.

### Consider whether to optimize for the present or the future
Having a feedback loop that drives a project towards correctness is not the same as having a correct project. One distinction that this mental framework drives me towards thinking about is the question of how important it is for a project to be correct now versus whether it is sufficient for it to be “fairly correct” and improve over time.

For some projects, such as [safety-critical software](https://alexgaynor.net/2019/nov/07/on-safety-critical-software/), it’s important that every release be as correct as we can make it (within some time or monetary budget). For many others, though, the cost of a few bugs up-front is acceptable, especially if we commit to setting up practices and infrastructure to fix them and ensure they stay fixed efficiently.

For the former types of problems, we need to make a lot of up-front investment in testing and correctness, and it’s likely worth bringing out some of the “big guns” of software correctness, such as formal methods, symbolic execution, large-scale coverage-guided fuzzing, and so on. For the latter category, though, it’s often faster and cheaper to invest in building the feedback loops I’ve described, which will ensure that we (with any luck) only see any given bug report once, and then to let early users and our production environment do our bug-finding.

One particular case for almost any project where it falls into the second camp is during the earliest development of a project,  before it has any users or any appreciable number of users. When I’m writing a program for the first time, I almost never worry about test coverage early on; instead I worry about setting up infrastructure that lets me cheaply capture any bugs I do find as test cases, and rely on later testing to increase coverage.

Once it comes time for a release, or if I later need to care more about correctness, I can use coverage metrics to guide which tests I add to the corpus, or developer fuzzers and other tooling to automatically find bugs, which I then also add to the corpus to prevent the bugs from re-occurring.

# Conclusion

The more I work professionally as a software engineer, the more I appreciate that the hardest problems in software engineering aren’t about getting a system to work once, but about dealing with our own past choices and with evolving needs, teams, and problem domains. I hope this post has helped to lay out one of the roles I see testing as playing in this challenge, and given a taste of why I care so deeply about testing in software projects.
