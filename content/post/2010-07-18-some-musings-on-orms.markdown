---
author: nelhage
author_email: nelhage@mit.edu
author_login: nelhage
author_url: http://nelhage.com
date: 2010-07-18T18:38:23Z
published: true
status: publish
tags:
- programming
- sql
- orm
- django
- dynamic analysis
- musings
title: Some musings on ORMs
url: /2010/07/some-musings-on-orms/
wordpress_id: 278
wordpress_url: http://blog.nelhage.com/?p=278
---

I'm pretty sure every developer who has ever worked with a modern
database-backed application, particularly a web-app, has a love/hate
relationship with their [ORM][orm], or object-relational mapper.

On the one hand, ORMs are vastly more pleasant to work with than code
that constructs raw SQL, even, generally, from a tool that gives you
an object model to construct SQL, instead of requiring (Cthulhu help
us all) string concatenation or interpolation.

On the other hand, there tends to be an enormous mismatch between the
objects the ORM is modelling on one end, and the relational database
on the back end. Basically, the object model encourages dealing with
one object at a time, whereas the relational database wants you to
push your queries, JOINs, and even your entire transactions back into
it, so that it can better optimize them for you. And so, you tend to
have a choice between writing shockingly inefficient code, and jumping
through weird hoops and strange syntax to give your ORM the tools to
generate efficient queries.

The most fundamental problem, perhaps, is the question of how much
data the ORM should request from the database. When I ask for a
collection of objects (using some syntax to generate the SQL `WHERE`
clause automatically), the ORM has to decide:

 - Which fields in the object should the ORM `SELECT` from the
   database, and
 - Which associated models (with foreign keys to or from the object in
   question) the ORM should `JOIN` in and pull out at the same time.

(You can think of the second as a more involved version of the first,
but involves more complexity, and is interesting for being potentially
recursive).

`SELECT`ing too much data in either case makes the database do too
much work, touch too many blocks on disk, read too many indexes, and
so on, and makes your queries slow. `SELECT`ing not enough data means
your ORM needs to go back and make additional queries when you ask for
fields or models that were not pulled in initially, killing your
performance even more.

I'd like to muse about two possible solutions, neither of which I'm
aware of implementations of, although to be honest I haven't looked
too hard. I'd love it if anyone could point me at work on either sort.

The static solution
-------------------

Given a language with a sufficiently awesome type system (Haskell
comes to mind :)), you could imagine encoding information about which
fields and models are or are not `SELECT`ed into the type of a record
object, making it statically an error to write a query that
under-selects. Given type inference, we can probably even go on to
statically infer (probably with a bit of programmer help in the form
of type annotations) which fields and models we need to `SELECT`,
based precisely on which fields of the result get accessed.

This is the kind of solution that occurs immediately to a certain type
of person (of which I know a lot) when they think about this type of
problem enough. It's fun to think about and there are probably some
papers in it (if no one's beaten you to it), but I suspect it's the
kind of thing that's unlikely to gain much traction among mainstream
developers, if only because it would require you write in Haskell or
equivalent.


The dynamic solution
--------------------

Another solution occurred to me recently, which I don't think I've
heard people talking about. I like it because I'm pretty sure it could
be grafted onto an existing ORM with little or no API change, which
means there's potential for incrementel adoption and improvement.

The key insight here is that, while the decision of what to `SELECT`
is critical for performance, it's (mostly)<a
href="#fn1"><sup>1</sup></a> irrelevant for correctness whether you
have to make additional queries later, or pull in too much data. A
good ORM, if it notices you ask for a field it doesn't have cached,
will just go ahead and make the `SELECT` for you.

This opens the door for heuristic solutions. A good ORM will already
have hooks for the programmer to specify which fields or models to
include or exclude -- [Django][django], to pick a popular example, has
the [`select_related`][select_related] and [`defer`][defer] methods on
its `QuerySet`s.

So, my proposal is this: Let's develop tools to profile an
application, and automatically generate those annotations in a
profile-driven manner. Since all accesses to your data are through the
ORM, we can instrument the ORM to determine which fields are selected
after a query, and remember that information for next time it sees the
same query.

In a webapp environment, the possibilities for optimization get
potentially even more exciting. A webapp framework like Django knows
an awful lot of information at any given moment, like what user is
logged into your app, and what URL is currently being generated. We
could imagine associating that data with the profile results, so that
the ORM could automatically discover relations between the application
structure and the ORM queries. It could "learn", for example, that
certain users are administrators, and so will access more fields on
certain models than other users. It could learn that even though
`/places/list/` and `/places/details/` pull up the exact same set of
models (maybe even through the same helper function!), the former
needs only their names and IDs, while the latter needs to pull in full
details, as well as rows from the associated `reviews` table.

All without a single programmer annotation. Of course, in cases where
the behavior is too complex for the tool's heuristics to pick up, or
where the programmer absolutely needs predictable behavior or needs to
micro-optimize, you could turn off the heuristics and fall back the
old behavior (with the same old annotations available to the
programmer).

Is anyone aware of any work in this space? It sounds like a
potentially extremely exciting project to me, but I probably don't
have the time to attempt to throw together an implementation.

[orm]: http://en.wikipedia.org/wiki/Object-relational_mapping
[django]: http://www.djangoproject.com/
[select_related]: http://docs.djangoproject.com/en/dev/ref/models/querysets/#django.db.models.QuerySet.select_related
[defer]: http://docs.djangoproject.com/en/dev/ref/models/querysets/#django.db.models.QuerySet.defer

<small><a name="fn1">1.</a>Ignoring, for example, race conditions
where you need to select a model and related models atomically, and
don't want or can't afford a transaction around the entire lifetime of
the objects in the app code.</small>
