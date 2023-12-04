---
title: "Profilers, performance engineering, and seeing the invisible"
slug: profiling-the-invisible
date: 2023-12-03T17:46:28-08:00
---
I was recently introduced to the paper “[Seeing the Invisible: Perceptual-Cognitive Aspects of Expertise](https://cmapspublic3.ihmc.us/rid=1G9NSY15K-N7MJMZ-LC5/SeeingTheInvisible.pdf) “ by Gary Klein and Robert Hoffman; it’s excellent and you should read it when you have a chance.

Klein and Hoffman discuss the ability of experts to “see what is not there”: in addition to observing data and cues that are present in the environment, experts (often implicitly or subconsciously) perceive **implications** of these cues, such as the absence of expected or “typical” information, the typicality or atypicality of observed data, and likely/possible past and future time trajectories of a system based on a point-in-time snapshot or limited duration of observation.

I want to talk about some of the observations in that piece in the specific context of performance engineering, and what profilers can and cannot do for you. In particular, I think this piece makes a great lens for discussing why profilers are both invaluable tools and yet also not the be-all and end-all of performance engineering.


## Profilers

In particular, I often encounter — sometimes implicitly — the meme that “all there is” to speeding up a piece of software is this loop:

1. Run a profiler
2. Identify the slow parts of the profile
3. Make them faster

In practice, though, when I or others attempt this loop as-written, it very rapidly reaches diminishing returns. The software will still be much slower than it seems like it “should” or “could” be (or just than we want), but it’s also not clear how to make much progress.

I see a number of reasons for this disconnect, and one could write a dozen pieces on the phenomenon, but with the above essay in mind, I’ll discuss the idea that:


> Profilers can only show you “what is there”; doing performance engineering skillfully requires seeing what is “not there” in the profile.


## Seeing the invisible

What do I mean by this? What is information that “is there” in the profile, and what isn’t?

Profilers show you “what’s there:” they show you, very concretely and literally, where a particular execution of a program is spending time, usually in terms of functions or stack frames. To be sure, this is very useful information.

However, it’s not enough! To be sure, sometimes you stumble on an obvious hot-spot caused by an obvious bug, but in many cases — especially in a program that’s already been optimized to some degree — you need additional information to decide what actions to take or optimizations to attempt. Seeing where the time is spent isn’t enough; we also need to understand which time-spent is feasible to remove or reduce, and how much effort or risk may be involved.

When I look at a section of a profile or trace, in addition to the information that is present, I find myself asking some version of two related questions:

- **Why** is the program spending time here?
- **How much time** “could” or “should” the program be spending here, given different optimizations or implementation choices?

Both of those are critical to make decisions about whether an operation is worth trying to optimize, but neither of them is, in general, directly visible”in the output of a profiler! A performance engineer needs a way of answering those questions in order to make progress past just plucking low-hanging fruit.

Even the “frame” of analysis that the profiler offers may not be the most useful frame in which to ask these questions. Profilers tend to organize information by time-order and/or grouped by function or stack frame; but often there’s another organization that would be much more productive. Some examples:

- Sometimes it’s more important to analyze time-spent in terms of I/O vs CPU time; a program may alternate doing some of each (or do both in parallel in varying ratios), and we want to break time down by various categories of I/O or CPU time, which may not neatly map to stack frames.
- Sometimes — a bit of a generalization of the above — the most useful axis is “which hardware resources is the program using, in which proportion?” Depending on the program, this opens up a world of possible perspectives:
    - Microarchitectural resources: CPU execution units, cache space, decoder capacity, etc
    - GPU vs CPU vs PCI or NVLink bus bandwidth
    - Ethernet bandwidth vs local disk bandwidth vs CPU throughput
    - And many, many others
- Sometimes we need to organize by another axis or organizational frame than the physical stack trace.


    A prototypical example for me is trying to profile a Python program using a C-level profiler; Typically, you’ll find that most of the runtime is spent in `_PyEval_EvalFrame`, which tells you little; instead, you need to “lift” your profile into the domain of the Python-level stack trace in order to gain useful understanding. (This is a problem shared by some automated tools of the form “code that operates on other code,” which [I’ve discussed a bit in the past](https://buttondown.email/nelhage/archive/tracing-jits-and-coverage-guided-fuzzers/)).


## The interplay with expertise

I think that Klein’s perspective on expertise gives us some useful frames to understand this situation.

Experts, especially domain experts in a particular application or programming language or framework, can often look at a profiler and “see” the answers to many of the above questions, even thought they are not literally present in the profile.

They can “read between the lines” to ask and even answer the “right“ or “better” questions, instead of stopping at the questions literally answered by the profiler. Additionally, and importantly, when they can’t do so for one reason or another, experts will be much faster to **recognize** that that the profiler is asking the wrong questions, and find or build another tool, instead of continuing to bash their head against the same profile in the hopes that it will somehow yield an understanding it does not contain.

This perspective also explains, I think, some of the disconnect I see in the discourse and the common understanding of performance engineering. If you look at an expert performance engineer, you will, in fact, observe that they spend a lot of time looking at profiles; from this, it’s temping to conclude that this is, in some sense, the main thing that they are doing.

However, these expert performance engineers, on inspection, are *using profilers differently* than a novice does, or than a naive observer might expect. And that usage is driven by their ability to *see different information* in a profile! When they look at a profile, they are not just looking at the literal information displayed, but rather using that as one of several inputs to enhance and build their model of the program, and to update and evolve their mental model of the system, and to test and formulate hypotheses which they test,  through further optimization and further profiling runs (potentially with different profilers).
