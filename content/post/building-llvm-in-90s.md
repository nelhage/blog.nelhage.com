---
title: "Building LLVM in 90 seconds using Amazon Lambda"
slug: building-llvm-in-90s
date: 2021-05-20T19:00:28-07:00
---
Last week, Frederic Cambus wrote about building LLVM quickly on some very large machines, culminating in [a 2m37s build on a 160-core ARM machine](https://www.cambus.net/speedbuilding-llvm-clang-in-2-minutes-on-arm/).

I don't have a giant ARM behemoth, but I have been working on a tool I call [Llama][llama], which lets you offload computational work -- including C and C++ builds -- onto Amazon Lambda. I decided to see how good it could do at a similar build.

I'm running Llama on my AMD Ryzen 9 3900 (12 cores, 24 SMT threads). I'm building [the same commit Cambus tested](https://github.com/llvm/llvm-project/commit/cf4610d27bbb5c3a744374440e2fdf77caa12040), and building using Clang/LLVM 13 from the LLVM apt repository:

> `Ubuntu clang version 13.0.0-++20210418052640+d480f968ad8b-1~exp1~20210418153358.383`

[llama]: https://github.com/nelhage/llama

Let’s start with the headline result: A full clang+LLVM build takes about 80s, a full minute faster than the largest single ARM node money can buy:


```shell
$ LLAMACC_LOCAL=1 CC=llamacc CXX=llamac++ \
  cmake -GNinja \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_PROJECTS=clang \
  -DLLVM_USE_LINKER=lld \
  -DLLVM_TARGETS_TO_BUILD=X86 \
  -DLLVM_PARALLEL_LINK_JOBS=8 \
  -DLLVM_BUILD_TOOLS=OFF \
  -DLLVM_BUILD_UTILS=OFF  \
  -DCMAKE_CXX_FLAGS_RELEASE="-O0" \
  -DCLANG_ENABLE_STATIC_ANALYZER=OFF  \
  -DCLANG_ENABLE_ARCMT=OFF \
   ../../llvm/
$ time ninja -j400
real    1m21.111s
user    8m29.731s
sys     1m19.925s
```

I’m doing a very similar build to Cambus’s final build — a stripped-down release configuration with optimization disabled. The notable differences are:

- I’m setting `CC=llamacc` and `CXX=llamac++` to build using Llama in Lambda. I also run `cmake` using `LLAMACC_LOCAL=1`; That setting makes llamacc just run builds locally during the `cmake` itself. `cmake` does some amount of `autoconf`-style feature detection, which is very serial, so we just want to run it locally.
- I’m setting `-DLLVM_PARALLEL_LINK_JOBS=`8 to limit the number of concurrent linker jobs; The linker is very memory-hungry, and without this limit high-concurrency builds cause my machine to run out of memory.

How much money did that build cost me? Llama includes a tool to estimate our spend:

```shell
 $ llama daemon -stats
 …
AWS Usage:
  Lambda runtime     ms     8611121
  Lambda runtime     MB-ms  15233073049  $0.25
  Lambda requests           2418         $0.00
  S3 Write requests         9841         $0.05
  S3 Read requests          185397       $0.07
  S3 Xfer in         MB     1825         $0.00
  S3 Xfer out        MB     241          $0.02
  Total              $                   $0.40
```

It cost us around forty cents, with close to zero ongoing spend if you're not currently running a build[^s3]. It also scales with how much compute you need -- we could edit a few files and do a partial rebuild and pay for only what we need.

[^s3]: It’s not quite zero because we use S3 for moving objects between the client and the cloud, and that will cost us a few cents until objects expire (after one month, in Llama’s default configuration).

## How Llama works

At a high level, Llama follows in the family of [distcc](https://github.com/distcc/distcc) and its descendants: We provide a drop-in replacement for `g++` which does the actual compute remotely, and then allow the user to run `make` or `ninja` with a `-j` argument much larger than the actual number of local cores.

In the case of llamacc, inspired by [Stanford's `gg`](https://github.com/StanfordSNR/gg)[^gg], the actual compilation happens inside Amazon Lambda. When invoked to build a source file, `llamacc` or `llamac++`:

- First runs the preprocessor locally to discover all header files required by that source file
- Uploads each source file to a content-addressed store in S3
- Invokes an Amazon Lambda function with a custom runtime, which
    - Downloads the required headers from S3
    - Invokes the compiler inside the Lambda function
    - Uploads the resulting object file to the same S3 store
- Downloads the resulting object from S3

Caching is used on both the client and in the Lambda function to avoid repeatedly uploading or downloading the same files, exploiting the fact that Amazon will reuse the same function instance to serve multiple requests.

[^gg]: I've [written previously](https://buttondown.email/nelhage/archive/papers-i-love-gg) about `gg` and how much I enjoy that paper.

## Potential improvements

This result is impressive, but I think there’s actually still some relatively low-hanging fruit for improvement.

### Removing unnecessary dependencies
If we use [ninjatracing](https://github.com/nico/ninjatracing) to visualize the build in Chrome’s trace viewer, we can see a number of bottlenecks where the entire build is waiting on one or two processes:

![](/images/posts/llvm-llama/profile.png)


I haven’t investigated to be sure, but I strongly suspect that these are related to cmake’s Ninja generator [emitting unnecessary dependencies](https://gitlab.kitware.com/cmake/cmake/-/issues/17097); if that bug were fixed, and LLVM’s build updated accordingly, I bet we could get much better utilization and an even-faster build!

### Optimizing long-pole files
We can also see from the above trace that a few files take disproportionately long to compile. For instance, `llvm/lib/Target/X86/X86ISelLowering.cpp` is extremely slow. That file turns out to contain 50,000(!) lines of C++ source, constituting 2MB (!!) of text. I suspect that splitting the largest ~5 files into smaller compilation units, so that they are no longer holding up the parallel build, might also be a substantial win.

### Llama improvements
Llama is still young and there’s a lot of work yet-to-be-done! For instance, I haven’t yet [implemented HTTP pipelining](https://buttondown.email/nelhage/archive/http-pipelining-s3-and-gg), which my experiments demonstrate to be a substantial win when downloading files from S3.


# Give Llama a try!

Do you have a large C or C++ codebase you wish were faster to compile? [Give Llama a try!](https://github.com/nelhage/llama#getting-started)

I’ve tried to make it fairly easy to get started. I’d love to hear from you if you give it a go.
