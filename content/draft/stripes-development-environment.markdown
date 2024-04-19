---
title: "Stripe's developer environment"
date: 2023-12-14T11:00:00-08:00
slug: stripes-dev-environment
---

# Introduction

I worked at Stripe for about seven years, from 2012 to 2019. During that time, I saw and contributed to many generations of Stripe's developer environment and developer tools -- the tools that engineers used daily to write and test code while working at Stripe.

Since leaving, I've found myself repeatedly describing features or design choices of that environment to friends and collaborators; this post is an attempt to describe the salient features of that environment as I remember it, because I think Stripe built something pretty good, and it's worth learning from. I'll also try to include some notes about the context, constraint, and motivations; It's worth learning from others' experiences, but in doing so it is critical to try to understand the constraints and environment that motivated them: it is very unlikely that a different organization will want to clone the system I describe wholesale.

Some caveats before I jump in: It's been nearly five years, and I have no doubt that I have misremembered some of the specific details, even though I'm confident in the overall picture. I'm also certain that Stripe has continued evolving and I make no claim this document represents the developer experience at Stripe as of right now.

And, finally, while I contributed to a number of the components described here, I can't and don't try to claim credit for this system or its design. Everything here was evolved over the course of many years, with contributions from many talented engineers, both to the overall design and vision and to the implementation.

# Stripe's monorepo development environment

I'll say more later, but some key context-setting: At the period I'm discussing, most of Stripe's codebase was written in Ruby, and lived in a single large monorepo. This codebase supported a number of services, which shared code extensively. Stripe did also have a large and growing number of services that did not live in the monorepo and were not implemented in Ruby, but they were in general less core to the business (concretely: the Stripe API was still almost-entirely in Ruby inside of the monorepo). The tooling and experience I discuss here were designed and built primarily to support development in the monorepo; other services and languages were supported incidentally if at all by this tooling.

Stripe's tooling was built and maintained by a succession of teams and individuals, but I'll collectively refer to the authors and owners of this tooling as the "Developer Productivity Team" or "devprod," which was the name of that team for the last few years of my tenure.

## Development code runs in the cloud

Every developer had a personal development instance ("devbox" for short) in Stripe's AWS environment (entirely disconnected from production, of course). During development, code ran on that instance (as opposed to on the developer laptop), be that for automated tests or for interactive testing of new code.

These developer instances were centrally-managed and provisioned, and were ephemeral -- a single command would destroy your devbox, provision a new one, and mark it as assigned to you. This design meant that most environment-configuration issues could be solved centrally by tooling teams or service owners, and developers mostly did not have to worry about updating their own development environments to keep up.

