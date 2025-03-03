---
title: "TIL: The ITTAGE indirect branch predictor"
slug: ittage
date: 2025-02-22T16:00:38-08:00
---

It's long been known that the indirect branch at the heart of a classic bytecode interpreter poses difficulties for CPUs, due to being difficult to predict and thus causing frequent CPU pipeline bubbles.

- [ ] TK: reference. "Art of Crafting Interpreters"? Other?

Surprisingly to me, I recently learned that this "fact" is no longer particularly true! (At least some) modern branch predictors can do a quite-good job of predicting the dispatch pattern in a bytecode interpreter, even **without** the "threaded code" optimization that duplicates the dispatch into each opcode body.

- [ ] TK: did i get his name right below?

(I learned this fact from Samuel Colesbury on a [recent issue I opened with CPython][colesbury-comment], as a result of investigating their [new tail-call based interpreter][tail-call-pr]).

To my knowledge, there are no public details on the branch predictors used by modern state-of-the-art CPUs, but the above claim comes from [a paper that studies prediction accuracy][interpreter-prediction] both on real hardware (an Intel Haswell CPU), and in simulation using the [ITTAGE][tage-paper] branch prediction algorithm, and demonstrates similar results from both.

The ITTAGE algorithm (and its sibling, the TAGE predictor) were new to me, and I find them really interesting, so I want to explain the key ideas of the algorithm, as I understand it, and then muse on a few connections and more-speculative ideas I had.

[tail-call-pr]: https://github.com/python/cpython/pull/128718
[my-issue]: https://github.com/python/cpython/issues/129987
[colesbury-comment]: https://github.com/python/cpython/issues/129987#issuecomment-2654837868
[interpreter-prediction]: https://inria.hal.science/hal-01100647/document


## Branch Prediction: A very short recap

I'm mostly not going to try to explain the need for branch prediction and the role of a branch predictor, and will assume some basic background. If you want a good introduction, from someone who (unlike me) has actually worked on CPU design and implemetnation, I refer you to [this post by Dan Luu][danluu].

That said, here's, a very brief recap, in part to frame the terminology and concepts as I think about them:

Modern CPUs are [deeply pipelined][pipeline], which means they have many instructions being executed concurrently. That, in turn, means that the CPU "frontend," responsible for fetching and decoding instructions, runs far ahead (in terms of logical execution order) of the "backend," which does the actual computation. This gap, in turn, poses a problem when executing conditional or indirect branches. When the frontend encounters a conditional branch, it needs to decide where to continue decoding instructions, but it can't know for sure whether or not the branch will be taken until the backend "catches up" and computes the relevant condition. However, if it waits around for the backend, this delay will creates a "bubble" in the CPU pipeline -- a region of time where the CPU has available resources to be executing instructions, but is sitting idle waiting for the frontend to continue fetching and decoding.

In order to minimize the frequency of such bubbles, the frontend will instead make a **prediction** about the direction (and, if necessary, the destination) of the branch, and continues fetching and executing, **speculatively**, along this path. Once the backend catches up, the prediction will be confirmed or disproved, and the speculative state can be committed (made visible outside the core), or reverted.

# Building up to TAGE

I'm going to take a lightning tour through some of the basic ideas in the field of branch prediction, in order to arrive at TAGE and ITTAGE. I'm going to be doing a lot of oversimplifying, and focusing on a subset of the high-level concepts and ideas that I find interesting. In particular, I am **not** a hardware person, and I will be glossing over a **lot** of implementation detail.

## The most basic dynamic predictor

A lot of branch prediction algorithms boil down to some version of "remember what happened last time, and assume it will happen again." In addition to numerous important implementation details, the core question raised by this description turns out to be: "What do we mean by "last time"?" That is to say: even if we had unlimited storage and time to compute, what previous state(s) of the machine should we regard as "the same" for the purposes of making a prediction?

One of the simplest "dynamic" predictors (which is to say, predictors that maintains state at execution time, and updates that state or "learns" over the course of program execution) is one that just assumes that each branch instruction will have stable behavior or time. That is to say, we conceptually store something like a `Map<PC, bool>`, update it after executing each branch, and then predict that each branch will repeat the last behavior.

