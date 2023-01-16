---
title: "Efficiency trades off against resiliency"
slug: efficiency-vs-resiliency
date: 2022-12-24T18:18:36-04:00
draft: false
---

What's the "right" level of CPU utilization for a server? If you look at a monitoring dashboard from a well-designed and well-run service, what CPU utilization should we hope to see, averaged over a day or two?

It's a very general question, and it's not clear it should have a single answer. For a long time, however, I had the vaguely felt conviction that higher must be better: we should aim for as close to 100% utilization as is practical on our servers. The logic is simple: Anything less than 100% represents unused hardware capacity, which means, in some sense, that we're wasting money. If a service isn't maxing out its CPU, we could move it onto a smaller instance, save some money, and contine to operate just fine.

This simplistic intuition, it turns out, is rarely quite right.

Suppose we achieve that ideal and our service is running close to 100% utilization. What happens now if we go viral somewhere and receive an unexpected burst of traffic? Or if we want to deploy a new feature which takes a little bit of extra CPU on each request?

If you were at 100% utilization, and then something happens to increase load, then we're in trouble! By running at 100% utilization, we leave ourselves no room to absorb incremental load. We will either degrade in some way, or have to scramble to add emergency capacity.

This toy example is an instance of a very general phenomenon: At some frontier, improvements to effiency almost always trade off against resiliency.

Past some point, making a system more efficient will mean making it less robust to change or disruption, and building in robustness tends to make a system less efficient. This isn't to say there are *no* win/wins; sometimes it is possible to move the [Pareto frontier][pareto] outwards, such as by fixing dumb performance bugs. But past some level of effort, you will be forced to make tradeoffs.

[pareto]: https://en.wikipedia.org/wiki/Pareto_front

## Other instances

This pattern occurs in domains beyond capacity planning. It applies at almost every level of almost any technical system or organization. Here's some other places where I've encountered this tradeoff:

### Redundancy

It's extremely common to run multiple instances of a service, with a load balancer set up such that if any instance fails, load will be transparently routed to the others. In sufficiently mature organizations, this pattern is even applied at the datacenter or region level, with an architecture that allows entire datacenters to fail and other datacenters to pick up the load.

In order for this to work, each instance must have enough spare capacity to absorb the incremental load from a failed-over instance. In steady-state, in the absence of failure, that capacity must be sitting idle, or, at best, serving lower-priority work that can be dropped at a moment's notice. The more redundancy we want, the more capacity we must hold idle in steady state.

### Optimization

Advanced performance optimizations often work by exploiting specific properties of the problem domain. By baking invariants into data structures and code organization, you can often win a huge amount of performance. However, the more deeply you rely on specific assumptions, the harder they become to change, which makes heavily-optimized code often much harder to evolve or add features to.

As a concrete example, in my [reflection on Sorbet][sorbet-fast], I talk about how we decided that type inference would be local-only and single-pass, and baked that assumption into our code and data structures. This choice resulted in substantial efficiency gains, but made the system in some sense more brittle: There are substantial classes of features one might potentially want which would be impossible or prohibitive to implement in the Sorbet codebase. I remain confident this was the correct choice for that project, but the tradeoffs are worth ackowledging.

Hillel Wayne, [in a blog post][clever], refers to a similar property as "clever code", which he defines as

> code which exploits knowledge about the problem.

He, too, talks about how clever code (in this sense) tends to be efficient but sometimes fragile.

[sorbet-fast]: https://blog.nelhage.com/post/why-sorbet-is-fast/#local-only-inference
[clever]: https://www.hillelwayne.com/post/cleverness/


### Distributed systems

One of my favorite systems papers ever is the [COST][cost] paper, which examples a number of big-data platforms, and observes that many of them have the desirable property of scaling (near-)linearly with available hardware, but do so at the cost of being *ludicrously* less efficient than a tuned single-threaded implementation.

This is a common tradeoff in my experience. Distributed computation frameworks are flexible and resilient in the sense of being able to handle near-arbitrary workloads by scaling up.  They can handle someone deploying inefficient code by scaling out, or handle hardware failures transparently. On the flip side, carefully-coded single-node solutions will tend to be faster (sometimes 10-100x faster!), but are much more brittle: If the dataset no longer fits on one node, or if you need to peform a 10x more expensive analysis, or if a new engineer on the team unwittingly commits slow code inside a tight inner loop, the entire system can become unsable.

