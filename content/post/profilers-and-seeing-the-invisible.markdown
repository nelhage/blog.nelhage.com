---
title: "Performance engineering, profilers, and seeing the invisible"
slug: profilers-seeing-the-invisible
date: 2023-12-18T08:00:00-08:00
description: |
  A meditation on expertise: Exploring the uses and weaknesses of profilers and profiling, through the lens of Gary Klein's "Seeing the Invisible"
---
I was recently introduced to the paper "[Seeing the Invisible: Perceptual-Cognitive Aspects of Expertise](https://cmapspublic3.ihmc.us/rid=1G9NSY15K-N7MJMZ-LC5/SeeingTheInvisible.pdf)" by Gary Klein and Robert Hoffman. It’s excellent and I recommend you read it when you have a chance.

Klein and Hoffman discuss the ability of experts to “see what is not there”: in addition to observing data and cues that are present in the environment, experts perceive **implications** of these cues, such as the absence of expected or “typical” information, the typicality or atypicality of observed data, and likely/possible past and future time trajectories of a system based on a point-in-time snapshot or limited duration of observation.

I want to talk about some of the ideas of that piece in the specific context of performance engineering, and what profilers can and cannot do for you. In particular, I think this piece makes a great lens for discussing why profilers are both invaluable tools and yet also not the be-all and end-all of performance engineering.


## Profilers

I often encounter — sometimes implicitly — the meme that “all there is” to speeding up a piece of software is this loop:

1. Run a profiler
2. Identify the most time-consuming portions of the profile
3. Make them faster

In practice, though, when I or others attempt this loop as-written, it very rapidly reaches diminishing returns. The software will still be much slower than it seems like it “should” or “could” be (or just than we want), but it’s also not clear how to make much progress.

I see a number of reasons for this disconnect, and suspect one could write a dozen different pieces on the phenomenon, but with the above essay in mind, I’ll discuss the idea that:


> Profilers can only show you “what is there”; skillful performance engineering requires understanding the information that shapes and informs the profile but is "not there" in the profile itself.


## Seeing the invisible

What do I mean by this? What is information that “is there” in the profile, and what isn’t?

Profilers show you “what’s there:” they show you, very concretely and literally, where a particular execution of a program is spending time, usually in terms of functions or stack frames. This is, certainly, useful information.

However, it’s not enough! Sometimes you stumble on an obvious hot-spot caused by an obvious bug, but in many more cases — especially once a program has been optimized somewhat already — you need additional information to decide what actions to take or optimizations to attempt. Seeing where the time is spent isn’t enough; we also need to understand which time-spent is feasible to remove or reduce, and how much effort or risk may be involved.

As some concrete example: If I see that a program is spending 80% of its time doing hashtable lookups, does that mean that I should be optimizing my hash table, or redesigning it to use fewer lookup operations? The profile cannot directly tell me, but this is the kind of information an expert may perceive in the profile, by virtue of integrating their expertise and background knowledge, regardless.

As another angle: a profile tells you how much time a program spent in a given operation; but not how much time it would spend if you optimized that operation.

Sometimes an operation is slow because it represents the "real work" of the program and is fundamental work that can't easily be reduced, and sometimes it's slow for inessential reasons, and potentially could even be removed entirely. A profiler will tell you that the program is spending time in the operation, but not whether it's already been optimized well, or whether it's work that could be removed or shifted elsewhere. However, once again, an expert reading the profile may emerge with answers to those questions.

Even the “frame” of analysis that the profiler offers may not be the most useful frame in which to ask these questions. Profilers tend to organize information by time-order and/or grouped by function or stack frame; but often another organization would be much more productive. For example:

- Sometimes it’s more important to analyze time-spent in terms of I/O vs CPU time; a program may alternate doing some of each (or do both in parallel in varying ratios), and we want to break time down by various categories of I/O or CPU time, which may not neatly map to stack frames.
- Sometimes — a bit of a generalization of the above — the most useful axis is “which hardware resources is the program using, in which proportion?” Depending on the program, this opens up a world of possible perspectives:
    - Microarchitectural resources: CPU execution units, cache space, decoder capacity, etc
    - GPU vs CPU vs PCI or NVLink bus bandwidth
    - Ethernet bandwidth vs local disk bandwidth vs CPU throughput
    - And many, many others
- Sometimes we need to organize by another axis or organizational frame than the physical stack trace.


    A prototypical example here is trying to profile a Python program using a C-level profiler. In that case, you’ll typically find that most of the runtime is spent in `_PyEval_EvalFrame`. True, as far as it goes, but not very useful; instead, you need to “lift” your profile into the domain of the Python-level stack trace in order to gain useful understanding. (This is a problem shared by some automated tools of the form “code that operates on other code,” which [I’ve discussed a bit in the past](https://buttondown.email/nelhage/archive/tracing-jits-and-coverage-guided-fuzzers/)).


## The interplay with expertise

I think that Klein’s essay on expertise gives us some useful frames to understand this situation.

Experts, especially domain experts in a particular application or programming language or framework, can often look at a profiler and “see” the answers to many of the above questions, even thought they are not literally present in the profile.

They can “read between the lines” to ask and even answer the “right“ or “better” questions, instead of stopping at the questions literally answered by the profiler. Additionally, and importantly, when they can’t do so for one reason or another, experts will be much faster to **recognize** that that the profiler is asking the wrong questions, and find or build another tool, instead of continuing to bash their head against the same profile in the hopes that it will somehow yield an understanding it does not contain.

This perspective also explains, I think, some of the disconnect I see in the discourse and the common understanding of performance engineering. If you look at an expert performance engineer, you will, in fact, observe that they spend a lot of time looking at profiles; from this, it’s temping to conclude that this is, in some sense, the main thing that they are doing.

However, these expert performance engineers, on inspection, are *using profilers differently* than a novice does, or than a naive observer might expect. And that usage is driven by their ability to *see different information* in a profile! When they look at a profile, they are not just looking at the literal information displayed, but rather using that as one of several inputs to enhance and build their model of the program, and to update and evolve their mental model of the system, and to test and formulate hypotheses which they test,  through further optimization and further profiling runs (potentially with different profilers).
