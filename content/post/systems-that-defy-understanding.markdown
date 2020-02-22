---
title: "Systems that defy detailed understanding"
slug: systems-that-defy-understanding
date: 2020-02-22T12:00:00-08:00
---
Last week, I [wrote about](https://blog.nelhage.com/post/computers-can-be-understood/) the mindset that computer systems can be understood, and behaviors can be explained, if we’re willing to dig deep enough into the stack of abstractions our software is built atop. Some of the ensuing discussion on Twitter and elsewhere lead me to write this followup, in which I want to run through a few classes of systems where I’ve found pursuing in-detail understanding of the system wasn’t the right answer. In these systems, detailed understanding of behaviors was either impossible, impracticable, or just a waste of time compared to other strategies for dealing with the system.

# The systems
## Complex distributed systems

The defining characteristic of distributed systems is failure. Once your system is spread across multiple nodes, we face the possibility of one node failing but not another, or the network itself dropping, reordering, and delaying messages between nodes. The vast majority of complexity in distributed systems arises from this simple possibility.

Because of this possibility, it is a fundamental requirement of high-reliability distributed systems that they be *robust* to failure of individual components. Such systems tend to run multiple copies of a service, and be able to retry requests, or to handle failure and proceed using some fallback strategy.

When you’re working on and building out a distributed system, you will inevitably encounter situations where some underlying component fails (say, an individual database node crashes), and in which the system as a whole does not gracefully recover (say, the client hangs instead of timing out and retrying a healthy node). When this happens, you have, broadly, two choices for how to remediate the issue: you can try to understand and fix the underlying failure (why did the node crash?), or you can try to fix the failure in the fault-tolerance layer (why didn’t the client time out?)[^priority].

I faced this sort of dilemma many times during my years on infrastructure teams at Stripe. In these cases, my systems debugging experience and my desire to understand systems tempted me towards wanting to understand what looked like the “root” case of the incident: why did the database crash? Could we track down the bug and report it upstream or fix it? Whether it was a bug in the database software itself, a kernel bug, or even a failure in the underlying cloud hardware, I was *sure* I could track it down. I ended up spending a lot of times digging through unfamiliar C++ code, scouring logs, and tracking down kernel bugs to understand various failures. While I did end up identifying a fair number of bugs (here are [two](https://nelhagedebugsshit.tumblr.com/post/140317144518/when-observing-your-database-stalls-it) [examples](https://nelhagedebugsshit.tumblr.com/post/82612335570/slow-mongo-foreground-index-builds)), it was always a slow and inconsistent process, and many others were eventually written off as unsolved mysteries.

Tracing behavior in detail in these systems is so hard for at least two reasons. The first is the sheer complexity and amount of code involved. Modern cloud deployments involve numerous different interacting components, most of which you didn’t write. A “relatively simple” system might touch Docker, Kubernetes, Kafka, Redis, and a hosted AWS service or three; the number and lines of code of dependencies easily drastically outpaces the dependencies of even a very sophisticated single-node program. All of these systems and lines of code add complexity that might potentially need to be understood to track down behavior.

The second reason, compounding the first, is the massively concurrent and asynchronous nature of distributed systems. You potentially have thousands of processes or more exchanging messages with a server, and an attempt to reconstruct a precise failure timeline might depend on the precise ordering or relative ordering of any or all of their requests, as well as any other asynchronous events running on the server. This is one of those cases where the “mostly-deterministic” nature of computers that I cited last time breaks down; concurrent execution leads to (effectively) random message ordering, massively exploding the space of possible traces you may have to consider.

### What works instead
As I came to see the “understand everything!” approach as largely misguided here, I started to conclude that it was usually a better use of my time to focus effort the systems-level failure, instead of the individual component failure.

*Given* that the individual node failed, how could the system as a whole react more robustly? Did we need to add or adjust timeouts or retries? Add health-checking or circuit-breaking or backpressure or fallbacks? Remove that dependency entirely somehow?

Such solutions may add complexity and new failure modes of their own. However, if you aspire to high reliability you need some such solutions regardless, because you aspire to be more reliable than any individual server or process, anyways.

If you do succeed in designing the system as a whole for resilience, then individual failures become much less important, and the details matter less — you can survive a broad class of individual failures, and diagnose them later, or even not at all. If a process keeps failing in the same way you can choose to investigate and debug (with the confidence that the damage is at least being mitigated while you do). And for one-off failures, your reaction can potentially just become “Well, that was weird. Guess I’ll replace the node and investigate if it ever happens again.” It’s a tremendously freeing feeling to get there!

[^priority]: Ideally you would probably try to address both, but given finite time and resources you often end up having to prioritize one of the two workstreams.


## Big balls of mud

Will Larson [writes about big balls of mud](https://lethain.com/reasoning-about-big-balls-of-mud/), and observes:

> Big balls of mud appear to have properties, but they don't. All mutation occurs through an immutable log, you say. Oh, but there are a few caveats. A couple of performance wins. Two adjacent products made a minor exception to hit launch timelines.
> …
> No properties remain, and abstract models of system behavior skew from actual behavior.

My experience here agrees with Will’s : When a system gets big enough fast enough, with a large enough team and under lots of competing product pressures, it risks losing nearly all internal logic and guarantees.

Last time I wrote

> Computer systems tend to be, well, systematic, and have some core algebra or logic that is comprehensible, which is smaller than a complete list of possible behaviors, and which generates or at least organizes all of those behaviors.

“Big balls of mud” like those Will describes don’t work this way. Insofar as any features work (and usually these systems *do* mostly work; among other reasons, by hypothesis they’ve survived to become popular), they work almost “by chance,” often due to specific pairwise interactions between distant parts of the codebase, and not because of any comprehensible layering or core invariants.

### What works instead
In such systems, we must resort to empirical methods. Instead of reasoning about the system and reading the source to answer questions, we find ways to ask questions about the running system. We can perform such queries by looking at existing logs and metrics, or by adding new instrumentation. Given, for instance, a `# this can never happen` comment, we might add a new log line or event emission at that point, deploy the system, and observe empirically if it happens and when, instead of trying to reason our way through the system.

In order to improve productivity on such systems, we invest in sophisticated [observability](https://docs.honeycomb.io/learning-about-observability/intro-to-observability/) tools, aiming to increase the number of questions we can ask without deploying custom code. We report high-granularity events on each request, and build systems like continuous runtime code coverage, so that we can observe many patterns of behavior in production just by writing the right query, instead of requiring *ad hoc* instrumentation and deploys.

It’s important to note that such systems can still be modeled, to a degree — typically they do have design properties which still *mostly* hold — but you must remain constantly aware of the distinction between the map and the territory. Mental models can be used to gain intuition and to form hypotheses, but these hypotheses absolutely must be empirically verified before they can be relied upon.

## Client-side Javascript

This example is the area I have the least personal experience with, but I’ve dabbled in it and worked adjacent to some very sophisticated teams who worked in this space.

If you run an even-moderately-sophisticated web application and install client-side error reporting for Javascript errors, it’s a well-known phenomenon that you will receive a deluge of weird and incomprehensible errors from your application, many of which appear to you to be utterly nonsensical or impossible. Sentry for instance, [has this blog post](https://blog.sentry.io/2017/03/27/tips-for-reducing-javascript-error-noise/) documenting techniques for trying to filter out the worst of it.

These errors come, in large part, from users running odd niche or out-of-date browsers with broken Javascript or DOM implementations, users with buggy browser extensions injecting themselves into your scope, adblockers blocking one of your `<script>`  tags, broken browser caches or middleboxes, and other weirder and more exotic failure modes.

These failures are, individually, mostly comprehensible! You can figure out which browser the report comes from, triage which extensions might be implicated, understand the interactions and identify the failure and a specific workaround. Much of the time.

However, doing that work is, in most cases, just a colossal waste of effort; you’ll often see any individual error once or twice, and by the time you track it down and understand it, you’ll see three new ones from users in different weird predicaments. The ecosystem is just too heterogenous and fast-changing for deep understanding of individual issues to be worth it as a primary strategy.

### What works instead
Past some point, I think you have to treat this largely as a statistical problem. Attempt to group errors together, and only investigate patterns that appear above some rate, or are trending in notable ways. Collect statistics on your users’ environments, and add sufficiently-common browsers or extensions to your pre-release testing processes.

For persistent issues that you can’t reproduce, you can resort to empirical methods; push code that queries facts about the environment and reports statistics back, or just guess at possible fixes, push them, and watch your statistics to see if they put a dent in the error rates. Push out deploys gradually and watch error rates in the wild instead of relying solely on your in-house testing.

These kinds of techniques are also used by browser teams themselves. Browsers, too, run in such heterogenous environments and at such scale that it’s just not feasible for their developers to understand all of the complexity of every environment they run in; they, too, must resort to experimentation, statistical summaries, and other empirical methods to understand the deployed behavior of their software. To this end, modern browser teams all have sophisticated tools for [controlling experiments remotely](https://chromium.googlesource.com/chromium/src/+/master/docs/configuration.md#features), [collecting privacy-preserving metrics](https://www.chromium.org/developers/design-documents/rappor) from users, and numerous other tools for empirical discovery.

This attitude was concisely expressed on Twitter by Adrienne Porter Felt, a long-time developer and manager on the Chrome team:

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Software seems like something we should be able to reason about, yet the reality is that it&#39;s often too complex. Since we don&#39;t know how it works, we measure it and experiment on it as if we are trying to discover properties of the natural world</p>&mdash; Adrienne Porter Felt (@__apf__) <a href="https://twitter.com/__apf__/status/1089877319123005440?ref_src=twsrc%5Etfw">January 28, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>


## Deep Learning

A few years ago, during a three-month sabbatical, I spent some time playing with deep learning toolkits and algorithms. I was particularly interested in reinforcement learning, so after reading a bunch of papers and blog posts, I wrote a toy implementation of an actor-critic system to play pong on an emulated Atari, a pretty [well-known task for deep reinforcement learning](https://deepmind.com/research/publications/playing-atari-deep-reinforcement-learning).

My system would make progress and I could watch its progress improving slowly … for a little while, and then it would diverge, with all of my parameters going to infinity and everything blowing up.

I spent literal months (on and off) trying to debug that ~200-line program. My experience with systems engineering lead me to go deeper into trying to understand its behavior, adding more and more metrics and summaries to my [tensorboard](https://www.tensorflow.org/tensorboard) setup, trying to comb over parameters and gradients and potentials step by step, hoping that maybe I could find the first moment where things started going wrong, and thereby somehow deduce where my problem was.

Eventually — somehow — I found the bug, mostly by accident: I had an [off-by-one error](https://github.com/nelhage/tf-experiments/commit/73be3cb581ac2e363be14995e8637b916bae7bb9#diff-966771c07ddf7937a3c6927b86e0f62e) when feeding frames into the training step. Because deep neural networks are so damn good at learning, they were still able to extract signal from the misaligned frames and start learning; but eventually the bad data won out and the system went off the rails. Fixing that data error resolved the problem and the network continued learning much longer and more effectively.

This experience taught me some really important lessons about deep learning. While working with deep learning “looks” like other programming in a lot of ways — certainly you’re writing and running programs — it’s also deeply different in a lot of important and practical ways. For one, *actually no one really understands* why these deep learning systems work. There are certainly many people with much better understanding and intuition than I have, but ultimately trying to understand these systems’ behavior too closely, especially as a novice to the field, is deeply the wrong way to approach it.

### What works instead
The kind of approach I settled on instead — which some friends who work professionally in deep learning echoed — is a far more incrementalist, empirical approach. Start with the simplest possible version of a system, demonstrate that it works as a baseline, and iterate in small steps from there. Make small changes, verify that it still works, and repeats. If it ever stops working, you know it was the most-recent change, and you can try to figure out what went wrong. If you’re reimplementing a paper or published algorithm, try to find their code and first make sure you are matching their behavior as precisely as possible, and only then try to modify from there.

Oh, and also double- and triple-check your data and your data infrastructure! This part of a system is “just plain-old code,” and your usual engineering strategies work, so double down on it. Write lots of assertions and checks that you’re feeding the right data in to your model and sanity-check it. Once you hit the neural networks errors become incredibly hard to trace, so try to enforce as much rigor as you can before then.

# Conclusion

Looking over these systems, I see some common themes. A lot of these systems are distributed, involving many different machines and interacting pieces of software. They may also be heterogenous, involving different implementations of a protocol or different versions of software running side-by-side. Relatedly, many of these systems have many participants and high rates of change, which means knowledge of the system has a shorter half-life of validity; in such systems, high-level architectural knowledge may remain valid, but the details are less worth learning since they’re subject to frequent change.

For the solutions, a big theme — also cited by several of the people I link to — is moving to empiricism and experiment, instead of abstract reasoning. I think this need additionally explains some of the recent [observability](https://docs.honeycomb.io/learning-about-observability/intro-to-observability/) movement; as we rely more and more on empirical observation of our systems, we need better and better tools for actually making and analyzing observations and characterizing the empirical behaviors.

Reflecting on both this post and the previous one, I want to call out pretty explicitly that I think there’s a lot of value in holding both mindsets as tools available for you, depending on the problems and needs of the situation. It’s both the case that computers are (mostly) comprehensible and can be understood in depth, and also the case that many computer systems are too complex or fast-moving for in-depth understanding, and are best managed by experiment and finding ways to ask the system itself questions about its actual behavior. These techniques can and should also be combined: Theoretical understanding of a system can inform which experiments and observations to perform, and observation refine and feed into mental models. Learning to master and deploy each mindset individually is an important step on the journey of a senior engineer; learning how to effectively switch between them and merge them when faced with a problem is a further step, and one I'm still getting better at.
