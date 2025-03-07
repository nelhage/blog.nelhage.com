---
title: "Performance of the Python 3.14 tail-call interpreter"
slug: cpython-tail-call
date: 2025-03-05T14:28:38-08:00
extra_css:
- css/tables.css
---

About a month ago, the CPython project merged [a new implementation strategy][tail-call-pr] for their bytecode interpreter. The headline results are very impressive, showing a 10-15% performance improvement **on average** across the entire [pyperformance][pyperformance] performance suite, across a variety of platforms.

Starting with the bottom line: I believe the impressive performance gains from the tail-calling interpreter are **primarily a side effect of working around a regression in LLVM 19.** When benchmarked against GCC, against clang 18, or against LLVM 19 with certain tuning flags, the performance gain drops into the 1-3%.

I want to be very clear up front that the tail-calling interpreter is (in my opinion) still an excellent piece of work (and still a real performance improvement). I also think it's a promising approach and hopefully more robust, in some ways I'll explore in the rest of this post.

Also, the impact of the LLVM regression doesn't seem to have been known prior to this work (and it still isn't fixed, as of this writing); thus, in that sense, this project really did produce a 10% speedup compared to a _status quo ante_ of "shipping Python built with clang 19." For instance, Simon Willison [reproduced the 10% speedup][simonw] "in the wild," as compared to Python 3.13, using builds from [`python-build-standalone`][python-build-standalone].

[python-build-standalone]: https://github.com/astral-sh/python-build-standalone
[simonw]: https://simonwillison.net/2025/Feb/13/python-3140a5/

