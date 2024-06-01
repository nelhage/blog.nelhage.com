---
title: "Inventing structured concurrency through the lens of error-handling"
slug: inventing-structured-concurrency
date: 2024-06-01T14:21:20-07:00
---
How should we handle and propagate errors in concurrent programs?

In single-threaded programs, we’ve mostly converged on a “standard” pattern: Errors are passed up the stack until we find a stack frame that’s prepared to handle them; as we do so we unwind the stack in-order, giving each frame an opportunity to perform cleanup actions or destroy resources.

This pattern clearly describes the explicit exception-handling mechanisms in many modern languages (C++, Python, Java), all of which also have mechanisms for cleanup on frame unwind (RAII, `finally` blocks, Python context managers). But it also describes the standard pattern in Rust (`Result` returns, the `?` operator, and calling `drop` on wind), as well as Go (the classic `if err != nil { return err }` pattern and `defer` for cleanup), and even most new `C` code, via the `goto error` pattern (c.f. [examples in the Linux kernel](https://livegrep.com/search/linux?q=goto%20err&fold_case=auto&regex=false&context=true)).

On the flip side, from today’s vantage this pattern may seem obvious or overdetermined, but it wasn’t so historically. Lisp’s [“restarts” mechanism](https://lisper.in/restarts), `longjmp` in C[^longjmp], “trap” mechanisms like UNIX signals, and the infamous [`on error`](https://learn.microsoft.com/en-us/office/vba/language/reference/user-interface-help/on-error-statement) clause in Visual Basic illustrate some alternate mechanisms that have been experimented with in the past. "Unwinding" is also predicated on **having** a structured call stack, a concept which also had to be [discovered and popularized](https://en.wikipedia.org/wiki/Structured_programming).

[^longjmp]: `longjmp` can be used as a primitive to help **implement** exceptions and unwinding. But by itself it’s a much lower-level primitive and admits a wide variety of alternate patterns.

My question is: what is the equivalent pattern in the presence of concurrent threads of execution? I intend here to be inclusive of both traditional threads, but also other mechanism of concurrency, such as Go’s “goroutines” or Python’s `asyncio` or Javascript `async/await` — I’m interested in [concurrency more so than hardware parallelism](https://go.dev/blog/waza-talk). Going forward, I’ll use the word “task” to refer to any logical sequence of linear execution which may be interleaved in time with any number of other such sequences.

If an exception can be handled within a single tasks, our existing patterns mostly work fine. But what about errors that need to cross between tasks? What if, for instance, one task is waiting on a second task, and the second encounters an error? How should we organize our code to make it easier to write correct code? Is there a concurrent version of the “unwind-and-cleanup” pattern?


## Unhandled errors

We’ll explore the question of “what happens when errors need to cross task boundaries?” by examining what popular programming languages do to **unhandled exceptions** inside of **concurrent programs**. In a single-threaded program, it’s natural that exceptions which “bubble up” out of the entrypoint should terminate the program, but in a concurrent program, it’s less clear: There may be other tasks that are making progress, and we may not wish to terminate them.

For concreteness, let’s consider this simple program[^python]:


    FIGURE:exception_thread.py

[^python]: Most of this discussion is intended to apply broadly across many languages and concurrency frameworks, but I’ll use Python example for concreteness. In addition to other benefits, Python supports both threading and cooperative asynchrony, which lets me explore multiple paradigms.

This program attempts to run two concurrent threads, one doing some “main” work function, and one doing some kind of work “in the background.” However, the background thread raises an exception immediately. What should happen?

The main thread (or other threads, in general) might still be making progress, and we may not want to unceremoniously cut it short. But, at the same time, something unexpected has clearly happened (if it were anticipated, the programmer would have caught the exception); do we really want to continue execution in such an unanticipated state?

And indeed, there tend to be broadly two schools of behavior here in programming languages and runtimes:

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

Killing the entire program is a heavyweight hammer. I have been involved in high-severity incidents where the proximate cause was an unhandled panic in a low-priority goroutine that crashed a critical daemon. But letting the program continue means we’re in a state that was almost certainly never tested or anticipated, and runs a high risk of deadlock if another task was waiting on the now-dead task. It also violates our single-thread intuition of “unhandled exceptions crash the program” and leads to a lot of confusion while developing and debugging.


## Where do I raise the exception?

The problem, in some sense, is that when we unwind to the root of the thread’s stack, we find ourselves in a dead end. We might like to “continue unwinding,” but it’s not clear what that means — we’ve reached the root of the stack, where do we unwind to? To what code should we hand the exception?

Python’s [`asyncio`](https://docs.python.org/3/library/asyncio.html) framework tries to resolve the contradiction and takes a different stance here than any of the threading systems above. In `asyncio`, a task (represented by a [`Task`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task) object) is an object you can wait on. If a Task raises an exception, waiting on it will re-raise the exception, giving you a mechanism to manually specify “where the exception should be forwarded.” If we port our program to `asyncio`, and remember to add an `await task`, we get:


    FIGURE:exception_asyncio.py

In this version, the `await task` re-raises the exception, so we don’t get the “All done” print, and we exit with an error code like you might expect:


    $ time python exception_asyncio.py
    Traceback (most recent call last):
      File "/Users/nelhage/Sync/code/structured-concurrency/exception_asyncio.py", line 16, in <module>
        asyncio.run(main())
      File "/Users/nelhage/.pyenv/versions/3.11.0/lib/python3.11/asyncio/runners.py", line 190, in run
        return runner.run(main)
               ^^^^^^^^^^^^^^^^
      File "/Users/nelhage/.pyenv/versions/3.11.0/lib/python3.11/asyncio/runners.py", line 118, in run
        return self._loop.run_until_complete(task)
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
      File "/Users/nelhage/.pyenv/versions/3.11.0/lib/python3.11/asyncio/base_events.py", line 650, in run_until_complete
        return future.result()
               ^^^^^^^^^^^^^^^
      File "/Users/nelhage/Sync/code/structured-concurrency/exception_asyncio.py", line 13, in main
        await task
      File "/Users/nelhage/Sync/code/structured-concurrency/exception_asyncio.py", line 6, in background_task
        raise ValueError("oops")
    ValueError: oops
    python exception_asyncio.py  0.06s user 0.02s system 1% cpu 5.158 total

    $ echo $?
    1

We can even use [`asyncio.gather`](https://docs.python.org/3/library/asyncio-task.html#asyncio.gather) to short-circuit, and raise the error as soon as it happens:


    await asyncio.gather(asyncio.sleep(5), task)

However, this approach has a massive downside: It relies on someone waiting on the `Task`. If you don’t, the exception will be printed … but only on program completion. If the program subsequently deadlocks because it was expecting the dead task to make progress, you get no feedback whatsoever other than silent hang.

It’s an oversimplification, but the situation is bad enough that I sometime summarize it as  “by default, `asyncio` completely swallows exceptions off the main task.” That’s not literally true, but in my experience it is a decent mental model to start with, and well describes the experience of many developers new to `asyncio`.

## What if we always waited on tasks?

The `asyncio` approach is arguably an improvement **as long as you remember to wait on every** `**Task**` you spawn. That suggests an improvement: Can we modify the API to **force** the developer to wait on tasks? We might replace `asyncio.create_task(coro)` with an asynchronous context manager, and make you write:


    # (n.b. this is not real API in any version of Python)
    async with asyncio.create_task(background_task()) as task:
      # …
      # `task` will be waited for on exit from this block, and any exception raised

Of course, we very often want to create many tasks — or a dynamically-chosen number of tasks — so we could borrow an idiom from [`contextlib.ExitStack`](https://docs.python.org/3/library/contextlib.html#contextlib.ExitStack) and have **one** context manager object which allows creating any number of tasks:


    FIGURE:task_launcher.py

We can port our running example over:

    FIGURE:exception_tasklauncher.py

And we’ll see it exit quickly in response to the exception, with a sensible stack trace.

So long as we only spawn new tasks via `TaskLauncher`, we now have a clear parent-child relationship, where every task “belongs to” a clear parent, and where parents are responsible for waiting on exceptions that escape their children. If any task raises an unhandled exception, it will bubble upwards through this hierarchy; if no one catches it, it will eventually bottom out in the root task and the `asyncio.run` call. We’ve gone a long way towards converting the concurrent exception-handling problem into the familiar single-threaded version!


## Two problems

Unfortunately, we’re not done. The implementation above has two serious issues, neither of which has a trivial fix.

#### Deadlocks
First, the deadlocks. We said above that a task’s parent will “eventually” wait on it. That’s only true so long as the parent actually exits the context manager. But what if the parent is **waiting on work** that will never complete, because some child task that would have processed it encountered an error?

Here’s a minimal example that shows the kind of problem:


    FIGURE: deadlock.py

Patterns that resemble this example are fairly common: many “fan-out” or “fan-out / fan-in” concurrency patterns have the basic flavor of “launch some tasks; those tasks perform some work; the parent waits for the work.”

There are plenty of simple solutions to this specific bug[^wait]. But can we find a pattern or language feature that makes this bug impossible — or, at a minimum, easy to detect and reason about locally? In my experience these kinds of deadlocks are pervasive following an error in many concurrency architectures.

[^wait]: Perhaps the simplest fix is to remove the `asyncio.Event`s entirely, and instead `asyncio.gather` the `Task` objects ourselves. That’s easy here, but isn’t always as simple when we may not have a 1:1 mapping between “units of work” and “child tasks.”

#### Resource leaks
In single-threaded programs, the primary challenge of error recovery is not merely unwinding control flow, but of making sure we “undo” or “clean up” any work that was in-flight and associated with the failed operation somehow. We might need to free memory, or close a file handle, or restore some data structure to a consistent state.

As written above, our `TaskLauncher` re-raises the **first** exception it sees immediately, and any other tasks are left to live indefinitely. With the parent task unwound, those tasks are likely to be orphaned and leaked forever, along with any resources they may have allocated. Moreover, it’s not entirely clear how we would change that behavior, at least in any general way.

In some sense, this issue arises because I chose to write `return_when=FIRST_EXCEPTION` in the `__exit__` implementation earlier. I did that to surface exceptions sooner -- it seems intuitive that if an unhandled exception has happened in any child, we want to find out promptly, rather than "at some arbitrary later point." If we replaced `FIRST_EXCEPTION` with `ALL_COMPLETED` we would instead wait for **all** childen to complete (success or failure). That would have the desirable property of strengthening our "tree-of-tasks" paradigm: no child ever outlives its parent. Howeve, in practice, making that change alone would make our deadlock problem **drastically* worse. Now, **any** child task can cause us to hang, and in general there's no reason to expect random tasks to exit promptly.

## Is this cancel culture?

If we want to **both** wait for all child tasks, **and** respond to errors reasonably promptly, we need a way to **force** the other tasks to exit promptly in response to an exception in any one of them. In other words, we need a **cancellation** mechanism.

More generally, I think I've become convinced of the following conclusion:

> Any general-purpose solution to ergonomic and reliable error handling in concurrent programs **requires** a robust cancellation mechanism.

We've considered the problem a bit in light of the specific paradigm I'm exploring in this post, but I suspect it's a more-general phenomenon. If you're running multiple concurrent tasks that are interrelated in some way, and one of them crashes, in general you probably need a way to make sure the others stop, too. The "general-purpose" caveat is important in the above claim; there are plenty of specific patterns that use some sort of ad-hoc mechanism to ensure termination even in the presence of errors, but I'm not sure I've "general patterns" that don't require one.

I find this conclusion unpleasant, because implementing and supporting cancellation is **hard**. Most attempts, historically, have been disasters in some way or another, and are sometimes even rolled back entirely. TKTK talk about Java and Ruby cancellation, and/or UNIX signals.

The problem, at least, is a bit easier in cooperative concurrency systems like `asyncio`, where we can limit cancellation to occur only at `await` points. More broadly, we have learned a lot as a field in the past decades, and some newer systems do seem to have general-purpose cancellation mechanisms that are more-or-less workable in practice. However, my goal with this post isn't to explore the challenges and choices in implementing cancellation, so for now we'll posit that we **can** cancel tasks (and I'll note that Python's `asyncio` does have a pervasive cancellation mechanism), and move on.

### A trees of tasks with cancellation

If we **do** have the ability to cancel tasks, we can use it alongside our "tree of tasks" idea to produce a fairly-general solution to concurrent error handling:

- If any task launched by a `TaskLauncher` raises an exception (including the parent task itself -- the one running the context manager), we cancel every task associated with the `Launcher` (again, including the parent).
- We give child tasks a mechanism to detect cancellation and to clean up their own resources. Typically this means re-using our normal error-handling mechanism in some form, by (e.g.) making cancellation raise a `TaskCancelled` exception which can be caught and re-raised, and which executes `finally` blocks.
- On exit from the context manager, we wait for **all** child tasks to exit, be that successfully, with an unhandled exception, or in response to a cancellation.
- Then, if any child task raised an error, we re-raise it into the parent task.

With this mechanism, concurrent errors now behave fairly similarly to single-threaded ones. They will be caught and propagated upwards if not handled. They can be caught and handled (including errors that originate from a child task) using our usual error-handling mechanisms. So long as we write code that cleans up after an error in the usual single-threaded way, we should also (for the most part, at least) get appropriate cleanup in the presence of multiple tasks.

In exchange, the primary new requirement on code is only that we nest tasks into a neat parent/child hierarchy, making sure to make use of an appropriate abstraction to create new tasks.

## `TaskGroup` and `trio.Nursery`: Structured Concurrency

For those of you who haven't already encountered the idea, here is where I admit that this paradigm is not at all my idea. It is one that already exists in several programming languages and frameworks, and has been slowly but steadily gaining popularity and adoption in recent years under the name ["Structured concurrency"][structured]. Some of the ideas and components have a long lineage, but the idea was first described and popularized as a whole in the [trio][trio] framework, and set forth in depth in an [essay naming and describing][njs-essay] the paradigm by Nathaniel J Smith, Trio's creator and lead author.

[structured]: https://en.wikipedia.org/wiki/Structured_concurrency
[njs-essay]: https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/
[trio]: https://trio.readthedocs.io/en/stable/index.html

As of Python 3.11, Python's own `asyncio` includes a [`TaskGroup`][taskgroup] class, which is their (production-ready) version of the `TaskLauncher` sketch I illustrated above; `trio` calls their own version a ["nursery"][nursery]. In Go, [errgroup][errgroup] package provides essentially the same semantics, building on Go's [context package][context] which provides cancellation support.

Structured concurrency has many advantages, and I and many others have found that writing programs in this style makes it **much** easier to write correct and safe concurrent code (although other challenges certainly remain!). I strongly recommend njs' [classic essay][structured], which I also linked previously, for a thorough exploration of the paradigm and its advantages.

[nursery]: http://TKTK-nursery
[taskgroup]: https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup
[errgroup]: http://TKTK-errgroup
[context]: http://TKTK-context


# Notes
-
- https://vorpus.org/blog/some-thoughts-on-asynchronous-api-design-in-a-post-asyncawait-world/
-
