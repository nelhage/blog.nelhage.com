---
title: "Building personal software with Claude"
slug: personal-software-with-claude
date: 2025-01-27T12:00:00-08:00
---

Earlier this month, I used Claude to port (parts of) an Emacs package into Rust, shrinking the execution time by a factor of 1000 or more (in one concrete case: from 90s to about 15**ms**).

This is a variety of yak-shave that I do somewhat routinely, both professionally and in service of my personal computing environment. However, this time, Claude was able to execute substantially the entire project under my supervision without me writing almost-any lines of code, speeding up the project **substantially** compared to doing it by hand.

Going in to this project, I still expected Claude to be only a small help -- on the order of "a better documentation search engine and also a better Stack Overflow." Despite working at Anthropic and reading [Simon Willison][simonw] from time to time, my expectations were apparently substantially out of date!

This experience has shifted a bunch of my thinking about the role of LLMs in software engineering and in my own work. These thoughts are still unfolding, but this piece is an attempt to capture my experience, and to think aloud as I ponder how to update my behaviors and beliefs and expectations.

[simonw]: https://simonwillison.net/

## The problem

Over the last year or so, I've become a heavy [Obsidian.md][obsidian] user, replacing a combination of [Emacs org-mode files][org-mode] and [Workflowy][workflowy]. There's a lot I like about Obsidian, but the salient detail for this post is (a) it stores data directly in vanilla Markdown files on local disk, and (b) there's a decent [Emacs mode][obsidianel] to interact with them. I still primarily use the app, even while on my laptop, but being able to drop into Emacs to do complex text manipulation, or to keep TODO lists in my editor as I code, is really valuable for me.

As my vault grew, though, I had a problem: `obsidian.el` became **unusably slow**. Simply opening any note would hang Emacs, at first for a few seconds, and eventually for a minute or more. The issue [turned out to be][slow] the `obsidian-update` function: `obsidian.el` regularly re-walks the entire vault, scanning every note for tags and metadata, in order to update its internal inventory, and this scan (which is written entirely in elisp), ends up being very slow.

## My plan

I'm a passable elisp programmer, but I have no experience profiling or optimizing it, and the [GitHub issue][slow] already contained a few other folks' attempts. I figured I'd take a different approach.

I planned to write a short program in Rust to scan and inventory my vault and output minimal metadata as JSON, and to modify `obsidian.el` to consume that output. I'm more comfortable in Rust, and I know it to be generally quite performant and to have excellent profiling tools.

Such a solution might be tricky to include upstream, but I was pretty optimistic that it'd be a relatively bite-size project for my own use, and certain I could solve my own problems that way.

## Using Claude

On a whim, I decided to see how much of this problem Claude could solve for me. I didn't expect it to work all that well, but I decided to give it a go anyways -- perhaps Claude would surprise me. I decided to start with a relatively well-defined request with a relatively clear goal, but asked it to do most of the project in one go:

> [Uploaded file: `obsidian.el`]
>
> The `obsidian-update` function is painfully slow. I want to speed it up by porting the relevant logic to a Rust program, which will output the relevant details in JSON. Can you write the first version of such a Rust program? It should accept the vault directory as a command-line argument, and can assume all other values are set to their default values.

In short: This just worked. Here's [the initial version](https://github.com/nelhage/nix-config/blob/6ffad936fa003ca8f35b1930806b3c15e36da41e/src/obsidian-scan/src/main.rs).

In a single prompt, with no chain-of-thought or other explicit reasoning, Claude read around 1,000 lines of Emacs Lisp, identified the ~200 lines involved in `obsidian-update`, identified the data boundary between that code and the rest of the file, designed a JSON format for that data, and then ported the relevant logic into about 150 lines of Rust.

The resulting code compiled and worked on the first try, and Claude even output a functioning `Cargo.toml` containing the needed dependencies.

I went further -- could it also write the Emacs side? I decided to be ambitious, and asked it to use [Emacs advice][advice] to patch `obsidian.el`, with the aim of producing something I could include in my configuration without forking or patching the upstream project. This prompt:

> [Uploaded file: `obsidian.el`]\
> [Uploaded file: `obsidian-scan/src/main.rs`]
>
> Now, please write some elisp code that patches `obsidian.el` to use `obsidian-scan`. Instead of editing the original file, please use the elisp "advice" feature, so I can include your snippet in my own configuration after loading obsidian.el

