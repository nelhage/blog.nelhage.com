---
title: "TIL: The ITTAGE indirect branch predictor"
slug: ittage
date: 2025-02-22T16:00:38-08:00
---

- [ ] TK better intro
  - [ ] Tease the interpreter fact

for complicated reasons, I [recently learned][colesbury-comment] a bit more about modern branch predictors. Specifically, I learned about the [TAGE and ITTAGE][tage-paper] branch prediction algorithms, which (as best I can tell) are used in some form by state-of-the-art CPUs.

I found this a fun tangent, but also one that appreciably updated my mental models for CPU performance, and which I hadn't even heard of, previously. I'm going to try to write up what I see as the the core ideas, and then I also want to mention a few connections or interesting ideas it raised for me.

[tail-call-pr]: https://github.com/python/cpython/pull/128718
[my-issue]: https://github.com/python/cpython/issues/129987
[colesbury-comment]: https://github.com/python/cpython/issues/129987#issuecomment-2654837868
[interpreter-prediction]: https://inria.hal.science/hal-01100647/document


## Branch Prediction: A very short recap

I'm mostly going to assume familiarity with the concept and motivation of a branch predictor. If you want more background, from someone who actually **does** understand hardware CPU design, I suggest [Dan Luu's writeup][danluu].

But to start with a very brief reminder: Modern CPUs are [deeply pipelined][pipeline], which means they have many instructions being executed concurrently. That, in turn, means that the CPU "frontend," responsible for reading and decoding instructions, is running far ahead of the "backend," which does the actual computation. This poses a problem when executing conditional or indirect branches. When the frontend encounters a conditional branch, it needs to decide where to continue decoding instructions. It can't know for sure whether or not the branch was taken until the backend is done computing the relevant condition; but if it waits for the backend, this creates a "bubble" in the CPU pipeline -- a region where the CPU **could** be executing instructions, but is sitting idle waiting for the frontend to continue fetching and decoding.

In order to minimize the frequency of such bubbles, the frontend will instead try to guess, or "predict," the direction of the branch. The CPU can then continue executing **speculatively** along that predicted path; once the backend "catches up" and computes the true answer, the speculative state can either be committed (made visible outside of the core), or rolled back.

# A "just-so" story

I'm going to give a bit of a "just-so" story building up to (my understanding of) TAGE and ITTAGE. I make no claims that this story is completely accurate in the details or describes the historical path that arrived at modern CPUs, but I believe the core ideas are basically right, and this corresponds to the model I have in my head.

## The inputs to a branch predictor

What information is available to a branch predictor, when making a prediction?

At the most basic, it knows the address (the program counter) of the branch it's trying to predict; this information **must** be available, since the frontend needed it to fetch and decode the branch instruction in the first place.

Beyond that, what else? My impression is that the answer, practically speaking, is "the program counter and branch history prior to this point," and not much else. This makes sense to me: On the one hand, we needed to use that information to produce the current (speculative) program counter value, at which we're fetching and prediction, and so we can choose to remember it and make use of it; On the other hand, we're speculating far and running very early in the CPU pipeline, and so virtually no other information has yet been computed about the logical architectural state of the CPU.

Thus, we can mostly think about a branch predictor as making a prediction based on the current PC and the branch history. Stylistically, we might think of that state like so:

```rust
type PC = usize;

struct Branch {
    address: PC,
    destination: PC,
    kind: BranchKind,
}

enum Outcome { Taken, NotTaken }

enum BranchKind {
    Unconditional,
    Conditional(Outcome),
}

type BranchHistory<const N: usize> = [Branch; N];
```

In practice, hardware predictors do not store all of that information; they use a subset, and may compute a rolling hash or other summary on the fly in order to save hardware. I find it conceptually helpful, though, to write out the full state we **could** use, for thinking about predictors in the abstract.

Once the backend catches up and evaluates the "true" result for each branch (taken or not-taken), the predictor is also informed of this "ground truth" behavior, which we can use to "train" our runtime structures to match the running code.

## Sizing history

Ignoring implementation details, one simple branch prediction strategy is to fix a history length, and then assume that a given (history, branch) pair will repeat its past behavior. Stylistically, I might describe that like:

```rust
type State<const HistorySize: usize> =
  HardwareMap<(BranchHistory<HistorySize>, PC), Outcome>;
```

I've invented the type `HardwareMap` as a reminder that hash tables or key-value maps in hardware behave differently and have different tradeoffs than in software. Notably, our map will typically be fixed-size (corresponding to some chunk of [SRAM][sram] inside the predictor), and will silently overwrite old entries as-needed. Many of the interesting questions in branch predictors come down to choosing the details of this implementation: How many bits do we use from which parts of the history and PC? Do we make our mapping [N-way set-associative][associative] or not? What replacement algorithm do we use? However, I'm going to mostly gloss over them (in part because I am extremely **not** an expert in that domain), and explore some of the higher-level architectural questions.

[sram]: https://en.wikipedia.org/wiki/Static_random-access_memory
[associative]: https://en.wikipedia.org/wiki/Cache_placement_policies#Set-associative_cache

In particular, I want to explore the question of choosing a history length.

Consider this function:

```rust
// Suppose that we create a single Context at program startup, and
// then use it repeatedly over the program's lifetime.
struct Context {
    metrics: Option<Box<dyn Metrics>>,
}

impl<'a> Context<'a> {
    fn increment(self: &Context<'a>, counter: &'static str) {
        if let Some(ref m) = self.metrics {
            m.increment(counter)
        }
    }
}
```

The `if let` conditional will compile into a branch, and that branch will exhibit highly predictable behavior over the course of a program. We don't actually need to use **any** history to predict it; just remembering what happened last time would suffice.

However, suppose that we configure our `Context` to tee to three metric sinks, like so:

```rust
struct MultiMetrics {
    inner: Vec<Box<dyn Metrics>>,
}

impl MultiMetrics {
    fn increment(self: &MultiMetrics, counter: &'static str) {
        for m in &self.inner {
            m.increment(counter)
        }
    }
}
```

The `for` loop will also end in a branch, which will exhibit repetitive "taken, taken, not-taken" behavior. If we examine enough history, we will also be able to predict it; but replaying the previous value will only give us a 2/3s hit rate.



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
