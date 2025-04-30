---
title: "Performance of the Python 3.14 tail-call interpreter"
slug: cpython-tail-call
date: 2025-03-09T15:00:00-07:00
description: >
  A deep dive into the performance of Python 3.14's tail-call interpreter: How the performance results were confounded by an LLVM regression, the surprising complexity of compiling interpreter loops, and some reflections on performance work, software engineering, and optimizing compilers.
extra_css:
- css/tables.css
---

About a month ago, the CPython project merged [a new implementation strategy][whatsnew] for their bytecode interpreter. The initial headline results [were very impressive][tail-call-pr], showing a 10-15% performance improvement **on average** across [a wide range of benchmarks][pyperformance] across a variety of platforms.

[whatsnew]: https://docs.python.org/3.14/whatsnew/3.14.html#whatsnew314-tail-call

Unfortunately, as I will document in this post, these impressive performance gains turned out to be **primarily due to inadvertently working around a regression in LLVM 19.** When benchmarked against a better baseline (such GCC, clang-18, or LLVM 19 with certain tuning flags), the performance gain drops to 1-5% or so depending on the exact setup.

When the tail-call interpreter was announced, I was surprised and impressed by the performance improvements, but also confused: I'm not an expert, but I'm passingly-familiar with modern CPU hardware, compilers, and interpreter design, and I couldn't explain why this change would be so effective. I became curious -- and perhaps slightly obsessed -- and the reports in this post are the result of a few weeks of off-and-on compiling and benchmarking and disassembly of dozens of different Python binaries, in an attempt to understand what I was seeing.

At the end, I [will reflect](#reflections) on this situation as a case study in some of the challenges of benchmarking, performance engineering, and software engineering in general.

I also want to be clear that I still think the tail-calling interpreter is a great piece of work, as well as a genuine speedup (albeit more modest than initially hoped). I am also optimistic it's a more robust approach than the older interpreter, in ways I'll explain in this post. I also really don't want to blame anyone on the Python team for this error. This sort of confusion turns out to be very common -- I've certainly misunderstood many a benchmark myself -- and I'll have some reflections on that topic at the end.

In addition, the impact of the LLVM regression doesn't seem to have been known prior to this work (and the bug wasn't fixed as of publishing this post, although it [since has been][llvm-pr]); thus, in that sense, the alternative (without this work) probably really was 10-15% slower, for builds using clang-19 or newer. For instance, Simon Willison [reproduced the 10% speedup][simonw] "in the wild," as compared to Python 3.13, using builds from [`python-build-standalone`][python-build-standalone].

[python-build-standalone]: https://github.com/astral-sh/python-build-standalone
[simonw]: https://simonwillison.net/2025/Feb/13/python-3140a5/

# Performance results

Here are my headline results. I benchmarked several builds of the CPython interpreter, using multiple different compilers and different configuration options, on two machines: an Intel server (a [Raptor Lake i5-13500][intel] I maintain in Hetzner), and my Apple M1 Macbook Air. You can reproduce these builds [using my `nix` configuration][nix-config], which I found essential for managing so many different moving pieces at once.

[intel]: https://www.intel.com/content/www/us/en/products/sku/230580/intel-core-i513500-processor-24m-cache-up-to-4-80-ghz/specifications.html
[nix-config]: https://github.com/nelhage/cpython-interp-perf/

All builds use LTO and PGO. These configurations are:
- `clang18`: Built using Clang 18.1.8, using computed gotos.
- `gcc` (Intel only): Built with GCC 14.2.1, using computed gotos.
- `clang19`: Built using Clang 19.1.7, using computed gotos.
- `clang19.tc`: Built using Clang 19.1.7, using the new tail-call interpreter.
- `clang19.taildup`: Built using Clang 19.1.7, computed gotos and some `-mllvm` tuning flags which work around the regression.

I've used `clang18` as the baseline, and reported the bottom-line "average" reported by `pypeformance`/`pyperf compare_to`. You can find the complete output files and reports [on github][benchmarks].

[benchmarks]: https://github.com/nelhage/cpython-interp-perf/tree/data/

| Platform             | clang18 | clang19      | clang19.taildup | clang19.tc   | gcc          |
|----------------------|---------|--------------|-----------------|--------------|--------------|
| Raptor Lake i5-13500 | (ref)   | 1.09x slower | 1.01x faster    | 1.03x faster | 1.02x faster |
| Apple M1 Macbook Air | (ref)   | 1.12x slower | 1.02x slower    | 1.00x slower | N/A          |

