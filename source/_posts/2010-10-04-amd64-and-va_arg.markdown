---
layout: post
status: publish
published: true
title: amd64 and va_arg
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 355
wordpress_url: http://blog.nelhage.com/?p=355
date: 2010-10-04 00:14:28.000000000 +02:00
tags:
- C
- amd64
- abi
- calling convention
---
A while back, I was poking around [LLVM][llvm] bugs, and discovered,
to my surprise, that LLVM [doesn't support][va_arg-bug] the `va_arg`
intrinsic, used by functions to accept multiple arguments, at all on
amd64. It turns out that `clang` and `llvm-gcc`, the compilers that
backend to LLVM, have their own implementations in the frontend, so
this isn't as big a deal as it might sound, but it was still a
surprise to me.

Figuring that this might just be something no one got around to, and
couldn't actually be that hard, I pulled out my copy of the [amd64
ABI][amd64abi] specification, figuring that maybe I could throw
together a patch and fix this issue.

Maybe half an hour of reading later, I stopped in terror and gave up,
repenting of my foolish ways to go work on something else. `va_arg` on
amd64 is a hairy, hairy beast, and probably not something I was going
to hack together in an evening. And so instead I decided to blog about
it.

The problem: Argument passing on amd64
--------------------------------------

On i386, because of the dearth of general-purpose registers, the
calling convention passes all arguments on the stack. This makes the
va_arg implementation easy -- A `va_list` is simply a pointer into the
stack, and `va_arg` just adds the size of the type to be retrieved to
the `va_list`, and returns the old value. In fact, the i386 ABI
reference simply specifies `va_arg` in terms of a single line of
code:

    #define va_arg(list, mode) ((mode *)(list = (char *)list + sizeof(mode)))[-1]

On amd64, the problem is much more complicated. To start, amd64
specifies that up to 6 integer arguments and up to 8 floating-point
arguments are passed to functions in registers, to take advantage of
amd64's larger number of registers. So, for a start, `va_arg` will
have to deal with the fact that some arguments may have been passed in
registers, and some on the stack.

(One could imagine simplifying the problem by stipulating a different
calling convention for variadic functions, but unfortunately, for
historical reasons and otherwise, C requires that code be able to call
functions even if their prototype is not visible, which means the
compiler doesn't necessarily know if it's calling a variadic function
at any given call site. <em>[edited to add: <strong>caf</strong>
points out in the comments that C99 actually explicitly does not
require this property. But I speculate that the ABI designers wanted
to preserve this property from i386 because it has historically
worked, and so existing code depended on it]</em>).

That's not all, however. Not only can integer arguments be passed by
registers, but small `struct`s (16 bytes or fewer) can also be passed
on the stack. A sufficiently small struct, for the purposes of the
calling convention, is essentially broken up into its component
members, which are passed as though they were separate arguments --
unless only some of them would fit into registers, in which case the
whole struct is passed on the stack.

So `va_arg`, given a `struct` as an argument, has to be able to figure
out whether it was passed in registers or on the stack, and possibly
even re-assemble it into temporary space.

The implementation
------------------

Given all those constraints, the required implementation is fairly
straightforward, but incredibly complex compared to any other platform
I know of.

To start, any function that is known to use `va_start` is required to,
at the start of the function, save all registers that may have been
used to pass arguments onto the stack, into the "register save area",
for future access by `va_start` and `va_arg`. This is an obvious step,
and I believe pretty standard on any platform with a register calling
convention. The registers are saved as integer registers followed by
floating point registers. As an optimization, during a function call,
`%rax` is required to hold the number of SSE registers used to hold
arguments, to allow a varargs caller to avoid touching the FPU at all
if there are no floating point arguments.

`va_list`, instead of being a pointer, is a structure that keeps track
of four different things:

    typedef struct {
      unsigned int gp_offset;
      unsigned int fp_offset;
      void *overflow_arg_area;
      void *reg_save_area;
    } va_list[1];

`reg_save_area` points at the base of the register save area
initialized at the start of the function. `fp_offset` and `gp_offset`
are offsets into that register save area, indicating the next unused
floating point and general-purpose register, respectively. Finally,
`overflow_arg_area` points at the next stack-passed argument to the
function, for arguments that didn't fit into registers.

Here's an ASCII art diagram of the stack frame during the execution of
a varargs function, after the register save area has been
established. Note that the spec allows functions to put the register
save area anywhere in its frame it wants, so I've shown potential
storage both above and below it.


    |     ...        |  [high addresses]
    +----------------+
    |   argument     |
    |    passed      |
    |  on stack (2)  |
    +----------------+  <---- overflow_arg_area
    |   argument     |
    |    passed      |
    |  on stack (1)  |
    +----------------+
    | return address |
    +----------------+
    |      ...       | (possible local storage for func)
    +----------------+
    |     %xmm15     | \
    +----------------+  |     
    |     %xmm14     |  |     ___
    +----------------+  |      |
    |      ...       |   \ register
    +----------------+    }save|
    |     %xmm0      |   / area|
    +----------------+  |      |
    |      %r9       |  |      |
    +----------------+  |      | fp_offset
    |      %r8       |  |  ___ |
    +----------------+  |   |  |
    |      ...       |  |   |  |
    +----------------+  |   | gp_offset
    |     %rsi       |  |   |  |
    +----------------+  |   |  |
    |     %rdi       | /    |  |
    +----------------+ <----+--+--- reg_save_area
    |     ...        | (potentially more storage)
    +----------------+ <----------- %esp
    |     ...        | [low addresses]

Because `va_arg` must tell determine whether the requested type was
passed in registers, it needs compiler support, and can't be
implemented as a simple macro like on i386. The amd64 ABI reference
specifies `va_arg` using a list of eleven different steps that the
macro must perform. I'll try to summarize them here.

First off, `va_arg` determines whether the requested type could be
passed in registers. If not, `va_arg` behaves much like it does on
`i386`, using the `overflow_arg_area` member of the `va_list` (Plus
some complexity to deal with alignment values).

Next, assuming the argument can be passed in registers, `va_arg`
determines how many floating-point and general-purpose registers would
be used to pass the requested type. It compares those values with the
`gp_offset` and `fp_offset` fields in the `va_list`. If the additional
registers would cause either value to overflow the number of registers
used for parameter-passing for that type, then the argument was passed
on the stack, and `va_arg` bails out and uses `overflow_arg_area`.

If we've made it this far, the argument was passed in
registers. `va_arg` fetches the argument using `reg_save_area` and the
appropriate offsets, and then updates `gp_offset` and `fp_offset` as
appropriate.

Note that if the argument was passed in a mix of floating-point and
general-purpose registers, or requires a large alignment, this means
that `va_arg` must copy it out of the register save area onto
temporary space in order to assemble the value.

So, in the worst case, `va_arg` on a type that embeds both a
floating-point and an integer type must do two comparisons, a
conditional branch, and then update two fields in the `va_list` and
copy multiple values out of the register save area into a temporary
object to return. That's quite a lot more work than the i386 version
does. Note that I don't mean to suggest this is a performance concern
-- I don't have any benchmarks to back this up, but I would be shocked
if this is measurable in any reasonable code. But I was surprised by
how complex this operation is.

[llvm]: http://llvm.org/
[va_arg-bug]: http://llvm.org/bugs/show_bug.cgi?id=1740
[amd64abi]: http://www.x86-64.org/documentation/abi.pdf
