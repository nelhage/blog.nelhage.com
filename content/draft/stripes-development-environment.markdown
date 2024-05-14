---
title: "Stripe's monorepo developer environment"
date: 2023-12-14T11:00:00-08:00
slug: stripe-dev-environment
---

I worked at Stripe for about seven years, from 2012 to 2019. Over that time, I used and contributed to many generations of Stripe's developer environment -- the tools that engineers used daily to write and test code. In my opinion Stripe did a pretty good job designing and building that developer experence, and since leaving, I've found myself repeatedly describing features or design choices to friends and collaborators.

This post is an attempt to record the salient features of that environment as I remember it. I'll also try to reflect on the context, constraints, and motivations for some of these choices; while I think they were generally good ones, they were deeply informed by the business and technical context, and it's unlikely anyone else should clone them identically.

Some caveats: It's been nearly five years, and I have no doubt that I have misremembered some of the specific details, even though I'm confident in the overall picture. I'm also certain that Stripe has continued evolving and I make no claim this document represents the developer experience at Stripe as of today.

In addition: I contributed to many of the components described here, but I can't and don't try to claim any overall credit for this system. Everything here was evolved over the course of many years, with contributions from many talented engineers, both to craft the vision, and to implement it.

# Technical and organizational context

First, some background for the tools I'll describe. At the time I'm discussing, most of Stripe's codebase was written in Ruby, and lived in a single large monorepo. This codebase supported a number of services, which shared code extensively. Stripe also had a large and growing number of services outside the monorepo, and often in another language, but they were (as a generalization) less core to the business (concretely: the Stripe API was still almost entirely a single Ruby service in the monorepo). The tooling and experience I discuss here were designed and built primarily to support development in the monorepo. Other services and languages were supported incidentally if at all.

Stripe's tooling was built and maintained by a succession of teams and individuals, but I'll collectively refer to its authors and owners as the "developer productivity" team ("devprod" for short), which was the name of that team for the last few years of my tenure.

## Investing in tooling

On the note of the that team: Stripe initally created a team with headcount dedicated to internal tooling and productivity, fairly early on in my time, but in the last three or four years it grew substantially and really came into its own. This team was also consistently staffed with a excellent, senior, engineers. More so than any of the specific choices I'll outline, I think the existence of the team, and its investment in **stability** and **reliability** of developer tooling (a bit later on, once the team had grown enough to put out immediate fires and have the room to plan and invest) were the major drivers in any success of the Stripe developer experience. Technical choices and engineering absolutely matter, but they need to be supported with enough headcount and with adequence **ongoing maintenance** in order to work.

This reliability and stability of the environment, especially as the team and codebase grew and evolved, will be recurring theme. The best tooling in the world will struggle to make up for "regularly losing a day debugging your environment," and so getting that aspect right can easily outweigh almost any other decisions you make. Many of the design choices were made with the goal of enabling the devprod team to more-easily and consistently support and monitor the tooling, and allow for centralized debugging and resolution of issues instead of pushing it onto individual engineers.

# Architecture of the developer environment

## Development code runs in the cloud

