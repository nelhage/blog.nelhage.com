---
title: "Stripe's monorepo developer environment"
date: 2023-12-14T11:00:00-08:00
slug: stripes-dev-environment
---

# Introduction

I worked at Stripe from 2012 to 2019. During that time, I saw and contributed to many different generations of Stripe's developer environment -- the ways that developers wrote and tested code within Stripe's monorepo.

This post is a description of (my memory of) that environment and the accompanying tooling, as of when I left four years ago. I'll also try to discuss some of the tradeoffs and motivations that led to these choices; different environments with different constraints likely **should** make different choices, and understanding the motivation and forces shaping a system is important in order to effectively learn from it.

Some caveats before I jump in: It's been four years, and I have no doubt that I have misremembered or imagined some of the specific details, even if I'm confident in the overall picture. I'm also certain that the excellent teams at Stripe have continued evolving their tooling since then, such that document is certainly out of date with regard to today's Stripe.

And, finally, while I contributed to some of the components described here, I want to be clear I do not claim credit for this system or its design -- everything here was evolved over the course of many years, with contributions from many, many talented individuals, both in vision-and-design, and in implementation.

## Context

The tooling I described here was developed over a range of time where Stripe's engineering team grew from perhaps two hundred to over a thousand. While many of the components here were based on tools that existed since Stripe's earliest days, the "mature" versions of these tools were built and maintained by Stripe's developer productivity team (often known as "dev-prod"), which ranged from perhaps 3-10 engineers over the relevant time period.

Stripe's technical systems were complex and varied and spanned a wide array of technologies and languages; but as of the time I left the majority of high-importance services at Stripe were all implemented in Ruby and lived primarily in a single monorepo with a large number of common libraries and shared tooling. However, the system was under constant development and churn, evolution, and new dependencies on external services -- be that a new open-source tool, a new cloud or third-party service, or an internal tool that lived outside of Ruby for one reason or another -- were a relatively frequent phenomenon. One major goal was to ensure that such migrations or modifications to the environment did not require every developer to perform manual work to garden their developer environment.

The tooling I will describe here was focused almost exclusively on those Ruby services in the monorepo; The developer tooling team, while aware of the diversity of services and languages in use, make a deliberate choice to prioritize the monorepo experience (which, again, was where most product development happened), and to try to make that experience as excellent as they could.

# The development environment

## Development code runs in the cloud

Every developer had a personal development instance in Stripe's cloud environment (but disconnected from production VPC, of course), on which they ran code during development, be that running tests or interactively testing new code.

These instances were centrally-managed and provisioned, and were ephemeral -- a single command would destroy your devbox, provision a new one, and reconfigure it as yours. This design meant that most environment-configuration issues could be solved centrally by tooling teams or service owners, and developers did not have to worry about updating their own development environments to keep up.

All devboxes were accessible to all users, so you could help a coworker debug or try out code on their devbox, and, again, environment and tooling issues on one developer's instance could be debugged by the developer tooling team or by the team owning the code, if needed.

## Devbox affordances

Stripe's developer productivity team built a plethora of tools to make working with devboxes easier. A central registry mapped every developer to their current devbox, which powered tools like:

- DNS resolved hostnames like `*.$username.stripe-dev.com` (I forget the exact domain we used) to that user's current devbox.
- A frontend service runs on every devbox that listens on SSL, handles any auth[nz] needed, and maps service names to distinct local ports:
  - e.g. the API service has some statically-assigned port, and  `api.nelhage.stripe-dev.com` will forward to that port
- A proxy service listens on all of those ports and starts services on demand. Thus, on first request to `api.nelhage.stripe-dev.com`, the API service will be started, but then it will remain running for future requests.
- Services on devboxes use Stripe's [autoloader][autoload] to only load source files on-demand, for faster startup time.
- Devboxes services also use the autoloader to watch (via `inotify`) every source file that has been loaded; if any of them are modified, the service restarts to ensure it's running on up-to-date code.

[autoload]: https://www.youtube.com/watch?v=lKMOETQAdzs

