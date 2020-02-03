---
slug: reflections-on-performance
title: "Reflections on software performance"
date: 2020-02-02T17:00:00-08:00
---
At this point in my career, I’ve worked on at least three projects where performance was a defining characteristic: [Livegrep](https://livegrep.com/search/linux), [Taktician](https://github.com/nelhage/taktician), and [Sorbet](https://sorbet.org/) (I [discussed sorbet](https://blog.nelhage.com/post/why-sorbet-is-fast/) in particular last time, and [livegrep in an earlier post](https://blog.nelhage.com/2015/02/regular-expression-search-with-suffix-arrays/)). I’ve also done a lot of other performance work on the tools I use, some of which ended up on my other blog, [Accidentally Quadratic](https://accidentallyquadratic.tumblr.com/).

In this post, I want to reflect on some of the lessons I’ve learned while writing performant software, and working with rather a lot more not-so-performant software.

## Performance is a feature

I’ve really come to appreciate that performance isn’t just some property of a tool independent from its functionality or its feature set. Performance — in particular, being notably fast — is a feature in and of its own right, which fundamentally alters how a tool is used and perceived.

One of the recurring pieces of feedback and appreciation we got from Stripe engineers following the Sorbet rollout was how *fast* the tool was. In a world of slow tooling[^0] developers genuinely appreciated a tool that felt fast and responsive.

[^0]: This is a place, in my opinion, where the Ruby ecosystem is especially egregious. For instance, Sorbet can typecheck Stripe’s codebase from a cold start faster than Ruby can *load* it, let alone execute any code.

I think the value of performance is sometimes well-understood intellectually — many engineers know of and talk about the [perceptual thresholds for response time](https://www.nngroup.com/articles/response-times-3-important-limits/) or [the impact of latency on conversion rates](https://blog.hubspot.com/marketing/page-load-time-conversion-rates) — but it’s rarely appreciated on a really visceral level, and also is often given lip service more so than real investment. It feels popular to bemoan slow software these days, and yet it also feels rare that a team is doing something about it — our tools still keep getting slower. My experiences lead me to the strong believe that while our tools do make it increasingly hard to write fast software, it’s very much still possible, and very much still worth it.

### Performance changes how users use software
It’s probably fairly intuitive that users prefer faster software, and will have a better experience performing a given task if the tools are faster rather than slower.

However, what is perhaps less apparent is that having faster tools *changes how users use a tool or perform a task*. Users almost always have multiple strategies available to pursue a goal — including deciding to work on something else entirely — and they will choose to user faster tools more and more frequently. Fast tools don’t just allow users to accomplish tasks faster; they allow users to accomplish entirely new types of tasks, in entirely new ways. I’ve seen this phenomenon clearly while working on both Sorbet and Livegrep:

With Sorbet, one of the primary goals of the project was to get users faster feedback about their code as they developed. Stripe has always maintained an extensive test suite with fairly high quality and code coverage, and whose run time is pretty consistently in the 10-15 minute range. While we hoped Sorbet would eliminate a few classes of tests and runtime errors, shrinking the test suite or getting additional safety in production was not a primary goal. Instead, a larger outcome we were hoping for was shrinking the feedback loop in development, and getting users actionable feedback on their code much faster than they were previously.

Many individual tests at Stripe would take 10-20s or more in the development environment. Because we succeeded at building Sorbet so that it could typecheck the entire codebase in that time window, it became the fastest way many developers could get decent feedback on their code, to check for basic typos, misused APIs, and other low-hanging classes of errors. Since it was their fastest option, we saw users reaching for Sorbet as their first line of checking their code fairly early on in our development and rollout. Getting even mediocre feedback and some confidence fast was much more important than anything that would take minutes.

Being fast also meant that users were (comparatively) more tolerant of false errors (Sorbet complaining about code that would not actually go wrong), as long as it was clear how to fix the error (e.g. adding a [`T.must`](https://sorbet.org/docs/nilable-types#tmust-asserting-that-something-is-not-nil)). If Sorbet had run with a similar runtime as our CI — 10 minutes or so — users would have had a very different relationship to it; it would have to provide very clear additional value on top of CI, and rarely require users to make changes only for Sorbet’s sake, since it would no longer have the advantage of providing early feedback to users.

Watching users use livegrep, I’ve seen a related upside from its performance. Livegrep aims to respond to most searches inside of 100ms, corresponding to [a well-documented threshold](https://www.nngroup.com/articles/response-times-3-important-limits/) at which a response appears “instantaneous” to most users. Because livegrep is so fast, users use it *interactively* in a way I’ve rarely seen people interact with other search engines: they enter an initial query, and then, if they get too many or too few results, they edit it based on the result list, which gets a new set of results, which they continue to refine or expand until they hit on the results they are seeking.

I built livegrep to use regular expressions largely as an interesting technical challenge — on paper, a syntax- or semantics-aware search sounds better for a lot of uses of Livegrep. However, I’ve never seen a syntax-aware tool that is nearly as fast as livegrep, and I’ve come to appreciate that the interactive nature of livegrep adds additional power, since it makes it cheap to iterate and refine your searches. And the fact that most engineers have some familiarity with regular expressions already, coupled with the ability to interactively experiment and explore and see results in real-time, makes it incredibly approachable in a way that a more-complex query syntax would not be.

## Performance needs effort throughout a project’s lifecycle

It seems increasingly common these days to not worry about performance at all, especially in the early days of a project. One hears advice like:

- "premature optimization is the root of all evil”
- “Make it work, then make it right, then make it fast.”
- “CPU time is always cheaper than an engineer’s time”

You hear similar sentiments from those who argue that Ruby or Python are [“fast enough;”](https://m.signalvnoise.com/ruby-has-been-fast-enough-for-13-years/) the idea seems to be that performance need only be an afterthought, and that code that works is far-and-away the bulk of the problem.

The general prevailing philosophy I see seems to be that one should first write an application in whatever form is most expedient, and only once it works, turn to a profiler and optimize individual hot spots, maybe even rewriting individual components into faster languages or technologies.

I think this may indeed be decent default advice, but I’ve also learned that it is really important to recognize its limitations, and to be able to reach for other paradigms when it matters. In particular, I’ve come to believe that the “performance last” model will rarely, if ever, produce truly fast software (and, as discussed above, I believe truly-fast software is a worthwhile target). I identify two main reasons why it doesn’t work:

### Architecture strongly impacts performance
The basic architecture of a system — the high-level structure, dataflow and organization — often has profound implications for performance. [As discussed last time](https://blog.nelhage.com/post/why-sorbet-is-fast/), one design decision we made in Sorbet was to only perform local type inference within methods. This decision had implications for the type system and the user experience, but also made Sorbet drastically simpler, faster, and easier to parallelize and incrementalize. This architectural decision was cheap to make up front, but would have been incredibly costly to change; the Flow team is in the middle of a [multi-year refactor](https://medium.com/flow-type/what-the-flow-team-has-been-up-to-54239c62004f) to move Flow and Facebook’s code base to a more local-only model. [^1]

If you want to build truly performant software, you need to at least keep performance in mind as you make early design and architectural decisions, lest you paint yourself into awkward corners later on.

[^1]: I want to be clear that I don’t criticize the Flow team for their choices. Their tool has been very successful inside and outside of Facebook, and they made the decisions they did for what I’m sure were excellent reasons at the time. And I admire their transparency in the linked blog post. I do, though, find the comparison quite illustrative about the impact of early architectural decisions.

### Performance isn’t just about hot spots
Sorbet, like most compilers and typecheckers, has very few hot spots; time is divided relatively evenly between the tool’s major passes, and spread among the code within those passes. This diffuse profile means that it would have been virtually impossible to start with a slow typechecker and make it fast by optimizing the hot spots, since there aren’t any.

Instead, a lot of the techniques I discussed last post — such as cache-optimized data structures, or writing in C++ — essentially make *every* line of code in the typechecker faster, in a way that adds up across the entire application.

One of my favorite performance anecdotes is the SQLite 3.8.7 release, which was [50% faster than the previous release](https://www.mail-archive.com/sqlite-users@mailinglists.sqlite.org/msg86235.html) in total, all by way of numerous stacked performance improvements, each gaining less than 1% individually. This example speaks to the benefit of worrying about small performance costs across the entire codebase; even if they are individually insignificant, they do add up. And while the SQLite developers were able to do this work after the fact, the more 1% regressions you can avoid in the first place, the easier this work is.


## Performant foundations simplify architecture

Another observation I’ve made is that starting with a performant core can ultimately drastically *simplify* the architecture of a software project, relative to a given level of functionality.

Before we wrote Sorbet, Stripe had some internal Ruby static analysis which performed a very limited subset of the analysis implemented by Sorbet (global constant resolution). This tooling was built in Ruby on top of the [Parser](https://github.com/whitequark/parser) gem, and was drastically slower than Sorbet, as well as being more challenging to parallelize because of Ruby’s GVL. In order to make it work, we ended up building an increasingly-complex caching solution using `fork`-based parallelization, just to keep it to an acceptable runtime. Sorbet was able to outperform this system without any caching, due to greatly superior baseline performance, and simple internal parallelism.

There’s a general philosophy here; attempts to add performance to a slow system often add complexity, in the form of complex caching, distributed systems, or additional bookkeeping for fine-grained incremental recomputation. These features add complexity and new classes of bugs, and also add overhead and make straight-line performance even worse, further increasing the problem.

When a tool is fast in the first place, these additional layers may be unnecessary to achieve acceptable overall performance, resulting in a system that is in net much simpler for a given level of performance.

In the case of Sorbet, while we did make use of some caches, and while Sorbet’s [LSP server](https://langserver.org/) has some complexity to perform incremental re-typechecking, both systems were drastically simpler than the ones in comparable typecheckers I am familiar with. We got away with this simplicity in large part because the base-case performance for Sorbet just isn’t that bad; Sorbet doesn’t have to go to extreme lengths to save work, because it’s often fast enough to just do the work instead.

One specific nice property is that Sorbet’s cache format has no forward- or backwards- compatibility: Sorbet releases known their git commit sha1, and will ignore cache files generated by a different version. This decision means that the cache format can be incredibly simple and minimal, and we never have to worry about data migrations or compatibility. The downside, of course, is that our users briefly see uncached performance every time we do a release, until the new caches are populated. Since the cache only saves a few seconds, and the cold-start performance is still < 20 seconds, we decided this was an acceptable tradeoff for the simplicity (and lower bug rate) it brought. In comparison, I have several friends who worked on the [MyPy](http://mypy-lang.org/) deployment at Dropbox, where a full uncached run there could take ten minutes or more. This drastic disparity meant they were facing an entirely different calculus, and had to worry a lot harder about when and whether it was OK to invalidate caches, adding complexity to their development process which we could entirely avoid.

As Stripe’s monorepo continues to grow, relying on Sorbet’s raw performance will work less and less well, and I expect Sorbet may eventually grow increasingly sophisticated caches and incremental execution; raw performance isn’t a panacea. However, *for a given level of scale*, it matters a lot, and we should not underestimate how helpful it was to be able to get to Sorbet’s level of successful deployment with a fundamentally simple design.


# Conclusion

I’ve really strongly come to believe that we underrate performance when designing and building software. We have become accustomed to casually giving up factors of two or ten or more with our choices of tools and libraries, without asking if the benefits are worth it.

Software can still be fast and responsive in 2020, and it’s very often well worth it to put in that effort. What’s your favorite software tool that’s *surprisingly* fast, in a way that makes it really pleasant to work with?
