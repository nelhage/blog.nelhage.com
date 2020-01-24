---
title: "Reflections on Sorbet Performance"
date: 2020-01-23T16:36:55-08:00
draft: true
---


# Reflections
## Performance is a feature

I first want to reflect on the value of performance, as well as our experiences and the feedback we got about rolling out Sorbet.

We knew from the start that performance was going to be a key feature of Sorbet. We built Sorbet in part to help accelerate the feedback loop while developers at Stripe were writing code. Stripe has a very extensive test suite, which runs in its entirety in 10-15 minutes in CI. Individual tests can take between a few hundred milliseconds and tens of seconds to run in the development environment prior to pushing to CI. We knew that if we could make Sorbet substantially faster than those benchmarks, it would have a strong shot at being developers’ first destination for feedback about their code, so long as that feedback was even slightly useful. In the end, we achieved typechecking times of ~10s from a cold start in development, and times as low as a few milliseconds for our eventual IDE integration (which used a lot more engineering, on top of that discussed here, to efficiently re-calculate after incremental updates).

Furthermore, a *lot* of the positive feedback we got from early users was about the tool’s speed. Users enjoyed the benefits of static types and type annotations, but they enjoyed even more having a tool that caught typos and many other simple errors drastically faster than the tools they were using before.

Performance is a feature, and an often underinvested-in one. And, unlike many features, it is one that is hard to win back after the fact. Too often, I see software projects whose implicit performance goal seems to be “just fast enough;” while this is often an acceptable choice, I encourage you not to underestimate the value of aiming for “joyfully fast,” instead. This is also a lesson I’ve learned in my own [livegrep](https://livegrep.com/search/linux) project; users would often prefer a more semantic search to livegrep’s regular expressions, but livegrep’s ability to turn around results in <100ms means that users can engage interactively and refine and iterate on searches in a way that is frustrating with many other tools, dramatically increasing the value of the tool.

## Performance is the sum of many choices

There is no one “secret” to Sorbet’s performance. The choice of cache-optimized core data structures is perhaps the largest single decision, but overall our performance comes from consistent attention to a lot of details, and consistent attention to regressions while we develop.

Sorbet, like most compilers and typecheckers, has very few hot spots; time is divided relatively evenly between the tool’s major passes, and spread the code within those passes. This diffuse profile means that it would have been virtually impossible to start with a slow typechecker and make it fast by optimizing the hot spots, since there aren’t any. We might have been able to take a slow tool and make it fast_er_, but getting to “objectively fast” required continuous attention from the beginning.

One of my favorite performance anecdotes is the SQLite 3.8.7 release, which was [50% faster than the previous release](https://www.mail-archive.com/sqlite-users@mailinglists.sqlite.org/msg86235.html) in total, all by way of numerous stacked performance improvements, each gaining less than 1% individually. These experiences, and others, lead me to a lot of skepticism about what I see as the prevailing philosophy that emphasizes “write it, then profile it, then optimize the hot spot” and tends to gloss over small performance differences. This advice may be an adequate general stance, but it falls badly short when performance is truly a goal. And, per the previous bullet, I also believe that we tend to value performance less than we should in general.


## Sorbet is fast without caches

In addition to all the work discussed in this post, Sorbet makes use of persistent caches to speed up repeated runs, and Sorbet’s [LSP-based](https://langserver.org/) IDE server contains substantial additional engineering to efficiency compute incremental responses in most cases, allowing for incremental re-runs in as few as a few milliseconds.

However, I think it’s a really important feature that Sorbet doesn’t *require* these caches or these incremental updates to be usably fast. Because an uncached run on Stripe’s entire monorepo (Sorbet’s current largest use case, to my knowledge) only takes 10-15s, it’s essentially always acceptable to drop the caches or to bail out of the incremental pathway if doing so makes something else easier. As an example, Sorbet’s cache format is intentionally not stable, and each release will disregard caches from older versions and start anew. This choice costs our users a slight performance hit when we ship a new release, but still keeps them in the cached path during day-to-day operations. It also substantially simplifies the architecture, testing, and development of the caching code. For instance, it means we are free to optimize or redesign the cache without having to worry about compatibility or migration paths. The caching strategy is also fairly simplistic and coarse-grained, because the baseline performance is fast enough anyways.

Similarly, software that is overly-dependent on caches can encourage users to alter their behavior in order to stay inside the cached path. You can imagine a version of Sorbet that is fast unless you touch certain core source files, at which point it becomes, not just slower, but “get a cup of coffee” slow. This behavior would materially discourage users from making changes to those core files, and thereby slow down and distort the incentives for development at Stripe in an undesirable way.

Performance impacts architecture and behavior. By having an adequately fast baseline performance, Sorbet’s overall architecture can be simpler, since we don’t have to do elaborate bookkeeping to implement fine-grained incremental computation or cache invalidation. In this way — perhaps surprisingly — it can actually end up being simpler to write fast software in the long run, than to write slow software and be forced to continually add complexity to make up for it.