Such a predictor can do reasonably well! In particular, a decent number of branches have behavior that is "static after startup;" imagine, for instance, code that looks like this:

```cpp
/* Populated based on a config file or command-line option */
bool logging_enabled;

void log(const char *msg) {
  if (logging_enabled) {
    fprintf(stderr, "%s", msg);
  }
}
```

The `if (logging_enabled)` branch will tend to repeat behavior very consistently, even though its behavior is unknowable ahead of time.

## Using branch history

Going beyond just the current program counter, what information is available to a branch predictor when making a prediction?

In some sense, the answer is "not much"; the predictor runs very near the top of the CPU pipeline, and can't really ask questions about the CPU state -- that's all computed later in the pipeline, which is "stuck in the past" from the predictor's perspective.

What the predictor **does** have, though, is information about the **branch history** that lead to the current program counter. It was consulted for each of those branches, after all, and we **must** know the (speculative) result of recent branches in order to compute the current program counter that we're now fetching and predicting at. Thus, most branch prediction algorithms (as far as I know) operate on an input of the current program counter, and a rolling window of recent branch history[^history].

[^history]: What, exactly do we mean by "branch history"? As best I can tell, the classical answer is just a ring buffer storing one bit per branch, for "taken" or "not taken." Depending the predictor, unconditional branches may be omitted, or stored as "taken." Some predictors additionally store some subset of the bits of the program counter from each recent branch.


    Branch predictors are extremely latency-sensitive, so my sense is that it's common to just use the low `k` bits of the PC, in a place where my software instincts would instead tell me to hash it to get better mixing.

Thus, we arrive at another family of predictors by indexing our map by `(History, PC)`, instead of just the current program counter. This means that the same branch instruction may now be stored multiple times, if we arrive at it via different paths.

A predictor like this can now predict (e.g.) loops which always have a constant number of iterations:

```cpp
for (int i = 0; i < 5; i++) {
  // ...
}
```

The end of that loop will contain a conditional branch to the top; that branch will be taken four times, and then not-taken once. A predictor which looks at a four-element branch history will be able to look at `([taken, taken, taken, taken], branchPC)` and predict "not taken," and predict "taken" otherwise.

## Sizing history

How large of a history should we use? If we're using a static length, we face a tricky tradeoff.

Longer histories allow us to learn more patterns and more-complicated patterns. Suppose our loop counts to `20`, instead of `5`. Or suppose the loop body contains a bunch of `if`s that compile into conditional branches. These patterns can still be learned, but not with only four branches worth of history.

However, longer histories mean we need to consider more distinct patterns (thus, we may need more SRAM in our predictor), and may also make us slower to learn simple patterns:

Recall our `logging_enabled` example from above. That branch doesn't need any history for us to predict it; a per-PC history table would suffice. If, however, we do use a four-element history, we may end up having to store 16 (2⁴) distinct entries, one for each possible four-element (taken, not-taken) history leading up to our `if`! That not only consumes more space, but means we potentially eat the cost of 16 mispredictions before we've "fully learned" this branch, instead of just one!

# The TAGE predictor

We can now understand the basic idea of the TAGE predictor. The TAGE algorithm:

- Stores a very-long branch history (hundreds or thousands of branches)
- Stores multiple tables:
  - A base table indexed only by PC
  - A series of tables indexed by exponentially- (aka geometrically-) spaced history lengths. That is, each table use a history length approximate `k` times longer than the previous, for some fixed `k`.
- Aims to -- for each branch encountered at runtime -- use the shortest history length **that makes good predictions**

That last bullet point is key! There are a number of ideas (and a lot of important implementation details and tuning!) that go into achieving it. I'm going to mostly stay with the big ideas; I'll make sure to link to some papers and code if you want the details.

## Tagged tables

The tables in TAGE are "tagged." What does that mean?

