---
layout: post
title: "What MongoDB got Right"
date: 2015-11-10 10:00:00 -0800
comments: true
categories:
---

[MongoDB][mongo] is perhaps the
[most][webscale]-[widely][broken-by-design]-[mocked][mongodbfacts]
piece of software out there right now.

While some of the mockery is out-of-date or rooted in
misunderstandings, much of it is well-deserved, and it's difficult to
disagree that much of MongoDB's engineering is incredibly simplistic,
inefficient, and immature compared to more-established databases like
PostgreSQL or MySQL.

You [can argue][genius], and I would largely agree, that this is
actually part of MongoDB's brilliant marketing strategy, of
sacrificing engineering quality in order to get to market faster and
build a hype machine, with the idea that the engineering
[will follow later][wiredtiger].

In this post, however, I want to focus on something different: I want
to explore some design decisions that I believe MongoDB got *right*,
especially compared to SQL databases, its main competitors[^sql].

Implementations evolve and improve with time, but interfaces last
nearly forever. In each of the cases I explore, I contend that MongoDB
legitimately out-innovated SQL databases, potentially positioning it
-- in a few years, once the implementation matures -- as a superior
database for a wide range of use cases.

This post is not an exhaustive enumeration: In the interests of space,
I've chosen a few areas to focus on, in which I think MongoDB has
really hit on some good ideas. There are a lot of other differences
between MongoDB and a SQL RDBMS; This post shouldn't be construed as
either a defense of or a criticism of any individual feature or design
choice not listed here.

[mongo]: https://www.mongodb.org/
[webscale]: http://www.mongodb-is-web-scale.com/
[broken-by-design]: http://hackingdistributed.com/2013/01/29/mongo-ft/
[random-log]: http://stackoverflow.com/questions/16833100/why-does-the-mongodb-java-driver-use-a-random-number-generator-in-a-conditional
[mongodbfacts]: https://twitter.com/mongodbfacts
[genius]: http://nyeggen.com/post/2013-10-18-the-genius-and-folly-of-mongodb/
[wiredtiger]: https://www.mongodb.com/press/wired-tiger

## Structured Operation Format

Let's start with the simplest one. Making the developer interface to
the database a structured format instead of a textual query language
was a clear win.

SQL is a great language for ad-hoc exploration of data or for building
aggregates over data sets. It's an absolutely miserable language for
programmatically accessing data.

Querying SQL data involves constructing strings, and then -- if you
want to avoid SQL injection -- successfully lining up placeholders
between your pile of strings and your programming language. If you
want to insert data, you'll probably end up constructing the string
`(?,?,?,?,?,?,?,?)` and counting arguments very carefully.

The situation is sufficiently untenable that it's rare to write apps
without making use of an ORM or some similar library, creating
additional cognitive overhead to using a SQL database, and creating
painful [impedance mismatches][impedance].

By using a structured (BSON) interface, MongoDB makes the experience
of programmatically interacting with the database much simpler and
less error-prone[^injection]. In a world where databases are more
frequently used as backend implementation details owned by an
application -- and thus accessed entirely programmatically -- this is
a clear win.

BSON probably wasn't the best choice, and MongoDB's implementation
isn't perfect. But it's a fundamentally better approach for how we use
databases today than SQL is.

[impedance]: https://en.wikipedia.org/wiki/Object-relational_impedance_mismatch

## Replica Sets

Now we'll turn to the more operational side of MongoDB. MongoDB, out
of the box, includes support for ["replica sets"][replset], MongoDB's
replication+failover solution. A MongoDB Replica Set consists of a
"primary", which services all writes, and a group of "secondaries",
which asynchronously replicate from the primary. In the event of a
failure of the primary, a [raft][raft]-like consensus algorithm
automatically elects and promotes a new primary node.

Nearly anyone who uses SQL databases in production ends up configuring
read replicas, in order to distribute read load, and provide a hot
spare in the event of the loss of the primary. However, with both
MySQL and PostgreSQL, configuring replication remains complex and
error-prone, both in terms of the breadth of options available, and in
the set-up and management of read replicas. I challenge anyone
unfamiliar with the respective systems to read
[mysql][mysql-replication] or [postgres][postgres-replication]'s
replication documentation and efficiently determine the correct setup
for a simple read slave.

And automated failover is even more fraught in the SQL world; Github
famously [disabled their automated failover][github-failover] after
their solution caused more outages than it resolved. Even once you
have a failover solution -- and everyone's looks slightly different --
client libraries are often not built with failover in mind, requiring
applications to manually detect failure and trigger reconnects.

MongoDB's replica sets are not without their warts, but the conceptual
model is fundamentally strong, and allows operators to, with minimal
fussing, configure read replicas, manually or automatically fail over
to a new server, add or remove replicas, and monitor replication
health, via simple, standard interfaces, all with minimal or no
downtime, and almost entirely using in-band administrative commands
(no editing of configuration files or command-line options).

