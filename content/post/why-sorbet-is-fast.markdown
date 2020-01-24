---
slug: why-sorbet-is-fast
title: "Why Sorbet is fast"
date: 2020-01-24T10:00:00-08:00
---
This is the second in an indefinite series of posts about things that I think went well in the [Sorbet](https://sorbet.org/) project. The [previous one covered our testing approach.](https://blog.nelhage.com/post/record-replay-in-sorbet/)

Sorbet is fast. Numerous of our early users commented specifically on how fast it was, and how much they appreciated this speed. Our informal benchmarks on Stripe’s codebase clocked it as typechecking around 100,000 lines of code per second per core, making it one of the fastest production typecheckers we are aware of.

In this post, I want to explore some of the reasons for Sorbet’s performance.

## C++

Sorbet is written in C++. We chose C++ for a number of reasons, but performance was definitely one of them. We knew from the start of the project that we wanted to build a fast tool, and believed C++ would give us our best shot at doing so.

Writing in C++ doesn’t automatically make your program fast, and a program does not need to be written in C++ to be fast. However, using C++ well gives an experienced team a fairly unique set of tools to write high-performance software. C++ compiles directly to native code, and gives us access to explicit data structure layout, control over allocation, access to some of the fastest data-structure libraries ever written.

Of the other languages we considered, Rust is the only one that I think *might* have offered us the same raw performance. Some other time I may write a fuller post about our discussions and reasons for language choice, but this is not that post.

## Design for cache locality

Good data structures are at the core of almost any successful software design, and Sorbet is no exception. Modern hardware is **incredibly** sensitive to hit rates on CPU caches, and so our core data structures were designed (among other objectives) to maximize cache locality, informed by [Dmitry](https://d-d.me/)’s extensive experience designing and working on high-performance compilers. I’ll touch on a few core details here.

### `GlobalState` and `*Ref`
All state used within a single typechecker instance, such as the global Ruby namespace and all the methods and constants defined within, live within an instance of the [`GlobalState`](https://github.com/sorbet/sorbet/blob/master/core/GlobalState.h) class.

One key design decision is that symbols and strings in a `GlobalState` are not individually heap-allocated. Instead, the `GlobalState` maintains large flat arrays of objects, and we refer to them using 32-bit indexes, known as `Ref`s ([`NameRef`](https://github.com/sorbet/sorbet/blob/master/core/NameRef.h) and [`SymbolRef`](https://github.com/sorbet/sorbet/blob/master/core/SymbolRef.h)). In addition to simplifying reasoning about object lifetimes, this design has several desirable performance properties:

- We **minimize allocator traffic** by amortizing allocations, since we only grow the backing arrays as needed, and the common case of allocating a `Symbol` is simply bumping an index into the backing array.
- We **enhance locality for the symbol array**. Symbols are stored in a flat array, not scattered throughout the heap, helping caches perform better.
- By using 32-bit indexes to refer to objects, instead of 64-bit pointers, we **fit more references into each cache line**. This also lets us get much more mileage out of the hardware caches. We pay the price of indexing into the array as opposed to directly dereferencing pointers, but this is a single instruction in practice, which is essentially free if it saves even a few cache misses.

### Fast, packed, data structures
We used data structures from Google’s [Abseil](https://abseil.io/) project for most of our core containers. This included:

- [`absl::InlinedVector`](https://github.com/abseil/abseil-cpp/blob/master/absl/container/inlined_vector.h), which is a `std::vector`-like container with the optimization that small vectors are stored directly in the structure, instead of requiring a separate allocation. This, again, avoids allocator traffic and enhances locality. `InlinedVector` pairs well with our `Ref` types: an `absl::InlinedVector` can store up to 4 32-bit values inline without even increasing the size of the structure compared to a `std::vector`
- [`absl::flat_hash_map`](https://abseil.io/docs/cpp/guides/container#hash-tables), which is one of the fastest general-purpose hash tables I know of, due in part to a design optimized for cache usage. See [Google’s CppCon 2017 talk](https://www.youtube.com/watch?v=ncHmEUmJZf4&t=3s) for more details[^1].

[^1]: Note that while we did benchmark `flat_hash_map` when we added it, the performance improvement was less substantial than I expected. Most of Sorbet’s maps are small, and for a long time we actually used a `vector<pair<key, value>>` for many of them, at completely acceptable performance; we believe the cache locality offered by that structure outweighed the requisite O(n) scan in most cases.

## Avoiding string operations

Operations on strings are, in general, much slower than operations on fixed-size data structures and integer comparisons. Sorbet must accept strings (in the form of Ruby source code) as input, but we aim to avoid string operations as much as at-all possible after the parse stage.

The key abstraction we use to this end are the `NameRef` and `SymbolRef` types, which I mentioned previously. A `GlobalState` stores a table of every string we have ever encountered, which are [interned](https://en.wikipedia.org/wiki/String_interning) so that the same string always has the same 32-bit `NameRef` identifier. Since the most common operation on names is comparison, this means we can compare two names using a simple integer comparison, and never even need to load the underlying UTF-8 bytes from memory.

For names that are known to be important at compile-time (e.g. built-in keywords and special-cased operators), we [generate the](https://github.com/sorbet/sorbet/blob/2a16bc6b644e6fc8a309f0b0f31fd3592a02bf1a/core/tools/generate_names.cc) [`NameRef`](https://github.com/sorbet/sorbet/blob/2a16bc6b644e6fc8a309f0b0f31fd3592a02bf1a/core/tools/generate_names.cc)  [identifiers at compile time](https://github.com/sorbet/sorbet/blob/2a16bc6b644e6fc8a309f0b0f31fd3592a02bf1a/core/tools/generate_names.cc), meaning that we save another level of indirection, and a comparison like “Is this identifier the special name `sig`?” compiles into essentially a single instruction.

`Symbol`s and `SymbolRef`s perform the same purpose, but for language-level symbols, such as classes and methods. Asking whether two types refer to the same ruby `Class` (a very common operation when comparing types) is also integer comparison of their `SymbolRef`s, again avoiding even looking at the underlying `Symbol`.

### Lazy error construction
When typechecking code that does not typecheck, just generating error messages — which involves a ton of string formatting and generation — can be easily the most expensive part of the typechecking pipeline!

Sorbet’s error-generation infrastructure uses an idiom which guards all error-message generation inside conditionals, such as the [following snippet](https://github.com/sorbet/sorbet/blob/2a16bc6b644e6fc8a309f0b0f31fd3592a02bf1a/core/types/calls.cc#L183-L187) for emitting arity errors:

```c++
if (auto e = ctx.state.beginError(callLoc, errors::Infer::MethodArgumentCountMismatch)) {
e.setHeader("Missing required keyword argument `{}` for method `{}`", arg.name.show(ctx),
method.data(ctx)->show(ctx));
return e.build();
}
```

This idiom allows the `beginError` function to avoid generating the error text entirely (while still recording that *some* error of type `MethodArgumentCountMismatch` happened). This speeds execution tremendously in contexts where we don’t care about actual error messages, such as when run under `--quiet`.

### Metrics infrastructure
Sorbet has a metrics system, which we use to keep track of statistics about the frequency of various code paths and various language features. While we disable most of these counters in production builds, we keep a small number always active to get visibility into the use of Sorbet in deployment.

We wanted to refer to counters by string names for simplicity and for ease of adding metrics, but we performed some [careful design work](https://github.com/sorbet/sorbet/blob/2a16bc6b644e6fc8a309f0b0f31fd3592a02bf1a/common/Counters.h#L12-L30) to avoid actually looking at the contents of these strings inside the bulk of metric operations.

## Simple type inference

When designing Sorbet, we were implementing and iterating on the type system, the type inference algorithms, and the implementation all in concert. We decided fairly early on to build a relatively simple type system and type inference algorithm. We made this decision primarily because we felt that this direction would result in a more-understandable tool, which would help our users reason about and have confidence in Sorbet and its behavior on their code. However, this choice also helped to simplify implementation and enhance performance. I will highlight two choices we made here that had substantial performance impact.

### Local-only inference
Sorbet performs inference within method bodies, but any symbols that are stored into the global symbol table (e.g. methods and constants) must have explicitly-specified types. We opted for this design largely because we wanted to encourage explicitness in code, and because we thought it would be more predictable and enable better error messages. However, it also had desirable performance properties, by limiting dependencies between typechecking different parts of a program.

Among these properties, this meant that once the global symbol table is fully constructed, local inference only mutates local state, and can be performed arbitrarily in parallel over read-only `GlobalState` instances. This fact allowed us to parallelize the actual inference half of the typechecker (which takes almost half of our time, and more in incremental runs) almost trivially.

### Forward-only inference
Sorbet performs typechecking in a single pass over each function’s [control-flow graph.](https://en.wikipedia.org/wiki/Control-flow_graph) The type of each expression is inferred as we process it, and we (with a few exceptions) avoid instantiating type variables or performing unification. We chose this design in large part because we wanted the tool’s behavior to be simple and predictable for our users (and to produce comprehensible error messages), but it had the desirable effect of a fast one-pass implementation with predictable performance.

## Continuous attention to performance

Early on in development, we talked about setting up a dedicated performance-monitoring setup that would continuously benchmark every revision and allow early detection of regression (ala https://speed.pypy.org/ or https://perf.rust-lang.org/ or https://arewefastyet.com/). We unfortunately never made time to prioritize it, and I’m confident we lost performance to many unnoticed regressions because of it.

However, that didn’t mean that we didn’t continually spend attention and effort on performance from the start of the project. From very early on, our CI ran Sorbet — with debug self-checks and with [the LLVM sanitizers](https://github.com/google/sanitizers/wiki/AddressSanitizer) enabled — over Stripe’s entire multi-million line repository. The combination of self-checks and sanitizers resulted in a 20x slowdown or more compared to a production build, which perversely had the positive effect of causing us to be much more sensitive to performance regressions!

Because we had to wait on CI in order to get our own work done, we were very sensitive to degradations in in CI performance, and the 20x slowdown meant we tended to notice even small regressions! Several of the performance optimizations mentioned here were initially implemented, not to make release builds faster, but in order to make our CI speed tolerable for our own purposes.[^0]

In addition, both Dmitry and I had substantial experience with and expertise and interest in performance engineering, and so we periodically ran profilers and other performance tools (and tracked our metrics) to maintain a continual sense of our performance, and what low-hanging performance fruit might be fixed.

[^0]: I’m struck here by an analogy to the Linux kernel. Kernel developers often use “time to build the kernel” as a go-to benchmark when proposing kernel optimizations, because it’s a performance-intensive benchmark that every kernel developer, by construction, has readily at hand. Not coincidentally, the Linux kernel is **drastically** faster than any other production kernel I know of for certain operations that kernel builds exercise heavily, such as filesystem lookups. I think there’s a general lesson here that converting some desirable system property into a direct personal pain point for developers of the system creates very powerful feedback loops to ensure that property.


## Lifting invariant checks

Sorbet’s internal data structures maintain a large number of sometimes-subtle invariants. When writing code against these data structures, we face two conflicting goals:

- We want to proactively double-check that these invariants remain true, in order to find bugs earlier and increase our confidence that these invariants are being maintained.
- We want to rely on the truth of these invariants *without checking them*, in order to write faster code that doesn’t need to repeatedly perform redundant checks.

In order to balance between these goals, we built a custom `assert`-like mechanism in the form of the `ENFORCE` macro. We [sprinkled the codebase](https://github.com/sorbet/sorbet/search?q=ENFORCE) with calls to `ENFORCE` to validate our assumptions and invariants, and defined [`sanityCheck`](https://github.com/sorbet/sorbet/search?q=sanityCheck) [methods](https://github.com/sorbet/sorbet/search?q=sanityCheck) on most core data structures, which further checked all intended invariant on those structures, and called them liberally. We even built [a small abstraction](https://github.com/sorbet/sorbet/blob/7f65113364da104b96c379fa633023b8f0e75014/core/DebugOnlyCheck.h) to allow types to carry additional metadata in debug builds; for instance, `NameRef`s in debug builds can check that they are only used with the correct `GlobalState`.

We then configured our build system to allow us to build with or without `ENFORCE` and `sanityCheck` methods enabled. Production builds compile these checks out entirely, but development and test builds run with all invariant checks enabled. We continuously test these assertion-checking builds in a number of ways. These tests include fuzz testing, as well as running a “shadow build” in Stripe’s CI that runs over every pull request to the monorepo, but reports failures only to the Sorbet developers, instead of to the PR author. This setup gives us confidence that our invariants are being properly enforced and are not bitrotting or silently failing, but also allows our users to experience the fastest possible Sorbet when they are waiting for feedback on their code.

We additionally ship a second version of Sorbet to our users which builds with full compiler optimization, but also enables debug symbols and assertions. In the rare event that a user experiences a Sorbet crash, we provide tooling to re-run the typechecker with this build, which typically garners us more information about what went wrong.

I want to close this section with a note that this approach is **potentially very dangerous**. In C++, a developer’s mistaken assumption can often lead to highly exploitable bugs, and not just crashes or harmless misbehavior. Sorbet at Stripe is essentially always run over source code that is about to be executed anyways, and so we deliberately decided that protecting Sorbet from malicious source code was not a high priority, and that performance was. If we were implementing an internet-facing network service, or any other tool with more attack surface, I would likely consider it an unacceptable risk to disable assertions in production.


## Profile-driven performance work

We were never able to implement actual compiler [PGO](https://en.wikipedia.org/wiki/Profile-guided_optimization), which I suspect would be worth at least a few percentage points.

However, we did implement our own internal [metrics framework](https://github.com/sorbet/sorbet/blob/master/common/Counters.h) (alluded to above), which we used to collect statistics about the behavior of our tool when running over Stripe’s real code. We then used this information by hand to tune various parameters of the code, such as the [initial size of global state data structures](https://github.com/sorbet/sorbet/blob/2a16bc6b644e6fc8a309f0b0f31fd3592a02bf1a/core/GlobalState.cc#L58-L61), the size of inline vectors, and the [order of cases in pattern-matching](https://github.com/sorbet/sorbet/blob/2a16bc6b644e6fc8a309f0b0f31fd3592a02bf1a/ast/desugar/Desugar.cc#L421-L424).  We also continually tracked time spent in various phases of the typechecker in CI and over Stripe’s code base, helping us better notice and characterize regressions.

## Fast serialization format

Sorbet by default ships with a large set of initial definitions, covering the built-in classes and methods in the Ruby standard library. These definitions are maintained [in the Sorbet source tree](https://github.com/sorbet/sorbet/tree/master/rbi) as `.rbi` files.

Instead of parsing+analyzing these files on startup, Sorbet’s build system first builds Sorbet, runs it over these definitions to produce a serialized binary form of the symbols. We then re-build Sorbet with that binary output linked into the binary, so that at startup, Sorbet does not need to open any additional files or parse any code, but can directly deserialize that blob into internal data structures.

To implement the serialization, we considered various open-source serialization formats like protocol buffers, but ended up with a [hand-rolled binary format](https://github.com/sorbet/sorbet/blob/master/core/serialize/serialize.cc) which substantially outperformed any other format we tested, due in large part to requiring minimal branching. We were able to get this speedup in part because we did not have to worry about any backwards- or forwards-compatibility, since the serialized format only needed to work on the exact same version of sorbet as it was built with.

We also [benchmarked multiple compression algorithms](https://github.com/sorbet/sorbet/blob/master/docs/compressors.md) to decide what compression format to use, optimizing primarily for decompression speed at startup.

As a result of these optimizations, `sorbet -e 1` — measuring the fastest possible end-to-end typechecking, including decompressing and loading the entire standard library — takes less than 30ms all-in.

## Miscellaneous toolchain optimization

We took advantage of a number of miscellaneous optimizations in our build system and environment, including:


- Production builds use Clang’s [link-time-optimization](https://llvm.org/docs/LinkTimeOptimization.html)
- We link against [jemalloc](http://jemalloc.net/) as our `malloc` implementation, over glibc’s built-in allocator.

We unfortunately were not as rigorous as we could have been about recording benchmark results for posterity, but we did benchmark these changes and found small but real improvements as a result of each.

# Conclusions

A lot of factors went into making Sorbet as fast as it is. Next time, I hope to write up a few reflections on the general lessons that guided this experience, and that I take away from it.