The net effect of this infrastructure is that developers can change code and then almost-immediately, and without any manual steps, access a copy of the service running their new code, at a fixed, shareable (within Stripe) URL. This feature works for nearly all internal services, but each devbox only runs the services that developer is actually working on, saving memory and reducing CPU contention at startup.

In addition, dependencies -- both Ruby gem dependencies, but also any external services (databases, etc) and AWS or other credentials, are centrally-managed on devboxes by Stripe's standard cloud tooling, and so for developers who are "just writing Ruby code" related to some feature, the development environment Just Works and they never need to worry about or debug configuration, Ruby dependencies or `bundler` errors, or anything else.

## Editors and source control

Code during development **ran** on ephemeral instances in the cloud, but that's only half of the development cycle. Before you can actually **run** code under development, you need to be able to access it from source control, and actually edit it; those interactions make up the other foundational component of a development environment.

At Stripe, source checkouts and editors lived on developer laptops, even though code primarily ran in the cloud. This choice was settled on for a number of reasons, but some of the main benefits include:

- Support for a wide variety of editor and IDE environments. Some editors (like `emacs` or `vim`) can be run reasonably-well over an SSH session, and some (like VSCode or emacs with [`tramp`][tramp] (at least in theory)) support running a local UI against a remote filesystem, but many do not. By keeping code on the laptop, Stripe enabled developers to use any editor they wanted without any complex configuration or hacks.
- Latency. Stripe had developers around the globe, and editing code on the other side of a 200ms network latency is a **painful** experience. This could be mitigated by standing up developer instances in regions close to every major office or concentration of developers, but at the time we did not have the necessary infrastructure already-in-place.
- Source code durability. By keeping the source-of-truth on laptops, it became easier to treat the execution environment as ephemeral and transient, since it can always be blown away and then restored from the laptop. This could also have been accomplished by storing source code on a network-accessible mount (such as [EFS][efs]), but those often have latency concerns for `git` operations and even importing or building code.

[tramp]: https://www.gnu.org/software/tramp/
[efs]: https://aws.amazon.com/efs/

There's one major obvious challenge of keeping source code on laptops, though: If we want to actually **run** the code in the cloud, how do we get it there?

Since the early days of Stripe, Stripe had maintained a "sync" script that glued together a file-watcher with `rsync` for this purpose; it would monitor your local checkout for changes and copy them to a devbox in the cloud. Historically, developers would be responsible for manually running it in a separate terminal (or `tmux` window) by hand.

Eventually, though, the Developer Productivity team took ownership of that tool and invested heavily in it, largely of the form of "polish" and attempting to make the experience seamless.

Perhaps most notably, they integrated into Stripe's laptop management and provisioning tooling to make the sync script run as a persistent service on every developer laptop, automatically. This service plugged into the above-mentioned devbox registry, and ensured that code was always synching to a developer's current devbox. The goal was that engineers never need to manually launch or restart the sync script, but can just rely on it as an always-on fact of life.

The sync script was also migrated to use [watchman][watchman] for file-watching; In our testing, `watchman` was by far the most robust file-watcher we found, and vastly reduce the frequency of missed updates or totally-broken change notification. We also made use of some of `watchman`'s more-advanced features, which I'll touch on later.

[watchman]: https://facebook.github.io/watchman/

On the whole, I believe that these investments largely succeeded and enabled most developers to stop worrying about how to run code sync or whether it had broken or gotten stuck.

## The `pay` command

Running `ssh` into devboxes and running commands there was well-supported, but the dev-prod team also implemented a `pay` command that was a unified frontend to a large fraction of the common developer operations.

Perhaps most commonly used, `pay test ...` wrapped our test runner, and would run specified tests on your devbox (over ssh), piping output back to the laptop console.

`pay curl` also wrapped `curl` and provided a helper to run manual `curl` commands against services on your devbox, which was often helpful when doing ad-hoc manual testing of API endpoints.