All devboxes were accessible (via `ssh` and the HTTP endpoints I'll talk about shortly) to all engineers, making it easy to show off work-in-progress or help a teammate debug environment or configuration issues. In particular, devprod could debug user issues directly, or a team that was adding a new service dependency could debug issues that arose with their development configuration.

## Editors and source control

I've talked about running code during development, but of course that's only relevant once you've actually gotten access to the code from source control, and made some changes. How did engineers at Stripe actually edit source and interact with `git`?

At Stripe, even though code **ran** in the cloud, git checkouts and editors lives locally, on developer laptops. Stripe settled on this approach for a number of reasons, including:

- Support for a wide variety of editor and IDE environments. Some editors (like `emacs` or `vim`) can be run reasonably-well over an SSH session, and some (like VS Code or emacs with [`tramp`][tramp]) support running a local UI against a remote filesystem, but many do not. By keeping code on the laptop, Stripe allowed developers to continue using any editor they wanted.
- Latency. Stripe had developers around the globe, but at the time did not maintain substantial infrastructure outside of the US. Editing code on the other side of a 200ms (or worse!) network latency is **painful**. Standing up more EC2 regions might have been an option, but would have brought many other complications.
- Source code durability. By keeping the source-of-truth on laptops and off of the devbox, it was much easier to treat the execution environment as ephemeral and transient. That property, in turn, was very helpful for keeping environments up-to-date and preventing drift over time.
  - This could could also have been achieved by keeping code on a network-accessible mount (e.g. [EFS][efs]) or even an EBS volume, but those also would add complexity, and many such options imposed painful latency on `git` and on service startup, both of which rely on large numbers of rapid filesystem accesses.

[tramp]: https://www.gnu.org/software/tramp/
[efs]: https://aws.amazon.com/efs/

Keeping code on laptops, and executing in the cloud, though, poses an obvious challenge: How do we **get** code from the laptop onto the devbox?

Since even before I had started, Stripe had maintained a "sync" script that glued together a file-watcher with `rsync` for this purpose; it would monitor your local checkout for changes and copy them to a devbox in the cloud. Historically, developers would manually start and monitor this script in an ad-hoc way in a separate terminal window or `tmux` pane.

Eventually, though, the Developer Productivity team took ownership of that tool and invested in it heavily, largely of the form of adding "polish" and with the aim of making it seamless and nearly-invisible.

Notably, they made syncing happen implicitly and with no configuration or intervention required: They worked with the IT team to use Stripe's laptop provisioning and management tools to install the sync script as a `launchd` service on every developer laptop, and used the above-mentioned registry to automatically discover the correct devbox.

Other polish work included migrating the sync script to use [watchman][watchman] for file-watching; In our testing, `watchman` was by far the most robust file-watcher we found, vastly reducing the frequency of missed updates or a "stuck" file-watcher. It also offered some advanced features that Stripe made use of -- more about that later.

[watchman]: https://facebook.github.io/watchman/

On the whole, these investments largely succeeded: most developers were able to treat code sync as simply a "fact of life," and rarely had to think about or debug it.

## HTTP services on the devbox

The developer productivity team built a plethora of tools to support working with devboxes. Most of these made use of a central registry which maintained an authoritative mapping of which devboxes were active and belonged to which users. A lot of tooling supported developing HTTP services, which accounted for a majority of the code at Stripe:

- DNS resolved hostnames along the lines of `*.$username.stripe-dev.com` to that user's current devbox
- A frontend service ran on each devbox terminating SSL, handled any auth[nz] needed, and mapping service names to the appropriate local instance
  - Each service at Stripe was statically assigned a local port, used by the above-mentioned frontend to route traffic.
  - For instance, the API service had some assigned port, and the frontend forwarded `api.nelhage.stripe-dev.com` there
- A separate proxy service actually listened on each of those ports, and demand-started the backing Ruby service.
  - Thus, on first request to `api.nelhage.stripe-dev.com`, the API service would start up, and then remain running to handle future requests directly.
- Those services used Stripe's [autoloader][autoload] on the devbox, in order to only loaded Ruby source code as-needed in order to speed startup time
- The autoloader also tracked which source files were loaded in which service, and then watched them using `inotify`; if a loaded file changed on disk, any services using it would be automatically restarted to pick up the changes

[autoload]: https://www.youtube.com/watch?v=lKMOETQAdzs

The net effect of this infrastructure was that developers could change code and effectively-immediately -- without any manual restarts -- access a copy of the service running their new code, at a fixed, shareable (within Stripe) URL. This URL was stable even if they a provisioned a new devbox. Additionally, even though this feature worked for nearly all internal services, each devbox only ran the services that were actually in-use by that developer, reducing the CPU and memory load.

In addition, dependencies -- both Ruby gem dependencies, but also any external services (databases, etc) or credentials, were centrally-managed on the devboxes by Stripe's standard configuration management. This meant that, for the most part, developers who were "just writing Ruby code" would not have to think about or debug environment or configuration issues. If they did run into problems, in most cases they could resolve them by creating a new devbox with updated configuration, which was itself a fast and mostly hassle-free operation.

## The `pay` command

In addition to those HTTP endpoints, devprod built and maintained a `pay` command-line tool that offered unified access to a range of devbox features and workflows.

Devboxes could be accessed via `ssh`, and direct access was widely used, but `pay` also wrapped some of the most common usage patterns to avoid the need.

For instance, `pay test ...` wrapped the common workflow of running automated tests, running specified tests on the devbox (using `ssh` under the hood) and piping output and exit status back to the laptop.

`pay curl` also wrapped `curl` and provided a helper to run manual `curl` commands against services on your devbox, which was often helpful when doing ad-hoc manual testing of API endpoints.

In addition to the slightly convenience these tools provided, they were able to add value by being aware of the above sync process: `pay test` would side-channel with the sync process (itself another `pay` subcommand -- `pay sync`) to ensure that synchronization was sufficiently up-to-date. This feature was another important part of making synchronization invisible; Historically, it had been a common "gotcha" to edit a file and then switch to a terminal and run a test, but for that test to start **before** the synchronization had landed, such that you were not actually running your edited code. By making `pay test` implicitly wait on the synchronization, this (extremely confusing!) mixup could be eliminated almost entirely.

This "sync barrier" piggybacked on watchman's [clock][watchman-clock] support. Using watchman's filesytem clock allowed `pay test` to check the current `watchman` timestamp and then ask the `pay sync` process "Have you synchronized to a timestamp equal to or greater than the one I observed?"

Since `pay sync` was running a filesystem watcher, in the common case where you edit a file and then run `pay test`, the synchronization process has already woken up and started a sync by the time `pay test` starts. Using the `watchman` clock meant that `pay test` could "piggyback" on this sync (and avoid waiting at all, if that synchronization is already complete), instead of having to trigger a new sync. Thus, in the usual case where synchronization is healthy, `pay test` could offer this liveness guarantee with near-zero added latency.

This side-channel also provided a useful mechanism to provide visibility to devprod about the health of the sync script. Developer laptops are laptops, and are expected to go offline and come back online regularly That means it's normal for the synchronization process to get temporarily stuck and not make progresss, and thus it's very noisy to try to report and triage errors from the synchronization proess. However, if a user is running `pay test`, that's a fairly clear sign of **intent** that the developer **expects** synchronization to be working; if the synchronization barrier discovers that `pay sync` has crashed, or if it times out, that would report an exception to a central error tracker, allowing devprod to monitor overall healthy of the synchronization system and investigate if needed.

[watchman-clock]: https://facebook.github.io/watchman/docs/cmd/clock

## LSP and editor tooling

As mentioned above, Stripe engineers could use any editor they chose; but by 2019 the Developer Productivity team had made the choice to invest in VS Code, declaring the preferred editor, and investing in editor and IDE tooling for VS Code users.

Some of this work consisted of internal plugins with small but meaningful quality-of-life affordances. For instance, VS Code users got the ability to launch a `pay test` command running the test under the cursor, and similar affordances.

However, the more substantial improvements happened as [Sorbet][sorbet] -- Stripe's Ruby typechecker -- matured and gained an LSP server implementation. Stripe configured VS Code to run the Sorbet LSP server on the devbox over `ssh`, communicating via stdin and stdout like a local LSP server. Doing this allowed Sorbet to make use of the devbox's large amount of RAM (LSP servers in general tend to be memory-hungry on large codebases, and Sorbet was no exception), and to run in a Linux environment that wasier for the Sorbet team to monitor, debug and test.

In general, this approach worked fairly well; LSP servers generally must tolerate some latency, and so the additional network hop and delay due to file synchronization mostly did not pose problems. For instance, the LSP protocol is designed to handle the (extremely common case) where the user has made changes in the editor but not yet saved them to disk, and thus has the editor send relevant edits directly to the server. This process, in effect, also automatically provides robustness to some latency in file synchronization.

[sorbet]: https://sorbet.org/
[sorbet-lsp]: https://sorbet.org/docs/vscode#installing-and-enabling-the-sorbet-extension

# Reflections on context

In my opinion, the tooling described here was pretty neat, worked pretty well, and created an effective and productive development experience for many of Stripe's engineers.

That said, I would not recommend that anyone treat this writeup as a playbook or uncritically attempt to recreate the same experience! The systems described here were desiged in a specific context for a specific organization with a specific codebase and environment; different contexts would almost certainly call for **some** different choices. I want to reflect here on a few of the details of Stripe's organization and technical stack that I think heavily shaped the choices I've outlined.

## Team scale

The tooling I described here was developed over a range of time where Stripe's engineering team grew from perhaps two hundred to over a thousand. While many of the components here existed in some form since Stripe's earliest days, the "mature" versions of these tools desscribed here were built and maintained by Stripe's developer productivity team (often known as "dev-prod"), which consisted of perhaps 10 engineers by the end of my tenure.

I've described a lot of fairly-involved custom tooling; building that tooling made sense for Stripe, which was large enough that it made sense to invest a team in building it and, more importantly, in **maintaining** it. Stitching together a file watcher and `rsync` in order to synchronize files from a laptop to a remote devbox is easy -- a day's work at most for a basic version. The magic of `pay sync` was not the basic functionality, but that it was owned and maintained by a team that invested in observability and ongoing maintenance to make it nearly magical: something that **always** worked, and which users didn't have to think about.

## Codebase

This infrastructure existed to support a large monorepo with relatively consistent architecture and patterns. Having that basis created points of leverage, where the developer productivity team could create shared tooling (like the autoloader, and the devbox service manager) that provided fairly deep integration with the environment and codebase and offered user-friend features. In an environment that included a much-larger variety of languages, runtimes, or patterns, it wouldn't make sense to specialize as much for any one, and infrastructure teams need to build more-generic tooling that treats user code as more of a opaque box.

Moreover, for various historical, business, and technical reasons, Stripe's codebase was fairly tightly coupled in a number of ways, such that it was clear to those of us on the developer productivity team that would not easily or rapidly be decomposed in any way. Thus, while we were continually investing in tooling to build stronger interface boundaries and abstractions, and allow individual teams and product services greater autonomy and separation, it was also clear that the basic code architecture was here to stay for some time, which also justified making relatively large investments in closely-coupled tooling.

I will note that even at Stripe this choice was somewhat contentious and not always agreed with; even though the majority of the code and business value lived in the Ruby monorepo, Stripe still did have a large number of other codebases running infrastructure components or parts of specialized pipelines; these often received less support from the tooling and infrastructure teams and this could be a source of tension.

This monorepo also, of course, was written in Ruby specifically. That choice heavily informed the details of the tooling described here, especially around the autoloader and service models in development. Stripe's monorepo was (to our knowledge) the largest Ruby codebase in existence, which meant that existing tools were often not sufficient to support our scale or use cases, further motivating custom development.

# Closing thoughts

Maintaining developer productivity as an engineering organization grows is **hard**. It is almost inevitable that per-engineer productivity drops to some extent as an organization and codebase grows, even though it's nearly impossible to quantify that effect with any precision.

The challenges arise at all levels of the organization, and are social and organizational as often as they are technical. I will never claim to have all the answers or that Stripe found optimal or even "consistently good-enough" solutions to all aspects of this challenge. However, I do think that by 2019 or so, Stripe had invested enough in developer tooling that we had in many ways **improved** the median developer experience compared to many smaller stages of growth, and I've attempted to convey here the basic choices and infrastructure supporting that experience. The development experience, of course, is only part of the story: the full lifecycle of code and of a feature continues onward into CI and deployment into production, where it will be further observed and debugged; writing about those systems would be at least one more post, equal to this one in length.
