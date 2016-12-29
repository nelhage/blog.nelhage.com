---
author: nelhage
author_email: nelhage@mit.edu
author_login: nelhage
author_url: http://nelhage.com
date: 2010-07-04T15:54:55Z
published: true
status: publish
tags:
- C
- software
- check
- cpp
- dsl
- preprocessor
title: Implementing a declarative mini-language in the C preprocessor
url: /2010/07/04/implementing-an-edsl-in-cpp/
wordpress_id: 272
wordpress_url: http://blog.nelhage.com/?p=272
---

[Last time][lasttime], I announced Check Plus, a declarative language for
defining [Check][check] tests in C. This time, I want to talk about the tricks I
used to implement a declarative minilanguage using the C preprocessor (and some
GCC extensions).

The Problem
-----------

We want to write some toplevel declarations that look like:

    #define SUITE_NAME example
    BEGIN_SUITE("Example test suite");

    #define TEST_CASE core
    BEGIN_TEST_CASE("Core tests");
    …

and so on, and somehow translate them into code that does the equivalent of:

    Suite *s = suite_create ("Example test suite");
    TCase *tc_core = tcase_create ("Core tests");
    …

First steps
-----------

Instead of having our macros somehow lay down code directly, we'll use them to
define some global data structures, which we will then loop across at runtime,
and make the appropriate Check library calls. We'll define some structs that
we're going to use for this purpose:

    typedef void (*test_func)(int);
    typedef void (*test_hook)(void);

    struct test_case {
        test_func  *funcs, *end_funcs;
        const char *test_name;
        test_hook *setup_hooks, *end_setup_hooks;
        test_hook *teardown_hooks, *end_teardown_hooks;
    };

    struct test_suite {
        struct test_case *cases, *end_cases;
        const char *suite_name;
    };

Since tests and test fixtures are code, we're going to need to keep function
pointers to them. We use typedefs to keep thos readable, and then we define
structs to represent a test suite, and a test case. We're going to represent
arrays (of test cases, fixture hooks, or test functions) using a pair of
pointers, to the start and end of the array. There's a specific reason for this
representation, which I'll get to shortly.

ELF Sections
------------

We can easily define global instances of these structs, but the problem is how
to get the pointers to the child arrays correctly. These macros are going to be
interspersed with other code, so we can't just use a literal array, like

    struct test_suite s = {
     .suite_name = "Example test suite",
     .cases = {
       …
     }
    };

because that "…" is going to have arbitrary code shoved in it.

The solution is to take advantage of a feature of the ELF object file format,
and some GCC extensions that let us take advantage of it.

ELF object files can contain an arbitary set of named sections, each
of which can contain code or data (or both, but that's not
typical). GCC allows us to specify that a given variable should live
in a given section using
`__attribute__((section("section_name"))`. So, by placing all our
`struct test_case`s for a given `test_suite` into a separate section,
we can cause them to get placed contiguously, which means we just need
to get pointers to the start and end. That we can accomplish by
declaring a zero-length array in the appropriate section, which (being
of size zero) won't generate any data, but will have the appropriate
address.

So, the code we're going to generate for the above will look something
like:

    struct test_case _begin_example_cases[];
    struct test_case _end_example_cases[];
    struct test_suite _suite_example = {
      .cases = _begin_example_cases,
      .end_cases = _end_example_cases,
      .suite_name = "Example test suite"
    };
    struct test_case _begin_example_cases[0]
      __attribute__((section(".data.example.cases")));

    struct test_case _test_example_core
      __attribute__((section(".data.example.cases"))) = {
      .test_name = "Core tests",
      …
    };

    …

    struct test_case _end_example_cases[0]
      __attribute__((section(".data.example.cases")));

Preprocessor tricks
-------------------

So far, I haven't actually talked about the preprocessor at all. I've
just shown the code we want our macros to generate. So now let's get
to the business of actually generating this code. I'll walk through
the definition of `BEGIN_SUITE', which expands to the code above. The
other definitions are all essentially analagous.

My first version of this library didn't use the `SUITE_NAME` and
`TEST_CASE` macros, instead requiring those arguments to be passed to
every macro. I like the decreased repetition that this method gives,
but we'll start with the explicit version, since it requires less
trickery to understand.

From the above example, we can see that `BEGIN_SUITE(example,
"Example")' is given the `example' token, and needs to construct both
symbols (`_suite_example`) and strings (`".data.example.cases"`)
containing that symbol. To do the first, we need to the processor
operator `##`, which concatenates two tokens into one. For example,
given

    #define PASTE(x,y) x##_##y