Baking replication, membership, and failover into the core also means
that drivers can be aware of it at a much higher level. The standard
MongoDB client drivers are aware of replica set membership, and
automatically discover the primary and reconnect as failovers happen
and the primary moves around or nodes come and go. Awareness of
secondaries also allows application developers to select desired
consistency and durability guarantees ("read preference" and "write
concern" in Mongo jargon) on a per-operation level.

In a world where we increasingly take for granted the devops mantra of
"cattle, not pets", and where databases are often the quintessential
"pet" (special, singleton, servers requiring lots of careful manual
attention), the MongoDB replica set model moves us strongly towards a
world where a database is a cluster of mostly-indistinguishable
machines that can be individually swapped out at will, and the system
will adapt appropriately.

[replset]: https://docs.mongodb.org/manual/replication/
[raft]: https://raft.github.io/
[github-failover]: https://github.com/blog/1261-github-availability-this-week
[mysql-replication]: http://dev.mysql.com/doc/refman/5.7/en/replication.html
[postgres-replication]: https://wiki.postgresql.org/wiki/Replication,_Clustering,_and_Connection_Pooling

## The Oplog

MongoDB's [oplog][oplog] is the data structure behind MongoDB's
replication. Similar in concept to MySQL's row-based replication, it
logs every modification made to the database, at level of individual
documents. By following this log and replaying the writes, secondaries
in a replica set are able to efficiently follow the primary and stay
up-to-date.

At a technical level, the oplog is not particularly novel or
innovative. In terms of how it fits into the overall system, however,
the design is compelling in a number of ways.

The oplog is exposed directly as a collection (`local.oplog.rs`) in
MongoDB, and can be accessed just like any other collection. This
design choice simplifies interacting with the oplog in a number of
ways: the same APIs can be used in client bindings, the standard
authentication and authorization mechanisms work, and the same data
format (BSON objects) is used, making entries easy for users to
interpret.

In addition, the oplog uses GTID-like position identifiers that are
identical across all nodes in a replica set. This allows clients to
seamlessly follow writes across the replica set as a logical unit,
without thinking too hard about the physical topology or which nodes
are primary.

Rather than define a new serialization format for changes, the oplog
reuses BSON, like everything else in MongoDB. In addition, the
structure of BSON entries is quite simple -- there are only five or so
types of operations. The conjunction of these two features makes it
easy to parse and interpret oplog entries.

The combination of all these features means that it's feasible -- even
easy -- to use the oplog not just as an internal implementation
detail, but as an application-visible [log][log] of all database
writes, which can be used to drive external systems, such as data
analysis pipelines or backup systems.

My own [MoSQL][mosql], for instance, replicates from MongoDB (via the
oplog) into PostgreSQL, in under 1000 lines of Ruby. [Meteor][meteor],
a platform for realtime javascript applications, leverages the oplog
to get [realtime notification][meteor-oplog] of changes to the
database without having to poll for changes.

# Conclusion

This post is not a holistic defense of MongoDB as it exists
today. MongoDB remains immature, and today's SQL databases remain some
of the most mature, battle-tested, reliable, and performant storage
solutions ever created. And MongoDB made many novel design decisions
other than the ones I've cited above; some are good ideas, some are
bad ideas, and some remain to be determined.

However, I do believe that, in significant ways, MongoDB is much
better designed for the ways we write and run applications in 2015
than the SQL RDBMS paradigm.

Put that way, of course, this realization shouldn't be surprising: SQL
databases -- while incredible pieces of technology -- haven't really
fundamentally changed in decades. We should probably expect a database
designed and written today to be a better fit for modern application
and operational paradigms, in at least some ways.

The interesting question, then, is whether the rest of the engineering
behind MongoDB can catch up. I, for one, am optimistic. MongoDB
continues to improve rapidly: It's adapted
[a real storage engine][wiredtiger], and Facebook is
[working on another][mongo-rocks]. MongoDB 3.2, now in RC,
[significantly improves leader elections][mongo32-raft]. Every day,
more experienced engineers turn their eyes on MongoDB, and while they
often do discover problems, those problems then tend to get fixed
fairly efficiently.

So while MongoDB today may not be a great database, I think there's a
good chance that the MongoDB of 5 or 10 years from now truly will be.

[oplog]: https://docs.mongodb.org/manual/core/replica-set-oplog/
[log]: https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying
[mosql]: https://github.com/stripe/mosql
[meteor]: https://www.meteor.com/
[meteor-oplog]: https://github.com/meteor/meteor/wiki/Oplog-Observe-Driver
[mongo-rocks]: http://blog.parse.com/announcements/mongodb-rocksdb-parse/
[mongo32-raft]: https://www.mongodb.com/presentations/replication-election-and-consensus-algorithm-refinements-for-mongodb-3-2

[^sql]: Despite all the "NoSQL" hype, MongoDB is structurally and
    functionally much more similar to a SQL database than it is to
    most of the more exotic NoSQL databases. In both MongoDB and SQL
    RDBMSes, the core abstraction is a single server, with a number of
    tables or collections that support multiple consistent secondary
    indexes and rich querying.

[^injection]: MongoDB did mess up slightly and allow a
    literal/operator ambiguity that permits
    [a form of injection attack][mongo-injection], but it's more
    limited and rarer than SQL injection.

[mongo-injection]: http://blog.websecurify.com/2014/08/hacking-nodejs-and-mongodb.html
