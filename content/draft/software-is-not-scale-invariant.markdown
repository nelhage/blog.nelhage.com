---
title: "Software is not scale-invariant"
date: 2018-09-04T01:39:23Z
---

One of the most remarkable properties of software is its remarkable
ability to scale to enormous impacts and user bases, and the ability
for comparatively small teams to produce outside impact on millions or
even billions of users.

Because of this remarkable ability to achieve scale and leverage, and
because software at large scale looks superficially similar to
software at small scale, it's sometimes easy to assume that software,
and the processes are software-development, are largely
_scale-invariant_: they look basically the same at any scale, from
one-person projects to Google-scale applications. Experience, however,
suggests otherwise: the practices that support effective software
development and deployment vary substantially as you grow, and
techniques that make perfect sense for side projects or small startups
break down for larger teams and deployments.

This post -- informed in large part by my experiences at Stripe as it
has scaled from tens to thousands of employees -- aims to explore some
of the ways software development and operations "in the small" and "in
the large" require different strategies and approaches. I've come to
believe that many disagreements about tools or methodologies boil down
to disagrements between developers operating at or optimizing for
different scales. I hope that by making explicit some of the
distinctions I see, I can help frame at least a few such conversations
more productively.

# Operational scale

I'm going to discuss two different aspects of "scale" when talking
about software at scale. I'll start by considering some phenomena that
change as the size of a deployment increases -- as a software
application handles millions or billions of users, terabytes,
petabytes, or exabytes of data, and so on.

## Performance

Small performance or efficiency changes matter a lot at large
scale. If your application fits on a single machine, or even a cluster
of 10 machines, it's hard to get excited about a 1% improvement in
performance, or runtime, or memory usage. In many cases, it's hard to
even measure single-digit percentage differences in the noise at that
scale. And while surely anyone would accept a free 1% optimization,
it's just not going to materially change the cost of running the
application, the problems that are tractable to solve, or any other
high-level goals you may have.

At scale, however, small fractional gains manifest as large absolute
deltas, and tend to directly translate into significant cost or other
real differences. On a 100-node cluster, a 1% optimization frees up an
entire server. On a million-node cluster, a 1% improvement wins back
10,000 servers! On a petabyte dataset, a 1% gain in compression
efficiency saves 10TB. At these scales, it becomes worthwhile to
invest in entire projects and teams dedicated to shaving off small
efficiencies: they literally pay for themselves.

This distinction, I think, explains part of the arguments over whether
dynamic languages like Python are "fast enough". Java or Go can often
outperform Python by [10x or more][go-v-python], but at small scale it
often just doesn't matter much -- modern CPUs are fast, you can put
caches in front, and other concerns dominate. In a large deployment,
though, 10x means the difference between 100 servers and 10,000 for
the same traffic, which becomes a hard distinction to ignore: In
addition to the raw cost of paying for the hardware or VMs, operating
at a 10,000-server scale requires an entirely different level of
operational expertise and sophistication, bringing further
second-order costs.

[go-v-python]: https://benchmarksgame-team.pages.debian.net/benchmarksgame/faster/python3-go.html

## Correctness

Similarly, at large scale, rare events become common. If one in a
million requests cause some kind of internal data corruption that may
require manual intervention (maybe due to a
[feral concurrency scheme][feral]), an app handling 10qps will see a
few a week. Annoying, but well within the range of a human to debug
and fix. If the causes are non-obvious, it's tractable and perhaps
even desirable to just ignore and fix the bad ones as users complain.

[feral]: http://www.bailis.org/papers/feral-sigmod2015.pdf

At a mere 10,000 requests per second, though -- still small by the
scale of internet giants -- we're contemplating 100 incidents per day.
Intervening manually would require an entire specialized operations
team, and likely special tooling support (and inevitable errors as
humans perform this repetitive work). Thus, correctness, especially
correctness in rare edge cases and unlikely failure regimes, becomes
qualitatively more important as a deployment grows. This phenomenon,
in turn, has downstream implications on the technologies and
methodologies that make sense at scale; it's one reason that
[model checkers][aws-tla] and even more mundane technologies like
[aftermarket][hack] [type][mypy] [systems][flow] are
disproportionately popular at large organizations.

[aws-tla]: https://cacm.acm.org/magazines/2015/4/184701-how-amazon-web-services-uses-formal-methods/abstract
[hack]: https://hacklang.org/
[mypy]: http://mypy-lang.org/
[flow]: https://flow.org/

## Efficiency-scalability trade-offs.

Many architectural choices in software boil down to tradeoffs between
efficiency and scalability. Efficiency, here, means "the total amount
of work needed to perform a given task", while scalability means "how
easily that work can be distributed among additional threads or
servers". Some examples:

- Doing data analysis on a [single computer][COST] tends to be more
  efficient than using a distributed system; You don't pay network and
  coordination overhead. However, given a Python+pandas script for
  processing a CSV in-memory, it's not clear what to do once your
  dataset outgrows your single machine's RAM.
- Modern SQL databases are marvels of engineering, and by pushing work
  into the database -- using rich SQL queries and even PL/SQL or
  similar tools -- you can save on network overhead, and give the
  query planner maximum scope to find the most efficient solutions to
  queries. However, a database instance only has so many CPUs, and
  scaling databases is notoriously tricky. If, on the other hand, you
  treat the database as a dumb key-value store and offload most logic
  to the application tier, you pay more per-request for serialization
  and networking overhead, increased RPC rates, and potentially
  overfetching data to filter on the client. However, adding more CPU
  is as simple as resizing an EC2 autoscaling group or tweaking a
  kubernetes config.

Notwithstanding earlier remarks about efficiency, at large scales, one
often has no choice but to opt for solutions that emphasize
scalability. If your datasets or workloads simply won't fit onto
single systems, or if your growth rates are fast enough that ease of
scaling is vital concern, then it can be well worth taking a hit in
absolute efficiency in order to be able to easily allocate more
hardware to a problem, without thinking twice.

A small deployment, however, may fit on a single machine with
[orders of magnitude to spare][^single]. In that case, using
"scalability-first tools brings not only losses in efficiency, but
added complexity of managing these tools, and bring no practical
benefit. It is often preferable to emphasize solutions that optimize
for simplicity, and for keeping a small and lean fleet, instead of
preserving scalablity options that you will never need.

[COST]: https://www.usenix.org/conference/hotos15/workshop-program/presentation/mcsherry

[^single]: Keep in mind that single computers get [really big][x1] these days!
[x1]: https://aws.amazon.com/ec2/instance-types/x1/

# Development



## Keeping it all in your head
## Individual productivity vs scalability

# Conclusion
