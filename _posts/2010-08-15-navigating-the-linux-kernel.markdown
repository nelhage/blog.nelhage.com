---
layout: post
status: publish
published: true
title: Navigating the Linux Kernel
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 291
wordpress_url: http://blog.nelhage.com/?p=291
date: 2010-08-15 21:52:58.000000000 +02:00
categories:
- linux
tags:
- linux
- kernel
- source-diving
---
In response to my query last time, `ezyang` [asked][question] for any
tips or tricks I have for finding my way around the Linux kernel. I'm
not sure I have much in the way of systematic advice for tracking down
the answers to questions about the Linux kernel, but thinking about
what I do when posed with a patch to Linux that I need understand, or
question I need to answer, I've come up with a collection of tips that
will hopefully be helpful to others looking to source-dive Linux for
whatever reason.

[question]: http://blog.nelhage.com/2010/08/suggestion-time-what-should-i-blog-about/#comment-2597

Know the layout
---------------

It sounds basic, but you probably shouldn't be doing any serious
source-diving into the Linux kernel without pausing to familiarize
yourself with the basic layout of the kernel sources. The most
interesting directories are:

- `fs/` -- This directory contains both the VFS implementation (the
  generic filesystem code and the top-level implementation of
  filesystem syscalls), and specific filesystems, in
  subdirectories. If you're looking for the implementation of a
  filesystem-related system call, it's probably in one of `fs/*.c`.

- `mm/` -- This contains the virtual memory and memory management
  subsystems. `mmap` lives here, as do all of the kernel's various
  memory allocators, including `kmalloc` and `vmalloc`.

- `kernel/` -- This contains the "core" kernel code. The scheduler
  lives here, as does the implementation of various primitives used
  throughout the kernel, like `printk` and various data
  structures. timer- and process- related system calls live here,
  including `fork` and `exit`, and most anything related to uids and
  pids.

- `net/` is the networking subsystem; much like `fs/` it contains both
  generic code and specific network protocol
  implementations. networking-related system calls are mostly in
  `net/socket.c`

- `arch/` -- Architecture-specific code lives here, in
  `arch/ARCHITECTURE/`. Per-architecture include files live in
  `arch/ARCHITECTURE/include/asm/`; Prior to 2.6.28 they were in
  `include/asm-ARCH/`. `arch/` directories tend to loosely parallel
  the top-level source directory, with `kernel/` and `mm/`
  subdirectories.

Know your git
-------------

I find `git` is one of the most invaluable tools at my disposal when
trying to understand the Linux source. There are large classes of
questions about the source that git makes it easy to answer that I
otherwise would have to resort to something much slower or more
cumbersome to figure out. Some things I've found particularly useful
include:

- `git grep` -- `git grep` works almost identically to `grep`, but
  instead of searching the files on disk, it searches the objects in
  git's object store. Because of the way this store is compressed and
  designed for locality, it's typically far faster at searching large
  trees than the equivalent recursive grep would be. In addition, it
  knows to ignore files that aren't in source control, such as object
  files.

- `git blame` -- This one should be familiar to anyone who's used
  subversion or most any other version control system. This will let
  you quickly find the commit that introduced a given line. This gives
  you several potential sources of information:

  - The commit message often includes helpful documentation on how a
    change was supposed to work or what the bug it was fixing was.
  - The diff is often a quick way to find other files that are related
    to the piece of code you're looking at, potentially giving you
    other places to look for more related code.

- `git log -S` -- while `git blame` can tell you when a specific line
  was introduced, `git log -S`, also known for some inscrutable reason
  as the "pickaxe", will let you know when a specific chunk of code
  was introduced. Here's how it works:

  Suppose I wanted to know when the `vmsplice` system call was
  introduced. A `git grep` will reveal the line in `fs/splice.c` that
  defines the system call:

        SYSCALL_DEFINE4(vmsplice, int, fd, const struct iovec __user *, iov,
                        unsigned long, nr_segs, unsigned int, flags)
    
  I could run `git blame`, but that points me at commit `836f92ad`,
  which was just one of the commits that introduced the
  `SYSCALL_DEFINEn` wrappers, which isn't what I'm looking for. I
  could continue `git blame`ing from there, but that's really not what
  I want.

  Instead, I can run:

        git log -Svmsplice fs/splice.c

  which yields two commits, the earliest of which is the one I want.

  So, how does this work? When you use the pickaxe, with `-Sstring`,
  git looks for commits that *add* or *remove* an instance of
  **string**. It doesn't look at the diff or anything -- it simply
  counts how often **string** appears before and after the commit, and
  includes commits where the numbers are different.

  So, `836f92ad`, which has the hunk:

        -asmlinkage long sys_vmsplice(int fd, const struct iovec __user *iov,
        -                            unsigned long nr_segs, unsigned int flags)
        +SYSCALL_DEFINE4(vmsplice, int, fd, const struct iovec __user *, iov,
        +               unsigned long, nr_segs, unsigned int, flags)

  doesn't change the count of `vmsplice` instances, and isn't flagged
  by the pickaxe. But the commit that introduced `sys_vmsplice` in the
  first place had to have, and so the pickaxe flags it.

Know your idioms
----------------

