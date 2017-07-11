---
date: 2017-07-10T21:15:00Z
slug: transparent-hugepages
title: Disable Transparent Hugepages
---

# tl;dr

["Transparent Hugepages"][kerneldoc] is a Linux kernel feature
intended to improve performance by making more efficient use of your
processor's memory-mapping hardware. It is enabled
("`enabled=always`") by default in most Linux distributions.

Transparent Hugepages gives some applications a
[small performance improvement][benchmarks] (~ 10% at best, 0-3% more
typically) at best, but can cause [significant][mongodb]
[performance][oracle] [problems][splunk], or even apparent
[memory][d-o] [leaks][golang] at worst.

To avoid these problems, you should set `enabled=madvise` on your
servers by running

    echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/enabled

and setting `transparent_hugepage=madvise` on your kernel command line
(e.g. in `/etc/default/grub`).

This change will allow applications that are optimized for transparent
hugepages to obtain the performance benefits, and prevent the
associated problems otherwise.

Read on for more details.

[rhel-guide]: https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Performance_Tuning_Guide/s-memory-transhuge.html
[mongodb]: https://docs.mongodb.com/manual/tutorial/transparent-huge-pages/
[kerneldoc]: https://www.kernel.org/doc/Documentation/vm/transhuge.txt
[oracle]: https://blogs.oracle.com/linux/entry/performance_issues_with_transparent_huge
[d-o]: https://www.digitalocean.com/company/blog/transparent-huge-pages-and-alternative-memory-allocators/
[lwn]: https://lwn.net/Articles/423584/
[benchmarks]: https://lwn.net/Articles/423590/
[marklogic]: https://help.marklogic.com/knowledgebase/article/View/16/0/linux-huge-pages-and-transparent-huge-pages
[golang]: https://github.com/golang/go/issues/8832
[splunk]: https://docs.splunk.com/Documentation/Splunk/6.5.2/ReleaseNotes/SplunkandTHP
[redis]: https://redis.io/topics/latency


# What are transparent hugepages?
## What are hugepages?

For decades now, processors and operating systems have collaborated to
use [virtual memory][virtmem] to provide a layer of indirection
between memory as seen by applications (the "virtual address space"),
and the underlying physical memory of the hardware. This indirection
protects applications from each other, and enables a whole host of
powerful features.

x86 processors, like many others, implement virtual memory by a
[page table][pagetable] scheme that stores the mapping as a large
table in memory [^tree]. Traditionally, on x86 processors, each table
entry controls the mapping of a single 4KB "page" of memory.

[^tree]: It's actually a tree structure of varying depth, but it's equivalent to a large sparse table.

While these page tables are themselves stored in memory, the processor
caches a subset of the page table entries in a cache on the processor
itself, called the [TLB][tlb]. A look through the output of `cpuid(1)`
on my laptop reveals that its lowest-level TLB contains 64 entries for
4KB data pages. 64\*4KB is only a quarter-megabyte, much smaller than
the working memory of most useful applications in 2017. This size mismatch
means that applications accessing large amounts of memory may
regularly "miss" the TLB, requiring expensive fetches from main memory
*just to locate their data in memory*

Primarily in an effort to improve TLB efficiency, therefore, x86 (and other)
processors have long supported creating "huge pages", in which a
single page-table entry maps a larger segment of address space to
physical memory. Depending on how the OS configures it, most recent
chips can map 2MB, 4MB, or even 1GB pages. Using large pages means
more data fits into the TLB, which means better performance for certain workloads.

## What are transparent hugepages?

The existence of multiple flavors of page table management means that
the operating system needs to determine how to map address space to
physical memory. Because application memory management interfaces
(like `mmap(2)`) have historically been based on the smallest 4KB
pages, the kernel must always support mapping data in 4KB
increments. The simplest and most flexible (in terms of supported
memory layouts) solution, therefore, is to just always use 4KB pages,
and not benefit from hugepages for application memory mappings. And
for a long time this has been the strategy adopted by the
general-purpose memory management code in the kernel.

For applications (such as certain databases or scientific computing
programs) that are known to require large amounts of memory and be
performance-sensitive, the kernel introduced the
[hugetlbfs][hugetlbfs] feature, which allows administrators to
explicitly configure certain applications to use hugepages.

[Transparent Hugepages][thp] ("THP" for short), as the name suggests,
intended to bring hugepage support automatically to applications,
without requiring custom configuration. Transparent hugepage support
works by scanning memory mappings in the background (via the
"`khugepaged`" kernel thread), attempting to find or create (by moving
memory around) contiguous 2MB ranges of 4KB mappings, that can be
replaced with a single hugepage.

# What goes wrong?

When transparent hugepage support works well, it can garner up to
about a 10% performance improvement on certain benchmarks. However, it
also comes with at least two serious failure modes:

## Memory Leaks

THP attempts to create 2MB mappings. However, it's overly greedy in
doing so, and too unwilling to break them back up if necessary. If an
application maps a large range but only touches the first few bytes,
it would traditionally consume only a single 4KB page of physical
memory. With THP enabled, `khugepaged` can come and extend that 4KB
page into a 2MB page, effectively bloating memory usage by 512x (An
example reproducer on
[this bug report](https://bugzilla.kernel.org/show_bug.cgi?id=93111)
actually demonstrates the 512x worst case!).

This behavior isn't hypothetical; Go's GC had to include an
[explicit workaround][golang] for it, and Digital Ocean
[documented][d-o] their woes with Redis, THP, and the `jemalloc`
allocator.

## Pauses and CPU usage

In steady-state usage by applications with fairly static memory
allocation, the work done by `khugepaged` is minimal. However, on
certain workloads that involve aggressive memory remapping or
short-lived processes, `khugepaged` can end up doing huge amounts of
work to merge and/or split memory regions, which ends up being
entirely short-lived and useless. This manifests as excessive CPU
usage, and can also manifest as long pauses, as the kernel is forced
to break up a 2MB page back into 4KB pages before performing what
would otherwise have been a fast operation on a single page.

Several applications have seen 30% performance degradations or worse
with THP enabled, for these reasons.

# So what now?

The THP authors were aware of the potential downsides of transparent
hugepages (although, with hindsight, we might argue that they
underestimated them). They therefore opted to make the behavior
configurable via the `/sys/kernel/mm/transparent_hugepage/enabled`
sysfs file.

Even more importantly, they implemented an "opt-in" mode for
transparent hugepages. With the `madvise` setting in
`/sys/kernel/mm/transparent_hugepage/enabled`, `khugepaged` will leave
memory alone by default, but applications can use the `madvise` system
call to specifically request THP behavior for selected ranges of
memory.

Since -- for the most part -- only a few specialized applications
receive substantial benefits from hugepage support, this option gives
us the best of both worlds. The applications of those applications can
opt-in using `madvise`, and the rest of us can remain free from the
undesirable side-effects of transparent hugepages.

Thus, I recommend that all users set their transparent hugepage
setting to `madvise`, as described in the [tl;dr](#tl-dr) section at
the top. I also hope to persuade the major distributions to disable
them by default, to save numerous more administrators and operators
and developers from having to rediscover these failure modes for
themselves.


[virtmem]: https://en.wikipedia.org/wiki/Virtual_memory
[pagetable]: https://en.wikipedia.org/wiki/Page_table
[tlb]: https://en.wikipedia.org/wiki/Translation_lookaside_buffer
[hugetlbfs]: https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt
[thp]: https://www.kernel.org/doc/Documentation/vm/transhuge.txt