In addition to the slightly convenience these tools provided (of not having to `ssh` manually), they were also sync-aware, and ensured that your code was synced **before** they would run the desired command. This ensured that you could edit some code, saving it in your editor, and then switch to a terminal and **immediately** run `pay test ...` without fear of missing a sync and inadvertently running on the old code.

This "sync barrier" piggybacked on watchman's [clock][watchman-clock] support. `pay test` could check the `watchman` clock, and then do a side-channel with `pay sync` to ensure that it had synced to at-least that clock value.

In the common usage pattern of "edit and then run a test", the sync operation for the edit would be already in-flight or (even already completed), and the clock-based check means that a `pay test` command can "piggyback" on the existing sync, and not need to incur any unnecessary latency or start a new sync in order to ensure it sees up-to-date code.

This side-channel also allows `pay` to handle the case where the sync script has crashed completely or is otherwise stuck, and report an error to a central Sentry instance, so that the dev-prod team can become aware of the rates of this failure, and investigate or intervene systemically.

[watchman-clock]: https://facebook.github.io/watchman/docs/cmd/clock

## LSP and editor tooling

As mentioned above, Stripe engineers could use any editor they chose, but by the end of my time there, the Developer Productivity team had made the choice to invest in VSCode in particular and make it the "blessed" choice, and build out particular tooling for it.

Some of this consisted of internal plugins with small but meaningful affordances; For instance, VS Code users got the ability to launch a `pay test` command to run the test under the cursor, and similar quality-of-life improvements.

However, the big improvements happened as [Sorbet][sorbet] -- Stripe's Ruby typechecker -- matured and gained an LSP implementation. Stripe configured VS Code to run the sorbet LSP server on the devbox over `ssh`, which gave access to IDE features for developers, but allowed Sorbet to make use of the devbox's larger amount of RAM (LSP servers are often memory-hungry on large codebases, and Sorbet is no exception), and run in a Linux environment that wasier for the Sorbet team to debug and test for.

In general, this approach worked well; LSP is designed to support remote servers and tolerates some amount of latency between the editor and the server. LSP needs to support making queries on unsaved files, and so allows the editor to push the latest edits to the server, allowing it to respond accurately even if the sync process was slightly behind.

[sorbet]: https://sorbet.org/
[sorbet-lsp]: https://sorbet.org/docs/vscode#installing-and-enabling-the-sorbet-extension

# Caveats and complications

Maintaining a smooth development experience as an organization grows in headcount and the codebase and product grow in complexity is a shockingly hard problem, and one that's never really done; as soon as you solve one set of problems, the codebase or business evolves or grows to present you with new ones.

The system designed here evolved bit by bit over many years, and I'm certain it has evolved substantially since them. In my opinion, though, it represented a remarkable achievement and the result of deliberate investment, and a team that did their best to develop and implement an overarching coherent vision for how the development environment would work. For many engineers, it did succeed at "just working" and enable them to focus on developing and shipping features and bugfixes witohut having to think too hard about their environment or other lower-level details.

The world is never simple, though, and the development of the system here coincided with the growth of services and languages and other complexity at Stripe, and for developers who worked outside of the "typical path" of writing Ruby code inside one of the "main" services, many of the components described here were much less effective, or unavailable at all. Such is often a tradeoff: It's easier to build a holistic, integrated, seamless experience if you're able to specialize and narrow the scope of whose use cases you're targeting; but that also risks leaving out important cases.

Still, on the whole, I was very impressed and happy with the tooling described here; I think Stripe, by the last few years of my tenure, had built a remarkable developer environment that effectively supported an ever-growing numbers of developers working on an ever-growing product, while mostly **decreasing**, on average, the degree to which those engineers had to fight with their own tools.

But it's worth recalling also that it wasn't perfect, and that it was a product of the specific problems and choices facing Stripe; take this story not as a manual or a blueprint for your project or company or developer productivity team, but as a set of ideas potentially worth borrowing, and as one example of an attempt to build a system that offered a holistic coherent vision of the development experience.





# TODO
- [X] make sure to mention caveats
- [X] intro, setting the stage
- [X] Am I using "we"?
- [ ] edit for "dev prod" and variants
