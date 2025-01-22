---
title: "TK building personal infrastructure with Claude"
slug: personal-software-with-claude
date: 2025-01-17T16:29:28-08:00
---

In order to speed up an Emacs package I use, I recently ported part of it into Rust, dropping the execution time by a factor of 1000 or more (from 90s to perhaps 15ms). This is a kind of coding I do somewhat routinely, either to fix problems in my personal computing environment, or at work.

However, for this project, I didn't write any code: I did the project almost entirely by asking Claude to write the code for me. I didn't really expect that to work, but it turns out my personal estimation of Claude's coding ability is behind the times, and it worked perfectly, smoothly executing both the initial rewrite, and a number of small fixes and feature additions.

This post is an attempt to document a bit of my experience working with Claude, as well as a number of my reflections and reactions.

## The problem

Over the last year or so, I've moved my personal note-taking and TODO lists into [Obsidian.md][obsidian], replacing a combination of [Emacs org-mode files][org-mode] and [Workflowy][workflowy]. There's a lot I like about Obsidian, but the salient detail for this post is (a) it stores data directly in vanilla Markdown files on disk, and (b) there's a decent [Emacs mode][obsidianel] to interact with your vault. I still primarily use the app, even while on my laptop, but being able to drop into Emacs to do more-complicated edits, or to keep TODO lists right there in my editor as I code, is really valuable for me.

As my vault grew, though, I had a problem: `obsidian.el` became **unusably slow**. Simply opening any note would hang Emacs, at first for a few seconds, and eventually for a minute or more. The issue [turned out to be][slow] the `obsidian-update` function: `obsidian.el` regularly re-walks the entire vault, scanning every note for tags and metadata, in order to update its internal inventory, and this scan (which is written entirely in elisp), ends up being very slow.

## My plan

As an Emacs user of two decades now, I'm a passable elisp programmer, but I have no experience profiling or optimizing elisp, and the [GitHub issue][slow] already contained a few other folks' attempts. I figured I'd take a different approach.

I planned to write a short program in Rust to scan and inventory my vault and output minimal metadata as JSON, and to modify `obsidian.el` to consume that output. I'm more comfortable in Rust, and I know it to be generally quite performant and to have excellent profiling tools. As a bonus, I could consume the JSON output from other tooling I might end up building.

## Using Claude

On a whim, I decided to see how much of this problem Claude could solve for me. I didn't expect it to work all that well, but I decided to check in on Claude -- perhaps it would surprise me. I made a well-scoped and explicit request, but asked it to do most of the project in one go:

> [Uploaded file: `obsidian.el`]
>
> The `obsidian-update` function is painfully slow. I want to speed it up by porting the relevant logic to a Rust program, which will output the relevant details in JSON. Can you write the first version of such a Rust program? It should accept the vault directory as a command-line argument, and can assume all other values are set to their default values.

Here's [the first version it produced](https://github.com/nelhage/nix-config/blob/6ffad936fa003ca8f35b1930806b3c15e36da41e/src/obsidian-scan/src/main.rs).

In a single prompt, with no chain-of-thought or other explicit reasoning, Claude read around 1,000 lines of Emacs Lisp, identified the ~200 lines involved in `obsidian-update`, identified what data they needed, and then ported all the relevant logic into about 150 lines of Rust.

The resulting code compiled and worked on the first try, and Claude even output a functioning `Cargo.toml` containing the needed dependencies.

I went further -- could it also write the Emacs side? I decided to make it a bit harder by telling it to use [Emacs advice][advice] to patch `obsidian.el`, so that I could load the upstream `obsidian.el` without forking it, and just add to my own configuration. Prompt:

> [Uploaded file: `obsidian.el`]\
> [Uploaded file: `obsidian-scan/src/main.rs`]
>
> Now, please write some elisp code that patches `obsidian.el` to use `obsidian-scan`. Instead of editing the original file, please use the elisp "advice" feature, so I can include your snippet in my own configuration after loading obsidian.el