`PASTE(foo, bar)' is equivalent to just having written "foo_bar"
literally in your code.

Secondly, to construct strings, we need the `#x' preprocessor
operator. This operator, when applied to an argument to a macro,
returns a string literal containing the tokens passed to the macro. A
common usage of this is for an `assert` macro, that can print the
failed expression:

    #define assert(x) if (!(x)) { die("Assertion failed: %s\n", #x); }

And finally, we're going to need the implicit concatenation feature of
the C language, which lets you type `"foo" "bar"` with the same
meaning as "foobar".

Putting it all together, and we get:

    #define BEGIN_SUITE(sym, name)                                          \
        struct test_case _begin_ ## sym ## _cases[];                        \
        struct test_case _end_ ## sym ## _cases[];                          \
        struct test_suite _suite_ ## sym = {                                \
            .cases       = (struct test_case*)_begin_ ## sym ## _cases,     \
            .end_cases   = (struct test_case*)_end_ ## sym ## _cases,       \
            .suite_name  = name,                                            \
        };                                                                  \
        struct test_case _begin_ ## sym ## _cases[0]                        \
        __attribute__((section(".data." #sym ".cases")))

Two last things to note: we deliberately leave off a trailing
semicolon, requiring the user to provide one, and we need to end every
line with a `\`, to include the entire body in the macro definition.

Another layer of indirection
----------------------------

The one final bit is to get rid of that explicit `sym` argument
everywhere, in favor of the `SUITE_NAME` define. Our first try might
look something like:

    #define BEGIN_SUITE(name)                                               \
        struct test_case _begin_ ## SUITE_NAME ## _cases[];                 \
        …

You'll quickly realize, however, that since `##` is applied at the
same time as function-like macro expansion, this will always give you
the symbol `_begin_SUITE_NAME_cases`. The solution is to force the
preprocessor to perform multiple phases of macro expansion, to cause
the `##` to be applied only after `SUITE_NAME` is expanded. In this
case, we require three layers of macros to get the order right:

    #define BEGIN_SUITE(name) _BEGIN_SUITE(SUITE_NAME, name)
    #define _BEGIN_SUITE(sym, name) __BEGIN_SUITE(sym, name)
    #define __BEGIN_SUITE(sym, name)                                        \
        struct test_case _begin_ ## sym ## _cases[];                        \

This causes the following sequence of expansions for
`BEGIN_SUITE("Example");`:

    BEGIN_SUITE("Example");                     // (a)
    _BEGIN_SUITE(SUITE_NAME, "Example");        // (b)
    __BEGIN_SUITE(sym, "Example");              // (c)

The key here is that when expanding a function-like macro, the
preprocessor performs the following steps, in order:

1. Apply any `#` or `##` operators.
2. Macro-expand any arguments to the call
3. Perform the substitution
4. Macro-expand the result of the substitution

In going from (a) to (b), only step (3) applies. When going from (b)
to (c), rule (2) means that `SUITE_NAME` gets expanded first, leaving
(c). Rule (1) means that the `_BEGIN_SUITE` macro is necessary, since
a call to `__BEGIN_SUITE(SUITE_NAME, "Example")` will get the `#` and
`##` applied to `SUITE_NAME` before it can be expanded.

Conclusions
-----------

So, that's basically all there is to the Check-Plus implementation. If
you're comfortable with that, you can try your hand at unravelling the
implementation of the Linux kernel's [tracing][define_trace]
[macros][ftrace], next ([sample usage][tracing])!

[lasttime]: http://blog.nelhage.com/2010/06/check-plus-an-edsl-for-writing-unit-tests-in-c/
[check]: http://check.sourceforge.net/
[define_trace]: http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=blob;f=include/trace/define_trace.h;h=1dfab54015113b83bce9f3302470c3a5ed95b5e7;hb=HEAD
[ftrace]: http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=blob;f=include/trace/ftrace.h;h=5a64905d7278a47fb683a0aceb63cef029dd467b;hb=HEAD
[tracing]: http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=blob;f=include/trace/events/kmem.h;h=3adca0ca9dbee10479d34d5a3e3562609ef89e86;hb=HEAD