One of the advantages of the centrally-controlled model of Linux,
where almost all changes, at least to the core code, are code-reviewed
extensively on the [LKML][lkml], is that the code tends to have a very
high standard of stylistic and idiomatic consistency. So once you
learn some of the common idioms of kernel development, you can
recognize them everywhere, and infer information about the structure
of a piece of code without having to go read all of the details.

A corollary that I've found here is: Trust your instincts. If you
think you recognize a pattern in the code, or if there is some way in
which it seems like the code "ought to be working", you're usually
well served by assuming that your hunch is right and proceeding based
off of that, and coming back later to check your assumptions if
necessary, instead of stopping at every stop to verify your
guesses. Because the code is, in general, of very high quality and
consistency, once you start developing familiarity with it, your
guesses will be right far more often than not.

I won't attempt to list an exhaustive list of design patterns and
idioms in the Linux kernel, but here are some it's pretty essential to
be familiar with:

- The `_ops` struct -- Linux uses an OO-esque style ubiquitously in
  the kernel, where structs of function pointers (basically a
  poor-man's `vtable`, to you C++ programmers), are passed around and
  stored to indicate how to work with some object. These `struct`s are
  known as "ops" structures, and typically have types name
  `FOO_operations`, and live in variables named `SOMETHING_ops` --
  `struct super_operations`, `struct inode_operations`, `struct
  file_operations`, and so on.

- `struct list_head`, defined in `include/linux/types.h`, with
  operations in `include/linux/list.h` is used basically anywhere the
  kernel needs to store linked lists. To save on space and reduce
  fragmentation, the kernel uses a trick where `struct list_head`s are
  stored inside the structures that are the element of a list, and
  pointer arithmetic is used to compute the one from the
  other. Familiarize yourself with `list.h`, since it's a rare piece
  of code that won't use at least some of its functionality.

- `container_of` and related idioms. The trick I mentioned previously,
  of storing a `list_head` inside a structure and using pointer
  arithmetic, is generalized in many places, through the
  `container_of` macro.

  Let's consider the problem of implementing a filesystem, say,
  `ext2`. Linux's VFS layer has a generic `inode` structure, that
  store filesystem-independent information about inodes. `ext2`,
  however, has some additional information it needs to store on each
  in-memory inode. A standard userspace approach would be for `struct
  inode` to contain a `void *userdata` pointer, and `ext2` could
  allocate a `struct ext2_inode_info`, and point `userdata` at that.

  This means that creating an inode needs two allocations, however,
  which is inefficient and causes fragmentation in the memory
  allocator, which is unacceptable in the kernel.

  Instead, ext2 embeds the `struct inode` *inside* `struct
  ext_inode_info`:

        /*
         * second extended file system inode data in memory
         */
        struct ext2_inode_info {
                __le32  i_data[15];
                …
                struct inode    vfs_inode;
                …
        };

  (See `fs/ext2/ext2.h` for the full definition)

  Then, whenever ext2 gets a callback from the VFS with a `struct
  inode`, it can retrieve the `ext2_inode_info` using:

        static inline struct ext2_inode_info *EXT2_I(struct inode *inode)
        {
                return container_of(inode, struct ext2_inode_info, vfs_inode);
        }

  This uses the `container_of` macro, which in this case is used to
  find the object of type `ext2_inode_info` which contains the object
  `inode` in the member named `vfs_inode`. The implementation of this
  macro is somewhat hairy and relies on GCC extensions when available,
  but you should be able to see that in the end it will compile down
  to a simple subtraction -- about as efficient as you could hope for.

Know your references
--------------------

While sourcediving is the ultimate way to answer any question about
the kernel, and is lots of fun to boot, don't forget about the
possibility of documentation answering your question, or at least
pointing you in the right direction. Some places that are essential to
look include:

- [*Understanding the Linux Kernel*][utlk] -- This book is an
  incredibly detailed walkthrough of the inner implementation of
  virtually every feature and subsystem in the kernel, as of version
  2.6.11. It's starting to show its age in some places, but it's still
  largely quite accurate, and is an essential guide to anyone who's
  serious about, well, understanding the Linux kernel.

- [LWN][lwn] -- LWN (Linux Weekly News) is an excellent publication,
  and anyone who hacks on Linux or cares about its development is
  well-advised to subscribe. Rarely does a new feature go into Linux
  without an incredibly detailed writeup on LWN, including the history
  of the feature, details of its development, and a low-level
  explanation of how it works and its APIs.

  Even without a subscription, old articles are all freely available,
  and you're well-advised to search LWN's [kernel index][lwn-kernel]
  for anything applicable to your problem.

- The LKML -- The Linux Kernel Mailing List is where almost all the
  action happens in the Linux development community. Few features go
  in without being hotly debated on this mailing list, and discussions
  often lend useful insight into the design and implementation of the
  feature in question.

  Because patches tend to be submitted to the LKML by email, a good
  first step to trying to find discusion on a specific patch is just
  to plug its subject (the first line of the commit message) into
  Google or your favorite LKML archive's search engine.


Well, this has been quite the braindump. I hope this turns out to be
useful to someone, and please comment if you have other advice or
resources you recommend for getting into the Linux source code.

[lkml]: http://lkml.org/
[utlk]: http://oreilly.com/catalog/9780596005658
[lwn]: http://lwn.net/
[lwn-kernel]: http://lwn.net/Kernel/Index/