[cost]: https://www.usenix.org/system/files/conference/hotos15/hotos15-paper-mcsherry.pdf


### Small teams vs larger organizations

Small teams -- including "teams" of one person -- can be incredibly productive and efficient. The smaller the team, the less communication overhead, and the easier it is to keep shared state inside of every engineer's head. There's less need to write documentation, less need to communicate changes, less time spend onboarding and training new members, and so on. Small teams can get further on "think carefully and try hard" (although not unboundedly far!) than large teams, and so often need to rely less on tooling like elaborate lint rules and other enforcement tools to ensure that invariants and style norms are respected.

Under the right circumstances, it's sometimes possible for a small team with careful design and experienced engineers to roughly match the raw output of teams 10x their size -- an enormous improvement in efficiency!

Small teams, however, can also be much more brittle and less resilient to changing business needs or technology landscapes. If one person departs from a 4-person team, you're down by 25% of your bandwidth -- and hiring and training a new person may be challenging, as the team is not practiced in doing so, and large amounts of knowledge live only in the engineers' heads. If a market change or business pivot resuls in a tool or product supported by a small team suddenly needing to grow massively in scale or scope, you may exceed the team's capacity, and growing the team will be challenging for all the same reasons.

### Automation versus human processes

In general, it's more efficient -- cheaper, faster, and often more reliable -- for a machien to automate a task than for a human to perform it manually.

Humans, however, are endlessly [adaptable][adaptable], while machines (both physical machines and software systems) are brittle and set in their ways. A system with humans in the loop has many more options for responding on the fly to changes in circumstances or unexpected events.

Even if we keep all the same humans around but augment them with automation to accelerate parts of their job, we risk [automation dependency][dependency], in which the humans become overly reliant on the automation and trust inappropriately, or where their ability to function without the automation atrophies such that they are no longer able to appropriately step in "on manual" when needed.

[adaptable]: https://how.complexsystems.fail/#12
[dependency]: https://www.aopa.org/news-and-media/all-news/2013/september/01/proficient-pilot-automation-dependency

### Slack

Many of these specific observations are really observations about [slack][slack] (not the software product).

Systems with healthy amounts of slack are -- at least in the short term, on simple measures -- by definition inefficient: That slack is time or resources that are going "idle" and which could have been spent producing more output.

Taking a broader view, though, that slack is key to resiliency: having "give" in the system lets the system cope with small disruptions or mishaps; developers or operators can use that slack to step in and handle unexpected load or improve underlying systems to fix issues before they become apparent. A system with no slack is efficient as long it works, but brittle and will break down quickly in the presence of any changes to its usual operating modes.

[slack]: https://www.amazon.com/Slack-Getting-Burnout-Busywork-Efficiency/dp/0767907698


# Conclusions

I believe it's very widely true that efficiency and reliability trade off against each other; this is true both as an absolute constraint at some point, but also as a general rule even for systems where neither goal has been the subject to extreme optimization pressure.

Unfortunately, this observation alone rarely tells us what to do with any _particular_ system. Is some concrete system we're looking at inefficient because of dysfunction or bad design or poor choices, or because it's harboring vast reserves of redundancy and flexibility and slack that will enable it to weather any change that may come? We cannot say without closer study and specific domain expertise.

Futhermore, sometimes there _are_ free lunches. Some design choices or decisions do shift the Pareto front outwards, and not just move us along it. They may be rare, but we should not ignore their possibility.

In addition, the optimal point in the design space varies depending on the system. Sometimes reliability or resiliency is critically important, and we are right to tolerate massive (seeming) inefficiency. Sometimes, however, extreme efficiency is the correct goal: perhaps our margins are thin enough for it to be our only choice, or perhaps we are sufficiently confident in the stability of the domain and of the demands upon the system that resiliency matters less.

Above all else, I would call for us, as engineers and designers and observers of systems, to work with an _awareness_ of this tradeoff and its implications. When we accuse a system of being wasteful and inefficient, it's worth pausing to ask what that "waste" may be buying. When we set out to optimize a system, pause to understand where the joints and flexibility in the current system are, and which ones are vital, and to do our best to preserve those. When we set metrics or goals for a system or a team or an organization and ask for efficiency, let us be aware that we will most likely get efficiency, but very little resiliency, unless we find a way to ask for it, too.

<!--

- hillel cleverness blog post?
- define "resiliency"?

- one-person vs team
- optimized code vs naive
- C++ w/ tightly-packed data structures vs Python
- microservices?
-->
