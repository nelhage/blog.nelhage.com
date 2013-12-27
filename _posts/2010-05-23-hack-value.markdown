---
layout: post
status: publish
published: true
title: Getting carried away with hack value
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 224
wordpress_url: http://blog.nelhage.com/?p=224
date: 2010-05-23 19:53:30.000000000 +02:00
categories:
- Software Engineering
tags: []
---
Recently, I've been working on some <a
href="http:&#47;&#47;barnowl.mit.edu&#47;">BarnOwl<&#47;a> branches that move more of
the core functionality of BarnOwl into perl code, instead of C
(BarnOwl is written in an unholy mix of C and perl code that call each
other back and forth obsessively).

Moving code into perl has many advantages, but one problem is speed --
perl code is obvious a lot slower than C, and BarnOwl has a lot of hot
spots related to its tendency to keep tens or hundreds of thousands of
messages in memory and loop over all of them in response to various
commands.

Unfortunately, one downside of the C&#47;perl mix is that it makes
profiling quite difficult. I can run the perl side under NYTProf and
get a good picture from the perl side of things, but I've been
unsatisfied about my visibility into the C side of things. The main
problem is that oprofile and gprof, my usual tools are statistical
profilers, which means that if they sample while the code is executing
inside Perl, they don't know which C code called it, and so while I
can tell if a lot of time is being spent in perl, I can't tell which C
functions made the calls that are being slow. What I really want is a
deterministic profiler that tracks every function entry and exit, so
that time spent in a perl call can be included in the total time of
the C functions earlier in the call chain.

After doing some quick googling and finding nothing compelling, I
decided to hack up something of my own -- this can't be that hard,
right?

The tricky part here is getting a way to intercept function calls and
returns, so that you can timestamp them and then figure out where the
time was being spent. My brilliantly hackish plan was:

 * If you compile your code with `-pg`, for profiling with gprof, GCC
   generates a call to the `mcount` function early in the prologue of
   every function. Normally that calls out into a function in glibc
   that keeps stats for gprof, but I can override that with
   `LD_PRELOAD`.
 * The next step is trapping function returns. I wrote a quick program
   that loads an ELF using [libbfd][1], disassembles any text sections
   using [udis86][2], and replaces any RET instructions with the
   one-byte `INT3` instruction, which generates a debug trap.
 
   I could then make my preloaded library install a `SIGTRAP` handler,
   which could inspect the trapped instruction pointer, emulate the
   RET, and then return.

The two parts work independently, but when I glued them together, I
hit trouble. It turns out that `gcc -pg` dumps some code into the
`.text` section of your binary, that ends up getting called into very
early in the shared-library load process -- in particular, before my
`LD_PRELOAD`ed library could install its `SIGTRAP` handler. Since I
had patched those functions to `INT3` instead of `RET`, this crashed
the binary with an unhandled `SIGTRAP` almost immediately.

After some debugging, I was able to resolve that issue by modifying
the patch program to overwrite the first byte of the offending
programs with a RET instruction (after patching out RETs, of course),
so that they effectively didn't get called. Since I'm using `gcc -pg`
only for the `mcount` calls, I don't want them, anyways.

After about four hours of hacking, I had a working system which could
take an executable and run it, generating a trace of function calls
and returns as the program executed. As I was settling down to write
some code to do something useful with the output, my friend [Josh
Oreman][josh] asked a question: "Couldn't you do this more easily with
-finstrument-functions?"

Having never heard of this, I pulled up the GCC [info page][gcc], and
found:

     Generate instrumentation calls for entry and exit to functions.
     Just after function entry and just before function exit, the
     following profiling functions will be called with the address of
     the current function and its call site.

          void __cyg_profile_func_enter (void *this_fn, void *call_site);
          void __cyg_profile_func_exit  (void *this_fn, void *call_site);

Oh. Sometimes it pays to do a little bit of documentation searching
before you get carried away with the hack value of your Awesome
Solution -- someone may have anticipated your problem and written a
real solution.

Oh well. It was a fun experiment, and I refreshed my memory about doing evil things with bfd, udis86, signals, and the dynamic linker.

I still haven't yet written the code to turn traces into userful profiling
information, but if I do produce a useful profiler tool, I'll post something on this blog.


[1]: http:&#47;&#47;en.wikipedia.org&#47;wiki&#47;Binary_File_Descriptor_library
[2]: http:&#47;&#47;udis86.sourceforge.net&#47;
[josh]: http:&#47;&#47;oremanj.scripts.mit.edu&#47;blog&#47;
[gcc]: http:&#47;&#47;gcc.gnu.org&#47;onlinedocs&#47;gcc-4.3.2&#47;gcc&#47;Code-Gen-Options.html
