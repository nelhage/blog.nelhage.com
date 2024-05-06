---
title: "Stripe's monorepo developer environment"
date: 2023-12-14T11:00:00-08:00
slug: stripe-dev-environment
---

# Introduction

I worked at Stripe for about seven years, from 2012 to 2019. Over that time, I used and contributed to many generations of Stripe's developer environment -- the tools that engineers used daily to write and test code.

Since leaving, I've found myself repeatedly describing features or design choices of that environment to friends and collaborators; this post is an attempt to record the salient features of that environment as I remember it, because I think it's helpful prior art for other teams building out their own tooling. I'll also try to include some thoughts about the context, constraint, and motivations for Stripe's choices; while I think their tools and choices are worth learning from, their specific choices were deeply informed by the local business and technical environment, and it's unlikely that identical choices would make sense elsewhere.

Some caveats: It's been nearly five years, and I have no doubt that I have misremembered some of the specific details, even though I'm confident in the overall picture. I'm also certain that Stripe has continued evolving and I make no claim this document represents the developer experience at Stripe as of right now.

And, finally, while I contributed to a number of the components described here, I can't and don't try to claim credit for this system or its design. Everything here was evolved over the course of many years, with contributions from many talented engineers, both at the level of overall design and vision and in the implementation.

# Stripe's monorepo development environment

Some initial context-setting for the tools I'm describing: At the time I'm discussing, most of Stripe's codebase was written in Ruby, and lived in a single large monorepo. This codebase supported a number of services, which shared code extensively. Stripe did also have a large and growing number of services that did not live in the monorepo and were not implemented in Ruby, but they were (as a generalization) less core to the business (concretely: the Stripe API was still almost-entirely in Ruby inside of the monorepo). The tooling and experience I discuss here were designed and built primarily to support development in the monorepo; other services and languages were supported incidentally if at all.

Stripe's tooling was built and maintained by a succession of teams and individuals, but I'll collectively refer to the authors and owners of this tooling as the "Developer Productivity Team" or "devprod," which was the name of that team for the last few years of my tenure.

Far more so than the specific engineering choices that I'll outline here, I think the mere existence of that team, which was created and relatively early and which grew faster than the organization as a whole for a while, was the determining factor in any successes of the development experience at Stripe (I worked on the team for a period of time, although not at its inception and also not at the time of my departure). In addition, as that team grew, it prioritized *reliability* and *stability* of developer tooling; the best tooling in the world will profoundly struggle to make up for "regularly losing a day debugging your environment," and so getting that aspect right can easily outweigh almost any other decisions you make. As I discuss the specifics of the choices Stripe made, you'll notice that a number of the design choices were made specifically to **enable** the devprod team to own and monitor tooling and developer experiences centrally, which was a huge enabler for that reliability.

## Development code runs in the cloud

A defining question for developer environments tends to be: Does code under development run locally on the developer's laptop, or does it need to be run elsewhere, usually inside an environment that is centrally defined and provisioned somehow.

At Stripe, we settled on running code in the cloud, for a variety of reasons. Every developer had a personal development instance ("devbox" for short) in Stripe's AWS environment (entirely disconnected from production, of course). During development, code ran on that instance, both for running automated tests and for interactive testing or experimentation.

These developer instances were provisioned by Stripe's standard configuration management tooling, and were ephemeral -- a single command would destroy your devbox, provision a new one, and assign it to you. A central registry kept track of which instance was active for which engineer, allowing tooling to track your devbox across reprovisioning.

This design meant that most environment-configuration issues could be solved centrally by tooling teams or service owners, and developers mostly did not have to worry about updating their own development environments to keep up.

