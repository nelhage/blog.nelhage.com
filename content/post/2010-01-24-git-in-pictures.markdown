---
author: nelhage
author_email: nelhage@mit.edu
author_login: nelhage
author_url: http://nelhage.com
date: 2010-01-24T23:30:02Z
published: true
status: publish
tags:
- git
- pictures
- ui
title: Git in pictures
url: /2010/01/24/git-in-pictures/
wordpress_id: 74
wordpress_url: http://blog.nelhage.com/?p=74
---

In my [previous blog post][git-post], I discussed how git is distinctive among version control system in the way in which it makes the backend model that is being used to store data the most important element of the tool, and that experts use it by having the complete model in their head, and thinking in terms of operations on this object model, rather than just in terms of knowing specific commands to accomplish specific tasks.

Learning to use git in this way requires two steps. First, you need to understand the git object model, so that you know how to express whatever you are trying to do in terms of operations on that object model. In general, I tend to recommend [Git for Computer Scientists][git-cs] as an excellent quick introduction to the git object model for hackers who are basically familiar with concepts like graphs and trees. I've also given a talk for SIPB entitled [Understanding Git](http://cluedumps.mit.edu/wiki/2009/09-29), which attempt to explain git starting from an understanding of the object model. I recommend either or both of those documents to anyone who wants to understand the backend git object model.

The second step, of course, is learning the git commands so that you can actually carry out the operations on the git objects and working copy that you want to accomplish. Git for Computer Scientists falls short on this point, although I think this is mostly because that's not its goal. This blog post is an attempt to provide a quick graphical reference for a number of common git commands, in the hopes of helping users who have gotten comfortable with the git model, but occasionally find themselves groping for the command they want to carry out some operation. I hope that the first image especially will also be useful as a reference for those already comfortable with git.

## Working Trees, Indexes, and HEADs, Oh My!

Personally, I find that the commands that often trip me up the most in git are the various commands for moving data between the working tree, index, and HEAD. I can always remember the basic ones like "add" and "commit", of course, but if I, say, want to update some files in the index to match some commit, it often takes a minute to remember I want `git reset`. The following image attempts to diagram the working tree, the index, and HEAD, the last commit, and show the common commands for moving data between any pair of them:

<a href="/images/posts/2010/01/index-3.png"><img src="/images/posts/2010/01/index-3.png" alt="Moving data around the git working copy." title="Git Index" class="aligncenter size-full wp-image-77" /></a>

The command above each pair of arrows moves data from left to right, and the one on the bottom from right to left. As you can see from the diagram, moving data directly between the working tree and the latest commit generally also updates the index, which is what makes it possible to mostly pretend that the index doesn't exist during normal usage, if you so choose. Also, of course, anywhere in this diagram where a command accepts `HEAD`, you can instead specify a an arbitrary commit object.

## Working with branches and commits

The next pair of diagrams, will both be starting from the following image, showing part of a git repository that contains two branches (`master` and `topic`), which refer to some commits within the git object store:

<a href="/images/posts/2010/01/base.png"><img src="/images/posts/2010/01/base.png" alt="A Git Repository" title="base" class="aligncenter size-full wp-image-80" /></a>

Starting from this image, we can adjust master by adding new commits in several ways. The following diagram shows how the picture would change after each of `git commit`, `git commit --amend` or `git merge topic`:

<a href="/images/posts/2010/01/creating.png"><img src="/images/posts/2010/01/creating.png" alt="Committing to the git repository" title="Committing" class="aligncenter size-full wp-image-81" /></a>

Note that these three commands aren't quite parallel -- the first two, assuming you've staged some files into the index, will both result in the same tree object, whereas the third will execute a merge and construct its tree that way.

Instead of committing to the repository, we might want to create new branches, or move refs around. The fourth diagram, below, shows various commands we could execute to create a branch, move to another branch, or move around the `master` pointer:
<a href="/images/posts/2010/01/branching.png"><img src="/images/posts/2010/01/branching.png" alt="Manipulating git refs." title="Branching"  class="aligncenter size-full wp-image-83" /></a>

Hopefully someone finds some of these useful as a quick reference, or perhaps feels inspired to do something similar but better -- my graphic design abilities are limited at best. Let me know if you find any of these helpful!

[git-post]: /2010/01/on-git-and-usability
[git-cs]: http://eagain.net/articles/git-for-computer-scientists/
[slides]: http://web.mit.edu/nelhage/Public/git-slides-2009.pdf