This, too, [mostly just worked](https://github.com/nelhage/nix-config/blob/6ffad936fa003ca8f35b1930806b3c15e36da41e/src/obsidian-scan/emacs/obsidian-scan.el). It originally had one bug (that I'm aware of): The Emacs side was prepending a `#` to tags, but Rust was already including one, resulting in tags like `##writing`. I asked Claude to fix it:

> `obsidian--tags-list` is ending up with entries with double `##`, like `##writing`. Do you think we should fix the Rust code or the elisp? Choose one and update it.

Claude chose the elisp, and rewrote the file with the trivial bugfix. This was characteristic of all my future interactions -- I would ask for an improvement or a fix, and it would pretty much Just Work.

I went back and forth a tiny bit more; notably, I asked Claude to output the (file -> tags) mapping, and to use it for `obsidian-tag-find` in Emacs. I've now added both the Rust program and the elisp patch to [my system configuration][obsidian-scan], and I've been quite happy so far!

The whole project took something like a single afternoon.

# Reflection

## Claude was much better than I realized

I didn't originally plan to use Claude to do the Rust rewrite; I only asked it to do the rewrite on a whim, figuring I'd give it a chance to impress me, or that I'd learn something about its current capabilities and limits. Even after the initial prompt wildly exceeded my expectations, I continued to be impressed throughout followup queries: Claude did multiple rounds of adding features and fixing bugs pretty much perfectly, and everything I tried "just worked."

I understand very well, intellectually, that these models are still improving at an exponential rate, and that impressions formed 6 or 12mo ago are likely badly out of date. But that abstract understanding isn't sufficient to form an accurate impression of the new _status quo_! The only way I know of to do so is to just keep using the models and interacting with them (and reading reports from others who do so). That fact is precisely why I tried this prompt, despite expecting failure, and also why I'm writing this reflection.

### "Should" you be impressed by this?

I'm fascinated by my own reactions to and thoughts about Claude, just in the course of writing this note.

I started out quite impressed -- I did not expect Claude to handle working on this much code so fluently, including across feature additions and modifications, and including fluently writing elisp, including using relatively obscure features and constructs. `obsidian.el` is about 800 lines long, which is 32KB of text and around 10,000 tokens. It was [only in 2023][clong] that Anthropic released the first public model that could process more than about 10,000 tokens, **at all**. Now Claude can not only handle that much code easily, but also identify the pieces relevant to a specific component, figure out what data they need, and reimplement them fluently in Rust, **in a single prompt**, without any chain-of-thought or explicit reasoning!

[clong]: https://www.anthropic.com/news/100k-context-windows

At the same time, _even as I've drafted this post_, I've noticed a familiar sort of goalpost-shifting inside myself! I find myself shifting from surprise and amazement into a sort of almost-jaded downplaying, along two lines:

1. Okay, but **shouldn't** I have expected this? I've seen many engineers marvel at Claude 3.5 Sonnet's coding acumen. I've personally seen it solve Anthropic interview questions; this exercise involved a bit more code, but I've also seen it read and answer questions about **much** larger pieces of code. I was surprised, but that's a problem with my expectations, more so than the model being **particularly** impressive!
2. Is this really that impressive? This project "only" involved around 1000 lines of code; many of the projects you work on touch hundreds of thousands or millions of lines that weren't in the training set. Rewriting elisp into Rust **sounds** impressive, but it's also "just" a translation problem: fairly well-specified at root, and language models are, in general, perhaps at their strongest at translation-shaped tasks.

I think both of these perspectives -- the awe, and also the downplaying -- have validity! All these things are true:
- LLMs are much, much better at coding than they were a year ago
- And, even more so, their present capabilities would have **absolutely** sounded like impossible sci-fi even 2-3 years ago
- But also, the current models are more-or-less "just a continuation of the same trend;" they're impressive, but also don't look all that different from where ML insiders and leaders were predicting we'd be as of a year ago
- Also, current models, while impressive, still fall well short of expert human performance.  They are useful in the right contexts, but not (yet!) able to "do my job"

This, I think, is pretty much just the nature of ML and LLMs right now. Exponential rates of performance improvement mean we're forever in the middle between the experiences of awe and amazement and sci-fi, and of "Oh, this is just more of the same, what's the big deal?"

<!--
$ cat ~/.elisp/elpa/obsidian-20240713.906/obsidian.el| scrubs count-tokens
10366
$ cat ~/code/nix-config/src/obsidian-scan/emacs/obsidian-scan.el| scrubs count-tokens
1251
$ cat ~/code/nix-config/src/obsidian-scan/src/main.rs| scrubs count-tokens
1589
-->

## I felt like I was fighting the tooling

Claude.app / [claude.ai][claudeai] did not feel well-designed for this problem. I spent a lot of time copy-pasting from the UI into on-disk files, and copy-pasting errors or other output back to Claude.

I found it challenging figuring out how to maintain appropriate context for my conversations. I ended up with three files -- the original `obsidian.el`, the Rust port `main.rs`, and the elisp adapter code, `obsidian-scan.el`. It wasn't clear to me if I should just use one long conversation, or try to pull all of them into a [Project][project] and create new conversations periodically (when?). Working in a single long conversation was straightforward, but gets slower as the context window gets longer, and I also find Claude eventually gets more confused by all the different versions of files it can see in the history. I tried using the "add to project" button to copy [Claude-created artifacts][artifacts] back to the project, but had two main challenges:
- I'd end up with out-of-date versions in the Project, since I was rapidly iterating with Claude on the generated artifacts
- Claude wouldn't let me rename/re-title the artifacts, so I'd end up with a file named "Obsidian.el Performance Enhancement with Rust Scanner," which made it hard to refer to them when asking questions or asking for changes to a specific one.

This is an area with a lot of active development right now; it's possible that using an appropriate [MCP server][mcp] or one of the many new AI-first IDEs would have made this simpler.

Overall, though, this experience reinforced my belief that the tooling and interface design around LLMs is lagging way behind the actual capabilities, and is an area of active experimentation and development, even aside from any future model improvements.

## Working between defined interfaces

When working with Claude, I found myself instinctively choosing to break down problems into ones with relatively well-defined and **testable** interfaces. For instance, instead of asking it to make changes to the Rust and elisp code in one query, I would ask for a feature to be added to the Rust side, which I would then spot-check by inspecting the output JSON, and then ask for the corresponding elisp changes.

I'm not sure how often this was **necessary**, but I think I was doing it for my own sake as much as Claude's; breaking pieces down based on interface boundaries and [testability][testability] helped **me** feel like I understood and could validate what was going on, and also that I knew how to intervene or debug if things were getting confused. It's not unlike how I might work with a more junior engineer, at least in part -- doing more of the system design or decomposition myself, and then delegating out well-defined components.

It would definitely also help with correctness and testing. I didn't bother verifying how well Claude's Rust code matched the original elisp, because I don't actually care until and unless it has bugs I notice; but "given this filesystem tree, you should find these notes, containing these tags and this other metadata" is a very easily-testable problem, and if I did care more, or if I find bugs, I can resolve them by encoding them into test cases and asking Claude to refine the code, rather than by digging into the guts myself.

I expect -- at least for the immediate future -- that this will remain a powerful pattern: human-defined interface boundaries or system decomposition, and Claude working "in between the lines." Presumably, as time goes on, Claude will be capable of handling larger and larger subsystems mostly-autonomously, and we will also gain confidence and build techniques for handling ones with fuzzier specifications.

[testability]: https://blog.nelhage.com/2016/03/design-for-testability/

## I'm tentatively excited

Even as I've been aware of the ongoing progress in coding performance by language models, I mostly haven't made much use of them myself, and I've approached them with a vague sense of dread and preemptive exhaustion. This comes from a number of places, but I think a large one is that a lot of the LLM usage I see feels antithetical to my personal approach to software and philosophy of software: I like to [deeply understand][understand] software systems, and build with careful craft, but a lot of the enthusiasm around LLMs feels targeted at replacing human understanding and spewing out ever-growing piles of mediocre glue code, rather than augmenting understanding and building with care.

I still have that fear, but this small project gave me another perspective: LLMs may be drastically lowering the cost to building personal tooling -- to certain types of [situated software][situated] -- and letting me, personally, build more of the one-off tooling to manage my digital life, which I feel like [used to be ubiquitous in my life][situated-newsletter], but is now tangled in a hellish mess of broken APIs, confusing OAuth, and other tangles of modernity. I have half a dozen projects languishing in my TODO list to extract data from `$service` or import data from `$google_project` into `$other_tool`, all of which I know I **could** build, given time, but which just seem not worth the squeeze when I just know I'm going to lose three days just figuring out how the hell to auth to Google again. If Claude can do that boilerplate for me, maybe I'll revisit those! And for many such projects, the code itself can be entirely throwaway or is fully described by its interface definition, and so I'm happy to **not** build it with craft, but treat it as a black box maintained by the helpful bots.

I **don't** think we're yet at a point where LLMs enable that sort of personal software, at scale, by people who **don't** already have much of the expertise and background context to do it themselves; but that does seem like it may change, and that, too, would be interesting to consider.

## Code is cheaper than ever

For a while now, I've increasingly held the perspective that lines of code, _per se_, are relatively cheap; the valuable resources in software development look more like the **understanding** of the code, and what it does and why, and the knowledge of **what** code to write[^fn-theory].

LLMs seem likely to **vastly** increase that dynamic, at least in the short term. You can now generate thousands of lines of code at a price of mere cents; but no human will understand them, and the LLMs are, for now, worse at debugging and refactoring and designing and maintaining those lines of code than they are at generating them. Thus, code is cheaper than ever, but I suspect that insight and good architectural design and **understanding**, at least for now, will become more valuable than ever.

I suspect this change also impacts which architectural patterns will make sense. Perhaps it will make it more important than ever to [write code that is easy to delete][delete], so you can just throw away whole modules and ask Claude N+1 or GPT M+1 to start over. Perhaps the bar to building "polyglot" systems, encompassing multiple different programming languages, will be much lower, if you're able to harvest the benefits of different language ecosystems or performance characteristics, and don't have to worry as much about **hiring** engineers comfortable in four esoteric languages. It's gonna be interesting.

[claudeai]: https://claude.ai/
[project]: https://support.anthropic.com/en/articles/9517075-what-are-projects
[artifacts]: https://support.anthropic.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them
[mcp]: https://docs.anthropic.com/en/docs/build-with-claude/mcp
[delete]: https://programmingisterrible.com/post/139222674273/write-code-that-is-easy-to-delete-not-easy-to

[^fn-theory]: I keep meaning to write something about Naur's [_Programming as Theory-Building_](https://pages.cs.wisc.edu/~remzi/Naur.pdf), one of my favorite papers, and one of the first I'm aware of to capture this insight.


# Footer: TODO

- [x] Intro
- [x] "should you be impressed" rewrite
- [ ] Title


[org-mode]: https://orgmode.org/
[workflowy]: https://workflowy.com/
[obsidian]: https://obsidian.md/
[obsidianel]: https://github.com/licht1stein/obsidian.el
[slow]: https://github.com/licht1stein/obsidian.el/issues/39
[advice]: https://www.gnu.org/software/emacs/manual/html_node/elisp/Advising-Functions.html
[obsidian-scan]: https://github.com/nelhage/nix-config/tree/main/src/obsidian-scan
[situated]: https://gwern.net/doc/technology/2004-03-30-shirky-situatedsoftware.html
[situated-newsletter]: https://buttondown.com/nelhage/archive/situated-social-software/
[understand]: /post/computers-can-be-understood
