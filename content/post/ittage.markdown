---
title: "The ITTAGE indirect branch predictor"
slug: ittage-branch-predictor
date: 2025-07-04T14:30:00-07:00
math: true
description: >
  Modern CPUs are actually pretty good at predicting the indirect branch inside an interpreter loop, _contra_ the conventional wisdom. We take a deep dive into the ITTAGE indirect branch prediction algorithm, which is capable of making those predictions, and draw some connections to some other interests of mine in the areas of fuzzing and reinforcement learning.
---
While [investigating the performance][blog] of the new [Python 3.14 tail-calling interpreter][interpreter], I learned ([via this very informative comment from Sam Gross][colesbury-comment]) new (to me) piece of performance trivia: Modern CPUs mostly no longer struggle to predict the bytecode-dispatch indirect jump inside a "conventional" bytecode interpreter loop. In steady-state, assuming the bytecode itself is reasonable stable, modern CPUs [achieve very high accuracy][interpreter-prediction] predicting the dispatch, even for "vanilla" `while / switch`-style interpreter loops[^threaded]!

[^threaded]: The "threaded"-style interpreter, which replicates the dispatch logic into each opcode, is still a performance win, but much less so than [on older CPUs][threaded-code-paper]; the wins these days are mostly just due to executing fewer total instructions, and branches in particular. There's [some more discussion](/post/cpython-tail-call/#further-weirdness) in my previous piece.

Intrigued, I spent a bit of time reading about just **how** branch predictors achieve this feat. I found the answer pretty fascinating, so I'm going to try to share the salient high-level features -- as I understand them -- as well as some interesting connections and ideas that came up in response.

