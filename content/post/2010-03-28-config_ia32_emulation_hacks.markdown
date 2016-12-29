---
author: nelhage
author_email: nelhage@mit.edu
author_login: nelhage
author_url: http://nelhage.com
date: 2010-03-28T20:07:43Z
published: true
status: publish
tags: []
title: 'Fun with the preprocessor: CONFIG_IA32_EMULATION hacks in Linux'
url: /2010/03/config_ia32_emulation_hacks/
wordpress_id: 201
wordpress_url: http://blog.nelhage.com/?p=201
---

<div id="outline-container-1" class="outline-2">
<div id="text-1">

<p>About two months ago, Linux saw <a href="http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-0307">CVE-2010-0307</a>, which was a trival
denial-of-service attack that could crash essentially any 64-bit Linux
machine with 32-bit compatibility enabled. LWN has an <a href="http://lwn.net/Articles/372321/">excellent writeup</a> of the bug, which turns out to be a subtle error related to
the details of the <code>execve</code> system call and with 32-bit compatibility
mode.
</p>
<p>
While dealing with this patch for <a href="http://www.ksplice.com/">Ksplice</a>, I ended up reading an awful
lot of the code in Linux that deals with handling 32-bit processes on
64-bit machines. In the process, I discovered a number of alternately
terrifying and clever hacks, the highlights of which I wanted to share
here.
</p>
</div>

<div id="outline-container-1.1" class="outline-3">
<h3 id="sec-1.1">Compatibility mode </h3>
<div id="text-1.1">

<p>On <code>x86_64</code> Linux kernels, the config option <code>CONFIG_IA32_EMULATION</code>
controls whether the kernel kernel supports i386 compatibility mode,
and the execution of "compatibility-mode" 32-bit processes.
</p>
<p>
For the most part, this support is very simple. As long as the OS sets
up a few bits in appropriate places, the hardware will switch to
32-bit mode and happily execute these compatibility processes in
32-bit mode. The kernel needs to contain a compatibility entry point
to handle system calls and such from 32-bit processes, which needs to
do a small bit of marshalling to convert 32-bit arguments to 64-bit
arguments, and handle the fact that i386 has different syscall numbers
than amd64, but that's about it. Most kernel interfaces are fairly
word-width agnostic; If you map the compatibility process into the
first 4G of the kernel's 64-bit address space, you can mostly just
zero-extend all arguments, and almost everything works fine.
</p></div>

</div>

<div id="outline-container-1.2" class="outline-3">
<h3 id="sec-1.2">But there are always details…</h3>
<div id="text-1.2">

<p>There are, however, a few devilish details that remain. Specifically,
places where the kernel cares about the details of either the pointer
size inside a process, or of the memory layout of a process. The
primary culprit is the ELF loader: The code which is responsible for
loading an executable out of a filesystem and into a process's address
space to start executing. This process is very much architecture
specific; While 64- and 32-bit ELF files are structured almost
identically, many of their fields are different sizes, as they need to
hold a pointer or offset of the appropriate size for the architecture.
</p>
<p>
Similarly, while a running process on Linux stores most relevant
information about its address space layout inside a <code>struct mm</code> in the
<code>struct task_struct</code>, or inside a <code>struct thread_info</code>, when first
constructing a new process, the ELF loader and <code>exec</code> system call need
to figure out how to set up an initial memory layout, which is very
dependent on the bittedness of the new process.
</p>
</div>

<div id="outline-container-1.2.1" class="outline-4">
<h4 id="sec-1.2.1">The ELF loader </h4>
<div id="text-1.2.1">

