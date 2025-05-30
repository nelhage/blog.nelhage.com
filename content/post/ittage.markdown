---
title: "The ITTAGE indirect branch predictor"
slug: ittage
date: 2025-02-22T16:00:38-08:00
---
While [investigating the performance][blog] of the new [Python 3.14 tail-calling interpreter][interpreter], I learned ([via this very informative comment by Sam Gross][colesbury-comment]) a fun and new (to me) piece of performance trivia: Modern CPUs no longer particularly struggle to predict the bytecode-dispatch indirect jump inside a "conventional" bytecode interpreter loop; in steady-state, assuming the bytecode itself is reasonable stable or predictable in an appropriate sense, modern CPUs [achieve very high accuracy][interpreter-prediction] when predicting the dispatch, even on "vanilla" `while / switch`-style interpreter loops[^threaded]!

[^threaded]: The "threaded"-style interpreter, which replicates the dispatch logic into each opcode, is still a performance win, but much less so than [on older CPUs][threaded-code-paper]; the wins these days are mostly just due to executing fewer total instructions, and branches in particular. There's [some more discussion](http://localhost:1313/post/cpython-tail-call/#further-weirdness) in my previous piece.

Intrigued, I spent a bit of time reading what we know about just **how** branch predictors are able to achieve this feat. I found the answer pretty fascinating, so I'm going to try to share the salient high-level features -- as I understand them -- and also talk about some interesting connections and ideas I had in response.