All devboxes were accessible (via `ssh` and the HTTP endpoints I'll talk about shortly) to all engineers, making it easy to show off work-in-progress or help a teammate debug environment or configuration issues. In particular, devprod could debug user issues directly, or a team that was adding a new service dependency could debug issues that arose with their development configuration.

## Editors and source control

Before you can run new code under development, though, you need to actually access the source code and edit it.

At Stripe, even though code **ran** in the cloud, git checkouts and editors lives locally, on developer laptops. Stripe settled on this approach for a number of reasons, including:

- Support for a wide variety of editor and IDE environments. Some editors (like `emacs` or `vim`) can be run reasonably-well over an SSH session, and some (like VS Code or emacs with [`tramp`][tramp]) support running a local UI against a remote filesystem, but many do not. By keeping code on the laptop, Stripe allowed developers to continue using any editor they wanted.
- Latency. Stripe had developers around the globe, but at the time did not maintain substantial infrastructure outside of the US. Editing code on the other side of a 200ms (or worse!) network latency is **painful**. Standing up more EC2 regions might have been an option, but would have brought many other complications.
- Source code durability. By keeping the source-of-truth on laptops and off of the devbox, it was much easier to treat the execution environment as ephemeral and transient. That property, in turn, was very helpful for keeping environments up-to-date and preventing drift over time.
  - This could could also have been achieved by keeping code on a network-accessible mount (e.g. [EFS][efs]) or even an EBS volume, but those also would add complexity, and many such options imposed painful latency on `git` and on service startup, both of which rely on large numbers of rapid filesystem accesses.

[tramp]: https://www.gnu.org/software/tramp/
[efs]: https://aws.amazon.com/efs/

### Automatic synchronization

Keeping code on laptops, and executing in the cloud, though, poses an obvious challenge: How do we **get** code from the laptop onto the devbox?

Even before I had started, Stripe had maintained a "sync" script that glued together a file-watcher with `rsync` for this purpose; it would monitor your local checkout for changes and copy them to a devbox in the cloud. Historically, developers would manually start and monitor this script in an ad-hoc way in a separate terminal or `tmux` window.

Eventually, though, the Developer Productivity team took ownership of that tool and invested in it heavily, largely with the goal of adding "polish" and making it seamless and nearly-invisible for other developers.

Notably, they made syncing happen implicitly and with no configuration or intervention required: They worked with the IT team to install the sync script as a `launchd` service on every developer laptop, and used the above-mentioned registry to automatically discover the correct devbox.

Other polish work included migrating the sync script to use [watchman][watchman] for file-watching; In our testing, `watchman` was by far the most robust file-watcher we found, vastly reducing the frequency of missed updates or a "stuck" file-watcher. It also offered some advanced features that Stripe made use of -- more about that later.

[watchman]: https://facebook.github.io/watchman/

On the whole, these investments largely succeeded: most developers were able to treat code sync as simply a "fact of life," and rarely had to think about or debug it.

As a quick note, one downside of synchronization is that it makes it harder to write "code that acts on code," such as automated migration tools, or even just linters or code-generation tools (for various reasons, Stripe ended up relying on a handful of code-generated components that needed to be checked in). This remained a bit of a pain point through my tenure; we made do with a mix of strategies:
- Run on the developer laptop, and deal with the environment challenges
- Run on the devbox and then somehow "sync back" generated files. We had a small protocol to run scripts in a "sync-back" wrapper, s.t. they could ask for file to be copied back to the laptop, but it remained somewhat awkward and non-ergonomic and occasionally unreliable.

## HTTP services on the devbox

A large fraction of the code at Stripe existed to support an HTTP service of some form, including most of the Stripe API. The developer productivity team built a suite of tooling to make developing these services fairly seamless. This included:

- DNS resolved hostnames along the lines of `*.$username.stripe-dev.com` to that user's current devbox
- A frontend service ran on each devbox that terminated SSL, handled any auth[nz] needed, and mapped service names to the appropriate local instance
  - Each service at Stripe was statically assigned a local port, used by the frontend to route traffic.
  - For instance, the API service had some assigned port, and the frontend would forward `api.nelhage.stripe-dev.com` there
- A separate proxy service listened on each of those ports, and demand-started the backing Ruby service.
  - Thus, on first request to `api.nelhage.stripe-dev.com`, the API service would start up, and then remain running to handle future requests directly.
- Those services used Stripe's [autoloader][autoload] on the devbox, and thus only loaded Ruby code as-needed, resulting in fast startup times.
- The autoloader also tracked which source files were loaded in which service, and watched for filesystem changes; if a loaded file changed on disk, any services using it would automatically restart to pick up the changes.

[autoload]: https://www.youtube.com/watch?v=lKMOETQAdzs

The net effect of this infrastructure was that developers could change code and then nearly immediately -- without any manual restart -- access a copy of the service running their new code, at a fixed, sharable (within Stripe) URL. This URL was stable even if they a provisioned a new devbox. Additionally, even though this feature worked for nearly all internal services, each devbox only ran the services that were actually in-use by that developer, reducing the CPU and memory load.

## The `pay` command

In addition to those HTTP endpoints, devprod built and maintained a `pay` command-line tool that offered unified access to a range of devbox features and workflows.

Devboxes could be and were accessed via `ssh`, but `pay` also wrapped some of the most common usage patterns to avoid the need.

For instance, `pay test ...` wrapped the common workflow of running automated tests, running specified tests on the devbox (using `ssh` under the hood) and piping output and exit status back to the laptop.

`pay curl` also wrapped `curl` and provided a helper to run manual `curl` commands against services on your devbox, which was often helpful when doing ad-hoc manual testing of API endpoints.

`pay typecheck` wrapped [Sorbet][sorbet], and ran the typechecker on the devbox.

In addition to the slightly convenience these tools provided, they were able to add value by being aware of the above sync process: Most commands that operated on source code would side-channel with the sync process (itself another `pay` subcommand -- `pay sync`) to ensure that synchronization was sufficiently up-to-date. This feature was an important component for making synchronization transparent. Historically, it was a common "gotcha" to edit a file and then start a test **before** synchronization had completed, resulting in running different code than you expected. By making `pay test` implicitly wait on the synchronization, this (extremely confusing!) mixup could be eliminated almost entirely.

This "sync barrier" piggybacked on watchman's [clock][watchman-clock] support. Using watchman's filesystem clock allowed `pay test` to check the current `watchman` timestamp and then ask the `pay sync` process "Have you synchronized to a timestamp equal to or greater than the one I observed?"

This meant that in the common case of editing a file and running a test, the timeline of events would look like so:

- The user edits and saves a file
- `pay sync` is woken up by `watchman`. It notes the filesystem clock, and starts an `rsync`
- The user starts a `pay test` command
- `pay test` asks `pay sync` to wait until synchronization is caught up
- In response, `pay sync` checks the filesystem clock again
- Because the filesystem has not changed since the earlier save, this timestamp matches one associated with the current `rsync`
- Thus, `pay sync` knows it only has to wait for the **already in-flight** `rsync` command before notifying `pay test`.

Using the `watchman` clock in this way allowed us to implement the "sync barrier" without any additional round-trips to the devbox. If the rsync had completed before the `pay test` command, then `pay test` could start immediately. We also made sure to configure ssh `ControlMaster` appropriately, so that each `ssh` invocation piggybacked on an existing session, additionally minimizing network round trips.

This `pay sync` side-channel also provided a useful mechanism to provide visibility to devprod about the health of the sync script. Developer laptops are laptops, and are expected to go offline and come back online regularly. That means it's normal for the synchronization process to get temporarily stuck while offline, and thus monitoring the health of synchronization is tricky. However, if a user runs a `pay` command that will work on the devbox, that represents a strong sign that the user **expects** synchronization to be healthy. Thus, if the IPC to the synchronization process fails or times out, that is an appropriate moment to report an error to Stripe's central exception tracker, allowing devprod to monitor the overall health of the synchronization system and the user experience.

[watchman-clock]: https://facebook.github.io/watchman/docs/cmd/clock

## LSP and editor tooling

As mentioned above, Stripe engineers could use any editor they chose; but by 2019 the Developer Productivity team had made the choice to invest in VS Code, declaring the preferred editor, and investing in editor and IDE tooling for VS Code users.

Some of this work consisted of internal plugins with small but meaningful quality-of-life affordances. For instance, VS Code users got the ability to launch a `pay test` command running the test under the cursor, and assorted other conveniences.

However, the more substantial improvements happened as [Sorbet][sorbet] -- Stripe's Ruby typechecker -- matured and gained an [LSP server implementation][sorbet-lsp] (LSP is the [Language Server Protocol][lsp], and defines an interface for a server to offer language-aware features such as "find definition" or autocomplete in a way that can be consumed by a variety of different editors).

Stripe configured VS Code to run the Sorbet LSP server on the devbox over `ssh`, communicating via stdin and stdout like a local LSP server. Doing this allowed Sorbet to make use of the devbox's large amount of RAM (LSP servers in general tend to be memory-hungry on large codebases, and Sorbet was no exception), and to run in a Linux environment that easier for the Sorbet team to monitor, debug and test.

In general, this approach worked fairly well; LSP servers generally must tolerate some latency, and so the additional network hop and delay due to file synchronization mostly did not pose problems. For instance, the LSP protocol is designed to handle the (extremely common case) where the user has made changes in the editor but not yet saved them to disk, and thus has the editor send relevant edits directly to the server. This process, in effect, also automatically provides robustness to some latency in file synchronization.

[sorbet]: https://sorbet.org/
[lsp]: https://microsoft.github.io/language-server-protocol/
[sorbet-lsp]: https://sorbet.org/docs/vscode#installing-and-enabling-the-sorbet-extension

# Reflections on context

In my opinion, the tooling described here was pretty neat, worked pretty well, and created an effective and productive development experience for many of Stripe's engineers.

That said, different organizations have different technical, social, and business constraints, and it's unlikely this system should be cloned identically by anyone else. I want to reflect here on a few of the details of Stripe's organization and technical stack that I think shaped the tooling I've outlined.

## Engineering organization size

The tooling I described here was developed over a range of time where Stripe's engineering team grew from a few hundred to over a thousand. While many of the components here existed in some form since Stripe's earliest days, the "mature" versions of these tools described here were built and maintained by Stripe's developer productivity team (often known as "devprod"), which consisted of perhaps 10 engineers by the end of my tenure.

I've described a lot of fairly-involved custom tooling; building that tooling made sense for Stripe, which was large enough that it made sense to invest a team in building it and, more importantly, in **maintaining** it. Stitching together a file watcher and `rsync` in order to synchronize files from a laptop to a remote devbox is easy -- a day's work at most for a basic version. The magic of `pay sync` was not the basic functionality, but that it was owned and maintained by a team that invested in observability and ongoing maintenance to make it nearly magical: something that **always** worked, and which users didn't have to think about. That work could only be supported by a team with the ongoing resources to maintain the tool.

Stripe was also still **growing** rapidly. In a rapidly growing organization, it's much more important to have a developer environment that "just works" out of the gate, with minimal setup pain for new engineers. That property was a large component of the motivation for the "devbox" model, for instance: Centrally managing the developer environments made it easier to centralize the work of ensuring they stayed healthy and functional.

## Codebase

This infrastructure existed to support a large monorepo with relatively consistent architecture and patterns. Having that basis created points of leverage, where the developer productivity team could create shared tooling (like the autoloader, and the devbox service manager) that provided fairly deep integration with the environment and codebase and offered user-friend features. In an environment that included a much-larger variety of languages, runtimes, or patterns, it doesn't make sense to specialize as much for any one, and infrastructure teams need to build more-generic tooling that treats user code as more of a opaque box.

Moreover, for various historical, business, and technical reasons, Stripe's codebase was fairly tightly coupled in a number of ways, such that it was clear to those of us on the developer productivity team that would not easily or rapidly be decomposed in any way (e.g. into a large number of diverse microservices). Thus, while we were continually investing in tooling to build stronger interface boundaries and abstractions, and allow individual teams and product services greater autonomy and separation, we also felt confident that the basic structure of the Ruby monorepo was not going to change drastically in any short timeframe, which also justified making relatively large investments in tooling to support it.

I will note that even at Stripe this choice was somewhat contentious and not always unanimous; even though the majority of the code and business value lived in the Ruby monorepo, Stripe still did have a large number of other codebases running infrastructure components or parts of specialized pipelines; these often received less support from the tooling and infrastructure teams and this was a recurring source of tension.

This monorepo, also, was written in Ruby specifically, which has a lot of implications on details of the tooling. In addition, Stripe's monorepo was (to our knowledge) the largest Ruby codebase in existence, which meant that existing tools were often not sufficient to support our scale or use cases, further motivating custom development.

# Closing thoughts

Maintaining developer productivity as an engineering organization grows is **hard**. It is almost inevitable that per-engineer productivity drops to some extent as an organization and codebase grows, even though it's nearly impossible to quantify that effect with any precision.

The challenges arise at all levels of the organization, and are social and organizational as often as they are technical. I will never claim to have all the answers or that Stripe found optimal or even "consistently good-enough" solutions to all aspects of this challenge. However, I do think that by 2019 or so, Stripe had invested enough in developer tooling that we had in many ways **improved** the median developer experience compared to many smaller stages of growth, and I've attempted to convey here the basic choices and infrastructure supporting that experience. The development experience, of course, is only part of the story: the full lifecycle of code and of a feature continues onward into CI and deployment into production, where it will be further observed and debugged; writing about those systems would be at least one more post, equal to this one in length.

{{%comment%}}
<!--
sequenceDiagram
  participant watchman
  participant devbox
  participant Sync as pay sync
  # participant pay test
  actor user

  note over user: Save and edit file
  note  over watchman: observe change
  watchman ->> Sync: notify of file change
  Sync -x watchman: check logical clock: T_0
  Sync -) +devbox: Start sync as-of T_0
  create participant pay test
  user->>pay test: Run test command
  pay test -) +Sync: start sync barrier
  Sync -x watchman: check logical clock: T_0
  devbox -) -Sync: sync completes
  Sync -) -pay test: sync barrier complete
  pay test->> +devbox: Run test via `ssh`
  devbox ->> -user: Stream output
  -->

<!--  LocalWords:  efs nz autoload lsp
 -->
{{% /comment %}}
