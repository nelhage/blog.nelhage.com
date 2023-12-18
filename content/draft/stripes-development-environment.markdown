---
title: "Stripe's monorepo developer environment"
date: 2023-12-14T11:00:00-08:00
slug: stripes-dev-environment
---

# Introduction

I worked at Stripe from 2012 to 2019. During that time, I saw and contributed to many different generations of Stripe's developer environment -- the ways that developers wrote and tested code within Stripe's monorepo.

This post is a description of (my memory of) that environment and the accompanying tooling, as of when I left four years ago. I'll also try to discuss some of the tradeoffs and motivations that led to these choices; different environments with different constraints likely **should** make different choices, and understanding the motivation and forces shaping a system is important in order to effectively learn from it.

Some cavaats before I jump in: It's been four years, and I have no doubt that I have misremembered or imagined some of the specific details, even if I'm confident in the overall picture. I'm also certain that the excellent teams at Stripe have continued evolving their tooling since then, such that document is certainly out of date with regard to today's Stripe.

And, finally, while I contributed to some of the components described here, I want to be clear I do not claim credit for this system or its design -- everything here was evolved over the course of many years, with contributions from many, many talented individuals, both in vision-and-design, and in implementation.

## Context

<!--
Some of the basic constraints here are:
An M1 or M2 core has faster single-threaded performance than any machine any cloud provider will sell you for any amount of money
But cloud nodes can cost-effectively provider many have more cores than a laptop
And much, much, more RAM
Cloud dev will be much closer in network distance and bandwidth to github and the cluster and other remote services
Containerized local dev on a mac means running in a VM, which means:
1. Your editor and your local commands (coo etc) are divided by what's effectively a network filesystem (albeit running over a virtual machine bridge, not a real network), which inevitably has bad latency properties, and which -- for every implementation I know of to date -- does not tend to support file-watching APIs well
2. The VM and your host environment share the limited memory resources on your laptop. Modern VM technology mean they don't have to statically partition it any more, but they still compete for a very limited resource
Remote dev tends to mean either:
Ephemeral container/VM instances, which is tough for "store of record for a developer's code" OR
Code lives across the network from the remote compute, which means network latency on fs access, and, again, usually means poor inotify support


Looking at these constraints, the point Stripe settled on was:
Remote development machines provisioned in the cloud environment
devboxes are ephemeral and can be nuked-and-re-created at any time
devboxes log into Splunk and dev-prod team members can log into your devbox and help debug environment issues
An internal frontend exposes stable hostnames -- e.g. api.nelhage.stripe.dev -- that always route to "your active devbox" to access services running on your devbox, and do appropriate auth[nz]
A proxy running on every devbox demand-starts all HTTP services in the monorepo, so that on first access to api.nelhage.stripe.dev, the API service is autostarted and traffic routes to it
All such services are run in an environment where (a) all imports are lazy, for faster startup, and (b) servers watch (via inotify ) every file that has been imported, and restart if any of them change
Developers' checkouts and editors live on their laptops
This is the lowest-latency option for the all-important editor UX, and also for local git commands like checkout or commit
It does not help with networked git performance but git can scale fairly well with some work, and also git fetches and pulls are less "in the tight loop" of developer productivity than "using your editor"
A coo sync-like syncing experience syncs code from the laptop to the devbox
This is centrally provisioned by the org's laptop management tooling and is always-on in the background and does not need to be managed or started by developers. It reports to Sentry or equivalent and dev-prod triages issues and does their best to ensure you never have to think about it.
Code in the monorepo (aside from the shim commands mentioned here, like the syncer) never runs on laptops. I
t's only ever run on the remote dev node.
Language dependencies etc are managed on that node by dev-prod-managed tooling and configuration.
Dev-prod provides a pay command that is a unified frontend to the vast majority of day-to-day operations that developers do in the codebase
e.g. pay test wraps the pytest equivalent, but transparently runs on the devbox
pay curl makes CLI requests to web services on your devbox
pay commands collude with the syncer, so when you run pay test it transparently waits until the syncer is caught up with the state as-of the pay test launch before executing code, so that -- as long as you're using pay commands -- you should never see stale / unsynced code on the devbox.
This also means that if the syncer is down it can raise an error (and, again, log to Sentry so dev-prod can triage these in an ongoing fashion)


Anthropic has different constraints and different goals and different problems so I would not adopt this exact same solution; it also took a lot of work to develop and evolved over time. But I do think it was actually, like, almost-uniquely-forced by the premises we chose to adopt and the needs of the organization, and also was absolutely fucking magically good once we were staffed to the point of building and supporting it properly.


