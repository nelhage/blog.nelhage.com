---
title: "The ITTAGE indirect branch predictor"
slug: ittage
date: 2025-02-22T16:00:38-08:00
---
While [investigating the performance][blog] of the new [Python 3.14 tail-calling interpreter][interpreter], I learned ([via this very informative comment by Sam Gross][colesbury-comment]) a fun and new (to me) piece of performance trivia: Modern CPUs no longer particularly struggle to predict the bytecode-dispatch indirect jump inside a "conventional" bytecode interpreter loop; in steady-state, assuming the bytecode itself is reasonable stable or predictable in an appropriate sense, modern CPUs [achieve very high accuracy][interpreter-prediction] when predicting the dispatch, even on "vanilla" `while / switch`-style interpreter loops!

(The "threaded"-style interpreter, which replicates the dispatch logic into each opcode, is still a performance win, but much less so than [on older CPUs][threaded-code-paper]; the wins these days are mostly just due to executing fewer total instructions, and branches in particular).

Intrigued, I spent a bit of time reading what we know about just **how** branch predictors are able to achieve this feat. I found the answer pretty fascinating, so I'm going to try to share the salient high-level features -- as I understand them -- and also talk about some interesting connections and ideas I had in response.

A quick caveat, though: I am not a hardware engineer or CPU designer, and I'm mostly going to be focused on some high-level ideas I find interesting; I'll probably get some things wrong. If you want an introduction to CPU branch prediction by someone who actually knows what they're talking about, I'll refer you instead to [Dan Luu's piece on the topic][danluu].

[danluu]: https://danluu.com/branch-prediction/

## The TAGE and ITTAGE branch predictors

- [ ] TK brief intro?

In general, modern state-of-the-art CPUs don't seem to document too much about their branch predictors, so we don't know -- or at least, I couldn't easily discover -- what branch prediction looks like in real cutting-edge CPUs. However, there **is** (at least) one published algorithm that is both practical and can predict bytecode interpreter loops -- the ITTAGE indirect branch predictor -- so that's what I'm talking about. The author of that predictor [wrote a paper][interpreter-prediction] exploring prediction on bytecode interpreters and found that his ITTAGE exhibited similar performance to Intel Haswell CPUs and suggested they use a variation of it, but I don't think we know for sure.


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

- [A case for (partially) tagged geometric history length branch prediction][tage-paper] -- As best I can tell, this is the paper that proposed TAGE and ITTAGE.
- [The L-TAGE Branch Predictor](https://jilp.org/vol9/v9paper6.pdf) -- an implementation of TAGE for a branch prediction competition in 2007 ("CBP-2").
- [A 64 Kbytes ISL-TAGE branch predictor](https://jilp.org/jwac-2/program/cbp3_03_seznec.pdf) -- description of an updated version for the successor competition in 2011 ("CBP-3").
- [A 64-Kbytes ITTAGE indirect branch predictor][ittage-predictor] -- Description of an ITTAGE predictor for the indirect branch track of the CBP-3 competition.
- [The program for JWAC2][jwac2], which hosted the CBP-3 competition, containing links to source code for the TAGE and ITTAGE implementations (in a microarchitectural simulator) for that competition.
- [BOOM (Berkeley Out Of Order Machine)'s documentation on their TAGE implementation][boom-tage]. BOOM is an open-source RISC-V core, intended for microarchitecture research.

[tage-paper]: https://inria.hal.science/hal-03408381/document
[ittage-predictor]: https://inria.hal.science/hal-00639041v1/document
[jwac2]: https://jilp.org/jwac-2/program/JWAC-2-program.htm
[boom-tage]: https://docs.boom-core.org/en/latest/sections/branch-prediction/backing-predictor.html#the-tage-predictor

## Why I find ITTAGE interesting

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

If anyone is aware of a project that's tried something like this -- or gets inspired to experiment by this post! -- please do let me know.

[ijon]: https://ieeexplore.ieee.org/document/9152719
[afl-whitepaper]: https://lcamtuf.coredump.cx/afl/technical_details.txt

### Curiosity-driven RL

I have to leave you with one more connection, before I go.

- [ ] TK openai papers
- [ ] TK "curiosity"


<!--  cutting room floor below this point -->


While [investigating the performance][blog] of the new [Python 3.14 tail-calling interpreter][interpreter], I learned ([via this excellent comment by Sam Gross][colesbury-comment]) about some (relatively) recent (and new to me) developments in CPU branch-prediction. In particular, I had previously believed the conventional wisdom that CPUs [struggle to predict][threaded-code-paper] the dispatch branches in bytecode interpreters, and that these prediction failures are a significant source of overhead for such interpreters.

This turns out to not be true for modern CPUs! Modern CPUs [are able to predict bytecode dispatch][interpreter-prediction] with a high level of accuracy, at least when the interpreted bytecode itself is reasonable stable and predictable. And, while there does not appear to be a ton of publically-available information on the branch prediction algorithms used by state-of-the-art processors, there is at least one published algorithm in the literature which is capable of this feat: the ITTAGE indirect branch predictor.

I found this algorithm very interesting a bit thought-provoking, so I'm going to attempt to summarize how it works -- as best I understand it -- and also share some of my reflections.

## Branch Prediction: A very short recap

I'm mostly not going to try to explain the need for branch prediction and the role of a branch predictor, and will assume some basic background. If you want a good introduction, from someone who (unlike me) has actually worked on CPU design and implemetnation, I refer you to [Dan Luu's introduction][danluu].

That said, here's, a very brief recap; feel free to skip ahead.

Modern CPUs are [deeply pipelined][pipeline], which means they have many instructions in various stages of execution, simultaneously. Since they nonetheless present a serial abstraction (the ISA), this means that the CPU "frontend," responsible for fetching and decoding instructions, must runs far ahead (in terms of logical execution order) of the "backend," which does the actual computation.

[pipeline]: https://en.wikipedia.org/wiki/Instruction_pipelining
[danluu]: https://danluu.com/branch-prediction/

This gap poses a problem when executing conditional or indirect branches. When the frontend encounters a conditional branch, it needs to decide where to continue decoding instructions, but it can't know for sure whether or not the branch will be taken until the backend "catches up" and computes the relevant condition. However, if it waits around for the backend, this delay will creates a "bubble" in the CPU pipeline -- a region of time where the CPU has available resources to be executing instructions, but is sitting idle waiting for the frontend to continue fetching and decoding.

Thus, in order to minimize the frequency of such bubbles, the frontend will instead make a **prediction** about the direction (and, if necessary, the destination) of the branch, and continue fetching, **speculatively**, along this path. Once the backend catches up, the prediction will be confirmed or disproved, and the speculative state can be committed (made visible outside the core), or reverted.

# Building up to TAGE

I'm going to take a lightning tour through some of the basic ideas used in branch prediction, in order to arrive at TAGE and ITTAGE. I'm going to be doing a lot of oversimplifying, and focusing on a subset of the high-level concepts and ideas that I find interesting. In particular, I am **not** a hardware person, and I will be glossing over a **lot** of implementation detail.

## Dynamic branch prediction

Many branch prediction algorithms work on the basic postulate that the program we're executing will behave in the same way it did "last time we were here." Thus, our task becomes to

1. Decide what counts as "the last time were here" -- what needs to be the same for us to consider two branches as "the same" for the purpose of prediction, and
2. Decide how to store and update a data structure which we can consult to determine the outcome of historical branches matching the present state.

Perhaps the simplest such scheme[^one-bit] tracks one bit of information per branch, indexed by the program counter.

We can't afford to store one bit for every possible PC, so a typical concrete implementation will pick some bit-width `k`, and store a table of size `2^k` bits, indexed by the low `k` bits of the program counter[^pc-hash]. Once the "true" outcome of a branch is known, we'll update the table (set `table[lowbits(PC,k)] = taken`. When making a prediction, we consult the corresponding entry and do what it says.

Note that if two branches have the same low bits, they may "fight" over the same table entry, and we may be using a history record written by another branch. We will make no effect to detect this case. Even if we knew that some other branch had clobbered our entry, the best we could do is to fall back on some "default" prediction engine, which isn't going to be that much better than "guessing based on someone else's data," and so it's not worth the added hardware or logic to disambiguate collisions.

A predictor like this handles rarely-changing or slowly-changing branches pretty well, such as `if (global_flag) { ... }`

### Saturating counters

Many branches turn out to have variable behavior, but have a strong bias in one direction or the other -- perhaps they are taken 90% of the time, or only 1% of the time. For example, consider the conditional branch implicit in a loop:

```cpp
for (int i = 0; i < 10; i++) {
  // ...
}
```

This branch will be taken 90% of the time, and not-taken the remaining 10%, on the final iteration. If we execute this loop repeatedly, a one-bit counter will incur **two** misses each time: it will get the final "not-taken" wrong, but it will also incorrectly remember that outcome, and get the "i=0" case wrong next time, as well.

We can improve that situation by storing **two** bits per branch, and using them as a saturating counter: we might add one on "taken" and subtract one on "not taken," skipping the update if the counter is at the maximum or minimum value.

On our simple "count to 10" example, we'll still get the final iteration wrong, but we'll only decrement the counter by one, so we'll still get the `i=0` case right next time (and then restore the counter to maximum).

[^one-bit]: And not just a toy example -- some real CPUs used variants of this system, in the early 90s or so.
[^pc-hash]: One could in principle use some more complex hash that mixes in more bits of the program counter, but my sense is that it's hard to beat the sheer simplicitly of looking the low bits, in terms of benefit-to-cost. The costs here are measured in transistor count (chip area, power), and latency for the predictor. If you have more budget, my sense is you're better off expanding the table or using a more sophisticated algorithm entirely.


## Using branch history

To do better, we need to go beyond just using the program counter. I find it interesting to try to understand: what other information is even **available** to a branch predictor when making a prediction?

In some sense, the answer turns out to be "not much!" The whole job of the CPU frontend (of which the branch predictor is a part) is to "run out ahead" of the actual execution units of the CPU, and fetch instructions long before they're actually executed. Thus, any information about, say, architectural registers is both (a) logically and physically located very far from the branch predictor, and (b) "stuck somewhere in the past," even if we could access it.

One source of information the predictor logically ought to be able to access, though, is **its own history.** When we're predicting a branch, we arrived at this point -- this instruction pointer value -- by way of some path of fetches and branching decisions, which were themselves produced (potentially speculatively!) also by the CPU frontend and branch predictor. Thus, it's fairly easy for us to track a rolling window of "the recent branch history,"[^history] and use that to distinguish machine states or make predictions.

[^history]: What, exactly do we mean by "branch history"? As best I can tell, the classical answer is just a ring buffer storing one bit per branch, for "taken" or "not taken." Depending the predictor, unconditional branches may be omitted, or stored as "taken." Some predictors additionally store some subset of the bits of the program counter for each branch.

Once again, the simplest option here is to incorporate that branch history into the key we use when looking in our table. Previously we had a table of 2-bit counters, indexed by `lowbits(PC, k)`; now we will index it by combining the PC and the branch history in some way[^gshare] -- `table[lowbits(combine(PC, history), k)]`.

[^gshare]: As I understand it, historically, the first predictors to incorporate branch history just concatenated the history buffer with the PC; If our table has `2^k` entries, we would keep an `h`-bit history buffer, and index our prediction table using `bitcat(history, lowbits(PC, k-h))`. Later on, the "gshare" predictor introduced the idea of combining them using `xor`: we can also keep a `k`-bit history, and index the table using `lowbits(PC, k) ⊕ history`.

If we have enough bits of history, we can now predict -- among other cases -- loops with a constant number of iterations. Considering our example from above:

```cpp
for (int i = 0; i < 10; i++) {
  // ...
}
```

If we can track a 10-bit history counter, and all goes right, we will effectively use a different table entry for each value of `i` / each pass through the loop, and will eventually achieve perfect prediction.

## Sizing history

How large of a history should we use? Larger is not automatically better.

Longer histories allow us to learn more total patterns, and more complex patterns. We can only learn our above loop perfectly if we track at least nine branches worth of history; if the loop body contains additional branches (e.g. `if (array[i] > 0) { ... }`, it might take more.

However, longer histories may make us **slower** to learn simple patterns, and require more table entried in order to do so. Remember our simple `if (global_flag) { ... }` example. Our PC-only predictor learned that branch in one or two passes through. Suppose, now, that line is inside of a utility function that is called from many, many different places. Each call site may have a unique fingerprint (in terms of branch history), or potentially many fingerprints, depending on whatever happened before the call. In the worst case, we may end up with exponentially-many entries (in the length of the history buffer), where previously we just needed one. Furthermore, if `global_flag` changes but only rarely, we may incur branch misses in **each** such case before we're fully updated.

Thus, while longer histories are probably better _in net_ up to some point, what we'd really like to do is use "just enough" history for each branch, but not more. And that's more-or-less what TAGE aims to do!

# The TAGE predictor

We can now understand the basics of the TAGE predictor. The TAGE algorithm:

- Stores a very-long branch history (hundreds or thousands of branches)
- Stores multiple tables:
  - A base table indexed only by PC
  - A series of tables (on the order of 10-20 tables), indexed by PC and geometrically-spaced history lengths. That is, each table uses a history length approximately `r` times longer than the previous table, for some constant `r`. This property gives "TAGE" the "GE" in its name, for the **ge**ometric sequence.
- Aims to dynamically select -- for each branch at runtime -- the shortest history length **sufficient to make good predictions**

That last bullet point is, of course, key! There are a number of ideas (and a lot of important implementation details and tuning!) that go into achieving it. I'm going to mostly stay with the big ideas; I'll link to some papers and code if you want the details.

## Prediction: Use the longest matching entry

We'll start by looking at how TAGE makes predictions, and then move on to understand how it updates state to ensure that these predictions remain accurate.

When predicting a branch, TAGE looks in every one of its history tables, and checks whether or not they contain entries matching the current (PC, branch history). If any tables match, we pick the matching entry associated with the **longest** history, and use the prediction from that entry.

### Tagged tables

We can already see one of the defining features of TAGE: in order to select which table to use, we need to be able to identify which tables actually contain history records for the current (PC, branch history) state.

To this end, TAGE's table entries are "tagged:" each table entry contains additional bits of metadata (partially) describing which (PC, history) pair is stored in that entry, allowing the predictor to distinguish between different histories which collide and reside in the same slot.

Suppose we're considering a given table, with a given (PC, history) tuple (with `history` truncated to match that table's history length). As with our earlier predictors, we'll combine the two fields and take the low `k` bits, and we'll examine the entry `table[lowbits(indexhash(PC, history), k)]`. Now, however, the entry in that table will **also** contain a `t`-bit tag; on access, we'll compute a **different** hash -- call it `taghash(PC, history)`, and compare it to the stored tag. If the tag doesn't match, it means this entry is storing information for a **different** (PC, history) pair, and we should ignore it. We can design `indexhash` and `taghash` in a variety of ways, but the key property we want is that they should encode **different** bits of information from the input, to ensure that collisions are (relatively) independent of each other.

In TAGE, the base predictor (indexed by PC only, with no history) is untagged; if no longer table matches, we will fall back to that one unconditionally, so there's no point in spending tag bits on it.

In addition, the number of tag bits per entry increases with the length of the history. Longer history lengths mean more total **possible** histories that we're trying to hash down into an index, and so it ends up being worthwhile to allocate more bits to distinguish them.

### The contents of each entry

In addition to a tag, each table entry in TAGE stores two fields:

- A prediction counter `ctr`. This counter works like our previously-described counter: `ctr` is incremented on "taken" and decremented on "not taken," and its sign produces the actual prediction. `ctr` is 2 or 3 bits long in the papers I've found.
- A "usefulness" field `u`. `u` helps track whether a given entry has been useful recently. This field is used when deciding where to allocate a new entry. `u` is 1 or 2 bits in the papers I've found.

## Updating the TAGE tables

On prediction, we use the longest matching table. So, how do we **update** the tables following a branch, and what information is actually stored in them?

The published papers have a number of details and variations, and my sense is that the details matter a lot, and that the optimal choices probably vary a lot with storage budget and other factors. The core ideas are the same, though, so that's what I'll focus on.

### Update `u` when a prediction was useful

The `u` field helps track which entries are providing value. It is set any time an entry is "useful," which is to say:

- A prediction derived from that entry was correct, **and**
- The next matching entry (in order of history length) yielded an incorrect prediction.


Thus, it's not sufficient that a table entry produce correct predictions for it to be marked useful; it also has to be correct, and also providing value above and beyond the shorter tables.

### Update `ctr`

We update the `ctr` on whichever table entry we used to make a prediction, whether or not that prediction was correct.

### On misprediction, try to allocate a new entry



- `u` flag/counter
- on correct: update `ctr`
- on incorrect: pick a longer table

# Reflections