Observe that the tail-call interpreter still exhibits a speedup as compared to clang-18, but that it's far less dramatic than the slowdown from moving to clang-19. The Python team has also observed larger speedups than I have (after accounting for the bug) on some other platforms.

You'll notice I didn't benchmark the tail-call interpreter on the older Clang release (what would be `clang18.tc`). The tail-call interpreter relies on new compiler features which only landed in Clang 19, meaning we can't test it on earlier versions. This interaction, I think, is a big reason this story was so confusing, and why it took me so many benchmarks to be **confident** I understood the situation.

# The LLVM regression

## A brief background

A classic bytecode interpreter consists of a `switch` statement inside of a `while` loop, looking something like so:

```c++
while (true) {
  opcode_t this_op = bytecode[pc++];
  switch (this_op) {
    case OP_IMM: {
      // push an immediate onto the stack
      break;
    }
    case OP_ADD: {
      // handle the add
      break;
    }
    // etc
  }
}
```

Most compilers will compile the `switch` into a jump table -- they will emit a table containing the address of each `case OP_xxx` block, index into it with the opcode, and perform an indirect jump.

It's [long been known](https://link.springer.com/content/pdf/10.1007/3-540-44681-8_59.pdf) that you can speed up a bytecode interpreter of this style by replicating the jump table dispatch into the body of each opcode. That is, instead of ending each opcode with a `jmp loop_top`, each opcode contains a separate instance of the "decode next instruction and index through the jump table" logic.

Modern C compilers support [taking the address of labels][address-of-label], and then using those labels in a "computed goto," in order to implement this pattern. Thus, many modern bytecode interpreters, including CPython (before the tail-call work), employ an interpreter loop that looks something like:

```c++
static void *opcode_table[256] = {
    [OP_IMM] = &&TARGET_IMM,
    [OP_ADD] = &&TARGET_ADD,
    // etc
};

#define DISPATCH() goto *opcode_table[bytecode[pc++]]

DISPATCH();

TARGET_IMM: {
    // push an immediate onto the stack
    DISPATCH();
}
TARGET_ADD: {
    // handle the add
    DISPATCH();
}
```

## Computed goto in LLVM

For performance reasons (performance of the compiler, not the generated code), it turns out that Clang and LLVM, internally, actually merges all of the `goto`s in the latter code into a **single** [`indirectbr` LLVM instruction][indirectbr], which each opcode will jump to. That is, the compiler takes our hard work, and deliberately rewrites into a control-flow-graph that looks essentially the same as the `switch`-based interpreter!

Then, during code generation, LLVM performs "tail duplication," and copies the branch **back** into each location, restoring the original intent. This dance is documented, at a high level, [in an old LLVM blog post][llvm-computed-goto] introducing the new implementation.

## The LLVM 19 regression

The whole reason for the deduplicate-then-copy dance is that, for technical reasons, creating and manipulating the control-flow-graph containing many `indirectbr` instructions can be quite expensive.

In order to avoid catastrophic slowdowns (or memory usage) in certain cases, LLVM 19 implemented [some limits on tail-duplication pass][llvm-bugpr], causing it to bail out if duplication would blow up the size of the IR past certain limits.

Unfortunately, on CPython those limits resulted in Clang **leaving all of the dispatch jumps merged**, and entirely undoing the whole purpose of the computed `goto`-based implementation! This bug was [first identified][llvm-bug] by another language implementation with a similar interpreter loop, but had not been known (as far as I can find) to affect CPython.

In addition to the performance impact, we can observe the bug directly by disassembling the resulting object code and counting the number of distinct indirect jumps:

```shell
$ objdump -S --disassemble=_PyEval_EvalFrameDefault ${clang18}/bin/python3.14 | \
  egrep -c 'jmp\s+\*'
332

$ objdump -S --disassemble=_PyEval_EvalFrameDefault ${clang19}/bin/python3.14 | \
  egrep -c 'jmp\s+\*'
3
```

### Further weirdness