I'm writing this note, thus, not to say anything bad about the tail-call work. However, there's been a lot of (understandable!) excitement for the tail-call work, and I want to have a centralized place to document my more-nuanced current understanding. I also think that this example makes a great case study of some of the challenges of benchmarking, performance engineering, and modern software engineering, and I will [reflect on those topics](#reflections) at the end of this post.

# Performance results

Here are my headline results, motivating the above claims. I benchmarked several Python configurations, using several different compilers, on both an Intel server (a [Raptor Lake i5-13500][intel] I maintain in Hetzner), and on my Apple M1 Macbook Air. You can reproduce these builds [using my `nix` configuration][nix-config], which I found essential for managing so many different moving pieces at once.

[intel]: https://www.intel.com/content/www/us/en/products/sku/230580/intel-core-i513500-processor-24m-cache-up-to-4-80-ghz/specifications.html
[nix-config]: https://github.com/nelhage/cpython-interp-perf/

All builds use LTO and PGO. These configurations are:
- `clang18`: Built using clang 18.1.8, using computed gotos.
- `gcc` (Intel only): Built with GCC 14.2.1, using computed gotos.
- `clang19`: Built using clang 19.1.7, using compute gotos.
- `clang19.tc`: Built using clang 19.1.7, using the new tail-call interpreter.
- `clang19.taildup`: Built using clang 19.1.7, computed gotos and some `-mllvm` tuning flags which work around the regression.

I've used `clang18` as the baseline, and reported the bottom-line "average" reported by `pypeformance`/`pyperf compare_to`. You can find the complete output files and reports [on github][benchmarks].

[benchmarks]: https://github.com/nelhage/cpython-interp-perf/tree/data/

| Platform             | clang18 | clang19      | clang19.taildup | clang19.tc   | gcc          |
|----------------------|---------|--------------|-----------------|--------------|--------------|
| Raptor Lake i5-13500 | (ref)   | 1.09x slower | 1.01x faster    | 1.03x faster | 1.02x faster |
| Apple M1 Macbook Air | (ref)   | 1.12x slower | 1.02x slower    | 1.00x slower | N/A          |

You can observe that the tail-call interpreter still exhibits a speedup when compared to clang18, but a much-less-dramatic one, and that the clang19 regression is quite stark.

# The regression

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

Modern C compilers support [taking the address of labels][address-of-label], and then using those labels in a "computed goto," in order to implement this pattern. Thus, many modern bytecode interpreters, including CPython (before the tail-call work), contain a bytecode interpreter loop that looks something like:

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

## The bug

For performance reasons (performance of the compiler, not the generated code), it turns out that clang and LLVM, internally, actually turn the latter code code, internally, into something that looks structurally more like the original `while(){switch}`-based implementation! That is, they merge all of the "computed goto" statements into a **single** [indirectbr][indirectbr] instruction in the LLVM IR, and branch to that instruction from the end of each opcode definition.

Then, during code generation, they implement a pass which implement "tail duplication" to copy the `indirectbr` back into every basic block, in order to restore the original intent. You can read more [in this old blog post from the LLVM blog][llvm-computed-goto].

Clang 19 included [a tweak to that tail-duplication pass][llvm-bugpr], which added some limits in order to try not to blow up the size of the internal IR.

Unfortunately, those limits resulted in `clang` **leaving all of the dispatch jumps merged**, and undoing the entire benefit of the computed-goto-based implementation! This bug was [first identified][llvm-bug] by another language implementation with a similar interpreter loop, but had not been known (as far as I can find) to affect CPython.

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

In fact, there's something even odder going on, which I can't fully explain.

Classically, the optimization of replicating the bytecode dispatch into each opcode is cited as speeding up interpreters anywhere from [20%](https://github.com/python/cpython/blob/c718c6be0f82af5eb0e57615ce323242155ff014/Misc/HISTORY#L15252-L15255) to [100%](https://link.springer.com/content/pdf/10.1007/3-540-44681-8_59.pdf). However, on modern processors with improved branch predictors, [more recent work](https://inria.hal.science/hal-01100647/document) finds a much smaller speedup, on the order of 2-4%.

Thus, I can't explain why the clang19 performance regression is as steep as it is. I can confirm that it's fixed by re-enabling tail duplication, but I cannot explain the 10% slowdown **only* as a result of tail merging.

### Do we need computed gotos?

In fact, as it turns out, I can measure the impact of tail-merging alone, due to a **very funny** (to me) set of facts.

I explained above that clang:
1. Compiles the `switch` into a jump table and an indirect jump, very similar to the one we create by hand using computed gotos
2. Compiles computed gotos into a control-flow graph that closely resembles the classic `switch` graph, which a single instance of the opcode dispatch, and
3. Is able to reverse the transformation during codegen in order to duplicate the dispatch

Those facts taken together, might lead you to ask, "Could we just start with the `switch`-based interpreter, and have the **compiler** do tail-duplication to get the same benefits?"

And it turns out: Yes.

Clang 18 (or Clang 19 with appropriate flags) actually compiles the "classic" pre-computed-goto interpreter into one that replicates the opcode-dispatch logic into each the body of each opcode.

Moreover, even though clang19 also fails to tail-duplicate the `switch` interpreter (due to the same regression), the slowdown in that case is only about 2% compared to clang18, consistent with the more-modern estimate:


| Benchmark           | clang18 | clang18.nocg | clang19.nocg | clang19      |
|---------------------|:-------:|:------------:|:------------:|:------------:|
| Performance change  | (ref)   | 1.01x faster | 1.02x slower | 1.09x slower |
| # of indirect jumps | 332     | 306          | 3            | 3            |

("nocg" = "no computed gotos"; I've also listed the number of indirect `jmp`s, as reported by the `grep` test above, to illustrate when tail-duplication is and isn't happening).

Thus, although I don't know what, some other behavior is magnifying the impact of the bug in clang19, but **only** for the computed-goto-based interpreter. I also note that my benchmarks claim that computed gotos are a 1% **slowdown** on clang18. I am not confident that number replicates, but it might be worth looking into!

## The fix

[LLVM pull request 114990][llvm-pr] is open and should fix this regression. It has not yet merged as of this writing, but hopefully will soon.

In the meanwhile, the [PR that caused the regression][llvm-bugpr] added a tunable option to choose the threshold at which tail-duplication will abort. We can restore similar behavior to clang 18 by simply setting that limit to a very large number. Note that this is a bit complex to do when using LTO, since the option impacts codegen, and so we need to make sure it actually gets passed through to the **linker**, not just the compiler. I was [able to get clang19](https://github.com/nelhage/cpython-interp-perf/blob/9ed03c33f85e908cc156d35b183c43479243d337/python.nix#L67-L70) to generate comparable performance to clang18 by configuring Python by passing these variables to `./configure`:

```shell
./configure [other flags] \
  "OPT=-g -O3 -Wall -mllvm -tail-dup-pred-size=5000" \
  "LDFLAGS=-fuse-ld=lld -Wl,-mllvm -Wl,-tail-dup-pred-size=5000"
```

# Reflections

I will freely admit that I got nerdsniped quite effectively by this topic, and have gone far deeper than was really necessary. That said, having done so, I think there are a number of interesting lessons and reflections to be taken away, which generalize to software engineering and performance engineering, and I will attempt to extract and meditate upon some of them.

## On benchmarking

When optimizing a system, we generally construct some set of benchmarks and benchmarking methodology, and then evaluate proposed changes using those benchmarks.

Benchmarking can sometimes **sound** easy, but it's fairly well-known to people who do a lot of performance work that it's an endlessly subtle and complex field, with lots of ways to confuse or deceive yourself, or to produce results that do not necessarily transfer from your benchmarks into production.

Any set of benchmarks or benchmark procedures embeds (often implicitly) what I like to call a "theory of performance." Your theory of performance is a set of beliefs and assumptions that answer questions like "which variables (may) effect performance, in what ways?" and "what is the relationship between results on benchmarks and the "true" performance in "production"?"

The benchmarks ran on the tail-call interpreter showed a 10-15% speedup when compared with the old computed-goto interpreter. Those benchmarks were accurate, in that they were accurately measuring (as far as I know) the performance difference between those builds. However, in order to generalize those specific data points into the statement "The tail-call interpreter is 10-15% faster than the computed-goto interpreter, more generally," or even "The tail-call interpreter will speed up Python by 10-15% for our users," we need to bring in more assumptions and beliefs about the world. In this case, it turns out the story was more complex, and those broader claims were not true in full generality.

(I want to be clear that I don't blame the Python developers! This stuff is **hard** and there are a million ways to get confused or to reach somewhat-incorrect conclusions. I had to do ~three weeks of intense benchmarking and experimenting to reach a better understanding. My point is that this is a very general challenge!)

### Baselines

This example highlights another recurring challenge, not only in software performance, but in many other domains: "What baseline do you compare against?"

In almost any domain, if you propose a new solution or method for some problem, you typically have a way of running your new method, and producing some relevant performance metrics.

Once you have metrics for **your** system, however, you need to know what to compare them against, in order to decide if they're any good! Even if you score well on some absolute scale (assuming there **is** a sensible absolute scale to evaluate), if your method is worse than an existing solution, it's probably not that interesting.

Thus, typically, you want to compare against "the current best-known approach." But sometimes that can be hard to do! Even if you understand the current approach in theory, you may or may not be an expert in applying it in practice. In the case of software this may mean something like, tuning your operating system or compiler options or other flags. The current-best approach may have published benchmarks, but they're not always relevant to you; for instance, maybe it was published years ago on older hardware, and so you can't do an apples-to-apples comparison with the public numbers. Or maybe their tests were run at a scale you can't afford to replicate.

I work in machine learning at Anthropic these days, and we see this all the time in ML papers. When a paper comes out claiming some algorithmic improvement or other advance, I've noticed that the first detail our researchers ask is often not "What did they do?" but "What baseline did they compare against?" It's easy to get impressive-looking results if you're comparing against a poorly-tuned baseline, and this turns out to explain a surprising fraction of supposed improvements.

## On software engineering

One other highlight, for me, is just how complex and interconnected our software systems are, and how rapidly-moving, and how hard it is to keep track of all the pieces.

If you'd asked me, a month ago, to estimate the likelihood that an LLVM release caused a 10% performance regression in CPython and that no one noticed for five months, I'd have thought that a pretty unlikely state of affairs! Those are both widely-used projects, both of which care a fair bit about performance, and "surely" someone would have tested and noticed.

And probably that **particular** situation was quite unlikely! However, with so many different software projects out there, each moving so rapidly and depending on and being used by so many other projects, it becomes practically-inevitable that **some** regressions "like that one" happen, almost constantly.

### Optimizing compilers

The saga of the computed-goto interpreter illustrates recurring tensions and unresolved questions around optimizers and optimizing compilers, to which we don't yet have agreed-upon answers as a field.

We generally "want" our compilers to respect the programmer's intent, and to compile the code that was written in a way that matches the programmer's intent and is understandable to the programmer. We also expect our compilers to optimize our code, and to transform it in potentially-complex-and-unintuitive ways in order to make it run faster. However, we have a dearth of patterns and idioms to explain to the compiler "why" we wrote code in various ways, and whether a given pattern was deliberately chosen for some performance reason or not.

Our compilers typically only **promise** to emit code with "the same behavior" as the code we write; performance is something of a best-effort feature on top of that guarantee.

Thus, we end up in this odd world where the compiler compiles the computed-goto interpreter "correctly," in the sense that the resulting binary produces all the same value we expect, but at the same time it produces an output **completely** at odds with the intention of the optimization. Moreover, we also see other versions of the compiler applying optimizations to the "naive" `switch()`-based interpreter, which implement the exact same optimization we "intended" to perform by rewriting the source code.

In hindsight, it appears that the "computed goto" interpreter, at a source code level, and "replicating the dispatch at a machine-code level" end up being almost-orthogonal notions! We've seen examples of every instance of the resulting 2x2 matrix, and because all of those `python` binaries compute the same values given the same inputs, our current tools are virtually unable to talk about the distinctions between them in a coherent way.

This confusion is one way in which I think the tail-calling interpreter (and the compiler features behind it) represent a genuine, and useful, advance in the state of the art. The tail-call interpreter is built on [the `musttail` attribute][musttail], which represents a relatively new **kind** of compiler feature. `musttail` does not affect the "observable program behavior," in the classic sense that compilers think, but is rather a conversation with the **optimizer**; it requires that the compiler be able to make certain optimizations, and requires that compilation **fail** if those optimizations don't happen.

I suspect that this framework represents a much more **style** for writing performance-sensitive code, especially over time and the evolution of the compiler. I look forward to more experiments with features in that category, and to many clever engineers finding more ways to exploit them for performance gains.

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

I was even able to build a custom version of LLVM (with the bugfix patch), and do Python builds using that compiler. Doing so required [about another 10 lines of code](https://github.com/nelhage/cpython-interp-perf/blob/afd2123bef6ec3fd872d628f2bb519b20684e161/llvm.nix#L14-L25).

That said, not everything was rosy. For one, `nix` is, by necessity, "weird" in various ways compared to the ways "normal people" use software, and I worry that some of that weirdness may have affected some of my benchmarks or conclusions in ways I didn't notice. For instance, early on I discovered that nix (by default) builds projects using certain hardening flags that [disproportionately impact the tail-call interpreter](https://github.com/python/cpython/issues/130961). I've handled that one, but are there more?

Nix is also incredible extensible and customizable, but figuring out **how** to make a specific customization can be a real uphill climb involving a lot of reading source and trial and error. I mentioned my patched LLVM build; the end result ended up being neat, but getting there required me to read a lot of `nix` source code, to mix and match two un- or under-documented extensibility mechanisms (`extend` and `overrideAttrs` -- not to be confused with normal `override`, which I used elsewhere), and one failed attempt which successfully patched `libllvm`, but then silently built a new `clang` against the unpatched version.

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
