---
author: nelhage
author_email: nelhage@mit.edu
author_login: nelhage
author_url: http://nelhage.com
date: 2010-02-14T20:03:15Z
published: true
status: publish
tags:
- unix
- git
- dotfiles
title: Versioning dotfiles in git
url: /2010/02/14/versioning-dotfiles-in-git/
wordpress_id: 134
wordpress_url: http://blog.nelhage.com/?p=134
---

I've been looking for a good solution for versioning and synchronizing
my dotfiles between machines for some time. I experimented with
keeping all of `~` in subversion for a while, but it never worked out
well for me.

I've finally settled on a solution that I like using git, and so this
is a writeup of my workflows for working with my dotfiles in git, in
the hopes that someone else might find it useful. You will not find
any scripts here, only a description of a workflow: It's simple enough that I have not felt the need to script any of the pieces, even though I potentially could.

## On the machines

On each machine, I have the dotfiles directory checked out into
`~/.dotfiles/`, and a symlink farm from the actual files in `~/` into
`~/.dotfiles`. If I need to edit files, I just edit them in place and
commit in `~/.dotfiles`. When adding new dotfiles, I just manually
create them in the checkout and create the symlink -- no fancy
scripts. I do this rarely enough that I find it doesn't bug me.

## Branches

I maintain one branch for each machine or group of machines I keep
dotfiles on. For instance, my laptop has a branch, my desktop has a
branch, and all of the machines I have accounts on at work share a
branch. In addition there is a "master" branch, which contains a
prototypical set of dotfiles without any machine-specific
customizations.

In an ideal world, any change to my dotfiles would go to either
`master` (if it is common to all machines) or to a specific machine
branch, and I could then merge `master` into each machine branch to
sync the state of my dotfiles around. Unfortunately, one of my main
desiderata is that I can edit and commit dotfiles in place. And
committing to a non-checked-out branch is awkward at best, and
checking out `master` in the `~/.dotfiles/` working copy is
undesirable, since that might disrupt other programs that are using my
dotfiles at the time. So in practice, I make commits to the branch of
whichever machine I made a given change on, and then push them to that
branch in the master repository on my server.

## Synchronizing

Periodically, I fetch that repository into another working copy, and
synchronize the branches. I check out `master`, and `git cherry-pick`
any commits off of each per-machine branch that should be shared among
all the machines. Once that's done, I check out each per-machine
branch in turn, and `git merge` master back into it. With that done, I
push the branches back, and pull them on each machine.

It's very important that this process -- including the merging back
from master -- be done in a separate working copy. Otherwise, a
conflicted merge results in conflict markers in your dotfiles. It was
particularly fun when I did this once and had a conflict in
`.gitconfig`, as `git` then immediately stopped working, and so I
couldn't `reset --hard` out of the merge or inspect history until I
either resolved the merge or moved the broken file aside.

In practice, this sequence happens maybe once a month, and takes half
an hour or so. I could probably script some of that away, but I find
it mostly acceptable as is.

## Final thoughts

I find the "commit and cherry-pick" workflow somewhat unfortunate, in that it results in two copies of every commit, and it makes "synchronize my dotfiles" a sufficiently expensive operation (in terms of effort) that it doesn't happy continually. It's conceivable that I would be better served by being disciplined about, after testing any change, immediately copying the files to another repository and comitting onto master and merging into the appropriate branch. However, I haven't experimented with that because I intensely dislike any workflow that turns a common, simple operation ("make a stupid fix to my dotfiles") into a more complex one. The current workflow does include a complex operation (the cherry-pick and merge step), but it's a batch job, so I don't mind scheduling it periodically, and it is an additional task I perform on top of the normal work of "messing with dotfiles", which has specific additional benefit ("keeping my dotfiles in sync"), so I find it much more palatable.
