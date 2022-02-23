---
title: "A Cursed Bug"
date: 2022-02-22T19:03:48-08:00
slug: a-cursed-bug
---
In my day job at [Anthropic](http://anthropic.com/), we run relatively large distributed systems to train large language models. One of the joys of using a lot of computing resources, especially on somewhat niche software stacks, is that you spend a lot of time running into the long-tail of bugs which only happen rarely or in very unusual configurations, which you happen to be the first to encounter.

These bugs are frustrating, but I also often enjoy them. They’re the opportunity for a good murder mystery, and they make for great stories if you ever track them down. This is the story of a bug we recently put to rest; a bug which, from our very first inklings of its existence, I repeatedly described as “**cursed**.” Now that we understand it, I want to share just how delightfully cursed it was.

Usually I like to tell these debugging stories in the “murder mystery” style, where I walk through the story of identifying and debugging the bug. In this case, I think the bug is more interesting than the debugging, so I want to focus in on the bug itself.


## Why was this bug “cursed”?

To begin with: Why do I call this bug “cursed”? I use that term as shorthand to refer to some combination of the following facts about this bug:

- It manifested as “this should never happen” failures extremely far-removed from the underlying bug — both in terms of happening long after the buggy code had executed, and in terms of the manifesting showing up many layers of abstraction higher in the stack than the cause.
- It manifested nondeterministically, and rarely – it manifested symptomatically only in around 0.1% of affected processes.
- It only manifested at all on our very largest distributed jobs, resulting in a very slow (and expensive!) debugging cycle.
- The bug (we eventually discovered) originated in the [RDMA](https://en.wikipedia.org/wiki/Remote_direct_memory_access) software stack we use. RDMA is always a bit spooky — the whole point being to allow (certain) memory to be transparently manipulated from a remote host without local intervention. And the RDMA software stack is complex and many-layered and has, in my experience, a checkered reputation for quality, at best.
# The bug

The bug manifested as follows: In our very largest distributed training jobs, sometimes — on less than 1% of nodes  — attempts to launch subprocesses would fail in very strange ways. These failures sometimes manifested as Python exceptions, something like so:


      …
      File "utils/command.py", line 89, in call
        proc = subprocess.run(
      File "lib/python3.8/subprocess.py", line 493, in run
        with Popen(*popenargs, **kwargs) as process:
      File "lib/python3.8/subprocess.py", line 858, in __init__
        self._execute_child(args, executable, preexec_fn, close_fds,
      File "lib/python3.8/subprocess.py", line 1704, in _execute_child
        raise child_exception_type(errno_num, err_msg, err_filename)
    OSError: [Errno 14] Bad address: 's5cmd'

And sometimes as segmentation faults coming from the child process in between `fork` and `exec`, with stack traces something like:


    #0  __malloc_fork_unlock_child () at arena.c:193
    #1  0x00007fe2a996fab5 in __libc_fork () at ../sysdeps/nptl/fork.c:188
    #2  0x00007fe2aa6e3941 in subprocess_fork_exec (self=<optimized out>, args=<optimized out>) at /usr/local/src/conda/python-3.8.10/Modules/_posixsubprocess.c:693
    #3  0x0000559a764598cf in cfunction_call_varargs (kwargs=<optimized out>, args=<optimized out>, func=0x7fe2a9704810) at /home/builder/ktietz/cos6/ci_cos6/python_1622837047642/work/Objects/call.c:758
    #4  PyCFunction_Call () at /home/builder/ktietz/cos6/ci_cos6/python_1622837047642/work/Objects/call.c:773
    #5  0x0000559a7641ada8 in _PyObject_MakeTpCall.localalias.6 () at /home/builder/ktietz/cos6/ci_cos6/python_1622837047642/work/Objects/call.c:159
    #6  0x0000559a764796f8 in _PyObject_Vectorcall (kwnames=0x0, nargsf=<optimized out>, args=0x559a773b7ff8, callable=<optimized out>) at /home/builder/ktietz/cos6/ci_cos6/python_1622837047642/work/Include/cpython/abstract.h:125


In either case, upon further inspection, we discovered that seemingly-random individual pages of memory were **just missing** from the heap in the `fork`ed child! The bad address was a perfectly-reasonable pointer, and the next and previous pages of memory existed, but one page was just … gone.


### Aside: `[Errno 14] Bad address`
The errno (“error number”) 14 in that first stack trace is [`EFAULT`](https://github.com/torvalds/linux/blob/f40ddce8/include/uapi/asm-generic/errno-base.h#L18). The Linux kernel returns `EFAULT` if you pass an invalid userspace address to a system call. In that way, it can sometimes indicate a similar failure mode as a segmentation fault, except that where a segmentation fault arises from trying to directly read or write to a bad pointer, `EFAULT` arises from trying to pass a similarly bad pointer to the OS kernel.

### Aside: `__malloc_fork_unlock_child`
What’s the deal with `__malloc_fork_unlock_child`? As mentioned above, `fork()` creates a new process with an exact copy of the memory image of the calling process. The `fork` system call predates multithreading in UNIX, and the introduction of multithreading posed some nasty challenges.

If a process is multi-threaded, all threads share the same memory. If one thread makes a copy of the entire memory image using `fork`, it has no way — in general — of making any guarantees about what the other threads are doing at the moment. They could be in the middle of mutating any data structures, or could be holding locks, which will then forever be held in the forked copy.

One of the most core bits of global shared state in most Linux processes is the memory allocator, generally referred to by the name of its main entry point — `malloc`. If we were to `fork` while another thread held a `malloc`-related lock, there would be no safe way to ever perform memory allocation again in the child. Thus, the C library, in its userspace wrapper around `fork`, walks the `malloc` heap metadata and (approximately speaking) takes out locks on every single bit of shared data. Then, after the `fork`, both the parent and the child release all of those locks, guaranteeing that the child sees a consistent snapshot of the allocator metadata.

Among other side effects, this means that `fork`, of necessity, looks at the metadata header for every region of memory (called “arenas” in glib) managed by `malloc`. When the child segfaulted in `__malloc_fork_unlock_child`, it was because the page of memory containing one of those headers had vanished in the child.

### Aside: `fork`/`exec`?
If you’re not familiar with how creating subprocesses works on UNIX, it’s an odd two-part dance under the hood. The only[^fork] way to create a new process on Linux is with the `fork` system call, which creates a (near-)identical **copy** of the current process, including its entire memory image (`fork` uses virtual memory tricks and copy-on-write to make this reasonably efficient). If you want to start an external process, you then arrange for the child to call `execve` (colloquially referred to as just `exec`), which then **replaces** the entire memory and state of the child process with the new binary loaded from disk.

This two-step results in a (brief) window where the child process is running code in between `fork` returning and `exec` nuking the entire state. This window is fraught in a number of ways, and it is during this window that we were seeing these failures arise.

[^fork]: This sentence isn’t literally true — there are some variants on `fork`, such as `vfork` and `clone` — but it’s true enough in the sense that all the variants are spiritually very similar.


## Why were the pages gone?

These pages were missing because of an interaction between [aws-ofi-nccl](https://github.com/aws/aws-ofi-nccl), an AWS plugin which lets NVIDIA’s [NCCL](https://developer.nvidia.com/nccl) GPU communication library use Amazon’s [EFA](https://aws.amazon.com/hpc/efa/) for high-performance communication between EC2 instances, and the [`libiverbs`](https://github.com/linux-rdma/rdma-core/tree/master/libibverbs) userspace library for RDMA on Linux.

In particular, `libiverbs` marks any memory segments which have been registered for use with RDMA using `madvise(…, MADV_DONTFORK)`, to ask the kernel not to copy them into the child on `fork`.

 `aws-ofi-nccl` [registered a 4-byte region](https://github.com/aws/aws-ofi-nccl/pull/77https://github.com/nzmsv/aws-ofi-nccl/blob/0ac4a5175cd038268927dc819e753b00b449e908/src/nccl_ofi_net.c#L1752-L1758) in the middle of a `malloc` ed buffer as an RDMA target. Instead of rejecting the request (since there is no way to manipulate `madvise` flags at a granularity of smaller than a page), `libiverbs` would “helpfully” [round up](https://github.com/linux-rdma/rdma-core/blob/3ff453edb8ce0e0e63e0d7de602577bd0a31aaee/libibverbs/memory.c#L638-L639) to the nearest full page, which resulted in an effectively random page vanishing from the child’s heap for each NCCL communicator! Most of the time, these pages would go unused in between `fork` and `exec` and so this behavior would go unnoticed; but sometimes they would contain part of glibc’s `malloc` metadata, or would contain string data used by Python between the `fork` and `exec`, manifesting in the two failure patterns described above!

In the process of debugging the issue, we were able to catch a “live” process in the broken state (where `fork`ing would fail). By looking at the address of the missing malloc arena and inspecting the parent’s address space in `/proc/pid/smaps`, we could observe the problematic state:


    7f5e20000000-7f5e20001000 rw-p 00000000 00:00 0
    Size:                  4 kB
    KernelPageSize:        4 kB
    MMUPageSize:           4 kB
    Rss:                   4 kB
    Pss:                   4 kB
    Shared_Clean:          0 kB
    Shared_Dirty:          0 kB
    Private_Clean:         0 kB
    Private_Dirty:         4 kB
    Referenced:            4 kB
    Anonymous:             4 kB
    LazyFree:              0 kB
    AnonHugePages:         0 kB
    ShmemPmdMapped:        0 kB
    FilePmdMapped:         0 kB
    Shared_Hugetlb:        0 kB
    Private_Hugetlb:       0 kB
    Swap:                  0 kB
    SwapPss:               0 kB
    Locked:                0 kB
    THPeligible:            0
    ProtectionKey:         0
    VmFlags: rd wr mr mw me dc nr sd


Looking at this mapping, we notice two salient facts:

- This is a 1-page (4k) range of memory. A page is the smallest region of memory that can be managed independently by the hardware virtual memory system. Most of the surrounding regions were much larger than a single page, because memory tends to get managed in much larger chunks on today’s large systems. The `madvise` call forced the kernel to split off a one-page region to track the effects of the `madvise` call.
- The `VmFlags` field includes the `dc` flag. Searching for `smaps` in [`man 5 proc`](https://man7.org/linux/man-pages/man5/proc.5.html), we find that `dc` means “do not copy area on fork” — the internal flag set by `madvise(…, MADV_DONTFORK)`.

### Aside: Why does `libiverbs` use `MADV_DONTFORK`?

`libiverbs` exists to interact with RDMA hardware.  RDMA hardware, like most forms of DMA, accesses memory in terms of **physical** memory addresses, unlike the **virtual** addresses code running on the CPU uses. This means, in order for things to go well, that any page of memory involved in RDMA needs to be pinned to a specific location in physical memory as long as it might be written to by a remote RDMA peer.

RDMA drivers “pin” memory pages, which prevents them from being relocated or swapped by the kernel. However, `fork` calls are now problematic. Following a `fork`, the kernel usually marks both processes’ entire address space as read-only; on an attempt to write to memory, it copies the underlying page so that neither process sees any modifications by the other.

Specifically, the first process to attempt to write to a copy-on-write (“CoW”) page will make a copy, leaving the original page (still visible to one or more other processes) intact.

This (potentially) interacts poorly with RDMA; If a process using RDMA `fork`s, any future attempts to write to pages used for RDMA would result in the parent copying the page, which would mean it would no longer see any remote writes to that page by RDMA peers.

Historically, to avoid this problem, the RDMA user-space would handle this case by explicitly requesting that any pages which are RDMA targets not get copied into the child at all. This ensures the parent process continues functioning properly, and (assuming the child then `execve`s to replace its address space) should normally be harmless in the child.


## The future: eager copying

While doing this writeup, I discovered that the previous section is no longer accurate for recent kernels! As of [this commit (released as part of Linux 5.12)](https://github.com/torvalds/linux/commit/4eae4efa2c299f85b7ebfbeeda56c19c5eba2768), Linux will **eagerly** copy pages which are marked as pinned on a `fork` call. This has the result that the parent process gets to hold on to the same physical page, but the child gets a copy and has its address space intact. `rdma-core`, as of version v35, has support for [checking whether or not a kernel is sufficiently new](https://github.com/linux-rdma/rdma-core/pull/975) to support this feature. Confusingly to me, this check takes the form of a new API call — [`ibv_is_fork_initialized`](https://www.mankier.com/3/ibv_is_fork_initialized) — and the existing call to request “fork support”, [`ibv_fork_init`](https://www.mankier.com/3/ibv_fork_init), was unmodified. Thus, on newer kernels, callers are able to detect that `MADV_DONTFORK` is unneeded and avoid enabling; but if someone explicitly enables, including by [setting the](https://github.com/linux-rdma/rdma-core/blob/ba3689ce1a5d01cd56219fa372e829fe83ac5f9e/libibverbs/init.c#L670-L673) [`RDMAV_FORK_SAFE`](https://github.com/linux-rdma/rdma-core/blob/ba3689ce1a5d01cd56219fa372e829fe83ac5f9e/libibverbs/init.c#L670-L673) [environment variable](https://github.com/linux-rdma/rdma-core/blob/ba3689ce1a5d01cd56219fa372e829fe83ac5f9e/libibverbs/init.c#L670-L673) (as we were), they will get the legacy (and buggy) `MADV_DONTFORK` behavior even if it is unneeded.

It turns out our kernels were too old to have this support, but even if they weren’t, the fact that our startup scripts set `RDMAV_FORK_SAFE` (presumably blindly copy-pasted on advice of some historical reference), it would not have shielded us from this bug.

# The fix
## The workaround

Once we realized the problem was related to `fork`, one natural workaround was to attempt spawning new processes via [`posix_spawn`](https://man7.org/linux/man-pages/man3/posix_spawn.3.html), instead of `fork`. `posix_spawn` is a (relatively) new API that attempts to encapsulate the creation of a subprocess into a single API call, instead of the `fork`/[setup code]/`exec` combination traditionally used. Under the hood, `posix_spawn` is implemented in most `libc`s I know of using the [`vfork`](https://man7.org/linux/man-pages/man2/vfork.2.html) system call. `vfork` is a call similar to `fork`, but instead of actually copying memory, it pauses the parent until the child calls `execve` or exits. This creates the risk of memory corruption if the child attempts to write to essentially any memory whatsoever; but if the child is properly-implemented, `vfork`+`exec` can be substantially more performant that the traditional `fork`+`exec` combination.

We were launching subprocesses using Python’s [`subprocess`](https://docs.python.org/3/library/subprocess.html) module. As of Python 3.8 (which we were running), `subprocess` supports `posix_spawn`, so we were actually a little bit surprised to discover it wasn’t being used. It turns out that `subprocess` only hits the `posix_spawn` path if [a long list of conditions](https://github.com/python/cpython/blob/4844abdd700120120fc76c29d911bcb547700baf/Lib/subprocess.py#L1584-L1598) holds on process creation. Notably, among other conditions, it requires that `subprocess.Popen` be given an absolute path to the program to execute, and that the `close_fds` flag be set to false. Since `close_fds=True` by default, virtually every Python program in existence is skipping the `posix_spawn` path!

Once we realized this (by reading `subprocess.py` — we found no documentation of this gotcha), we were able to modify our subprocess creation code to meet those conditions, and the bug stopped manifesting! We were able to deploy this workaround several days before we actually identified the root bug, and resume training stably.


## The actual fix

Once we had identified the fix was related to `fork` and `MADV_DONTFORK`, our AWS support contacts were able to (impressively quickly!) identify the troublesome registration in `aws-ofi-nccl`, [and fairly quickly pushed a fix](https://github.com/aws/aws-ofi-nccl/pull/77) that resolved the problem.

I also [reported a bug](https://lore.kernel.org/linux-rdma/CAPSG9dZ-dkWPcbXECQeZyvOHu7M+vfrX+jJDe+fxY6_iSnQyKw@mail.gmail.com/) to the rdma-core project asking that they error (or warn) on these registrations instead of blithely trashing the processes’ address space, but I received a (frankly) [kind of baffling response](https://lore.kernel.org/linux-rdma/20220107150655.GZ6467@ziepe.ca/). In light of my note above about eager copying in `fork`, I think they may be referring to the fact that `ibv_fork_init` is no longer needed on recent systems, but in light of the fact that that mode is still supported and is fairly easy to unintentionally enable, it still seems clear to me that there’s a bug worth fixing.

# Closing Thoughts

I don’t feel like I have any great takeaways here. This was a very satisfying bug to finally track down and fix; it was the kind of bug that can linger, rarely triggered, never diagnosed, for years, worked around or tolerated, but never debugged. We got lucky, in a way, that we were able to track it down; a completely unrelated update in a library we used caused us to start shelling out early on in every training process’ lifetime, which was sufficient to increase the rate of incidence of this bug high enough that it became unbearable; but that also meant it was high enough to be reproducible and debuggable. For months prior to that we’d just tolerated occasional incidences, working around with retry loops at various levels.

The experience also feels like a vindication of a sorts for my conviction that [computers can be understood](https://blog.nelhage.com/post/computers-can-be-understood/). When you start dealing with large distributed systems, you see a **lot** of intermittent weird bugs, and it’s basically inevitable that you’ll never track down or diagnose all of them. But it turns out that sometimes, with a bit of luck and persistence, even the hairiest ones can sometimes be tracked down.