A quick caveat, though: I am not a hardware engineer or CPU designer, and I'm mostly going to be focused on some high-level ideas I find interesting; I'll probably get some things wrong. If you want an introduction to CPU branch prediction by someone who actually knows what they're talking about, I'll refer you instead to [Dan Luu's piece on the topic][danluu].

[danluu]: https://danluu.com/branch-prediction/

## The TAGE and ITTAGE branch predictors

In general, modern state-of-the-art CPUs don't seem to document too much about their branch predictors, so we don't know -- or at least, I couldn't easily discover -- what branch prediction looks like in real cutting-edge CPUs. However, there **is** (at least) one published algorithm that is both practical and can predict bytecode interpreter loops -- the ITTAGE indirect branch predictor -- so that's what I'm talking about. The author of that predictor [wrote a paper][interpreter-prediction] exploring prediction on bytecode interpreters and found that his ITTAGE exhibited similar performance to Intel Haswell CPUs and suggested they use a variation of it, but I don't think we know for sure.

### A brief preview

Before I dive in, I'll give you a brief summary of where we'll be going. TAGE and ITTAGE:
- Predict branch behavior mapping from _(PC, PC history) -> past behavior_, and hoping that the future resembles the past.
- (IT)TAGE stores many such tables, using a geometrically-increasing series of history lengths
- The algorithm attempts to dynamically choose the correct table for each branch. It does this by adaptively moving to a longer history on error, and with a careful replacement policy to preferentially keep useful entries.

Now, onto the longer version! Or, if that's enough for you, [skip ahead][reflections] to my reflections and the connections that make this topic interesting to me.

[blog]: https://blog.nelhage.com/post/cpython-tail-call/
[interpreter]: https://docs.python.org/3.14/whatsnew/3.14.html#whatsnew314-tail-call
[threaded-code-paper]: https://link.springer.com/chapter/10.1007/3-540-44681-8_59
[interpreter-prediction]: https://inria.hal.science/hal-01100647/document
[colesbury-comment]: https://github.com/python/cpython/issues/129987#issuecomment-2654837868

### Dynamic branch prediction 101

Many dynamic branch prediction algorithms work off of a simple premise: Keep track of a table of historical data in some form, and, when asked to predict a branch, look up what happened "last time" and assume that history will repeat itself.

One key design question, then, is what we mean by "last time" -- how do we identify or name particular states? What information has to match in order for us to count an older branch as "the same situation"?

Perhaps the simplest option -- chosen by some of the earliest branch predictors -- is to just use the program counter of the branch, and hope that any given branch exhibits mostly-consistent behavior at runtime. Implementing such a strategy involves many decisions about how to store this information, how much storage to allocate, how to map PC values to a table, and so on, but at a very high level I summarize this family of predictors by thinking of them as something like `FixedSizeMap<PC, BranchData>`.

What data should we store about a past branch? The simplest answer is a single bit -- "was this branch taken?" -- or the target address, for indirect branches. In practice, it's often useful to store a few bits (as few as two!) for a (saturating) counter. We might add one for "branch taken," and subtract one for "not taken." For indirect branches, we add one on a correct prediction, and subtract one on incorrect, and only replace the saved target address if the counter is zero. In both cases, using a few bits of counter gives us a measure of hysteresis and will improve performance on branches which are **mostly** consistent, with rare deviations; that turns out to be usually helpful in practice.

Of course, indexing our history **just** by the branch address is fairly limiting. Many branches exhibit dynamic data-dependent behavior and we need to somehow distinguish different branches in a finer-grained way, if we want better accuracy.

For the most part, though, branch predictors can't actually look at any program data when making predictions. The whole idea of branch prediction, after all, is that we need to guess the result of a branch well before we've actually executed the instructions leading up to that branch. However, we do have one source of additional bits of context or information that's relatively easy to use: We can track the **history** of the program counter, leading up to a given branch.

That's (more-or-less) readily available to a CPU frontend and branch predictor, since they're responsible for predicting and tracking it, in the first place! They can just keep some additional bits of state, and remember the series of branches immediately prior to the current state, in a fixed-size circular buffer. In the most basic version, we will store a history of one bit per jump, a "1" for "taken" and a "0" for not taken; more sophisticated predictors may also record unconditional jumps, or fold in a few bits of the program counter at each jump, or perhaps just for indirect jumps.

Thus, another family of branch predictors stores a table that we might describe as a `FixedSizeMap<(PC, BranchHistory), BranchData>`. In this case, we have even more implementation decisions in practice, around how to design and address the table, but once again I'm going to gloss over them because I'm interested in the high-level ideas.

### Sizing our branch history

How large of a history should we use? Larger is not automatically better.

Longer histories allow us to learn more total patterns, and more complex patterns. For a program with stable behavior, in steady state, a larger history will potentially allow us to learn more of the program's behavior, and distinguish between different situations more finely.

However, longer history means we **need** more space in our table to learn simple patterns, and potentially more time, since we need to encounter every distinct state before we can learn it. Imagine a simple function that looks like:

```cpp
void some_helper() {
  if (global_flag) {
    // log stuff
  }
}
```

Assuming `global_flag` is fairly static at runtime, we only **need** the PC to predict this branch. If we also track 10 bits of history, in the very worst case, we could end up using 1024 (2^10) **different** table entries to learn this very simple branch, depending on how many and which random code paths call this helpers. In addition, if `global_flag` changes rarely, we might need to re-learn each of those cases, separately!

## The TAGE algorithm: big ideas

I can now explain the TAGE algorithm, or at least the big ideas.

TAGE keeps track of branch history, as I've described, but unlike simpler predictors, it tracks hundreds or even thousands of bits of history, allowing it to potentially learn patterns over very long distances.

In order to make use of this lengthy history, TAGE stores **multiple** history tables (perhaps on the order of 10-20), indexed by a geometric series of history lengths (i.e. table N uses history length Lₙ ≈ L₀·kⁿ for some k). TAGE then attempts to adaptively choose, for each branch, the shortest history length (and corresponding table) that **suffices to make good predictions.**

How does it do that? I am going to only stick to the high-level ideas here (as I understand them), since that's what I find interesting. I'll link to some papers and code later on if you want to get into the details!

### Tag bits in each table

So far, I've entirely glossed over the question of how these lookup tables are actually implemented, and precisely what gets stored and how.

In many simpler branch predictors, history tables "don't know" anything about which keys they are storing. For instance, for a predictor indexed by PC, we might just have an array of _2ᵏ_ counters, and use the low _k_ bits of PC to select an entry. If two branches have the same address modulo _2ᵏ_, they will collide and use the same tabe entry, but we will make no effort to detect the collision or behave differently. This choice makes tables extremely cheap and efficient, and turns out to be a good tradeoff in many cases. Intuitively, the branch predictor will always be wrong sometimes -- for a wide variety of reasons -- and (a) this is just another reason, and we can estimate the cost empirically, and (b) if we **did** detect a collision, it's not clear what we could do, anyways; simple branch predictors often have extreme power/latency/size/etc constraints.

TAGE, however, stores multiple tables, and needs to distinguish which tables actually store information about the current branch, vs unrelated entries. Thus, in addition to the other payload, each table entry stores a **tag**, containing additional bits of metadata describing which key is stored in the entry.

Given a (PC, branch history) tuple `T`, TAGE uses two different hash functions for each table, `H_index` and `H_tag`; the branch state `T` will be stored in the table at index `H_index(T)`, with a tag value of `H_tag(T)`. On lookup, we check the value at `H_index(T)` and compare the tag to `H_tag(T)`. If the tags disagree, this entry is currently storing information about some **other** branch. If the tag agrees, it's still possible we collided in both hashes, but we can tune our choice of hashes and their bit-widths to make collisions rare-enough to not majorly impact our accuracy.

These **ta**g bits give TAGE the "TA" in its name; the "GE" comes from the **ge**ometric series of history lengths.

### Basic prediction algorithm

Given this preamble, the basic prediction algorithm of TAGE is fairly simple. Each table entry stores a counter (called _ctr_ in the papers), as described above -- it is incremented on "branch taken," and decremented on "not taken."

To make a prediction, TAGE checks **every** table, using the appropriate history length. It considers all the table entries which have matching tags (thus, which are likely actually storing data about this (PC, history) state), and uses the prediction from the matching entry corresponding to the longest history length.

The base predictor -- in the simplest case, a table indexed only by PC -- does not use tag bits, and thus will always match and be used if no matching entry is found with a longer history.

(In practice, depending on the implementation, there may be a few other special cases or complications; I'll link some papers below for details).

### Move to a longer history on prediction error

Once a branch resolves and we know the correct answer, we need to update the predictor. TAGE updates the `ctr` field on the entry which was used to make the prediction. However, if the prediction was **wrong**, it also attempts to allocate a new entry, in one of the tables using a **longer** history length. The goal, thus, is to gradually try longer and longer histories until we find a length that works.

### Track the usefulness of table entries

Since table entries are tagged, we need a way to decide when we will replace a table entry and reuse it for a new branch. For the predictor to work well, we aim to keep entries which are likely to make useful predictions in the future, and discard ones which won't.

In order to approximate that goal, TAGE tracks which entries have been useful **recently**. In addition to the tag and the counter, each table entry also has a _u_ ("useful") counter (typically just 1 or 2 bits). This counter approximates whether a table entry has been useful recently.

When we allocate new table entries -- described above -- we will only ever overwrite a slot with _u_=0, and we will initialize new slots with _u_=0; they must prove their worth in order to stick around.

The _u_ counter is incremented any time a given table entry:
- Is used for a prediction, and
- That prediction turns out to be correct, and
- The prediction from that entry is **different** from the prediction given by the matching entry with the next-longest history.

Thus, it's not enough for an entry to yield correct predictions; it also needs to yield a correct prediction that counterfactually would have been incorrect.

The _u_ counters are periodically reset or decremented, to prevent stale entries from lingering forever. The precise algorithm here varies a lot between published versions.

### From TAGE to ITTAGE

I've been describing the behavior of TAGE, which predicts one bit of information (taken/not-taken) for conditional branches. ITTAGE, which predicts the target of indirect branches (the whole reason I'm writing about this system!) is essentially identical; the primary changes are only:

- Each table entry also stores a branch target address
- The _ctr_ counter is retained, but becomes a "confidence" counter. It is incremented and decremented as before, and the predictor will update the target address with a new address iff the existing counter is 0 (minimum confidence).

In fact, the same tables may be combined -- with some subtleties I'll gloss over here -- into a joint predictor, called "COTTAGE" [in the paper][tage-paper].

### References

I haven't found an enormous amount written about TAGE and ITTAGE, but I will include here the best links I have found, in case you're curious and want to dig into more details! Reading these papers, it really jumped out at me the extent to which it's **not** enough to have the right high-level ideas; a high-performing implementation of TAGE or ITTAGE (or any branch predictor) is the result of a good design, and a **ton** of careful tuning and balancing of tradeoffs. Here's the links:

[A case for (partially) tagged geometric history length branch prediction][tage-paper]
:    As best I can tell, this is the paper that proposed TAGE and ITTAGE

[The L-TAGE Branch Predictor](https://jilp.org/vol9/v9paper6.pdf)
:    An implementation of TAGE for a branch prediction competition in 2007 ("CBP-2").

[A 64 Kbytes ISL-TAGE branch predictor](https://jilp.org/jwac-2/program/cbp3_03_seznec.pdf)
:    The description of an updated version for the successor competition, in 2011 ("CBP-3").

[A 64-Kbytes ITTAGE indirect branch predictor][ittage-predictor]
:    The description of an ITTAGE predictor for the indirect branch track of the same competition.

[The program for JWAC2, which hosted the CBP-3 competition][jwac2]
:    In particular, containing links to source code for the TAGE and ITTAGE implementations submitted to that competition (in a microarchitectural simulator).

[BOOM (Berkeley Out Of Order Machine)'s documentation on their TAGE implementation][boom-tage]
:    BOOM is an open-source RISC-V core, intended for microarchitecture research.

[tage-paper]: https://inria.hal.science/hal-03408381/document
[ittage-predictor]: https://inria.hal.science/hal-00639041v1/document
[jwac2]: https://jilp.org/jwac-2/program/JWAC-2-program.htm
[boom-tage]: https://docs.boom-core.org/en/latest/sections/branch-prediction/backing-predictor.html#the-tage-predictor

## Why I find ITTAGE interesting

[reflections]: #why-i-find-ittage-interesting

On one hand, I find ITTAGE interesting because I occasionally have cause to think about the performance of interpreter loops or similar software, and it represents an important update to how I need to reason about those situations.

ITTAGE interests me for a broader reason, however!

I've [written in the past][fuzzing-and-tracing] about a class of software tools (including both coverage-guided fuzzers and tracing JITs), which attempt to understand some program's behavior or state space primarily by looking at the behavior of the program counter of time, and how those tools struggle -- in related ways -- on interpreters and other programs where program state is "hidden in the data" instead of in the control flow.

[fuzzing-and-tracing]: https://buttondown.com/nelhage/archive/tracing-jits-and-coverage-guided-fuzzers/

I didn't write about this connection in that post, but branch predictors are, in some sense, also members of the same class, or at least closely related. They also have to understand (some aspects of) program behavior primarily "just" by watching PC, and -- historically -- they've also struggled with interpreters.

Thus, learning about ITTAGE and its success predicting interpreter behavior raises for me the question: Is there anything to be learned from the ITTAGE algorithm for those tools? In particular, I wonder about...


### ITTAGE for coverage-guided fuzzing and program state exploration

As I sketched in [my older post][fuzzing-and-tracing], coverage-guided fuzzing is a technique which attempts to automatically explore the behavior of a target program, by generating candidate inputs, and then observing which inputs generate "new" behavior in the program. In order for this loop to work, we need some way of characterizing or bucketing program behavior, so we can decide what counts as "new" or "interesting" behavior, versus behavior we have already observed. I will admit that I am not up-to-date on the current frontiers in this field, but [historically this has been done][afl-whitepaper] using "coverage"-style metrics: counting which PC values, or which branches (which is essentially to say: pairs of program counters, (PC, PC')) are observed, potentially with some coarse counts and bucketing -- e.g. we might declare that seeing a given branch once is different from twice, but 3-5 times all constitute the "same" behavior.

This approach is, in practice, fabulously effective. However, it can struggle on certain shapes of programs -- such as interpreters -- where the "interesting" state does not map well onto sets of program counters or branches. One of my favorite illustrations of this problem is the [IJON paper][ijon], which attacks the problem using human-added annotations.

My questions, then, is thus: Could something like the TAGE/ITTAGE approach help coverage-guided fuzzers in better exploring the state space for interpreters, and similar programs? Could we, for instance, train a TAGE-like predictor on existing corpus entries, and prioritize candidate mutants based on their rate of prediction errors? Might this allow a fuzzer to (e.g.) effectively explore the state space of code in an interpreted language, only by annotating the interpreter?

There are a large number of practical concerns or implementation details, but in principle this seems like it might allow more-nuanced exploration of state spaces, and discovering "novel behavior" which can only be identified by means of long-range correlations and patterns.

Perhaps the most important caveat, is that such an algorithm would necessarily be slower than "just keeping a bunch of counters," and most such fuzzers live or die on their performance -- the more inputs you can try, the more likely you are to find the interesting ones. Perhaps, though, such a system could be enabled selectively, when we find that progress has slowed, or perhaps it would have value on more expensive fuzz targets.

In addition, TAGE/ITTAGE **in particular** are designed and tuned around **hardware** performance characteristics and tradeoffs; the landscape in software is very different and if such an idea does work, I suspect it looks moderately different; but it seems plausible to me there's a place for borrowing the core idea of "dynamically picking a history length on a per-branch basis."

An even whackier idea might be to use the **actual hardware branch predictor**. Modern CPUs contain performance counters which let you observe the number of branch prediction failures while executing a given piece of code. We could imagine executing the corpus of existing examples to train the branch predictor, and then observing the actual hardware misprediction count as a novelty signal. This approach also has a ton of challenges, because of the opacity of the hardware branch predictor and the inability to explicitly control it; however, it has the virtue of potentially being much, much cheaper than a software predictor.

If anyone is aware of a project that's tried something like this -- or gets inspired to experiment by this post! -- please do let me know.

[ijon]: https://ieeexplore.ieee.org/document/9152719
[afl-whitepaper]: https://lcamtuf.coredump.cx/afl/technical_details.txt

### Curiosity and Reinforcement Learning

As outlined in the section above, my best guess for how you might apply a TAGE/ITTAGE-like algorithm to fuzzing is by treating "prediction error" as a reward signal, and spending time on inputs which have high prediction error.

As I was thinking through that idea, I realized that, at some level of abstraction, that's precisely the technique used by an older line of research in reinforcement learning!

In 2018, OpenAI published [two][openai-curiosity] [papers][openai-rnd] about "curiosity-driven learning," exploring techniques to enhance reinforcement learning by adding reward terms that encourage exploration, even absent reward signal from the environment. The two papers differ in the details of their approach, but share the same basic idea: Along with the policy network -- which is the one that determines the actions to take -- you train a **predictor** network, which attempts to predict some aspect of the environment. You then reward the policy model for discovering actions or states with a high prediction error, which -- if all goes right -- encourages exploration of novel parts of the environment or state space.

As best I can tell, this technique worked quite well; the [second paper][openai-rnd] achieved state-of-the-art performance on [Montezuma's Revenge][montezuma], an Atari game which was famously hard for reinforcement learning algorithms, since it requires extensive exploration and manipulation of keys and equipment prior to receiving any score. However, I don't know, off the top of my head, what the future trajectory of that work and that approach has been.

I was aware of those papers, and followed the work at the time, but hadn't been consciously aware of them when I started trying to fit the "ITTAGE" and "coverage-guided fuzzing" pieces together in my head. The confluence does suggest to me that there may be something here; although, at the same time, in 2025 it might end up being easier to just throw a neural net at the problem, instead of a custom-designed-and-tuned prediction algorithm!

[montezuma]: https://en.wikipedia.org/wiki/Montezuma%27s_Revenge_(video_game)
[openai-curiosity]: https://arxiv.org/pdf/1808.04355
[openai-rnd]: https://openai.com/index/reinforcement-learning-with-prediction-based-rewards/
