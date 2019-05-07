---
title: "Reader/Reader blocking in reader/writer locks"
slug: rwlock-contention
date: 2019-05-07T08:00:00-07:00
---
# tl;dr

In writer-priority reader/writer locks, as soon as a single writer enters the acquisition queue, all future accesses block behind any in-flight reads. If any readers hold the lock for extended periods of time, this can lead to extreme pauses and loss of throughput given even a very small number of writers.

# Abstract

This post describes a phenomenon that can occur in systems built on [reader/writer locks](https://en.wikipedia.org/wiki/Readers–writer_lock), where slow readers and a small number of writers (e.g. even a single writer) can lead to substantial latency spikes for the entire system. This phenomenon is well-known in certain systems engineering communities (e.g. among some kernel or database developers), but is often surprising when first encountered, and has important implications for the design of such systems.


# Background

[Reader/writer locks](https://en.wikipedia.org/wiki/Readers–writer_lock) (sometime styled `rwlocks`) are a common concurrency primitive that aim to provide higher concurrency than a traditional mutex by allowing multiple readers to proceed in parallel.

A reader/writer lock can be acquired, as the name suggests, either for *read* or *write* access. Any number of threads may hold it for read access concurrently, but if a writer holds the lock, no other thread may concurrently hold it for either read or write access.

In order to avoid starving writers, most reader/writer lock implementations are *writer priority.* This means that once a writer starts waiting on the lock, no future reader may acquire the lock until after the writer has acquired and dropped the lock. This behavior is necessary in order to prevent writer starvation; Otherwise, under contention, readers may continually overlap their sessions, and never leave a moment at which a writer can acquire an exclusive lock.


# The problem

This writer priority behavior, however, essentially creates the possibility for *readers to block on other readers*, something that a reader/writer lock is supposed to avoid. Consider a situation where, in this order:

- At time T₁, a long-running reader R₁ acquires a read lock
- At time T₂, a writer W attempts to acquire a writer lock
- At time T₃, a reader R₂ attempts to acquire a read lock
- At time T₄, R₁ drops the read lock

Because the writer W has priority, R₂ will be blocked until W can acquire and then release the lock. However, W is blocked until R₁ releases its read lock at T₄. Thus, between times T₃ and T₄, R₂ is blocked, effectively waiting on R₁ to complete (and then W after it).

If readers ever hold locks for extended periods of time, such that T₄ is much later than T₁, this can lead to disastrous system pauses: a very small amount of write load may be sufficient to allow long-running readers to halt all other threads until they complete.


## Exacerbating factors

This description may seem like a niche problem: It requires a confluence of a few different factors in order to occur. However, nature of reader/writer locks and the situations in which they’re used actually makes it much more common than one might expect.

First, note that any system designed to use a reader/writer lock by hypothesis is likely to see a high rate of read volume and less frequent write traffic, because that’s exactly the kind of situation where a reader/writer lock provides value over a vanilla mutex. So reader/writer locks tend to *rely* on the additional concurrency, and on readers not blocking other readers.

Furthermore, absent contention with writers, long-lived readers have no impact on the throughput or latency of lock acquisitions, because other readers can peacefully coexist with them. As a general matter, software has a constant tendency to get slower, absent deliberate and careful efforts to hold the line, as developers add features and complexity. And if slower processes don’t immediately impact performance (because they only hold read locks), they’re more likely to go unnoticed. Thus, it’s not unlikely that over time read lock durations will creep upwards, *mostly* without effect, until one happens to coincide with an attempt to grab a write lock.

# Examples

Here’s a few examples of real systems in which I’ve observed this problem for real:

## Linux kernel `mmap_sem`

The Linux kernel uses [a reader/writer lock](https://github.com/torvalds/linux/blob/v5.0/include/linux/mm_types.h#L394) to protect the structures controlling address space layout within a process (and shared between threads). It is in general a well-known source of contention, but I’ve recently [debugged and documented](https://nelhagedebugsshit.tumblr.com/post/140317144518/when-observing-your-database-stalls-it) a case in which unexpected long-lived readers using this lock can result in egregious latency spikes for other readers.

As an interesting aside, I looked up the documentation for the kernel’s reader/writer locks, and discovered [this advice](https://github.com/torvalds/linux/blob/v5.0/Documentation/kernel-hacking/locking.rst#readwrite-lock-variants):

> If your code divides neatly along reader/writer lines, and the lock is held by readers for significant lengths of time, using [reader/writer] locks can help.

Based on the phenomenon described in this post, I believe this evidence to be overly simplistic at best, and actively harmful at worst.

## MongoDB MMAPv1 storage engine

MongoDB’s classic storage engine (known as “mmapv1”) traditionally used a single reader/writer lock to guard access to each database. By design, the lock should never be held for extended periods of time, and readers are supposed to periodically release (“yield”) the lock to allow other processes to make progress.

However, readers holding the lock for too long was and is a recurring source of performance-limiting concurrency problems; here’s [just one example](https://jira.mongodb.org/browse/SERVER-33527?focusedCommentId=1829093#comment-1829093) from their issue tracker describing exactly this issue, and I’ve debugged many others in the course of operating a large MongoDB deployment.

## SQL

SQL databases, even ones with powerful MVCC concurrency engines, tend to use reader/writer locks in the implementation, and it’s not hard to unintentionally lock entire tables against all access for extended periods of time. Let’s look at an example that works on (at least) both PostgreSQL and MySQL:

We’ll create a small table with some data:

    CREATE TABLE rwlock (n integer);
    INSERT INTO rwlock VALUES (1), (2), (3);

Then, in one client, we’ll simulate a slow read-only analytical query:

    SELECT pg_sleep(60) from rwlock; -- PostgreSQL
    SELECT sleep(60) from rwlock; -- MySQL

While that runs, we execute a schema migration:

    ALTER TABLE rwlock ADD COLUMN i integer;

This command hangs, waiting for the `SELECT` to complete. To me this behavior seems mildly disappointing, but also not that surprising. However, if we then attempt to execute a read that should be fast:

    SELECT * from rwlock;

We find that that this read *also* hangs until the initial query completes!

This is a surprising result to me: Running a slow analytical query alongside an online query is something I understood to be safe, and I also understood running an `ADD COLUMN` to be a safe operation to perform online. However, combining them lets us accidentally completely lock our database!

MySQL’s `ALTER TABLE` supports the [option to specify `ALGORITHM=INSTANT`](https://dev.mysql.com/doc/refman/8.0/en/alter-table.html#alter-table-performance), documented as


> Operations only modify metadata in the data dictionary. No exclusive metadata locks are taken on the table during preparation and execution, and table data is unaffected, making operations instantaneous.

But testing reveals that even that option is not sufficient to prevent the potentially-unbounded blocking behavior described here.

I am, of course, far from the first to note this behavior: Among others, GoCardless [wrote a good postmortem](https://gocardless.com/blog/zero-downtime-postgres-migrations-the-hard-parts/) describing a run-in with exactly this problem.

# What to do about it?

So, we’ve seen that reader/writer locks can have surprising behavior, leading to high latency and leading to readers effectively holding exclusive locks. What can we do about it?

At a high level, my experiences and this exploration have made me much more wary of designs that rely on reader/writer locks. Because of the behavior explored here, their additional concurrency, can effectively becomes a false promise, but one that works just well for systems designers and users to come to rely on it. That’s in many ways the worst possible outcome, setting systems up for unexpected failures when that concurrency vanishes.

With that in mind, here’s some recommendations. None of these is appropriate for every situation, but hopefully together they provide some helpful starting points.


- Use finer-grained concurrency and normal exclusive locks. A vanilla exclusive lock offers lower concurrency, but does so in a more consistent way — contention is *always* bad, as opposed to sporadically bad, and so performance is more predictable.
    - As a concrete example, the [RadixVM research paper](https://people.csail.mit.edu/nickolai/papers/clements-radixvm-2014-08-05.pdf) replaces the `mmap_sem` with a radix tree; nodes in the tree are locked using a trivial spinlock, but by design contention is extremely rare, and so the system as a whole achieves greatly improved concurrency.
- Various concurrency schemes can provide the guarantee that readers *never* block, offering an even stronger guarantee of reader concurrency than reader/writer locks:
    - [read-copy-update](https://www.kernel.org/doc/Documentation/RCU/whatisRCU.txt) is a technique used in the Linux kernel which allows for completely synchronization-free reads.
    - Various schemes using immutable data structures and atomic pointer updates can provide similar properties in other environments.
    - Some optimistic concurrency schemes, like [Michael-Scott queues](https://www.cs.rochester.edu/u/scott/papers/1996_PODC_queues.pdf), offer progress guarantees even in the face of contention.
- Time out writer acquisitions.
    - If potentially starving writers is acceptable, adding a timeout to write-side acquisitions will bound how long readers can be forced to pause. GoCardless’ [library for online Rails migrations](https://gocardless.com/blog/zero-downtime-postgres-migrations-a-little-help/) implements this option for Postgres, and also discusses the challenges and implications a bit.
- If you do have to use a reader/writer lock, think carefully about how long a reader or writer can hold the lock, and instrument the system so that long-lived critical sections are observable *before* they become problematic.

# Conclusion

A number of people I’ve talked to have described this phenomenon as unsurprising, and I think it is when explained explicitly. But the number of times I’ve run into performance problems related to long-lived readers in reader/writer locks makes me believe that it’s not necessarily obvious *a priori*, or perhaps not obvious in the right ways. Dealing with these problems and writing this post has made me much more cautious about ever reaching for an rwlock as a solution to a problem, and I hope it helps raise awareness of this problem and potential performance pitfall.