This, too, [pretty much just worked](https://github.com/nelhage/nix-config/blob/6ffad936fa003ca8f35b1930806b3c15e36da41e/src/obsidian-scan/emacs/obsidian-scan.el). It had one bug (that I'm aware of): The Emacs side was prepending a `#` to tags, but Rust was already including one, resulting in tags like `##writing`. I asked Claude to fix it:

> `obsidian--tags-list` is ending up with entries with double `##`, like `##writing`. Do you think we should fix the Rust code or the elisp? Choose one and update it.

Claude chose the elisp, and rewrote the file with the trivial bugfix. This was characteristic of all remaining interactions on the project -- I would ask for an improvement or a fix, and Claude would execute it with at most small hiccups.

I went back and forth in this style just a few more times. Notably, I asked Claude to output the (file -> tags) mapping, and to use it for `obsidian-tag-find` in Emacs. I've now added both the Rust program and the elisp patch to [my personal system configuration][obsidian-scan], and I've been quite happy so far!

The whole project took something like a single afternoon.

# Reflection

## Claude is much better at code than I realized

I asked Claude the original prompt mostly not expecting much. But I'd heard that Claude 3.5 Sonnet is much-improved at writing code, and in general I try to periodically try to use LLMs even when I expect failure, since that's the only way I've found to really stay calibrated on their capabilities.

I was impressed, first, by the initial rewrite: I figured Claude would be alright at writing ~100 lines of code given a clear spec, but extracting an implicit spec from within a thousand lines of elisp, and executing against that spec, surprised me. I was further impressed by the process of iterating on the system; Claude continued to be very effective at reading moderate amounts of code from the context window, and updating them or responding to them in fairly sophisticated ways. Pretty much everything I tried "just worked."

I understand very well, intellectually, that these models are improving at an exponential rate, and that impressions formed 6 or 12mo ago are almost surely badly out of date. But that abstract understanding isn't sufficient to form an accurate impression of the new _status quo_! The only way I know of to do so is to just keep using the models and interacting with them (and reading reports from others who do so). That fact is precisely why I tried this prompt, despite expecting failure, and also why I'm writing this reflection.

### "Should" you be impressed by this?

I'm fascinated by my own reactions to and thoughts about Claude, just in the course of writing this note.

As described above, my initial reaction was "surprised and impressed," with Claude exceeding expectations. The original `obsidian.el` is about 10,000 tokens long; It was [only in 2023][clong] that Anthropic released the first public model that could process that much text **at all**. And for my task, Claude needed to not just extract a single fact, or some summarized gist, of the input, but accurately understand and interpret **many** small details, scattered around the file. Models couldn't do that until recently! And it did so in a **in a single prompt**, without any chain-of-thought or explicit reasoning! So yeah, I was impressed.

[clong]: https://www.anthropic.com/news/100k-context-windows

At the same time, _even as I drafted this post_, I noticed a familiar sort of goalpost-shifting in my own mind! I find myself shifting from surprise and amazement into a sort of almost-jaded minimization, from at least two perspectives:

1. Okay, but **shouldn't** I have expected this? I've seen many engineers marvel at Claude 3.5 Sonnet's coding acumen. I've personally seen it solve Anthropic interview questions; this exercise involved a bit more code, but I've also seen it read and answer questions about **much** larger pieces of code. I was surprised, but that's a problem with my expectations, more so than the model being all that impressive!
2. Is this task really that impressive? This project "only" involved around 1000 lines of code; many of the projects I work on touch hundreds of thousands or millions of lines of code! Rewriting elisp into Rust might **sound** impressive, but it's also "just" a translation problem: fairly well-specified at root, and language models are generally quite strong at tasks that "look like" translation.

I think both of these perspectives -- the awe, and also the minimization -- are valid in different ways! All these things are true:
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

## I'm tentatively excited

I feel like this project made me feel a sense of personal excitement and enthusiasm for building software with Claude / other LLM assistance, in a way I hadn't personally felt before.

Specifically, I felt the joy and excitement of writing software to "scratch my own itch" -- to solve a problem I had, and build a tool I wanted -- in a way I mostly haven't felt in years. In my younger years, especially around undergrad, I was [surrounded by][situated-newsletter] bits and pieces of software that I had written, or my friends or people like us had written, in order to solve our own problems and give ourselves new capabilities. We felt a tremendous amount of power and optimism and possibility, and a real lived belief that we could shape our own computing environment, and build our own tools and craft the experiences we wanted. This ethos existed on a range of scales, describing both [small scripts used by a few friends][situated], but also many of the largest and most successful open-source projects of the time.

In the years since, the world has changed, and I have changed. Our data has become more and more isolated into various siloed cloud services owned and operated by monopolies ranging from indifferent to malevolent. Everything has become more complicated, layered behind towers and towers of abstraction and complexity and inscrutable OAuth. I, too, have spent my career reading about working at those kinds of establishments, and have learned to reflexively write all software as thought it were going to be deployed at massive scale and live for years. I even **enjoy** that mode of working, and the challenge of building robustly and finding firm foundations, but coupled with the abject despair I feel every time I have to figure out how to authenticate to a Google API, or migrate to a new version of a Javascript framework, it means I've mostly stopped building small pieces of software for myself and my friends.

I now feel, for the first time in a while, a glimmering of that old excitement! Claude is far better than I will ever be at handling arcane authentication systems, or knowing the current version of React, or whatever else. It feels like we're living in a world where -- now, or at least soon -- it will suffice to accurately describe the tool or script that I **wish** existed, and Claude can do the gruntwork. And if time passes, and the API is deprecated, and we need to migrate to v27 of some framework, Claude will probably also be able to do that work. Heck, we may live in a world where every few years you just throw out all of the code, and start over in a new session with Claude, with the same problem statement but a copy of the current documentation, and work from scratch. I feel a glimmer of hope and excitement: can I relearn to build small software, for my own use, or my community's, where Claude helps to handle so many of the innumerable nuisances that have crept into software engineering in the last twenty years, and also lets me stay at a high-level design and concept level that prevents my obsessive perfectionism from getting stuck in the details?

## I was fighting the tooling, not working with it

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