For any of our predictors, we need to store (in hardware) a table mapping from some key, to our prediction (and potentially other state). For [our first predictor](#the-most-basic-dynamic-predictor), for instance, our key was just the program counter at each branch.

Typically, in hardware, we'll implement this by creating a 2^k-entry [SRAM][sram], and directly indexing it using the low `k` bits of the program counter. For instance, we might have a 1024-row table, indexed by the low 10 bits. In each row, we store a single bit (was the branch taken, or not?)[^counter].

Of course, we need to worry about what happens when multiple branches exist in the program, at the same offset modulo 2^k, such that they map onto the same row of the table.

Suppose we want to detect collisions of this form, and act on them somehow. Any colliding addresses **by definition** have their low 10 bits in common, so if we want to detect a collision, we need to store the remaining 22 bits of the address (assuming a 32-bit architecture) in the table, so that we can check what address the table "thinks" its storing.

Those 22 bits, in hardware lingo, are called a "tag"; they let the row know which address it is storing, so that we can check on retrieval whether we're talking about the same thing.

Recall that the actual **payload**, though, may just be a bit or two. Storing a 22-bit tag, in order to store 2 bits of payload, is a hefty tax! And if the table doesn't have an entry for the current PC, what are we going to do, anyways? In many cases, it will be a better choice to just omit the tag, and blindly use the row at `PC % 1024`, and hope things work out. If we have the budget for more bits, we should probably expand the number of entries -- maybe we'll have 16384 rows, and use the low 14 bits -- instead of tagging entries.

In TAGE, however, we have multiple tables (potentially as many as 20 or so!). Thus, we care whether any given table has a "hit" for our current situation; if it doesn't, we would prefer to use a table that does, instead of just using information for a different branch.


[^counter]: In practice, we usually store a 2-bit counter, which we might increment on "taken" and decrement on "not taken"; this allows us to handle branches which **almost-always** resolve one way but allow for occasional deviations. This doesn't change the overall point about storage, though.

[sram]: https://en.wikipedia.org/wiki/Static_random-access_memory
[associative]: https://en.wikipedia.org/wiki/Cache_placement_policies#Set-associative_cache




<!--
## How I think about branch predictors

Discussions of branch predictors tend to be full of clever optimization tricks to squeeze state into a small number of bits and into formats that are efficient to implement in hardware. Transistors, chip area, and power are all contended resources, and branch predictors must pay for themselves and be fast enough to "keep up" with the rest of the pipeline, and thus are highly-optimized and shrunk to use the minimum amount of state possible.

Reading about branch prediction, I notice myself dividing the problem into two conceptually-separable pieces:

1. A theory of branch predictability

   In order for branch prediction to be a profitable endeavor, we typically have to believe that program execution exhibits **some** sort of predictable structure, which in practice means that it is repetitive in some way. Our belief about the **nature** of this repetitive structure, and in particular, about what information is necessary and/or sufficient to predict branching behavior, I will refer to our "theory" of branch structure or prediction.

   For example, one of the simplest kinds of dynamic branch predictors stores the result from the **last** time it saw a given branch, and assumes the branch will have the same result next time. Our theory, in this case, is that any given branch instruction will tend to repeat the same behavior (taken or not-taken), many times in a row.

2. Implementation strategies and tradeoffs

   As mentioned, efficiency along various measures is paramount to successful branch prediction. Thus, given a theory, we have to actually encode it into hardware somehow.

   Continuing our example from above, we may start with the theory that any given branch instruction exhibits repetitive information. However, in hardware, we cannot simply instantiate and unbounded `Map<PC, bool>` to store the result from every branch we've ever seen. Instead, we will pick an **approximation** to our theory, tuned based on experiment to have a good tradeoff between resource cost and performance.

   In practice, for that theory, this might look like a 1024-bit table, indexed by the low 10 bits of the instruction pointer. If we encounter multiple branch instructions at the same address (mod 1024), they will collide in this table, but we hope that in practice we can still achieve good results.

Of course, these two components are necessarily tightly intertwined; if a theory produces very good predictions in the abstract, but cannot be efficiently implemented, it doesn't necessarily help us. Nonetheless, I find this separation a helpful framework, and in this post I'm mostly going to focus on the prediction **theory** of TAGE and ITTAGE, since that's the part I found most novel!
-->

- intro: interpreters



[pipeline]: https://en.wikipedia.org/wiki/Instruction_pipelining
[danluu]: https://danluu.com/branch-prediction/
[tage-paper]: https://inria.hal.science/hal-03408381/document
[boom-tage]: https://docs.boom-core.org/en/latest/sections/branch-prediction/backing-predictor.html#the-tage-predictor