Here's one defining question for a developer environments: Does code under development run locally on the developer's laptop, or does it need to be run remotely somewhere (often on instances or containers inside the organization's cloud environment)?

For many years, Stripe engineers made use of both options, depending on developer preference and idiosyncratic decisions and details of what worked in which environment when. When the developer productivity team made the decision to invest in a single blessed environment, we settled on running code in the cloud. An EC2 instance (outside of the production VPC, of course) was allocated to each developer (a "devbox" for short). During development, code ran on that instance, both for running `minitest` tests and for interactive testing and experimentation.

These developer instances were provisioned by Stripe's standard configuration management tooling, and were ephemeral -- a single command could destroy your instance and provision a new one (a number of warm spares were kept, making this operation typically ~instantaneous). A shared registry kept track of which instance was active for which engineer, for consumption by tooling.

This design meant that most environment-configuration issues could be solved centrally by tooling teams or service owners, and developers mostly did not have to worry about updating their own development environments to keep up.

All devboxes were accessible (via `ssh` and the HTTP endpoints I'll talk about shortly) to all engineers. This both made it easier to demo work-in-progress, and also made devboxes **much** easier to support: A teammate could log in to help you debug an issue they've already encountered, and the devprod team could debug issues and roll out fixes centrally. If one team was added a service dependency that was breaking another team's environment, they could help debug it directly with a member of that team.

## Editors and source control

Before you can run new code under development, you need to actually access code via source control and edit it.

At Stripe, even though code **ran** in the cloud, git checkouts and editors lived locally, on developer laptops. Stripe settled on this approach for a number of reasons, including:

- Support for a wide variety of editor and IDE environments

  Some editors (like `emacs` or `vim`) can be run reasonably-well over an SSH session, and some (like VS Code or emacs with [`tramp`][tramp]) support running a local UI against a remote filesystem, but many do not. By keeping code on the laptop, Stripe allowed developers to continue using any editor they wanted.
- Latency

  Stripe had developers around the globe, but at the time did not yet maintain substantial infrastructure outside of the US. Editing code on the other side of a 200+ms network latency is **painful**. Standing up devboxes globally was an option but would have been complicated for numerous reasons.
- Source code durability

  By keeping the source-of-truth on laptops and off of the devbox, it was much easier to treat the execution environment as ephemeral and transient. That property, in turn, was very helpful for keeping environments up-to-date and preventing drift over time.

  Source code could have been kept on a network-accessible store separate from the devbox (e.g. [EFS][efs] or an EBS volume), but aside from the complexity cost, those options have relatively high latency, and source code tends to be dependent on (at least) fast file-metadata lookups; `git` operations over NFS or on EBS tend to be painfully slow without heroic levels of tuning and optimazation.

[tramp]: https://www.gnu.org/software/tramp/
[efs]: https://aws.amazon.com/efs/

### Automatic synchronization

Keeping code on laptops, and executing in the cloud, though, poses a new challenge: How does code, once edited, **get**  from the laptop onto the devbox?

Even before I joined, Stripe had maintained a "sync" script that glued together a file-watcher with `rsync` for this purpose; it would monitor your local checkout for changes and copy them to a devbox in the cloud. Historically, developers would manually start and monitor this script in an ad-hoc way in a separate terminal or `tmux` window.

Eventually, the developer productivity team took ownership of that tool and invested in it heavily, largely with the goal of adding "polish" and making it seamless and nearly-invisible for other developers.

Notably, they made syncing happen implicitly and with no configuration or intervention required: They worked with the IT team to install the sync script as a `launchd` service on every developer laptop, and used the above-mentioned registry to automatically discover the correct devbox.

They also invested heavily in reliability and self-healing on errors and network problems. One "internal" but significant improvement involved migrating to [watchman][watchman] for file-watching: In our testing , it was by far the most robust file-watcher we found, vastly reducing the frequency of missed updates or a "stuck" file-watcher. We were also able to use some of its more-advanced features, which I'll discuss in a bit.

[watchman]: https://facebook.github.io/watchman/

On the whole, these investments largely succeeded: most developers were able to treat code sync as simply a "fact of life," and rarely had to think about or debug it.

One downside of synchronization is that it makes it harder to write "code that acts on code," such as automated migration tools, or even just linters or code-generation tools (Stripe ended up relying on a handful of code-generated components which needed to be checked in for various reasons). This remained a bit of a pain point through my tenure; we made do with a mix of strategies:
- Run on the developer laptop, and deal with the environment challenges
- Run on the devbox and then somehow "sync back" generated files. We had a small protocol to run scripts in a "sync-back" wrapper, s.t. they could ask for file to be copied back to the laptop, but it remained somewhat awkward and non-ergonomic and occasionally unreliable.

## HTTP services on the devbox

Much of the code at Stripe executed inside of HTTP services, including notably the Stripe API. The developer productivity team built tooling to support developing these services, including:

- DNS resolved hostnames along the lines of `*.$username.stripe-dev.com` to that user's current devbox
- A frontend service ran on each devbox that terminated SSL, handled any auth[nz] based on client certificates, and mapped service names to the appropriate local instance
  - Each service at Stripe was statically assigned a local port, used by the frontend to route traffic.
  - For instance, the API service had some assigned port, and the frontend would forward `api.nelhage.stripe-dev.com` there
- A separate proxy service listened on each of those ports, and demand-started the backing Ruby service.
  - Thus, on first request to `api.nelhage.stripe-dev.com`, the API service would start up, and then remain running to handle future requests directly.
- Those services used Stripe's [autoloader][autoload] on the devbox, and thus only loaded Ruby code as-needed, resulting in fast startup times.
- The autoloader also tracked which source files were loaded in which service, and watched for filesystem changes; if a loaded file changed on disk, any services using it would automatically restart to pick up the changes.

[autoload]: https://www.youtube.com/watch?v=lKMOETQAdzs

The net effect of this infrastructure was that developers could change code and then nearly immediately -- without any manual restart -- access a copy of the service running their new code, at a fixed, sharable (within Stripe) URL. This URL was stable even if they a provisioned a new devbox. Additionally, even though this feature worked for nearly all internal services, each devbox only ran the services that were actually in-use by that developer, reducing the CPU and memory load.

## The `pay` command

Devprod also built and maintained a `pay` command-line tool that offered unified access to a range of devbox features and workflows.

Devboxes could be and were accessed via `ssh` (encapsulated as `pay ssh`), but `pay` also wrapped the most common usage patterns:

- `pay test ...` would execute `minitest` tests on the devbox (using `ssh` under the hood), and piping back output and exit status locally.
- `pay curl` wrapped `curl` and provided a helper to run manual `curl` commands against services on your devbox, which was often helpful when doing ad-hoc manual testing of API endpoints.
- `pay typecheck` wrapped [Sorbet][sorbet] to typecheck code on the devbox

In addition to the convenience these tools provided, they were aware of the code-sync process: Most commands that operated on source code would side-channel with the sync process to ensure that synchronization was appropriately up-to-date. This feature was a critical improvement to the transparency of synchronization: Historically, it was a common "gotcha" to edit a file and then start a test **before** synchronization had completed, and thus run tests against and old version while **believing** you were testing your new code. Often this results in a false belief that a change didn't work, and can result in wild goose chases and immense frustration.

By initiating workflows from the laptop, `pay` subcommands were guaranteed an up-to-date view of the filesystem; by colluding with synchronization it could ensure this state was propagated to the devbox,

The straightforward approach to such a "sync barrier" requires (e.g) `pay test` to trigger a new synchronization, which adds frustrating latency to workflows, even if you haven't changed anything at all. By making use of watchman's [clock][watchman-clock] feature, we were able to do better and avoid almost-all unnecessary roundtrips. In the common case of editing a file and running a test, the timeline of events would look like:

- The user edits and saves a file
- `pay sync` is woken up by `watchman`. It notes the filesystem clock, and starts an `rsync`
- The user starts a `pay test` command
- `pay test` asks `pay sync` to wait for synchronization to catch up
- In response, `pay sync` first checks the filesystem clock once again
- Because the filesystem has not changed since the earlier save, this timestamp matches one associated with the current `rsync`
- `pay sync` now knows it only has to wait for the **already in-flight** `rsync` command, prior to notifying `pay test`

We also made sure to configure ssh `ControlMaster` and other `ssh` settings, in order to reuse connections where possible.

This `pay sync` side-channel also provided a useful location for visibility into the health of the synchronization process. Developer laptops are laptops, and routinely go offline and come back online. Thus, synchronization failures, or even long periods where synchronization lags, may be normal, making it difficult to usefully collect error statistics from the sync script. However, if a user runs a `pay` command that will work on the devbox, that represents good evidence that the user **expects** synchronization to be healthy. Thus, if the sync-barrier call to the synchronization process fails or times out, that is an appropriate moment to report an error to Stripe's central exception tracker. That report, in turn, allowed devprod to monitor the overall health of the synchronization system and the user experience.

[watchman-clock]: https://facebook.github.io/watchman/docs/cmd/clock

## LSP and editor tooling

As mentioned above, Stripe engineers could use any editor they chose, but by 2019 the developer productivity team had made the choice to invest in VS Code. They declared that the preferred editor, and invested in tooling for that environment in particular.

Some of this work consisted of internal plugins with small but meaningful quality-of-life improvements, such as support for running `pay test` on the specific test under the cursor.

However, more substantial improvements happened as [Sorbet][sorbet] -- Stripe's Ruby typechecker -- matured and gained an [LSP server implementation][sorbet-lsp] (LSP is the [Language Server Protocol][lsp], and defines an interface for a server to offer language-aware features such as "find definition" or autocomplete in a way that can be consumed by a variety of different editors).

Stripe configured VS Code to run the Sorbet LSP server on the devbox over `ssh`, communicating via stdin and stdout like a local LSP server. Doing this allowed Sorbet to make use of the devbox's large amount of RAM (LSP servers in general tend to be memory-hungry on large codebases, and Sorbet was no exception), and to run in a Linux environment that easier for the Sorbet team to monitor, debug and test.

In general, this approach worked fairly well; LSP servers generally must tolerate some latency, and so the additional network hop and delay due to file synchronization mostly did not pose problems. For instance, the LSP protocol is designed to handle the (extremely common case) where the user has made changes in the editor but not yet saved them to disk, and thus has the editor send relevant edits directly to the server. This process, in effect, also automatically provides robustness to some latency in file synchronization.

[sorbet]: https://sorbet.org/
[lsp]: https://microsoft.github.io/language-server-protocol/
[sorbet-lsp]: https://sorbet.org/docs/vscode#installing-and-enabling-the-sorbet-extension

# Reflections

In my opinion, the tooling described here was pretty neat, worked pretty well, and created an effective and productive development experience for many of Stripe's engineers. I want to reflect here on a few of the details of Stripe's organization and technical stack that I think shaped the tooling; these are areas likely worth considering if you're working on developer tooling at a different organization.

## Engineering organization size

The tooling I described here was developed over a range of time where Stripe's engineering team grew from a few hundred to over a thousand. Over that time, devprod grew from less than one FTE into a team of perhaps a dozen engineers. Much of the tooling described here was built near that latter scale, when devprod was 3+ engineers.

This scale -- the scale of devprod, and in turn the scale of the overall organization, such that it could afford 10 FTEs on tooling -- was a major factor in our choices. I've described a lot of fairly-involved custom tooling; we needed enough engineers to build **and maintain** it, and enough "customer" engineers for that investment to pay off. I will again stress maintenance and reliability: Stitching together a file watcher and `rsync` in order to synchronize files from a laptop is easy -- the first version existed by 2012 and represented under a day of initial investment. The eventual magic was not that such a tool existed, but that it was reliable enough, and stayed reliable enough across growth and changing needs, that you **didn't have to think about it** as a user.

In additionl, Stripe was also **growing** rapidly. In a rapidly growing organization, it's much more important to have a developer environment that "just works" out of the gate, with minimal setup pain for new engineers. In a more-static organization, engineers can learn the quirks of the tools and how to work around them, and that's a mostly-one-time cost. In one that's rapidly growing, those costs are constantly borne by every new hire and by the engineers training them, and the return to stability and seamlessness go up.

This property was a large component of the motivation for the "devbox" model. Local development on the laptop has advantages, but it's much harder to manage the environment centrally or help users debug, and so it almost-inevitably requires more expertise and involvement from individual developers. Moving to centrally-provisioned, ephmeral devboxes, let us centralize most of that maintenance, and let us recommend a blanket "pull `master` and re-create your devbox" as first-line debugging advice that resolved most problems.

## Codebase

I've mentioned that this infrastructure supported a large monorepo with relatively consistent architecture and patterns, both across time and across the codebase. Having that stable target created points of leverage, where the developer productivity team could create shared tooling (like the autoloader, and the devbox service manager) that provided fairly deep integration with the environment and codebase and offered user-friendly features. In an environment that included a much-larger variety of languages, runtimes, patterns, it doesn't make sense to specialize as much for any one, and infrastructure teams need to build more-generic tooling that treats user code as more of a opaque box.

Moreover, for various historical, business, and technical reasons, Stripe's codebase was fairly tightly coupled in a number of ways; it was clear to those of us on the developer productivity team that it would not easily or rapidly be decomposed in any way (e.g. into a larger number of microservices). We did continually invest in tools and patterns for modularity and abstraction within the monorepo, but the confidence in the overall "stickiness" of the structure of the Ruby monorepo also justified our large investments in specialized tooling.

I will note that even at Stripe this choice was somewhat contentious and not always unanimous. Even though the majority of the code and business value lived in the Ruby monorepo, Stripe still did have a large number of other codebases running infrastructure components or parts of specialized pipelines; these generally received less support from the tooling and infrastructure teams, which was a recurring source of tension.

The fact that the monorepo was written in Ruby, specifically, also shaped a lot of our decisions in detail. A compiled language with more-decoupled source and artifacts may have pushed us in other directions, for instance. In addition, Stripe's monorepo was (to our knowledge) the largest Ruby codebase in existence, meaning we were often on our own and that existing tooling struggled to support our scale in one way or another.

## Closing thoughts

Maintaining developer productivity as an engineering organization grows is **hard**. It is almost inevitable that per-engineer productivity drops to some extent as an organization and codebase grows, even though it's also nearly-impossible to quantify that effect.

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