<p>Linux's ELF loader lives mostly in <a href="http://lxr.linux.no/linux+v2.6.33/fs/binfmt_elf.c"><code>fs/binfmt_elf.c</code></a>, which takes then
definition of an ELF header from <a href="http://lxr.linux.no/linux+v2.6.33/include/linux/elf.h"><code>include/linux/elf.h</code></a>. The latter
file defines the structs for both 32- and 64- ELF files
(e.g. <code>Elf32_Ehdr</code> and <code>Elf64_Ehdr</code>), and then uses an <code>#ifdef</code> near
the bottom to select the appropriate definition.
</p>
<p>
In order to support loading both 32- and 64- bit ELF files in the same
kernel, Linux uses a cute hack on the <a href="http://lxr.linux.no/linux+v2.6.33/fs/compat_binfmt_elf.c"><code>fs/compat_binfmt_elf.c</code></a>
file. This file uses <code>#define</code> to set the ELF class to <code>ELFCLASS32</code>,
indicating that <code>elf.h</code> should use the 32-bit definitions, <code>#define=s a few more thing, and then just =#include</code> 's <code>binfmt_elf.c</code>, causing
the ELF loader to get compiled a second time!:
</p>



<pre class="example">
/*
 * Rename the basic ELF layout types to refer to the 32-bit class of files.
 */
#undef  ELF_CLASS
#define ELF_CLASS       ELFCLASS32

#undef  elfhdr
#undef  elf_phdr
#undef  elf_shdr
#undef  elf_note
#undef  elf_addr_t
#define elfhdr          elf32_hdr
#define elf_phdr        elf32_phdr
#define elf_shdr        elf32_shdr
#define elf_note        elf32_note
#define elf_addr_t      Elf32_Addr

/* Some more #defines elided */

/*
 * We share all the actual code with the native (64-bit) version.
 */
#include "binfmt_elf.c"
</pre>




<p>
The ELF structs themselves, however, aren't the only thing that
depends on the architecture. The details of initializing a new process
depend on the architecture as well. So, throughout <code>binfmt_elf.c</code>,
there are a number of calls to macros that handle various
platform-specific elemnts of ELF loading.
</p>
<p>
<code>compat_binfmt_elf.c</code> then just goes through and uses <code>#define</code> to
replace all of these with appropriate <code>COMPAT_</code> versions, defined by
the architecture:
</p>



<pre class="example">
#undef  ELF_ARCH
#undef  elf_check_arch
#define elf_check_arch  compat_elf_check_arch

#ifdef  COMPAT_ELF_PLATFORM
#undef  ELF_PLATFORM
#define ELF_PLATFORM            COMPAT_ELF_PLATFORM
#endif

/* ... */
</pre>




<p>
The Linux developers do love their preprocessor.
</p></div>

</div>

<div id="outline-container-1.2.2" class="outline-4">
<h4 id="sec-1.2.2"><code>TASK_SIZE</code> </h4>
<div id="text-1.2.2">

<p>In the linux kernel, the <code>TASK_SIZE</code> macro defines the highest address
available to a user process. Once a process is running, this
information (along with a whole host of other information about the
memory layout). However, in various places, including the ELF loader,
the <code>TASK_SIZE</code> macro (along with a few others, like <code>STACK_TOP</code>) are
needed.
</p>
<p>
However, <code>TASK_SIZE</code> obviously must be different between 32- and 64-
processes. Conveniently, almost all code that uses <code>TASK_SIZE</code> cares
about the current process (such as the ELF loader), and so the
introduction of compatibility mode just changed the macro as follows
(<a href="http://lxr.linux.no/linux+v2.6.33/arch/x86/include/asm/processor.h"><code>arch/x86/include/asm/processor.h</code></a>):
</p>



<pre class="example">
#define TASK_SIZE (test_thread_flag(TIF_IA32) ? \ IA32_PAGE_OFFSET :
                                        TASK_SIZE_MAX)
</pre>




<p>
<code>test_thread_flag</code> reads a bit out of the flags field on the current
process's <code>thread_info</code> struct. And so the <code>TASK_SIZE</code> macro
pseudo-magically changes value depending on whether the process
calling it is running in 32-bit compatibility mode or not!
</p></div>
</div>
</div>
</div>
