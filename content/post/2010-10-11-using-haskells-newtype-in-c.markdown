---
author: nelhage
author_email: nelhage@mit.edu
author_login: nelhage
author_url: http://nelhage.com
date: 2010-10-11T13:11:25Z
published: true
status: publish
tags:
- C
- programming
- newtype
- haskell
- JOS
- operating systems
- memory
title: Using Haskell's 'newtype' in C
url: /2010/10/using-haskells-newtype-in-c/
wordpress_id: 364
wordpress_url: http://blog.nelhage.com/?p=364
---

A common problem in software engineering is avoiding confusion and
errors when dealing with multiple types of data that share the same
representation. Classic examples include differentiating between
measurements stored in different units, distinguishing between a
string of HTML and a string of plain text (one of these needs to be
encoded before it can safely be included in a web page!), or keeping
track of pointers to physical memory or virtual memory when writing
the lower layers of an operating system's memory management.

Unless we're using a richly-typed language like Haskell, where we can
use [`newtype`][newtype], the best solutions tend to just rely on
convention. The much-maligned [Hungarian Notiation][hungarian] evolved
in part to try to combat this sort of problem -- If you decide on a
convention that variables representing physical addresses start with
`pa` and virtual addresses start with `va`, then anyone who encounters
a `uintptr_t pa_bootstack;` can immediately decode the intent.

It turns out, though, that we can get something very much like
`newtype` in familiar old C. Suppose we're writing some of the paging
code for a toy x86 architecture. We're going to be passing around a
lot of physical and virtual addresses, as well as indexes of pages in
RAM, and it's going to be easy to confuse them all. The traditional
solution is to use some `typedef`s, and then promise to be very
careful to mix them up:

    typedef uint32_t physaddr_t;
    typedef uint32_t virtaddr_t;
    typedef uint32_t ppn_t;

We have to promise not to mess up, though -- the compiler isn't going
to notice if I pass a `ppn_t` to a function that wanted a
`physaddr_t`.

This example was inspired by JOS, a toy operating system used by MIT's
Operating Systems Engineering class. JOS remaps all of physical memory
starting at a specific virtual memory address (`KERNBASE`), and so
provides the following macros:

    /* This macro takes a kernel virtual address -- an address that points above
     * KERNBASE, where the machine's maximum 256MB of physical memory is mapped --
     * and returns the corresponding physical address.  It panics if you pass it a
     * non-kernel virtual address.
     */
    #define PADDR(kva)                                          \
    ({                                                          \
            physaddr_t __m_kva = (physaddr_t) (kva);            \
            if (__m_kva < KERNBASE)                                     \
                    panic("PADDR called with invalid kva %08lx", __m_kva);\
            __m_kva - KERNBASE;                                 \
    })

    /* This macro takes a physical address and returns the corresponding kernel
     * virtual address.  It panics if you pass an invalid physical address. */
    #define KADDR(pa)                                           \
    ({                                                          \
            physaddr_t __m_pa = (pa);                           \
            uint32_t __m_ppn = PPN(__m_pa);                             \
            if (__m_ppn >= npage)                                       \
                    panic("KADDR called with invalid pa %08lx", __m_pa);\
            (void*) (__m_pa + KERNBASE);                                \
    })

Because the typedefs are unchecked by the compiler, though, it is a
common mistake to use a physical address where a virtual address is
meant, and nothing will catch it until your kernel triple-faults, and
a long, painful debugging session ensues.


Inspired by Haskell's `newtype`, though, it turns out we can get the
compiler to check it for us, with a little more work, by using a
singleton `struct` instead of a `typedef`:

    typedef struct { uint32_t val; } physaddr_t;

If we wanted to be overly cute, we could even use a macro to mimic
Haskell's `newtype`:

    #define NEWTYPE(tag, repr)                  \
        typedef struct { repr val; } tag;       \
        static inline tag make_##tag(repr v) {  \
                return (tag){.val = v};         \
        }                                       \
        static inline repr tag##_val(tag v) {   \
                return v.val;                   \
        }

    NEWTYPE(physaddr, uint32_t);
    NEWTYPE(virtaddr, uint32_t);
    NEWTYPE(ppn,  uint32_t);

Given those definitions, `PADDR` and `KADDR` become:

    #define PADDR(kva)                                          \
    ({                                                          \
        if (virtaddr_val(kva) < KERNBASE)                       \
                panic("PADDR called with invalid kva %08lx", virtaddr_val(kva)); \
        make_physaddr(virtaddr_val(kva) - KERNBASE);            \
    })

    #define KADDR(pa)                                           \
    ({                                                          \
        uint32_t __m_ppn = physaddr_val(pa) >> PTXSHIFT;        \
        if (__m_ppn >= npage)                                   \
                panic("KADDR called with invalid pa %08lx", physaddr_val(pa)); \
        make_virtaddr(physaddr_val(pa) + KERNBASE);             \
    })

