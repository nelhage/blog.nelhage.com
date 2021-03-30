---
title: "Some opinionated thoughts on SQL databases"
slug: some-opinionated-sql-takes
date: 2021-03-30T10:32:31-07:00
---
People who work with me tend to realize that I have Opinions about databases, and SQL databases in particular. Last week, I [wrote about a Postgres debugging story](https://buttondown.email/nelhage/archive/22ab771c-25b4-4cd9-b316-31a86f737acc) and tweeted about [AWS’ policy ban on internal use of SQL databases](https://twitter.com/nelhage/status/1374518454921826304), and had occasion to discuss and debate some of those feelings on Twitter; this article is an attempt to write up more of them into a single place I can refer to.

I believe most of the commentary here applies to most SQL engines, but the focus of this post is going to be mostly on MySQL and Postgres, and occasional SQLite, since those are the engines I have familiarity with. Between them they also cover the vast majority of open-source SQL engine usage.

These opinions are derived from years of using and/or running SQL and other databases primarily as backends for web apps or web APIs. My experience is largely with these kinds of applications, which tend to have relatively predictable data access patterns, and care about low latency, high throughput, and some degree of high availability. I acknowledge that this is not the only purpose that SQL engines are used for, but it’s the one I have experience with and it’s a very common one these days.

# SQL Databases have incredible storage engines

The storage engine in a SQL database is the layer responsible for actually managing on-disk data, persisting it to disk and reading it back. They underpin all other functionality in the database. In a (non-distributed) SQL database, they also play a leading role in the transactional functionality of the database, managing most of the complexity behind atomic commit and mediating much of a database’s consistency and isolation properties. When we speak of “ACID,” those features are largely properties of, or at least rooted in properties of, the storage engine.

The open-source SQL database have absolutely some of the best storage engines in the world. They offer — with appropriate tuning and care —  excellent throughput and high utilization of the underlying hardware, while offering strong durability guarantees and transactional semantics. They have different strengths and weaknesses, but if you want to store some low-level records on a disk durably, it is hard to do substantially better, in a general-purpose way, than MySQL’s InnoDB, PostgreSQL, or SQLite’s storage layer[^sqlite].

These storage engines, in fact, are so good that they are a reason to use these databases, just by themselves. It can make sense to use a SQL database even with a trivial `(id int primary key, data blob)` key-value schema, just to get access to the performance and durability of the underlying storage layer.

[^sqlite]: You may be surprised to see SQLite on this list, but its storage engine really is an incredible piece of engineering. Among other evidence, [FoundationDB](https://www.foundationdb.org/), a modern, distributed, ACID database engine, actually uses a fork of SQLite’s storage engine as its local storage on each node.

# I dislike SQL as an API

Unfortunately, in my opinion, those incredible storage engines — some of the finest pieces of systems engineering available to us — are hidden behind what I find to be a very frustrating interface. Let’s look at what I dislike about SQL.


## String concatenation is a bad API

Even if you are sure to always use query parameters for actual data (and you should be), writing code against a SQL database almost always ends up constructing query strings by pasting strings together at some point, potentially deep within the bowels of an ORM. At best, this is awkward and cumbersome, and at worse it is error-prone. Furthermore, for simple queries like indexed point lookups, the overhead of constructing and parsing these strings can actually be a meaningful performance cost.

For programmatic use by an application, I would much rather an API that is structured data all the way down, such as a GRPC schema or even MongoDB’s BSON. It’s just much easier to work with programmatically to consume or produce or analyze, and eliminates whole classes of potential bugs.


## I dislike query planners

I can’t deny that query planners are very *useful*, and that there is something magical about the dream of declarative query execution, where you just write down a predicate, and the engine ✨ magically✨ makes it efficient.

However, for application development, the vast majority of the time, you *know* what your data access patterns are, and you have designed indexes around them, and value predictability. For an online application with consistent data access patterns and high throughput, performance is *part of the database’s interface*; if a database continues serving queries but with substantially worse latency, that’s an incident!

The query planner is the antithesis of predictability; *most* of the time it will choose right, but it’s hard to know when it won’t or what the impact will be if it doesn’t. Query planners change behavior based on estimates of data distribution, so even running `EXPLAIN` at CI time does not suffice to provide guarantees. SQL just engines make it really hard to guarantee predictable performance in query excecution.

Postgres, in particular, stubbornly refuses to have any pragma for forcing selection of an index at all, which infuriates me. It’s not a question of whether or not their planner is smart enough (although I have run into cases where it makes bad mistakes!), but it’s about the mindset of development and data management; At some point, transparency, explicitness, and predictability are really important values.

Query planners also make performance subject to weird-action-at-a-distance; very similar queries will execute very differently depending on what indexes exist and which ones the planner chooses, and it’s very hard to predict. At scale, developers often end up building some kind of application-level enforcement that all queries are well-indexed; I would much rather have at least the option to flat-out disable the query planner and require all queries have a `HINT`, or even better have an (optional) explicit index-based query engine, where you specify which index to walk, index bounds, and optional additional predicate to evaluate. This would make performance much more obvious from reading code.

The query planner problem gets worse in many ways as your queries get more complicated; I’ve seen instances of semantically-identical queries getting executed in vastly different ways when rewritten to use or avoid JOINs, subqueries, or CTEs, which means that even if you know a query *can* be executed efficiently, you often end up trying somewhat at random to rewrite it in between forms to find the one where the database is able to figure it out. I’ve also run into cases where I *know* a query could be efficiently executed over the raw B-Trees of the underlying index, but the query engine is missing some key feature meaning it can’t formulate the required plan.


## I don’t like the type system

The SQL type system is from another era and it shows. It can’t decide if it’s hyper-focused on how to lay out records in memory (c.f. `CHAR(8)`) or on being a rich expressive type system to let you push computation into the database (c.f datetime types, and the million and one postgres extensions) or on being as absolutely minimal as possible (c.f. SQLite). This just causes a lot of awkward confusion and mismatch at the application layer. I’d much rather something like protobuf’s type system which is pretty-minimally focused on describing the on-disk representation, and lets users define structs to layer semantics on top. In general I think a stronger separation between “storage type” and “semantic type” might be a productive step here.

Also, there are just so many rough edges and weird corners, especially between engines; Why does Postgres have `bytea` but everyone else has `blob`? Why is a `bigint` a 64-bit integer in most engines, even though that word means “arbitrary-precision integer” in most programming languages?

These aren’t *huge* problems, but they’re unforced errors in terms of creating mental overhead and impedance mismatch between the engine and the application.


## SQL is a decent ad-hoc query and reporting language

With all that said to the bad about SQL, SQL is a pretty decent ad-hoc query language or exploratory tool. If I have a large dataset I want to explore interactively, my first move is typically to import it into SQLite to play with interactively, since it’s so easy to ask all kinds of different questions about it in a fairly concise and expressive fashion.

There are plenty of nits to pick about SQL for interactive use, but in net this is where it really shines as a language, in my opinion.

This advantage also carries over, to some degree, to exploratory *development*; if you are building our an early-stage application where you are iterating rapidly, the ability to use SQL and to use the query planner to just write whatever queries you need at the moment and then make them efficient later — if there even is a later, for that feature — is genuinely valuable.

# Migrations are far more painful than they need to be

Honestly, migrations might be my biggest pet peeve with SQL databases. They’re just way harder and riskier than they need to be. Let’s look at two areas where they bug me.

### Migrations are too imperative
SQL databases have imperative schema definitions (`CREATE TABLE …` and then `ALTER TABLE …` or `CREATE INDEX…`). It feels clear to me that the vast majority of schema definition should be *declarative*: You just tell the database what the schema should be, it introspects the current schema, and then it evolves it to the new schema. If it can’t do so automatically, it raises an error, and you need to add additional pragmas or metadata to your schema to tell it what to do.

This is the approach taken, in some fashion, by most ORMs, which is part of why it feels so clearly right to me. It frustrates me that SQL databases don’t work this way natively, and that all of us are left reinventing the wheel over and over again, or relying on heavy-weight ORM and application frameworks just to make our databases usable. SQL databases have so damn much complexity already, and are so damn proud of their declarative query language; why isn’t data definition — which is a *much* better fit, in my opinion, for a declarative model — also declarative?

This problem is reasonably well-solved by many ORMs and frameworks, but it’s a real source of annoying design choices and boilerplate when starting a new project from scratch. I think the startup costs here, as well as the pain to iteration — it feels so heavyweight to write and run a migration just to add a new field to an object! — is a real reason for the popularity of “document” databases which just don’t have this problem. I think one major factor here is that SQL engines feel like a relic of an era that did more “big design up front”; if you do most of your design before implementing a system, it’s fine to have a fairly heavy-weight migration process. But these days, we emphasize tight iteration and evolution, and it feels infuriating to me to have to write and run a migration just to effectively add a new field to a `struct`, even if I don’t want to index or query on it!

### Zero-downtime migrations are way too fraught
It is *incredibly* subtle and non-obvious in SQL engines to figure out which migrations — in particular, which `ALTER TABLE`  or `CREATE INDEX` commands — can safely be executed concurrently with other data access and mutation.

This tend to be for two reasons. First, many migration commands, by default, lock the table for access while they rewrite every row on disk, which is a sure-fire recipe for downtime on a large table. Database engines typically document which operations these are, but don’t always have a way to *enforce* “If this operation would stop the world, abort instead of attempting it.” This means you need to use additional tooling, or just rely on expertise and manually cross-referencing between migrations and documentation, in conjunction with testing, to make sure you don’t cause an incident just because you added a new field to a struct.

MySQL has an `ALGORITHM=INSTANT` pragma on `ALTER TABLE`, which is intended to provide an automated safety net, but it turns out that under the right circumstances, [even that option doesn’t suffice](https://blog.nelhage.com/post/rwlock-contention/#sql) to avoid locking-related outages! You can tune lock timeouts and various limits to get more assistance from the database, but it will inevitably take you a few tries to find all the new ways your database can accidentally DoS itself.

There are [numerous](https://www.braintreepayments.com/blog/safe-operations-for-high-volume-postgresql/) [writeups](https://gocardless.com/blog/zero-downtime-postgres-migrations-the-hard-parts/) [and tools](https://gocardless.com/blog/zero-downtime-postgres-migrations-a-little-help/) [to help](https://benchling.engineering/move-fast-and-migrate-things-how-we-automated-migrations-in-postgres-d60aba0fc3d4) by encoding the innumerable piles of lore and techniques that have been learned with blood, sweat, and outages by operators over the years, and it’s infuriating to me that the databases themselves cannot better address this pain point themselves.

The second class of incident is migrations that avoid heavy locking, but do large amounts of background I/O to the table. A classic example is index creation (with `CREATE INDEX CONCURRENTLY`, in Postgres). On a busy system, these operations can easily thrash disk bandwidth and/or cache badly enough to cause incidents (once again: for an OLTP database, performance *is* part of the interface contract). SQL databases provide weak tools for observing and throttling or controlling this additional I/O. At a certain scale, operators switch to creating indexes by building indexes one replica at a time and failing over the primary write replica, but this isn’t something there is great out-of-the-box tooling for.

# SQL databases have Too Damn Many features

SQL databases have become enormously complex and customizable, supporting many, many, more features than the core feature set of “tables with rows of data you can query and update.” These include exotic data types, language extensions and indexing strategies, things like user-visible locking mechanisms or storage options like clustered or partitioned tables, and various options to push logic into the database like triggers and stored procedures.

I’ve been known to jokingly comment that I hate all features, and I do have a general tendency to be vary wary of unnecessary features and added complexity. But, of course, every feature is added because it has a constituency, and the line between “useful” and “unnecessary” complexity is always debatable.

My biggest problem with the feature surface of SQL databases is that it is *incredibly* hard to tell what the operational implications of any feature is. In my experience, every SQL database contains within it a *subset* of its features which are both:

- Performant by design; that is, they do not have any horrific performance implications if used with large data or under high concurrency or throughput
- Sufficiently mature and well-tested that they live up to their performance/reliability/durability goals under heavy usage.

However, it’s rarely documented *which* those features are! I [wrote earlier this week](https://buttondown.email/nelhage/archive/22ab771c-25b4-4cd9-b316-31a86f737acc) about a nasty performance cliff I encountered in Postgres; In response, a Postgres expert I talked to commented that Postgres subtransactions are “basically cursed” and essentially unfit for use at high concurrency. This sort of knowledge is widespread among experts but not explicitly documented or written down in any systematic way, making these tools incredibly fraught to actually use safely.

One of the problems here is that SQL engines attempt to be everything to everyone; they are used for online purposes, offline data warehouses, analytics, for [building entire applications](https://postgrest.org/en/v7.0.0/) within the database, and more. However, as a result, many of these features were only really intended for one family of use case, but it’s not well-documented or certainly enforced.

I would *love* if it a SQL engine shipped with a global pragma of some sort that disabled all of these “everyone knows they’re unsafe in a low-latency high-throughput environment” features and only allowed a small core of functionality that is known to be well-tested and relatively safe for use backing your webapp. It would, of course, be a painful judgment call by the developers to actually draw precise boundaries around such a feature set; but such a call would also be a valuable coordination point to standardize usage patterns by data-intensive users, and to redouble investment in the production-readiness of that core.

# My personal choice: MySQL

Whenever I rant about my pet peeves with SQL engines, someone always asks me what I would recommend instead. If you’re starting a webapp these days with relatively “vanilla” data modeling needs, what technology do I recommend using for data storage?

The devil is in the details, but for me, as a default, and despite all these pet peeves, I would start with MySQL.

This answer often surprises people, since MySQL has earned a bit of a bad rap for its rocky early days. These days, however, MySQL, with InnoDB[^rocks] as a storage engine is one of the most battle-tested and well-engineered pieces of software we have. It’s been run (and still is in some cases) at planetary scale by some of tech’s largest companies, including both Facebook and Google. They have, in turn, invested enormous amounts of engineering effort in hardening it for performance and for stability and operability at scale. These days, MySQL even has ready-made clustering features available essentially off-the-shelf via [Vitess](https://vitess.io/).

[^rocks]: Or, potentially, but only if you know your data patterns need its compression, write throughput, or other features, [MyRocks](http://myrocks.io/).

MySQL, to be clear, suffers from all of the complaints above. It still must be used with care, and if I were to develop an application against it, I would attempt to use as minimal a feature set as I comfortably could. But at the end of the day, its sheer maturity and flexibility would win me over for most purposes, over any of the newer or trendier NoSQL engines.

As for Postgres, I have enormous respect for it and its engineering and capabilities, but, for me, it’s just too damn operationally scary. In my experience it’s *much* worse than MySQL for operational footguns and performance cliffs, where using it slightly wrong can utterly tank your performance or availability. In addition, because MySQL is, in my experience, more widely deployed, it’s easier to find and hire engineers with experience deploying and operating it. Postgres is a fine choice, *especially* if you already have expertise using it on your team, but I’ve personally been burned too many times.


# Closing thoughts

I have a real love/hate relationship with SQL databases. They are incredibly powerful tools, and when used well can drastically simplify architectures and help solve entire classes of consistency and durability problems. At the same time, every time I interact with one, I feel like the experience is one of a thousand avoidable papercuts, and that the experience could be so much better without losing almost any of their strengths. SQL as an API is in many ways a relic from another era, and while it’s held up remarkably well, it also feels like it shows its age. The operational problems also terrify and enrage me. Databases are always going to be challenging and sources of complexity and danger, but it feels like SQL engines barely even *try* to offer predictability performance or to build reliable guard rails against accidentally taking the entire site down.

My frustrations with SQL engines give me an optimistic lens on the “NoSQL” fad/movement/whatever you want to call it. Whether or not they are succeeding, in systems like MongoDB, [I see a real attempt to modernize](https://blog.nelhage.com/2015/11/what-mongodb-got-right/) how a database works, and to bring modern sensibilities — such as easy scale-out, rapid iteration of the application, 24/7 availability, “cattle, not pet” cloud deployment — to our databases. I see attempts to simplify systems to make them more understandable and predictable, and in order to offer better guarantees. I see efforts to design interfaces better-suited to our modern style of application development, where the app code is king, and is rapidly evolving and iterating, and where planned downtime for maintenance is no longer a norm.

“NoSQL” encompasses a wide range of visions and efforts and goals. But to me, personally it stands for an optimism that we can do *even better* than SQL engines; that, while powerful, these tools are in many ways relics of another era, and that we have different needs, new technologies, new sensibilities, and just a few decades of experience as an industry, and that we should continue exploring and experimenting and iterating, even if it inevitably means a few wrong steps along the way.