A quick caveat: I am not a hardware engineer or CPU designer, and I'm mostly going to be focused on some high-level ideas I find interesting. I'll probably get some things wrong. If you want an introduction to CPU branch prediction by someone who actually knows what they're talking about, I'll refer you to [Dan Luu's piece on the topic][danluu].

[danluu]: https://danluu.com/branch-prediction/

## The TAGE and ITTAGE branch predictors

In general, modern state-of-the-art CPUs don't seem to document too much about their branch predictors, so we don't know -- or at least, I couldn't easily discover -- what branch prediction looks like in real cutting-edge CPUs. However, there **is** (at least) one published algorithm that is both practical and can predict bytecode interpreter loops -- the ITTAGE indirect branch predictor -- and that's what I'm talking about. The author of that predictor [wrote a paper][interpreter-prediction] exploring prediction on bytecode interpreters and found that his ITTAGE exhibited similar performance to Intel Haswell CPUs and suggested they use a variation of it, but I don't think we know for sure.

ITTAGE is a variant of the TAGE predictor; TAGE predicts taken/not-taken behavior for conditional branches, while ITTAGE predicts the destination of indirect jumps. Both predictors have a very similar structure, so I will mostly lump them together for most of this post.

### A brief preview

Before I dive in, I'll give you a brief summary of where we'll be going. Both TAGE and ITTAGE:
- Predict branch behavior by mapping from _(PC, PC history) -> past behavior_, and hoping that the future resembles the past.
- Stores many such tables, using a geometrically-increasing series of history lengths
- Attempt to dynamically choose the correct table (history length) for each branch.
- Do so by adaptively moving to a longer history on error, and using a careful replacement policy to preferentially keep useful entries.

Now, onto the longer version! Or, if that's enough for you, [skip ahead][reflections] to some reflections and some of the connections that make this topic particularly interesting to me.

[blog]: https://blog.nelhage.com/post/cpython-tail-call/
[interpreter]: https://docs.python.org/3.14/whatsnew/3.14.html#whatsnew314-tail-call
[threaded-code-paper]: https://link.springer.com/chapter/10.1007/3-540-44681-8_59
[interpreter-prediction]: https://inria.hal.science/hal-01100647/document
[colesbury-comment]: https://github.com/python/cpython/issues/129987#issuecomment-2654837868

### Dynamic branch prediction 101

Many dynamic branch prediction algorithms work off of a simple premise: Keep track of a table of historical data in some form, and, when asked to predict a branch, look up what happened "last time" and assume that history will repeat itself. Thinking in C++-flavored pseudocode, I tend to mentally model this approach as something like:

```c++
struct BranchDetails {
  // The information we use to identify a branch
};

struct BranchHistory {
  // The information we store about eaach branch

  // Predict the outcome of a branch based on past state
  bool predict() const { /* ... */ };

  // Update our state based on a resolved branch
  void update(bool taken) { /* ... */ };
};

// We store a mapping from one to the other. This is a fixed-size chunk of
// hardware, so it stores a fixed number of entries. We'll talk a bit about
// replacement strategy and some details later on.
 using PredictorState = FixedSizeMap<BranchDetails, BranchHistory>;

void on_resolve_branch(PredictorState &pred, BranchDetails &branch, bool taken) {
  pred[branch].update(taken);
}

bool predict_branch(PredictorState &pred, BranchDetails &branch) {
  return pred[branch].predict();
}

```

So, what do we use for the `BranchDetails` and `BranchHistory`? Perhaps the simplest option -- used by some early CPUs! -- is to just use the branch address to identify the branch -- essentially, track state for each branch instruction in the program text -- and to just use one bit of information for the history:

```c++
struct BranchDetails { uintptr_t addr; };
struct BranchHistory {
  bool taken_;

  bool predict() { return taken_; }
  void update(bool taken) { taken_ = taken; }
};
```

The next-simplest strategy replaces our `bool` per-branch state with a small counter (as few as 2 bits!), to give us a measure of hysteresis. We increment or decrement the counter on branch resolution, and use the sign for our prediction. It turns out that most branches are heavily "biased" one way or another -- e.g. a "branch taken" rate of 10% or 90% is more common than 50% -- and a small amount of hysteresis can let us absorb occasional outlier behaviors without forgetting everything we know:

```c++
struct BranchHistory {
  int2_t counter_;

  bool predict() { return counter_ >= 0; }
  void update(bool taken) {
   saturating_increment(&counter_, taken ? 1 : -1);
  }
};
```

### Moving beyond just `PC`

Indexing branches by `PC` is simple and efficient, but limiting. Many branches exhibit dynamic data-dependent behavior and we need to somehow distinguish with more granularity, if we want better accuracy.

Because branch predictors live in the CPU frontend, and have to make predictions well before an instruction is actually executed -- or potentially even fully-decoded -- and so they don't have much access to other CPU state to use in their predictions. However, there is one type of context they can access practically "for free": the **history** of the program counter and recent branches, since the predictor had to help generate those, to start with!

Thus, a branch predictor can maintain a circular buffer of some fixed size, store a rolling "branch history" or "PC history" in some form, and use that state to distinguish different branches. In the simplest case, we might store one bit per previous branch, writing a "1" for branch-taken, or "0" for not-taken. In more sophisticated predictors, we might include unconditional, and/or write a few bits of the PC value into the history:

```c++
constexpr int history_length = ...;

struct BranchDetails {
  uintptr_t pc;
  bitarray<history_length> history;
}
```
### Sizing our branch history

How large of a history should we use? Is larger always better?

Longer histories allow us to learn more total patterns, and more complex patterns. For a program with stable behavior, in steady state, a larger history will potentially allow us to learn more of the program's behavior, and distinguish between different situations more finely.

However, longer history means we **need** more space in our table to learn simple patterns, and potentially more time, since we have more possible states, and we need to encounter each one separately to learn it. Imagine a simple function that looks like this:

```cpp
bool logging_active;

void log(const char *msg) {
  if (logging_active) {
    printf("%s\n", msg);
  }
}
```

Suppose this function isn't inlined, and is called from many different places in an executable.

Assuming `logging_active` is static or mostly-static at runtime, this branch is highly predictable. A simple PC-only predictor should achieve near-perfect accuracy. However, if we also consider branch history, the predictor no longer "sees" this branch as a single entity; instead it needs to track every path that arrives at this branch instruction, separately. In the worst case, if we store `k` bits of history, we may need to use 2^k different table entries for this one branch! Worse, we need to encounter each state individually, and we don't "learn anything" about different paths, from each other.

## The TAGE algorithm: big ideas

We now have the necessary background to sketch a description of the TAGE predictor.

TAGE keeps track of branch history, as I've described, but unlike simpler predictors, it tracks hundreds or even thousands of bits of history, allowing it to potentially learn patterns over very long distances.

In order to make use of all this history without unwanted state explosion, TAGE stores **multiple** history tables (perhaps on the order of 10-20), indexed by a geometric series of history lengths (i.e. table N uses history length \\( L_n ≈ L_0\cdot{}r^n \\) for some ratio \\(r\\)). TAGE then attempts to adaptively choose, for each branch, the shortest history length (and corresponding table) that **suffices to make good predictions.**

How does it do that? Here are the core ideas (as I understand them). I'll link to some papers and code later on if you want to get into the details!

### Tag bits in each table

So far, I've entirely glossed over the question of **how** these lookup tables are actually implemented, and in particular concretely how we implement "lookup the entry with a given key."

In many simple branch predictors, history tables "don't know" anything about which keys they are storing, and just directly index based on some of the bits of the key.

For instance, for a predictor indexed by PC, we might just have an array of \\(2^k\\) counters, and use the low \\(k\\) bits of PC to select an entry. If two branches have the same address modulo \\(2^k\\), they will collide and use the same table entry, and we will make no effort to detect the collision or behave differently. This choice makes tables extremely cheap and efficient, and turns out to be a good tradeoff in many cases. Intuitively, we already have to handle the branch predictor being wrong, and such collisions are just another reason they might be wrong; detecting and reacting to collisions would take more hardware and more storage, and it turns out we're better off using that to lower the error rate in other ways, instead.

TAGE, however, stores multiple tables, and needs to use different tables for different branches, which in turn necessitates **knowing** which tables are actually storing information for a given key, vs a colliding key. Thus, in addition to the other payload, each table entry stores a **tag**, containing additional bits of metadata describing which key is stored in the entry.

Given a (PC, branch history) tuple _T_, TAGE uses two different hash functions for each table, `H_index` and `H_tag`. The branch state `T` will be stored in the table at index `H_index(T)`, with a tag value of `H_tag(T)`. On lookup, we check the value at `H_index(T)` and compare the tag to `H_tag(T)`.
- If the tags disagree, this entry is currently storing information about some **other** branch, and we won't use it (but may decide to overwrite it)
- If the tag agrees, we assume this state matches our branch, and we use and/or update it. Note that we still only check the hashes, and so it's possible we collided with a different branch in **both** hashes, but we will design them and choose their sizes so this condition is sufficiently rare in practice.

These **ta**g bits give TAGE the "TA" in its name; the "GE" comes from the **ge**ometric series of history lengths.

### Basic prediction algorithm

Given this setup, the basic prediction algorithm of TAGE is fairly simple. Each table entry stores a counter (called _ctr_ in the papers), as described above -- it is incremented on "branch taken," and decremented on "not taken."

To make a prediction, TAGE checks every table, using the appropriate history length. It considers all the table entries which have matching tags, and uses the prediction from the matching entry corresponding to the **longest** history length.

The base predictor -- in the simplest case, a table indexed only by PC -- does not use tag bits, and thus will always match, and be used as a fallback if no longer history matches.

### Move to a longer history on prediction error

Once a branch resolves and we know the correct answer, we need to update the predictor.

TAGE always updates the `ctr` field on the entry which was used to make the prediction. However, if the prediction was **wrong**, it also attempts to allocate a new entry, into one of the tables using a **longer** history length. The goal, thus, is to dynamically try longer and longer histories until we find a length that works.

### Track the usefulness of table entries

Since table entries are tagged, we need a way to decide when we will replace a table entry and reuse it for a new branch. For the predictor to work well, we aim to keep entries which are likely to make useful predictions in the future, and discard ones which won't.

In order to approximate that goal, TAGE tracks which entries have been useful **recently**. In addition to the tag and the counter, each table entry also has a _u_ ("useful") counter (typically just 1 or 2 bits), which tracks whether the table has recently produced useful predictions.

When we allocate new table entries -- as described above -- we will only ever overwrite a slot with _u_=0, and we will initialize new slots with _u_=0; thus, new entries must prove their worth or risk being replaced.

The _u_ counter is incremented any time a given table entry:
- Is used for a prediction, and
- That prediction turns out to be correct, and
- The prediction from that entry is **different** from the prediction given by the matching entry with the next-longest history.

Thus, it's not enough for an entry to yield correct predictions; it also needs to yield a correct prediction that counterfactually would have been wrong.

In addition, the _u_ counters are periodically decayed (or just set to zero) in some form, to prevent entries from lingering forever. The precise algorithm here varies a lot between published versions.

### From TAGE to ITTAGE

I've been describing the behavior of TAGE, which predicts one bit of information (taken/not-taken) for conditional branches. ITTAGE, which predicts the target of indirect branches (the whole reason I'm writing about this system!) is virtually identical; the primary changes are only:

- Each table entry also stores a predicted target address
- The _ctr_ counter is retained, but becomes a "confidence" counter. It is incremented on "correct prediction" and decremented on "incorrect." On incorrect prediction, we will update the predicted target to the new value iff _ctr_ is at the minimum value. Thus, _ctr_ tracks our confidence in this particular target address, while _u_ tracks the usefulness of this entire entry in the context of the entire predictor.

In fact, the same tables may be combined into a joint predictor, called "COTTAGE" [in the paper][tage-paper].

### References

I haven't found an enormous amount written about TAGE and ITTAGE, but I will include here the best links I have found, in case you're curious and want to dig into more details! Reading these papers, it really jumped out at me the extent to which it's **not** enough to have the right high-level ideas; a high-performing implementation of TAGE or ITTAGE (or any branch predictor) is the result of both a good design, and a **ton** of careful tuning and balancing of tradeoffs. Here's the links:

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

On one hand, I find ITTAGE interesting because I occasionally have cause to think about the performance of interpreter loops or similar software, and it represents an important update to how I need to reason about those situations. Very concretely, it informed my [CPython benchmarking][blog] from last post.

However, I also find it fascinating for some broader reasons, and in connection to some other areas of interest.

I've [written in the past][fuzzing-and-tracing] about a class of software tools (including both coverage-guided fuzzers and tracing JITs), which attempt to understand some program's behavior in large part by looking at the behavior of the program counter over time, and about how those tools struggle -- in related ways -- on interpreters and similar software for which the program state is "hidden in the data" and where the control flow alone is a poor proxy for the "interesting" state.