I am confident that the change to the tail-call duplication logic caused the regression: if [you fix it](#the-fix), performance matches clang-18. However, I can't fully explain the **magnitude** of the regression.

Historically, the optimization of replicating the bytecode dispatch into each opcode has been cited to speed up interpreters anywhere from [20%](https://github.com/python/cpython/blob/c718c6be0f82af5eb0e57615ce323242155ff014/Misc/HISTORY#L15252-L15255) to [100%](https://link.springer.com/content/pdf/10.1007/3-540-44681-8_59.pdf). However, on modern processors with improved branch predictors, [more recent work](https://inria.hal.science/hal-01100647/document) finds a much smaller speedup, on the order of 2-4%.

We can verify this 2-4% number in practice, because Python still supports the "old-style" interpreter, which uses a single `switch` statement, via a configuration option. Here's what we see if we benchmark that interpreter (".nocg" for "no computed gotos" in the following table):

| Benchmark           | clang18 | clang18.nocg | clang19.nocg | clang19      |
|---------------------|:-------:|:------------:|:------------:|:------------:|
| Performance change  | (ref)   | 1.01x faster | 1.02x slower | 1.09x slower |


Notice that `clang19.nocg`" is only 2% slower than `clang18`, even though the base `clang19` build is 9% slower! I interpret that "2%" as a fairer estimate for the cost/benefit of duplicating opcode dispatch, alone, and I don't fully understand the other one.

### Do we need computed gotos?

I haven't mentioned the `clang19.nocg` benchmark, which you may notice claims to be **faster** than `clang19`. It was at this point that I discovered an additional, and very funny, twist to the story.

I explained earlier that Clang and LLVM:
1. Compiles the `switch` into a jump table and an indirect jump, very similar to the one we create by hand using computed gotos
2. Compiles computed gotos into a control-flow graph that closely resembles the classic `switch` graph, which a single instance of the opcode dispatch, and
3. Is able to reverse the transformation during codegen in order to duplicate the dispatch

Those facts taken together, might lead you to ask, "Couldn't we just start with the `switch`-based interpreter, and have the **compiler** do tail-duplication, and get the same benefits?"

And it turns out: Yes.

clang-18 (or clang-19 with appropriate flags), when presented with the "classic" `switch`-based interpreter, **goes ahead and duplicates the dispatch logic into each the body of each opcode anyways**. Here's another table, showing the same builds with the number of indirect jumps, using the `objdump | grep` test from earlier:

| Benchmark           | clang18 | clang18.nocg | clang19.nocg | clang19      |
|---------------------|:-------:|:------------:|:------------:|:------------:|
| # of indirect jumps | 332     | 306          | 3            | 3            |

Thus, there's a case to be made that the entire "computed goto" interpreter turns out to be entirely unnecessary complexity (at least for modern Clang). The compiler is perfectly capable of performing the same transformation itself, and (apparently) the computed gotos don't even suffice to guarantee it!

That said, I did also test GCC, and GCC (at least as of 14.2.1) does not replicate the `switch`, but does implement the desired behavior for when using computed goto. So at least in that case we see the expected behavior.

## The fix

[LLVM pull request 114990][llvm-pr] merged shortly after I published this post, and fixes the regression. I was able to benchmark it before merge and confirm it restores the expected performance.

For releases before that fix, the [PR that caused the regression][llvm-bugpr] added a tunable option to choose the threshold at which tail-duplication will abort. We can restore similar behavior on clang-19 by simply setting that limit to a very large number[^lto].


[^lto]: Note that setting this option is a bit complex to do when using LTO. Tail duplication happens during code-generation, and for LTO builds, the code generation actually happens at **link time**, not compile-time. Thus, we need to make sure the flag is passed down to `lld`, not just to the compiler. I was [able to get it to work](https://github.com/nelhage/cpython-interp-perf/blob/9ed03c33f85e908cc156d35b183c43479243d337/python.nix#L67-L70) by configuring Python with these variables at `./configure`-time:

    ```shell
    ./configure [other flags] \
      "OPT=-g -O3 -Wall -mllvm -tail-dup-pred-size=5000" \
      "LDFLAGS=-fuse-ld=lld -Wl,-mllvm -Wl,-tail-dup-pred-size=5000"
    ```

# Reflections

I will freely admit that I got nerdsniped quite effectively by this topic, and have gone far deeper than was really necessary. That said, having done so, I think there are a number of interesting lessons and reflections to be taken away, which generalize to software engineering and performance engineering, and I will attempt to extract and meditate upon some of them.

## On benchmarking

When optimizing a system, we generally construct some set of benchmarks and benchmarking methodology, and then evaluate proposed changes using those benchmarks.

Any set of benchmarks or benchmark procedures embeds (often implicitly) what I like to call a "theory of performance." Your theory of performance is a set of beliefs and assumptions that answer questions like "which variables (may) effect performance, in what ways?" and "what is the relationship between results on benchmarks and the "true" performance in "production"?"

The benchmarks ran on the tail-call interpreter showed a 10-15% speedup when compared with the old computed-goto interpreter. Those benchmarks were accurate, in that they were accurately measuring (as far as I know) the performance difference between those builds. However, in order to generalize those specific data points into the statement "The tail-call interpreter is 10-15% faster than the computed-goto interpreter, more generally," or even "The tail-call interpreter will speed up Python by 10-15% for our users," we need to bring in more assumptions and beliefs about the world. In this case, it turns out the story was more complex, and those broader claims were not true in full generality.

(Once again, I really don't want to blame the Python developers! This stuff is **hard** and there are a million ways to get confused or to reach somewhat-incorrect conclusions. I had to do ~three weeks of intense benchmarking and experimenting to reach a better understanding. My point is that this is a very general challenge!)

### Baselines

This example highlights another recurring challenge, not only in software performance, but in many other domains: "What baseline do you compare against?"

Any time you propose a new solution or method for some problem, you typically have a way of running your new method, and producing some relevant performance metrics.

Once you have metrics for **your** system, however, you need to know what to compare them against, in order to decide if they're any good! Even if you score well on some absolute scale (assuming there **is** a sensible absolute scale to evaluate), if your method is worse than an existing solution, it's probably not that interesting.

Typically, you want to compare against "the current best-known approach." But sometimes that can be hard to do! Even if you understand the current approach in theory, you may or may not be an expert in applying it in practice. In the case of software this may mean something like, tuning your operating system or compiler options or other flags. The current-best approach may have published benchmarks, but they're not always relevant to you; for instance, maybe it was published years ago on older hardware, and so you can't do an apples-to-apples comparison with the public numbers. Or maybe their tests were run at a scale you can't afford to replicate.

I work in machine learning at Anthropic these days, and we see this all the time in ML papers. When a paper comes out claiming some algorithmic improvement or other advance, I've noticed that the first detail our researchers ask is often not "What did they do?" but "What baseline did they compare against?" It's easy to get impressive-looking results if you're comparing against a poorly-tuned baseline, and that observation turns out to explain a surprising fraction of supposed improvements.

## On software engineering

One other highlight, for me, is just how complex and interconnected our software systems are, and how rapidly-moving, and how hard it is to keep track of all the pieces.

If you'd asked me, a month ago, to estimate the likelihood that an LLVM release caused a 10% performance regression in CPython and that no one noticed for five months, I'd have thought that a pretty unlikely state of affairs! Those are both widely-used projects, both of which care a fair bit about performance, and "surely" someone would have tested and noticed.

And probably that **particular** situation was quite unlikely! However, with so many different software projects out there, each moving so rapidly and depending on and being used by so many other projects, it becomes practically-inevitable that **some** regressions "like that one" happen, almost constantly.

### Optimizing compilers

The saga of the computed-goto interpreter illustrates recurring tensions and unresolved questions around optimizers and optimizing compilers, to which we don't yet have agreed-upon answers as a field.

We generally expect our compilers to respect the programmer's intent, and to compile the code that was written in a way that preserves the programmer's intent.

We also, however, expect our compilers to optimize our code, and to transform it in potentially-complex-and-unintuitive ways in order to make it run faster.

These expectations are in tension, and we have a dearth of patterns and idioms to explain to the compiler "why" we wrote code in various ways, and whether we were **deliberately** trying to trigger a certain output, or make a certain performance-related decision, or not.

Our compilers typically only **promise** to emit code with "the same behavior" as the code we write; performance is something of a best-effort feature on top of that guarantee.

Thus, we end up in this odd world where clang-19 compiles the computed-goto interpreter "correctly" -- in the sense that the resulting binary produces all the same value we expect -- but at the same time it produces an output completely at odds with the intention of the optimization. Moreover, we also see other versions of the compiler applying optimizations to the "naive" `switch()`-based interpreter, which implement the exact same optimization we "intended" to perform by rewriting the source code.

In hindsight, it appears that the "computed goto" interpreter, at a source code level, and "replicating the dispatch at a machine-code level" end up being almost-orthogonal notions! We've seen examples of every instance of the resulting 2x2 matrix! Because all of those `python` binaries compute the same values when run, our current tools are essentially unable to talk about the distinctions between them in a coherent way.

This confusion is one way in which I think the tail-calling interpreter (and the compiler features behind it) represent a genuine, and useful, advance in the state of the art. The tail-call interpreter is built on [the `musttail` attribute][musttail], which represents a relatively new kind of compiler feature. `musttail` does not affect the "observable program behavior," in the classic sense that compilers think, but is rather a conversation with the **optimizer**; it requires that the compiler be able to make certain optimizations, and requires that compilation fail if those optimizations don't happen.

I'm hopeful this framework will turn out to be a much more robust style for writing performance-sensitive code, especially over time and as compilers evolve. I look forward to continued experiments with features in that category.

Concretely, I find myself wondering if it would be viable to replace the computed-goto interpreter with something like a (hypothetical) `[[clang::musttailduplicate]]` attribute on the interpreter `while` loop. I'm not expert enough in all the relevant IRs and passes to have confidence in this proposal, but perhaps someone with more familiarity can weigh in on the feasibility.

[musttail]: https://clang.llvm.org/docs/AttributeReference.html#musttail


## One more note: on `nix`

I want to close with a call-out of how helpful `nix` was for this project. I have been experimenting with nix and NixOS for my personal infrastructure over the last year or so, but they turned out to be a total lifesaver for this investigation.

In the course of these experiments, I have built and benchmarked **dozens** of different Python interpreters, across four different compilers (`gcc`, `clang-18`, `clang-19`, and `clang-20`) and using numerous combinations of compiler flags. Managing all of that by hand would have strained my sanity, and I'm **certain** I would have made numerous mistakes during which I mixed up which compiler and which flags went into which build, and so on.

Using `nix`, I was able to keep all of these parallel versions straight, and build them in a reproducible, hermetic style. I was able to write some short abstractions which made them very easy to define, and then know with absolute confidence **where** any given build in my `nix` store came from, with which compilers and which flags. After a small amount of work to build some helper functions, the core definitions of my build matrix is [shockingly concise](https://github.com/nelhage/cpython-interp-perf/blob/afd2123bef6ec3fd872d628f2bb519b20684e161/python.nix#L105-L143); here's a taste:

```nixos
{
    base = callPackage buildPython { python3 = python313; };
    optimized = withOptimizations base;
    optLTO = withLTO optimized;

    clang18 = withLLVM llvmPackages_18 optLTO;
    clang19 = withLLVM llvmPackages_19 optLTO;
    clang20 = withLLVM llvmPackages_20 optLTO;

    clang18nozero = noZeroCallUsed clang18;
    clang18nocg = withoutCG clang18;

    clang19taildup = withTailDup clang19;
}
```

I was even able to build a custom version of LLVM (with the bugfix patch), and do Python builds using that compiler. Doing so required [all of about 10 lines of code](https://github.com/nelhage/cpython-interp-perf/blob/afd2123bef6ec3fd872d628f2bb519b20684e161/llvm.nix#L14-L25).

That said, not everything was rosy. For one, `nix` is, by necessity, "weird" in various ways compared to the ways "normal people" use software, and I worry that some of that weirdness may have affected some of my benchmarks or conclusions in ways I didn't notice. For instance, early on I discovered that nix (by default) builds projects using certain hardening flags that [disproportionately impact the tail-call interpreter](https://github.com/python/cpython/issues/130961). I've handled that one, but are there more?

In addition, Nix is incredibly extensible and customizable, but figuring out how to make a specific customization can be a real uphill battle, and involve a lot of trial and error and source-diving. My patched LLVM build ended up being pretty short and clean, but getting there required me to read a lot of `nixpkgs` source code, mixing and matching two under-documented extensibility mechanisms (`extend` and `overrideAttrs` -- not to be confused with `override`, used elsewhere), and one failed attempt which successfully patched `libllvm`, but then silently built a new `clang` against the unpatched version.

Still, `nix` was clearly enormously helpful here, and on net it definitely made this kind of multi-version exploration and debugging **much** saner than any other approach I can imagine.


[llvm-bugpr]: https://github.com/llvm/llvm-project/pull/78582
[llvm-pr]: https://github.com/llvm/llvm-project/pull/114990
[address-of-label]: https://gcc.gnu.org/onlinedocs/gcc/Labels-as-Values.html#Labels-as-Values
[llvm-computed-goto]: https://blog.llvm.org/2010/01/address-of-label-and-indirect-branches.html
[indirectbr]: https://llvm.org/docs/LangRef.html#indirectbr-instruction
[tail-call-pr]: https://github.com/python/cpython/pull/128718
[pyperformance]: https://pyperformance.readthedocs.io/
[dispatch-bug]: https://github.com/python/cpython/issues/129987
[llvm-bug]: https://github.com/llvm/llvm-project/issues/106846
