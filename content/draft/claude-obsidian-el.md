---
title: "Speeding up obsidian.el with Claude"
slug: personal-software-with-claude
date: 2025-01-17T16:29:28-08:00
---

TKTK intro

# What I built


## Context: The problem

Over the last year or so, I've moved my personal note-taking and TODO lists into [Obsidian.md][obsidian], replacing a combination of [Emacs org-mode files][org-mode] and [Workflowy][workflowy]. There's a lot I like about Obsidian, but the salient detail for this post is (a) it stores data directly in vanilla Markdown files on disk, and (b) there's a decent [Emacs mode][obsidianel] to interact with your vault. I still **primarily** use the app, but being able to drop into Emacs to do more-complicated edites, or to keep TODO lists right there in my editor as I code, is really valuable for me.

As my vault grew, though, I had a problem: `obsidian.el` became **unusably slow**; As an example, the `obsidian-daily-note` command, which just jumps to today's daily note, would hang for multiple seconds before opening the note. The issue [turned out to be][slow] the `obsidian-update` function: `obsidian.el` regularly re-walks the entire vault, scanning every note for tags and metadata, in order to update its internal inventory, and this scan (which is written entirely in elisp), ends up being very slow.

## My idea for a fix

As an Emacs user of two decades now, I'm a passable elisp programmer, but I have no experience profiling or optimizing elisp, and the [github issue][slow] already contained a few other folks' attempts.

I figured I'd take a different approach: I'd write a short program in Rust to scan and inventory my vault and output minimal metadata as JSON, and then modify `obsidian.el` to consume that output. I'm more comfortable in Rust, and I know it to be generally quite performant and to have excellent profiling tools. As a bonus, I could consume the JSON output from other tooling I might end up building.

## Using Claude

On a whim, I decided to see how much of this problem Claude could solve for me. I didn't expect it to work all that well, but to my shock, Claude produced a working Rust implementation in one go, with a quite simple prompt:

> [Uploaded file: `obsidian.el`]
>
> The `obsidian-update` function is painfully slow. I want to speed it up by porting the relevant logic to a Rust program, which will output the relevant details in JSON. Can you write the first version of such a Rust program? It should accept the vault directory as a command-line argument, and can assume all other values are set to their default values.

Here's [the first version it produced](https://github.com/nelhage/nix-config/blob/6ffad936fa003ca8f35b1930806b3c15e36da41e/src/obsidian-scan/src/main.rs). Around 150 lines of Rust, featuring parallelism with `rayon`, filesystem interactions, and `serde` parsing and output. It compiled and worked on the first try, and Claude even output a functioning `Cargo.toml` containing the required dependencies.

I went further -- could it also write the Emacs side? I decided to make it a bit harder by telling it to use [emacs advice][advice] to patch `obsidian.el`, so that I could load the upstream `obsidian.el` without forking it, and just add to my own configuration. Prompt:

> [Uploaded file: `obsidian.el`]
> [Uploaded file: `obsidian-scan/src/main.rs`]
>
> Now, please write some elisp code that patches `obsidian.el` to use `obsidian-scan`. Instead of editing the original file, please use the elisp "advice" feature, so I can include your snippet in my own configuration after loading obsidian.el

This, too, [mostly just worked](https://github.com/nelhage/nix-config/blob/6ffad936fa003ca8f35b1930806b3c15e36da41e/src/obsidian-scan/emacs/obsidian-scan.el). It originally had one bug (that I detected): The Emacs side was prepending a `#` to tags, but Rust was already including one, resulting in tags like `##writing`. I asked Claude to fix it:

> `obsidian--tags-list` is ending up with entries with double `##`, like `##writing`. Do you think we should fix the Rust code or the elisp? Choose one and update it.

Claude chose the elisp, and rewrote the artifact with the trivial change.

After playing with this solution a bit, I went back and forth, tweaking just a bit more; notably, I asked Claude to output the (file -> tags) mapping, and to use it for `obsidian-tag-find` in Emacs. I've now added both the Rust program and the elisp patch to [my system configuration][obsidian-scan], and I've been quite happy so far!

# Reflection

- Claude is better than I expected
- The tooling (Claude.ai / Claude.app) wasn't set up for this
- Reflection on a possible pattern: Human-defined interfaces, Claude works in between them
  - Gonna be a moving target, though!
- Potentially excitement for a new era of [situated software][situated] (a topic [dear to my heart][situated-newsletter])
  - boilerplate, auth, APIs, overhead
  - previously I was pessimistic




[org-mode]: https://orgmode.org/
[workflowy]: https://workflowy.com/
[obsidian]: https://obsidian.md/
[obsidianel]: https://github.com/licht1stein/obsidian.el
[slow]: https://github.com/licht1stein/obsidian.el/issues/39
[advice]: https://www.gnu.org/software/emacs/manual/html_node/elisp/Advising-Functions.html
[obsidian-scan]: https://github.com/nelhage/nix-config/tree/main/src/obsidian-scan
[situated]: https://gwern.net/doc/technology/2004-03-30-shirky-situatedsoftware.html
[situated-newsletter]: https://buttondown.com/nelhage/archive/situated-social-software/