[fuzzing-and-tracing]: https://buttondown.com/nelhage/archive/tracing-jits-and-coverage-guided-fuzzers/

I didn't write about this connection in that post, but I've always considered branch predictors to be another member of this class of tool. As [mentioned above](#moving-beyond-just-pc), they also understand program execution mostly through the lens of "a series of program counter values," and they, too -- at least historically -- have struggled to behave well on interpreter loops.

Thus, learning about ITTAGE and its success predicting interpreter behavior naturally raises the question, for me: Is there anything to be learned from the ITTAGE algorithm for those other tools?

In particular, I wonder about...


### ITTAGE for coverage-guided fuzzing and program state exploration?

As I sketched in [that older post][fuzzing-and-tracing], coverage-guided fuzzing is a technique which attempts to automatically explore the behavior of a target program, by generating candidate inputs, and then observing which inputs generate "new" behavior in the program.

In order for this loop to work, we need some way of characterizing or bucketing program behavior, so we can decide what counts as "new" or "interesting" behavior, versus behavior we have already observed. I will admit that I am not up-to-date on the current frontiers in this field, but [historically this has been done][afl-whitepaper] using "coverage"-like metrics, which count the occurrences of either PC values or of branches (which essentially means (PC, PC') pairs). These counts are potentially bucketed, and we "fingerprint" execution by the list of `[(PC, bucketed_count)]` values generated during execution.

This approach is, in practice, fabulously effective. However, it can struggle on certain shapes of programs -- including often interpreters -- where the "interesting" state does not map well onto sets of program counters or branches. One of my favorite illustrations of this problem is the [IJON paper][ijon], which demonstrates some concrete problems, and attacks them using human-added annotations.

My questions, then, is thus: Could something like the TAGE/ITTAGE approach help coverage-guided fuzzers to better explore the state space for interpreters, and interpreter-like programs? Could we, for instance, train a TAGE-like predictor on existing corpus entries, and then prioritize candidate mutants based on their rate of prediction errors? Might this allow a fuzzer to (e.g.) effectively explore the state space of code in an interpreted language, only by annotating the interpreter?

There are a large number of practical challenges, but in principle this seems like it might allow more-nuanced exploration of state spaces, and discovering "novel behavior" which can only be identified by means of long-range correlations and patterns.

I will note that TAGE/ITTAGE **specifically** are designed and tuned around hardware performance characteristics and tradeoffs; the performance landscape in software is very different, and so if such an idea **does** work, I suspect the details look fairly different and are optimized for an efficient software implementation, it seems plausible to me there's a place for borrowing the core idea of "dynamically picking a history length on a per-branch basis."

An even whackier idea might be to use the **actual hardware branch predictor**. Modern CPUs allow you to observe branch prediction accuracy via hardware performance counters, and we could imagine executing the corpus of existing examples to train the branch predictor, and then observing the actual hardware misprediction count as a novelty signal. This approach also has a ton of challenges, in part because of the opacity of the hardware branch predictor and the inability to explicitly control it; however, it has the virtue of potentially being much, much cheaper than a software predictor. It does make me wonder whether there are any CPUs which expose the branch predictor state explicitly -- even in the form of "save or restore predictor state" operations -- which seems like it would make such an approach far more viable.

If anyone is aware of a project that's tried something like this -- or is inspired to experiment -- please do let me know.

[ijon]: https://ieeexplore.ieee.org/document/9152719
[afl-whitepaper]: https://lcamtuf.coredump.cx/afl/technical_details.txt

### Curiosity and Reinforcement Learning

As outlined in the section above, my best guess for how you might apply a TAGE/ITTAGE-like algorithm to fuzzing is by treating "prediction error" as a reward signal, and spending time on inputs which have high prediction error.

As I was thinking through that idea, I realized that it sounded familiar because, at some level of abstraction, that's a classic idea from the domain of reinforcement learning!

Perhaps most notably, in 2018, OpenAI published [two][openai-curiosity] [papers][openai-rnd] about "curiosity-driven learning," exploring techniques to enhance reinforcement learning by adding reward terms that encourage exploration, even absent reward signal from the environment. The two papers differ in the details of their approach, but share the same basic idea: Along with the policy network -- which is the one that determines the actions to take -- you train a predictor network, which attempts to predict some features of the environment, or the outcomes of your actions. You then reward the policy model for discovering actions or states with a high prediction error, which -- if all goes right -- encourages exploration of novel parts of the environment.

As best I can tell, this technique worked fairly well; the [second paper][openai-rnd] achieved state-of-the-art performance on [Montezuma's Revenge][montezuma], an Atari game which was famously hard for reinforcement learning algorithms, since it requires extensive exploration and manipulation of keys and equipment prior to receiving any score. However, I don't know, off the top of my head, what the future trajectory of that work and that approach has been.

I was aware of those papers, and followed the work at the time, but hadn't been consciously aware of them when I started trying to fit the "ITTAGE" and "coverage-guided fuzzing" pieces together in my head. The confluence does suggest to me that there may be something here; although, at the same time, in 2025 it might end up being easier to just throw a neural net at the problem, instead of a custom-designed-and-tuned prediction algorithm!

[montezuma]: https://en.wikipedia.org/wiki/Montezuma%27s_Revenge_(video_game)
[openai-curiosity]: https://arxiv.org/pdf/1808.04355
[openai-rnd]: https://openai.com/index/reinforcement-learning-with-prediction-based-rewards/