Oh also: Our LSP server (Sorbet) ran on the devbox, and VSCode talked to it over ssh. This gave it the benefit of running in a controlled environment where the Sorbet authors could test and debug it and capture crashes etc, and of having the devbox's greater amount of RAM (LSP servers are typically memory-hungry in large codebases).

LSP is well-architected for this; fundamentally the protocol assumes you may be working in the editor in a not-yet-saved file and so the in-editor state is canonical over the on-disk state. Also, an LSP server generally has to update internal symbol tables and such in response to code changes in a way that can be expensive, so there's a general tolerance for asynchrony between the code state and the server state, and if your sync is reasonably fast you're basically operating "within the regime of asynchrony that's already anticipated" for most purposes.


# Claude-reformatted

 Here is the text reformatted in Markdown:

# Some of the basic constraints here are:

- An M1 or M2 core has faster single-threaded performance than any machine any cloud provider will sell you for any amount of money
- But cloud nodes can cost-effectively provider many have more cores than a laptop
- And much, much, more RAM
- Cloud dev will be much closer in network distance and bandwidth to github and the cluster and other remote services

## Containerized local dev on a mac means running in a VM, which means:

1. Your editor and your local commands (coo etc) are divided by what's effectively a network filesystem (albeit running over a virtual machine bridge, not a real network), which inevitably has bad latency properties, and which -- for every implementation I know of to date -- does not tend to support file-watching APIs well
2. The VM and your host environment share the limited memory resources on your laptop. Modern VM technology mean they don't have to statically partition it any more, but they still compete for a very limited resource

## Remote dev tends to mean either:

- Ephemeral container/VM instances, which is tough for "store of record for a developer's code" OR
- Code lives across the network from the remote compute, which means network latency on fs access, and, again, usually means poor inotify support

# Looking at these constraints, the point Stripe settled on was:

- Remote development machines provisioned in the cloud environment
- devboxes are ephemeral and can be nuked-and-re-created at any time
- devboxes log into Splunk and dev-prod team members can log into your devbox and help debug environment issues
- An internal frontend exposes stable hostnames -- e.g. api.nelhage.stripe.dev -- that always route to "your active devbox" to access services running on your devbox, and do appropriate auth[nz]
- A proxy running on every devbox demand-starts all HTTP services in the monorepo, so that on first access to api.nelhage.stripe.dev, the API service is autostarted and traffic routes to it
- All such services are run in an environment where (a) all imports are lazy, for faster startup, and (b) servers watch (via inotify ) every file that has been imported, and restart if any of them change

- Developers' checkouts and editors live on their laptops
  - This is the lowest-latency option for the all-important editor UX, and also for local git commands like checkout or commit
  - It does not help with networked git performance but git can scale fairly well with some work, and also git fetches and pulls are less "in the tight loop" of developer productivity than "using your editor"

- A coo sync-like syncing experience syncs code from the laptop to the devbox
  - This is centrally provisioned by the org's laptop management tooling and is always-on in the background and does not need to be managed or started by developers. It reports to Sentry or equivalent and dev-prod triages issues and does their best to ensure you never have to think about it.

- Code in the monorepo (aside from the shim commands mentioned here, like the syncer) never runs on laptops. I
t's only ever run on the remote dev node.
Language dependencies etc are managed on that node by dev-prod-managed tooling and configuration.

- Dev-prod provides a pay command that is a unified frontend to the vast majority of day-to-day operations that developers do in the codebase
  - e.g. pay test wraps the pytest equivalent, but transparently runs on the devbox
  - pay curl makes CLI requests to web services on your devbox
  - pay commands collude with the syncer, so when you run pay test it transparently waits until the syncer is caught up with the state as-of the pay test launch before executing code, so that -- as long as you're using pay commands -- you should never see stale / unsynced code on the devbox.
    - This also means that if the syncer is down it can raise an error (and, again, log to Sentry so dev-prod can triage these in an ongoing fashion)

- Our LSP server (Sorbet) ran on the devbox, and VSCode talked to it over ssh. This gave it the benefit of running in a controlled environment where the Sorbet authors could test and debug it and capture crashes etc, and of having the devbox's greater amount of RAM (LSP servers are typically memory-hungry in large codebases).

  - LSP is well-architected for this; fundamentally the protocol assumes you may be working in the editor in a not-yet-saved file and so the in-editor state is canonical over the on-disk state. Also, an LSP server generally has to update internal symbol tables and such in response to code changes in a way that can be expensive, so there's a general tolerance for asynchrony between the code state and the server state, and if your sync is reasonably fast you're basically operating "within the regime of asynchrony that's already anticipated" for most purposes.
-->