We have to use some accessor and constructor functions, but in
exchange, we get strong type-checking: If you pass `PADDR` a physical
address (or anything other than a virtual address), the compiler will
catch it.

The wrapping and unwrapping is slightly annoying, but we can for the
most part avoid having to do it everywhere, by pushing the wrapping
and unwrapping down into some utility functions. For instance, a
relatively common operation at this point in JOS is creating a
page-table entry, given a physical address. If you want to construct
the PTE by hand, you need to use `physaddr_val` every time. But a
better plan is a simple utility function:

     static inline pte_t make_pte(physaddr addr, int perms) {
         return physaddr_val(addr) | perms;
     }

In addition to losing the need to unwrap the `physaddr` everywhere, we
gain a measure of clarity and typechecking -- if you remember to use
`make_pte`, you'll never accidentally try to insert a virtual address
into a page table.

We can add similar functions for converting between types, as well a a `struct
Page`, used to track metadata for a physical page. As an experiment, I went and
reimplemented JOS's memory management primitives using these definitions, and
only needed to use `FOO_val` or `make_FOO` a very few times outside of the
header files that defined `KADDR` and friends.

Performance
-----------

While the typechecking is nice, any C programmer implementing a
memory-management system is probably going to want to know: How much does it
cost me? You're creating and unpacking these singleton `struct`s everywhere --
does that have a cost?

The answer, though, in almost all cases is "no" -- A half-decent compiler will
optimize the resulting code to be completely identical to the code without the
`struct`s, in almost all cases.

Also, the in-memory representation of the `struct` is going to be exactly the
same as the bare value -- it's even guaranteed to have the same alignment and
padding constraints, so if you need to embed a `physaddr` inside another struct,
or into an array, the representation is identical to the `physaddr_t` typedef.

On i386, parameters are passed on the stack, so that means that passing the
struct is identical to passing the `uint32_t`. On amd64, as described last week,
small structures are passed in registers, and so, again, the calling convention
is identical.

Unfortunately, the i386 ABI specifies that returned `struct`s always go on the
stack (while integers go in `%eax`), so you do pay slightly if you want to
return one of these typedef'd objects. `amd64` will also break it down into a
register, though, so on a 64-bit machine it's again identical.

If you're worried, though, you can always use the preprocessor to make the
checks vanish for a production build:

    #ifdef NDEBUG
    #define NEWTYPE(tag, repr)                  \
        typedef repr tag;                       \
        static inline tag make_##tag(repr v) {  \
                return v;                       \
        }                                       \
        static inline repr tag##_val(tag v) {   \
                return v;                       \
        }
    #else
    /* Same definition as above */
    #endif

Because the types have identical representations, you can safely
serialize your structs and exchange them between code compiled with
either version. On amd64, you can probably even call between
compilation units defined either way.

The next time you're writing some subtle C code that has to deal with multiple
types with the same representation, I encourage you to consider using this
trick.


Addendum
--------

I didn't invent this trick, although as far as I know the `NEWTYPE` macro is my
own invention (<em>Edited to add: A commenter points out that I'm not the first
to [use the `newtype` name in
C](http://notanumber.net/archives/33/newtype-in-c-a-touch-of-strong-typing-using-compound-literals),
although I think I prefer my implementation</em>).

. I learned this trick from the Linux kernel, which uses it for a
very similar application -- distinguishing entries in different levels of the
x86 page tables. `page.h` on amd64 includes following definitions [Taken from an
old version, but the current version has equivalent ones):

    /*
     * These are used to make use of C type-checking..
     */
    typedef struct { unsigned long pte; } pte_t;
    typedef struct { unsigned long pmd; } pmd_t;
    typedef struct { unsigned long pud; } pud_t;
    typedef struct { unsigned long pgd; } pgd_t;


I claimed above that the struct and the bare type will have the same alignment
and padding. I don't believe this is guaranteed by C99, but the SysV amd64 and
i386 ABI specifications both require:

<blockquote>
Structures and unions assume the alignment of their most strictly aligned
component. Each member is assigned to the lowest available offset with the
appropriate alignment. The size of any object is always a multiple of the
object’s alignment.
</blockquote>

(text quoted from the amd64 document, but the i386 one is almost identical).

And C99 requires (§6.7.2.1 para 13):

<blockquote>
… A pointer to a structure object, suitably converted, points to its initial
member (or if that member is a bit-field, then to the unit in which it resides),
and vice versa. There may be unnamed padding within a structure object, but not
at its beginning.
</blockquote>

I believe these requirements, taken together, should be enough to ensure that
the `struct` and the bare type will have the same representation.


[hungarian]: http://en.wikipedia.org/wiki/Hungarian_Notation
[newtype]: http://blog.ezyang.com/2010/08/type-kata-newtypes/
[metric]: http://articles.cnn.com/1999-09-30/tech/9909_30_mars.metric.02_1_climate-orbiter-spacecraft-team-metric-system?_s=PM:TECH
