---
title: "Inventing structured concurrency through the lens of error-handling"
slug: inventing-structured-concurrency
date: 2026-01-05T09:00:00-08:00
---
How should we think about error-handling in concurrent programs?

In single-threaded programs, we’ve mostly converged on a standard pattern, with a diverse zoo of implementations and concrete patterns. When an error occurs, it is propagated up the stack until we find a stack frame which is prepared to handle it. As we do so, we unwind the stack frames in-order, giving each frame the opportunity to clean up or destroy resources as appropriate.

This pattern clearly describes the explicit exception-handling mechanisms in many modern languages (C++, Python, Java), all of which also have mechanisms for cleanup on frame unwind (RAII, `finally` blocks, Python context managers). But it also describes the standard pattern in Rust (`Result` returns, the `?` operator, and calling `drop` on wind), as well as Go (the classic `if err != nil { return err }` pattern and `defer` for cleanup), and even most modern `C` code, via patterns like `goto error` pattern (c.f. [examples in the Linux kernel](https://livegrep.com/search/linux?q=goto%20err&fold_case=auto&regex=false&context=true)).

From today’s vantage, that description may seem so general as to be contentless, but that has not always been so. Some other (mostly-abandoned) error-handling approaches include Lisp’s [“restarts” mechanism](https://lisper.in/restarts), `longjmp` in C[^longjmp], “trap” mechanisms like UNIX signals, and the infamous [`on error`](https://learn.microsoft.com/en-us/office/vba/language/reference/user-interface-help/on-error-statement) clause in Visual Basic. "Unwinding" is also predicated on **having** a structured call stack, a concept which also had to be [invented and popularized](https://en.wikipedia.org/wiki/Structured_programming).

[^longjmp]: `longjmp` can be used as a primitive to help **implement** exceptions and unwinding. But by itself it’s a much lower-level primitive and admits a wide variety of alternate patterns.

Today I want to ask: how should we update this pattern for **concurrent** programs, where there is no single stack? How do we organize our code to handle error conditions, in the presence of multiple concurrent tasks[^task]?

[^task]: I'm interested in logical [concurrency more so than hardware parallelism](https://go.dev/blog/waza-talk), so I'll be using the word "task" to refer to separate linear sequences of execution, which may be interleaved in time with any number of other such sequences.

## Unhandled errors

If an error is raised and handled within a single task, our usual idioms work fine. We're concerned about what happens when the error involves multiple tasks.

One entrypoint we can use to explore that question is to ask what happens when our single-threaded "stack-unwinding" approach "runs out of stack" -- we reach the top of a given task's stack, without handling the error.

In a single-threaded program, it’s natural that exceptions which “bubble up” out of the entrypoint should terminate the program, but in a concurrent program, it’s less clear: There may be other tasks that are making progress, and we may not wish to terminate them.

For concreteness, let’s consider this simple program[^python]:


    FIGURE:exception_thread.py

[^python]: Most of this discussion is intended to apply broadly across many languages and concurrency frameworks, but I’ll use Python example for concreteness. In addition to other benefits, Python supports both threading and cooperative asynchrony, which lets me explore multiple paradigms.

This program attempts to run two concurrent threads, one doing some “main” work function, and one doing some kind of work “in the background.” However, the background thread raises an exception immediately. What should happen?

Running a version of this program across various language implementations, we will discover there are, broadly, two common approaches:

- Print the error, terminate the thread, and carry on until all threads (or the main thread, or all “non-daemon” threads) have exited. (Java, Python)
- Print the error, and immediately terminate the entire program (Go, Rust, C++)

In Python, the program logs the exception immediately, but then continues exiting — it waits for the `time.sleep` to elapse and then exits – with a successful exit code, even!


    $ time python exception_thread.py
    Exception in thread Thread-1 (background_thread):
    Traceback (most recent call last):
      File "/Users/nelhage/.pyenv/versions/3.11.0/lib/python3.11/threading.py", line 1038, in _bootstrap_inner
        self.run()
      File "/Users/nelhage/.pyenv/versions/3.11.0/lib/python3.11/threading.py", line 975, in run
        self._target(*self._args, **self._kwargs)
      File "/Users/nelhage/Sync/code/structured-concurrency/exception_thread.py", line 7, in background_thread
        raise ValueError("oops")
    ValueError: oops
    All done, exiting!
    python exception_thread.py  0.03s user 0.02s system 0% cpu 5.124 total

    $ echo $?
    0

Neither option is particularly satisfactory.

Killing the entire program is a heavyweight hammer. I have, for instance, personally been involved in high-severity incidents where the proximate cause was an unhandled panic in a background monitoring goroutine, which was crashing a critical daemon. If the error had not been fatal, we would have had a much better time.

On the flip side, letting the program continue means we’re in a state that was almost certainly never tested or anticipated, and we run a high risk of deadlock if another task was waiting on the now-dead task. It also creates a lot of pain during development and debugging, where we, as programmers, generally want to get feedback about the error and move on to fixing it, as quickly as possible.

## Where do I raise the exception?

What we'd like, in some sense, is to have a better place to deliver the exception. In a single-threaded program, that place is "the caller." In the presence of concurrency, tasks don't have a caller to which they will eventually return, so what should we do instead?

Python’s [`asyncio`](https://docs.python.org/3/library/asyncio.html) framework takes a different approaach than any of the above languages. In `asyncio`, a task is represented by a [`Task`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task) object, which is an object that can be waited on, much like an event or a lock or a socket or any other asynchronous object. Waiting on a `Task` blocks until it completes; if the task raises an exception, it will be redelivered to anyone waiting on the `Task`.

This approach is unopinionated; it avoids taking an opinion on **who** should handle an exception that escapes a task, but it gives the programmer the tools to make that decision themselves.

However, it comes with a steep downside. If **no one** waits on a `Task`, and that task raises an exception, the exception is effectively swallowed until program exit, at which point it will be printed with a warning. If someone was waiting on that task to, say, produce output via some queue, then the program will simply hang forever, silent, inscrutable, and mysterious.

Indeed, he situation is bad enough that I sometime summarize it as “by default, `asyncio` completely swallows exceptions off the main task.” That’s not literally true, but in my experience it is a decent mental model to start with, and usefully describes many developers' experiences with `asyncio`.

## What if we always waited on tasks?

There's a lot to recommend the `asyncio` approach, **as long as you remember to wait on every** `**Task**`. How can we enforce that invariant, as a rule? The simplest rule might be: if you spawn a task, you are also responsible for waiting on it. We could encode this pattern by making `asyncio.create_task(coro)` into an asynchronous context manager, which will wait on the task before exiting.

```python
# (n.b. this is not real API in any version of Python)
async with asyncio.create_task(background_task()) as task:
  # …
  # `task` will be waited for on exit from this block, and any exception raised
```

Of course, we very often want to create many tasks — or a dynamically-chosen number of tasks — so we could borrow an idiom from [`contextlib.ExitStack`](https://docs.python.org/3/library/contextlib.html#contextlib.ExitStack) and have **one** context manager object which allows creating any number of tasks; something like:

```python
async with TaskLauncher() as tasks:
  tasks.create_task(background_task())

  # do the main work via a second task
  tasks.create_task(asyncio.sleep(5))

  # All tasks will be waited for on exit from the region
```

So long as we only spawn new tasks via `TaskLauncher`, we now have a clear parent-child relationship, where every task “belongs to” a clear parent, and where parents are responsible for waiting on exceptions that escape their children. If any task raises an unhandled exception, it will bubble upwards through this hierarchy; if no one catches it, it will eventually bottom out in the root task and the `asyncio.run` call. We’ve gone a long way towards converting the concurrent exception-handling problem into the familiar single-threaded version!

## Two problems

Unfortunately, we’re far from done. The above sketch has two serious, related, challenges, neither of which has a trivial fix.

#### Deadlocks

First, the deadlocks. We said above that a task’s parent will “eventually” wait on it. That’s only true so long as the parent actually exits the context manager. But what if the parent is **waiting on work** that will never complete, because some child task that would have processed it encountered an error?

Here’s a minimal example that shows the kind of problem:


    FIGURE: deadlock.py

This example is a stylized version of a common pattern: many “fan-out” or “fan-out / fan-in” concurrency patterns have the basic flavor of “launch some tasks; those tasks perform some work; the parent waits for the work.”

There are plenty of simple solutions to this specific bug[^wait]. However, we'd like to find a pattern that fixes it "automatically" or in some general fashion, or at least avoids having that easy trap just lying there waiting to bite us. In my experience, it's extraordinarily easy to run into these sorts of deadlocks, especially while first developing a concurrent system.

[^wait]: Perhaps the simplest fix is to remove the `asyncio.Event`s entirely, and instead `asyncio.gather` the `Task` objects ourselves. That’s easy here, but isn’t always as simple when we may not have a 1:1 mapping between “units of work” and “child tasks.”

#### Resource leaks

In single-threaded programs, the challenge of error recovery is not merely unwinding control flow, but of making sure we “undo” or “clean up” any work that was in-flight and somehow associated with the failed operation. We might need to free memory, or close a file handle, or restore some data structure to a consistent state.

In a concurrent program, the resources associated with the failed operation may include multiple different tasks of execution! If one task associated with our `TaskLauncher` fails, we need to ensure that **all** spawned tasks somehow get stopped or least have some opportunity to clean up abandoned state.

The simplest fix, in some sense, would be for `TaskLauncher` to wait for **all** spawned tasks before it exits. However, in practice, making that change alone makes our deadlock problem **drastically** worse.

## I hate to cancel, but...

If we want to both:
- Wait for all child tasks before returning, to ensure we know about any additional errors, and that we clean up any associated resources, and also
- Respond promptly to errors in any child task, without waiting a potentially-unbounded amount of time,

then I think we basically need a way to request that an arbitrary task exit early and promptly, in response to events which happen in a different task. In other words, we need a **cancellation** mechanism.

We've reached this conclusion in the light of the specific paradigm we're developing here, but I think it's much broader, and also fairly intuitive on reflection. In any concurrency paradigm, you will have **some** version of "multiple cooperating concurrent tasks," and that means that you need an answer to "what happens if one of them dies unexpectedly." And, in turn, it's hard for me to imagine a fully-general answer other than "we ask the other tasks to cancel and terminate early."

It's perfectly possible, mind, to implement specific concurrent programs or patterns without a general-purpose cancellation mechanism, by some a combination of ad-hoc mechanisms, and careful reasoning and construction. But I struggle to envision a general, composable, concurrent paradigm without one.

I find this conclusion unpleasant, because implementing and supporting cancellation is **hard**. It introduces an additional error path into virtually every piece of code, and one which is by its very nature asynchronous and hard to reason about or test. Historical attempts at cancelation mechanisms, like C's [pthread_cancel], Java's [Thread.stop], and Ruby's [Thread.terminate] are incredibly-subtle and error-prone at best, or [fundamentally unusable](https://jvns.ca/blog/2015/11/27/why-rubys-timeout-is-dangerous-and-thread-dot-raise-is-terrifying/) at best.

[pthread_cancel]: https://man7.org/linux/man-pages/man3/pthread_cancel.3.html
[Thread.stop]: https://docs.oracle.com/javase/8/docs/technotes/guides/concurrency/threadPrimitiveDeprecation.html
[Thread.terminate]: https://ruby-doc.org/3.4.1/Thread.html#method-i-terminate

In the context of concurrency, we do have some advantages, at least. If we're already writing concurrent code, a cancellation mechanism adds one **more** "thing which may happen asynchronously," but we at least we already have some version of the problem. In cooperative concurrency systems like `asyncio`, we can also limit cancellation to occur only at `await` points, which reduces the scope for potential chaos. Go, meanwhile, encodes cancellation into the [Context][context] object and requires code to check for cancellation explicitly, with a different set of tradeoffs.

[context]: https://pkg.go.dev/context

More generally, we have learned a lot as a field in the past decades, and some of these newer systems do seem to have general-purpose cancellation mechanisms that are more-or-less workable in practice. This post isn't intended to be a deep dive into the challenges and design space for cancellation mechanisms, though, so for now I'll assume we do have some sort of cancellation API[^asyncio-cancel], and move on.

[^asyncio-cancel]: And indeed, I'll note that Python's `asyncio` does have a [pervasive cancellation mechanism][asyncio-cancel-api].

[asyncio-cancel-api]: https://docs.python.org/3/library/asyncio-task.html#task-cancellation

### A trees of tasks with cancellation

If we do have the ability to cancel tasks, we can use it alongside our "tree of tasks" idea to produce a fairly general solution to concurrent error handling:

- If any task launched by a `TaskLauncher` raises an exception (including the parent task itself -- the one running the context manager), we cancel every other task (both the child tasks, and the parent task itself).
- We give child tasks a mechanism to detect cancellation and to clean up their own resources. Typically this means re-using our normal error-handling mechanism in some form; e.g. cancellation might raise a `CancelledError` exception, which tasks can cancel and re-raise, or they can use a `finally` block or context manager.
- On exit from the `TaskLauncher` context, we wait for **all** child tasks to exit, be that successfully, with an unhandled exception, or in response to a cancellation.
- Then, if any child task raised an error, we re-raise it into the parent task.

With this mechanism, concurrent errors now behave fairly similarly to single-threaded ones. They will be caught and propagated upwards if not handled. They can be caught and handled (including errors that originate from a child task) using our usual error-handling mechanisms. So long as we write code that cleans up after an error in the usual single-threaded way, we should also (for the most part, at least) get appropriate cleanup in the presence of multiple tasks.

In exchange, we do impose a requirement of additional structure on our concurrent code: We must nest tasks into a parent/child hierarchy, and we must make sure their lifetimes nest appropriately.

# Structured Concurrency

This is the part where I admit that none of this is a new idea, and none of it is my invention (although I haven't seen another writeup approaching the idea in this form). This paradigm, of a tree of tasks with nested lifetimes, and automatic cancellation up and down the tree, has been slowly but steadily gaining popularity and adoption in recent years under the name ["Structured concurrency"][structured].


Many of the ideas and components have a long lineage, but the idea was [first named][sc-named], as far as I know, in 2016, and probably best-popularized by the [trio][trio] framework, along with [an in-depth essay exploring the idea][njs-essay] by Nathaniel J Smith, Trio's creator and lead author.

[sc-named]: https://sustrik.github.io/250bpm/blog:71/

[structured]: https://en.wikipedia.org/wiki/Structured_concurrency
[njs-essay]: https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/
[trio]: https://trio.readthedocs.io/en/stable/index.html

As of Python 3.11, Python's own `asyncio` includes a [`TaskGroup`][taskgroup] class, which is essentially a (production-ready) version of the `TaskLauncher` sketch I illustrated above; `trio` calls their own version a ["nursery"][nursery]. In Go, [errgroup][errgroup] package provides essentially the same semantics, building on Go's [context package][context] which provides cancellation support.

Structured concurrency has many advantages, and I and many others have found that writing programs in this style makes it **much** easier to write correct and safe concurrent code (although other challenges certainly remain!). I strongly recommend njs' [classic essay][njs-essay], which I also linked previously, for a more-thorough exploration of the paradigm and its advantages.

[nursery]: https://trio.readthedocs.io/en/stable/reference-core.html#nurseries-and-spawning
[taskgroup]: https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup
[errgroup]: https://pkg.go.dev/golang.org/x/sync/errgroup
