---
layout: post
status: publish
published: true
title: On git and usability
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 64
wordpress_url: http://blog.nelhage.com/?p=64
date: 2010-01-18 00:57:31.000000000 +01:00
tags:
- svn
- git
- ui
- usability
- subversion
---
I've been helping a number of people get started working with `git`
over the last couple of weeks, as [Ksplice](http://ksplice.com/) has
brought on some new interns, and we've had to get them up to speed on
our internal git repositories. (As you might expect from a bunch of
kernel hackers, we use git for absolutely everything). While that
experience is what prompted this post, it wasn't really anything I
haven't seen before as [SIPB](http://sipb.mit.edu) transitioned from a
group that mostly versioned code in SVN or
[SVK](http://svk.bestpractical.com/) to one that used git almost
exclusively, practically overnight, as these things go.

I love git, and use it everywhere. One of the things I particularly
love about git is the fact that I understand essentially every element
of git's data model and how it works, which means that I can invent
odd workflows or script git to do things that I never could have with
SVN or even SVK. Part of the reason for this is that git, internally,
really is fundamentally simple in a way subversion isn't. There's some
complexity (which I don't and don't have to understand) behind
implementing its model efficiently, but I can describe the basic git
data model in about a single slide (And in fact [I have][git-slides]).

[git-slides]: http://web.mit.edu/nelhage/Public/git-slides-2009.pdf

At the same time, though, I consistently find that people who come to
git from subversion find it insufferably complex and hard to learn,
and spend quite a while wrangling with it, before they eventually
fight it to a draw where they only need to ask their local git expert
to dig them out of a hole every once in a while, or else until they
just give up and declare git to be space-alien and go back to
something they "understand".

## The problem

Having helped enough of these people, I've begun to understand the
problem. Fundamentally, the way (most) people learn and think about
subversion is different from the way git experts think about
git. subversion's internal model is fairly complex, but **you are not
expected to understand it**. You just have to know the half dozen
commands you'll ever need, and you use them, and everything is fine.

Git's model is fundamentally fairly simple (a DAG of immutable commit
objects where branches are named mutable pointers into it), but **you
are expected to understand it fully** to use git effectively.

A subversion user who wants to get the latest updates from the server
just knows to run `svn up` and go back to their life. A git user has
to (in general) think about whether they want to rebase their local
changes onto the remote changes or merge them, and which remote branch
they want (this is all, of course, expressed essentially in terms of
the git data model), and then maps that back into the commands they
want to perform this task.

Subversion users aren't used to thinking like this, and so when they
ask a git user for help, they get one of two classes of answers,
neither of which they like. Either they just get back a list of half a
dozen possible commands they could choose (since the git user has
mapped their request into the git object model, and can think of a
bunch of ways to implement that operation, depending on your mood and
the phase of the moon), or else they get a "What are you really trying
to?", since their request has multiple interpretations in the language
of git objects. Neither answer is a single command that will always
solve their problem, which is what they want, so they can just go back to whatever they were trying to do, and so they end concluding that git is needlessly strange and confusing.

I think there are at least two reasons for this different perspective
from git users. One is historical -- git was designed to work this
way. Linus wrote git as a dumb content tracker, and intended other
people to write various UIs on top of it, but not really for anyone to
use it as-is. So fundamentally, git is designed to just implement this
basic model, and not to export a single way to do anything.  The
other, perhaps more fundamental reason, is that git is designed to be
infinitely flexible, and so it's fairly rare that you *can* give an
answer for "How do I do X with git?", since the answer will often
depend on, "Well, what are your project's conventions?"

## What do we do about this?

So, as git users, hackers, and evangelists, what can we do to improve
this situation? I think that basically, we need to understand what's
going on here, and try to have better answers for what the subversion
users regard as simple questions. We should encourage people to get to
know and love the git object model, but realize that most of the time,
they just want to get work done, and we need to try to have simple
answers to simple questions.

To that end, I think there are three main remaining areas in the git UI
that I think new users find confusing, and that we need to figure out
how to improve. These are pulling and pushing changes, and the whole
issue of the git index. Here are my thoughts on what's wrong, and what
we can improve on:

* ''git-push'': This command is a disaster in a number of
   ways. `matching` is utterly the wrong default for `push.default` (I
   believe 1.7 is going to fix this). The `git push $REMOTE $BRANCH`
   syntax is not optimized for the common "subversion-like" case of a
   single remote, and leads countless users to attempt to `git push
   master`, at which point they get the less than totally helpful
   error message:

        fatal: 'master' does not appear to be a git repository
        fatal: The remote end hung up unexpectedly

   In general, the error messages are pretty bad. I believe it was
   [`@defunkt`](http://twitter.com/defunkt) who nominated "src refspec
   does not match any" as the "worst error message of all time", and
   it's hard not to see his point.


* ''git-pull'' is also problematic. The vast majority of users in a
  project live at the edges, pushing code inwards, and they almost
  never actually want to create a merge commit. I almost always end up
  telling users "No, really, you always just want `pull --rebase`",
  which leaves them with a poor impression of git's UI (if you always
  want that flag, why isn't it just mandatory?), and which is hard to
  explain, because the whole concept of "rebasing" is difficult to
  really grok without really understanding the commit DAG.

  `branch.<name>.rebase` and `branch.autosetuprebase` are a partial solution, but is there a reason I can't just have a repository wide default option that makes `--rebase` the default? With the former solution, I'm never confident I've actually set it on all of my branches, or that it will get set if some script creates a branch without using `git-branch` or `git-checkout`.

  Similarly, it sucks that you can't pull if you have any uncommitted
  changes at all, especially for subversion users who are used to this
  Just Working.


* Finally, we really need to improve the story with the index. As an
  experienced git user, I love the index, and use it for many reasons
  all the time. However, as anyone who has tried to explain it (or
  even to learn git) probably knows, it's a confusing idea at first.

  Generally, when someone is starting out with git, I try to ignore
  the index, and suggest they just always use `git commit` with a
  path. I then have to handwave, of course, over why they need `git
  commit -a` instead of `git commit`, but that's a mostly minor
  problem.

  The bigger issue comes when a user finds themselves in a merge
  conflict, and finds that it's basically impossible to do anything
  without fully understanding the index, and all of the weird commands
  for moving content between the index, the working tree, and
  `HEAD`. Just the other day, three of us git experts had to stop and
  think for a while about how to answer the question "So, if I'm in a
  conflicted merge, and I want to just conditionally take “their”
  changes, what the hell do I do?". That's not a good sign for new
  users being able to figure out what to do here.

  I don't fully know what the right answer is, but I think it's clear
  the tools need some help here.
