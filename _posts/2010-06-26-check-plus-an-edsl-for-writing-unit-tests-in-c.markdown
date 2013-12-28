---
layout: post
status: publish
published: true
title: ! 'Check Plus: An EDSL for writing unit tests in C'
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 263
wordpress_url: http://blog.nelhage.com/?p=263
date: 2010-06-26 15:54:53.000000000 +02:00
tags:
- C
- programming
- check
- testing
- macros
---
[Check][check] is an excellent unit-testing framework for C code, used
by a number of relatively well-known projects. It includes features
such as running all tests in separate address spaces (using
`fork(2)`), which means that the test suite can properly report
segfaults or similar crashes without the test runner crashes.

My main complaint about Check is that (unsurprisingly for a framework
written in C), it's not very declarative. After you define all your
tests as separate functions, you need to write code to manually
collect them into "test cases", which you then collect into "test
suites", which you can then run. While this is an understandable
interface for a C library to provide, as a user I found that it
rapidly annoyed me, and so I wrote Check Plus.

[Check Plus][chkp] is a simple library I wrote that uses some preprocessor
tricks and GNU C extensions to provide a declarative EDSL within C for
declaring and collecting your test cases. Check-Plus is a single
self-contained header file distributed under the LGPL (like Check), so
it should be easy to pull into your project.

Example
------

Check-Plus lets you declare your Check suites and test cases in a
declarative manner. A simple example, shipped with Check-Plus, would
look like:

    #include <string.h>
    #include <stdlib.h>
    
    #include "check-plus.h"
    
    #define SUITE_NAME example
    BEGIN_SUITE("Example test suite");
    
    #define TEST_CASE core
    BEGIN_TEST_CASE("Core tests");
    
    TEST(strcmp)
    {
        fail_unless(strcmp("hello", "HELLO") != 0);
        fail_unless(strcasecmp("hello", "HELLO") == 0);
    }
    END_TEST
    
    TEST(strcat)
    {
        char buf[1024] = {0};
        strcat(buf, "Hello, ");
        strcat(buf, "World.");
        fail_unless(strcmp(buf, "Hello, World.") == 0);
    }
    END_TEST
    
    END_TEST_CASE;
    #undef TEST_CASE
    END_SUITE;
    #undef SUITE_NAME

Having done that, you can extract the declared test suite using
`construct_test_suite(example)`, and run it using the standard Check
test runner tools.

You can, of course, define multiple test cases within a single suite,
and you can even define test fixtures using

    DEFINE_FIXTURE(setup, teardown);

within the `BEGIN_TEST_CASE` ... `END_TEST_CASE` block.

The equivalent code, without Check-Plus, would look similar, but
instead of the `BEGIN_` and `END_` blocks to define the suite and test case, you'd have to write
something like:

    Suite *
    example_suite (void)
    {
      Suite *s = suite_create ("Example test suite");
    
      /* Core test case */
      TCase *tc_core = tcase_create ("Core tests");
      tcase_add_test (tc_core, test_strcmp);
      tcase_add_test (tc_core, test_strcat);
      suite_add_tcase (s, tc_core);
    
      return s;
    }

It's not a huge difference, but it's big enough that I really
appreciate it once the test suite gets large. For more documentation on Check, check out [its manual][manual].

Let me know if you find this useful for anything. Next week, I'll do a
writeup on the tricks behind the implementation.
    
[check]: http://check.sourceforge.net/
[chkp]: http://github.com/nelhage/check-plus
[manual]: http://check.sourceforge.net/doc/check_html/index.html
