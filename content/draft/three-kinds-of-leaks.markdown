---
title: "Three kinds of memory leaks"
slug: three-kinds-of-leaks
date: 2018-04-29T08:30:00-07:00
---

So, you've got a program that's using more and more over time as it
runs. Probably you can immediately identify this as a likely symptom
of a memory leak.

But when we say "memory leak", what do we actually mean? In my
experience, apparent memory leaks divide into three broad categories,
each with somewhat different behavior, and requiring distinct tools
and approaches to debug. This post aims to describe these classes, and
provide tools and techniques for figuring out both which class you're
dealing with, and how to find the leak.

# Type (1): Unreachable allocations

This is the classic C/C++ memory leak. Someone allocated memory with
`new` or `malloc`, and never called `free` or `delete` to release the
memory after they were done with it.

```c
void leak_memory() {
  char *leaked = malloc(4096);
  use_a_buffer(leaked);
  /* Whoops, forgot to call free() */
}
```

## How to tell if you have this category

- If you're writing in C or C++, especially C++ without ubiquitous use
  of smart pointers to manage memory lifetimes, this is your first
  guess.

- If you're in a garbage-collected runtime, it's possible that a
  native-code extension
  [has a leak of this type](https://blog.nelhage.com/2013/03/tracking-an-eventmachine-leak/),
  but you should rule out types (2) and (3) first.

## How to find the leak

- Use [ASAN][asan]. Use ASAN. Use ASAN.
- Use another leak detector. I've used
  [Valgrind](http://valgrind.org/) or
  [tcmalloc's heap tools](http://goog-perftools.sourceforge.net/doc/heap_profiler.html),
  and there are other tools in other environments.
- Some allocators can dump a heap profile containing all un-free'd
  allocations. If you're leaking, after enough time, nearly all active
  allocations will come from the leak, so it should be easy to spot.
- When all else fails,
  [take a core dump and stare at it really hard][coredump]. This
  should never be your first tool.

[asan]: https://github.com/google/sanitizers/wiki/AddressSanitizer
[coredump]: https://blog.nelhage.com/2013/03/tracking-an-eventmachine-leak/#searching-for-c-object-leaks

# Type (2): Unexpectedly long-lived allocations

These leaks aren't "leaks" in the classical sense, because the memory
is still referred to from somewhere, and may even eventually be freed
(if the program makes it that far without running out of memory).

There are a lot of specific causes in this category. Some common
patterns look like:

- Unintentionally accumulating state onto a global structure; e.g. an
  HTTP server that pushes every `Request` object it receives onto a
  global list.
- Caches without an appropriate expiry policy. For instance, an ORM
  cache that caches every object ever loaded, that is active during a
  migration job that loads every record in a table.
- Capturing too much state inside a closure. This is
  [particularly common][meteor-closure] in Javscript, but not unique
  to that environment.
- More broadly, inadvertently holding on to every element of an array
  or stream when you thought you were processing it in an online
  streaming manner.

[meteor-closure]: https://blog.meteor.com/an-interesting-kind-of-javascript-memory-leak-8b47d2e7f156

## How to tell if you have this category

- If you're writing in a garbage-collected runtime, this is probably
  your first guess.
- Compare the heap size reported by your GC's statistics to the memory
  size reported by the operating system. If this is your leak
  category, the numbers will be comparable, and, in particular, will
  tend to track each other over time.

## How to find the leak

- Use the heap profiling or dumping tools provided by your
  environment. I know of [guppy][guppy] in Python or
  [memory_profiler][memory_profiler] in Ruby, or I've just scripted
  [ObjectSpace][objspace] directly in Ruby.

[guppy]: https://pypi.org/project/guppy/
[memory_profiler]: https://github.com/SamSaffron/memory_profiler
[objspace]: http://ruby-doc.org/stdlib-2.5.0/libdoc/objspace/rdoc/ObjectSpace.html

# Type (3): Free but unused or unusable memory

This is the trickiest category to characterize, but it also a very
important one to understand and be aware of.

This class arises in the gap between memory that is regarded as "free"
by the allocator inside a VM or runtime, and the memory that is
regarded as "free" by the operating
system. [Heap fragmentation][fragmentation] is the most common cause,
but not the only one; Some allocators will just flat-out never return
memory the host operating system once it's ever been allocated.

We can see a demo of this concept with the following short Python
program:

```python
import sys
from guppy import hpy
hp = hpy()

def rss():
    return 4096 * int(open('/proc/self/stat').read().split(' ')[23])

def gcsize():
    return hp.heap().size

rss0, gc0 = (rss(), gcsize())

buf = [bytearray(1024) for i in range(200*1024)]
print("start rss={}   gcsize={}".format(rss()-rss0, gcsize()-gc0))
buf = buf[::2]
print("end   rss={}   gcsize={}".format(rss()-rss0, gcsize()-gc0))
```

We allocate 200,000 1kb buffers, and then keep every other one. At
each point, we print memory usage, as seen by the operating system
("RSS"[^rss]), and as seen by Python's own GC.

On my laptop, output looks something like:

```
start rss=232222720   gcsize=11667592
end   rss=232222720   gcsize=5769520
```

We can see that Python has in fact freed half the buffers, as
evidenced by `gcsize` dropping nearly to half its peak, but that it
has not managed to return a single byte to the operating system; That
remaining memory is available for use by this Python process, but not
by any other process on this machine.

Such free-but-unused segments may or may not pose a problem; If a
Python program does this, and then goes on to make a bunch more 1kB
allocations, the space will get reused, and we'll be fine.

But if we did this during setup, and made minimal further allocations,
or if future allocations were all sized at 1.5kB and couldn't fit into
those 1kB buffers left behind, all that memory may remain unusable
forever.

On environment in which this problem class is particularly pronounced
is multiprocess server environments for languages like Ruby or
Python. Suppose you have a setup where:

- Each server runs N single-threaded workers to handle requests
  concurrently. Let's say N=10 for concreteness.
- In normal usage, each worker uses a fairly steady amount of memory;
  Let's say 500MB for concreteness.
- At some low rate, requests arrive that need much more memory than
  the median request. For concreteness, suppose that once a minute, we
  receive a request that requires an additional 1GB of memory during
  its execution, which is freed at the end of the request.

Once a minute the "whale" request will arrive, and get assigned to one
of the 10 workers, probably at ~random. Ideally, that worker will
allocate 1GB of RAM during the request, and then release that memory
back to the OS afterwards, so it's available for future usage. The
server as a whole will need `10 * 500MB + 1GB = 6GB` of RAM to handle
this request pattern indefinitely.

However, let's imagine that, due to heap fragmentation or otherwise,
the VM is unable to ever release memory to the operating system. That
is, the memory it needs from the OS is equal the largest amount of
memory it has *ever* needed at a time. Now, once a particular process
serves the high-memory request, that process's memory footprint is
bloated by 1GB -- forever.

At server launch, you'll see memory usage of `10 * 500MB = 5GB`. As
soon as the first large request hits, one worker will bloat by 1GB,
_and then not drop back down_. Overall memory usage will jump to
`6GB`. As each future request comes in, sometimes it will land on a
process that has served a "whale" request before, and memory usage
will be unchanged, but sometimes it will also land on a new worker,
and total memory usage will grow by another 1GB, until the point where
each worker has served that request at least once, and you are using a
total of `10 * (500MB + 1GB) = 15GB` of RAM, far more than our ideal
6GB! Furthermore, looking at fleet usage over time, you'll see a slow
rise from `5GB` to `15GB`, which will look very much like a "true"
leak.

[fragmentation]: https://stackoverflow.com/questions/3770457/what-is-memory-fragmentation

## How to tell if you have this category

- Compare the heap size reported by your allocator with the RSS[^rss]
  reported by the operating system. If type (3) is your problem, these
  numbers will tend to diverge over time.
  - I like to make my app servers report both numbers into my
    time-series infrastructure periodically, for easy graphing.
  - On Linux, get the OS view using field 24 of `/proc/self/stat`, and
    the allocator view from a language/VM-specific API.

## How to find the leak

This category, as mentioned, is a bit trickier, since it often results
from all components working "as designed". That said, there are a
number of useful tricks for mitigating or reducing the impact of this
kind of "virtual leak":

- Restart your processes more often. If the problem grows slowly,
  restarting all your app processes every 15m or once an hour might
  make it a non-issue. Nearly every large Ruby or Python shop I talk
  to ends up doing this.
- Even more aggressively, you can teach workers to self-restart once
  their memory footprint crosses a threshold, or a certain amount of
  growth. Be sure to put some thought into ensuring your entire fleet
  can't start spontaneously restarting in sync.
- Use a different allocator. Both [tcmalloc][tcmalloc] and
  [jemalloc][jemalloc] tend to have much better fragmentation
  properties over time than the default allocator, and are very easy
  to experiment with using an `LD_PRELOAD` drop-in.
- Identify if you have single requests that take far more memory than
  others. At [Stripe][stripe], our API servers measure RSS[^rss]
  before and after servicing every API request, and log the
  delta. It's then an easy query in our log aggregation systems to
  identify if specific endpoints, users, or other patterns are
  responsible for spiking memory.
- Tune your GC/allocator. Many GCs or allocators contain tunables that
  can control how aggressively it tries to return memory to the OS,
  how much it optimizes for avoiding fragmentation, or other useful
  parameters. This is a tricky area; Make sure you know what you're
  measuring and optimizing before before you start, and find+consult
  with an expert in your particular VM, if possible.

[tcmalloc]: http://goog-perftools.sourceforge.net/doc/tcmalloc.html
[jemalloc]: http://jemalloc.net/
[stripe]: https://stripe.com/

[^rss]: "Resident set size", the amount of RAM actually being consumed by a program, as distinct from virtual memory size or other statistics the OS might care about.
