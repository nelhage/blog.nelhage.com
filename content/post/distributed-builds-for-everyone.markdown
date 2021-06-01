---
title: "Distributed cloud builds for everyone"
date: 2021-05-31T16:05:17-07:00
---
CPU cycles are cheaper than they have ever been, and cloud computing has never been more ubiquitous. All the major cloud providers offer generous free tiers, and services like GitHub Actions offer free compute resources to open-source repositories. So why do so many developers still build software on their laptops?

Despite the embarrassment of riches of cheap or even free cloud compute, most projects I know of, and most developers, still do most of their software development — building and running code — directly on their local machines. Cloud builds — be they distributed, or just running on a single giant instance larger than the machine in front of you — are relatively common inside large, sophisticated organizations, but rare for individual developers, open-source project, or smaller operations.

Recently, I blogged about [building LLVM in 90 seconds](https://blog.nelhage.com/post/building-llvm-in-90s/) using AWS Lambda and my Llama project. Today I want to share the larger vision behind Llama’s design, and why I believe Llama’s approach could finally make distributed cloud builds ubiquitously accessible, even to smaller open-source projects and individual hobbyist developers.

I believe that Llama’s design — enabled in large part by AWS Lambda — can remove the most significant barriers to entry to using cloud compilation. With more maturity and implementation work, I think it has the potential to make cloud builds almost the default for a wide range of developers, and I’m incredibly excited about that future.

Let’s take a look at the features and design decisions that make this possible.


## Cheap, with near-zero carrying costs

Llama is cheap to use, and costs virtually nothing when you’re not actively running a build.

My LLVM build cost around $0.40. If you were using ec2 directly, forty cents buys you about 10 minutes of on-demand compute on a [96-core c5 instance](https://aws.amazon.com/ec2/instance-types/c5/). I haven’t benchmarked, I estimate that that instance can likely build LLVM in about 3 minutes. On paper that makes Llama up to 3x more expensive … but only if you manage to start and stop the ec2 instance **exactly** when you need it. If you were to leave up that instance  while idle even for an hour, it would be incredibly hard to ever catch up to Llama. Llama, on the other hand, automatically scales down when you’re not doing a build, and Lambda’s impressive cold-start performance means it’s always there when you need it.

Furthermore, Lambda transparently scales to intermediate levels of utilization. If you do an incremental rebuild that only builds, say, 10% of the source files, your cost with Llama will be about 10% of the overall build, with no additional tuning or configuration.

So Llama is not only price-competitive with ec2’s best case, but is much simpler for developers to manage, and removes the risk of leaving an instance running and being stuck with a $3000 bill (the rough monthly cost of that c5 instance).

Llama always costs (nearly) nothing when you aren’t running a build. If you only do one build a month, you’ll still only pay a few cents a month. If you keep a suspended c5 instance around, by comparison, you’ll probably use an EBS disk, which costs 8¢/GB-mo. That cost can add up on its own if you keep large build trees around, but more importantly it creates cognitive overhead for infrequent users: when is it worth keeping your instance around in case you need to do more builds, and when is it worth turning it off to save a few bucks, at the cost of higher startup time and configuration cost the next time you want it?

## Fits into your existing workflow

If you’re using a cloud instance to build your software, somehow you have to get your code onto it. The two most obvious choices are:

- Keep your source checkout remote, and develop using `ssh` or something like [VS Code’s Remote Development](https://code.visualstudio.com/docs/remote/remote-overview)
- Keep your checkout locally, and use `rsync` or similar to copy source into the cloud, and artifacts back to your laptop.

Either option imposes a change to your workflow, requiring you to think about the local and remote machines separately, and split your work, to at least some extent, between two machines. And if you do the first option, you additionally have to pay for the dev machine while you’re just doing development or reading code, even when you’re not doing any builds.

By contrast, Llama leverages S3 and file-level caching to move individual files and artifacts between your workstation and Lambda transparently and on-demand. You can continue developing locally and running `make` in exactly the same way you would without Llama. Everything works exactly the same, except your builds are faster!

Llama’s approach, to be explicit, does come with some downsides. It is somewhat slower than systems which require fewer network round-trips, and it adds complexity to the implementation. It also makes builds dependent on your workstation’s network speed; builds from a slow wireless connection may end up bottlenecked on bandwidth instead of compute. Nonetheless, I strongly believe in this choice, because I think there’s enormous power in presenting a completely transparent user experience, and asking for as small a behavior change from the user as possible.


## Compatible with virtually every build system

Some build systems — most notably [Google’s Bazel](https://docs.bazel.build/versions/master/remote-execution.html) — natively support transparent remote build execution. However, accessing this feature requires you to rewrite your project’s entire build in Bazel (in addition to the cost and complexity of running or finding a remote build executor). This is a fairly sizeable task, since Bazel is quite strict and very opinionated. And, even if you succeed, few of your users will already have Bazel installed, and its heavyweight dependencies will scare off many potential users and contributors.

Llama uses the classic `distcc` paradigm of providing a drop-in replacement (`llamacc`) for the compiler ( `gcc` or `clang`), and relying on the build system’s own parallelism[^smp]. This approach lets it support virtually every C and C++ build system out there, out of the box, with minimal fuss.

It might be more performant to specialize for, say, `cmake` or `ninja` — and it may still make sense to do so in the future — but here as well, I have optimized in Llama’s design for broad adoptability and low barriers to entry for new projects and new developers.

[^smp]: Especially in today’s multicore world, any build system worth talking about has some means to run builds using multiple cores, which is regularly exercised by developers who build on top of it.


## Minimal configuration

I’ve tried to design Llama to require as little configuration as possible; the ultimate goal is that, with one or two commands, you can set up Llama with a cloud compiler environment mirroring your local desktop, and use that image across any and all of your projects.

This feature is enabled in large part by AWS and by the technologies of infrastructure-as-code, which lets me define a single [CloudFormation template](https://github.com/nelhage/llama/blob/main/cmd/llama/internal/bootstrap/template.json) that sets up all of Llama’s major dependencies in one fell swoop. I’ve also provided tooling to help build compiler toolchains and manage system headers between local nodes and the cloud environment, but that’s another place where Llama could be even more polished. AWS Lambda’s recent Docker support — announced while I was first building Llama — helps a lot here, letting Llama developers and users use standardized and widely-understood Docker images and Dockerfiles to build compiler images.


# Conclusion

For decades now, technologists have talked about the vision of [“utility computing”](https://en.wikipedia.org/wiki/Utility_computing) — computing as a metered, on-demand service, in much the same way as we think of power or running water as a utility.

The modern cloud era has been hailed by many as the realization of this vision, in which we rent computing an hour-at-a-time (and more recently, [a second-at-a-time](https://aws.amazon.com/blogs/aws/new-per-second-billing-for-ec2-instances-and-ebs-volumes/)), and virtually no one owns their own hardware at scale any more.

However, cloud environments remain fairly involved to configure and to operate, and tend to demand a lot of expertise of their users. For me, the vision of utility computing can never be complete until the act of running computation — using the full power and scale of the cloud, at least as far as you’re willing to pay — is about as easy and worry-free as turning on the faucet in your home for fresh water.

Llama is my attempt to work towards this vision for — at the least — the specific problem of software builds. Thanks to Amazon Lambda, and the general advancement of cloud technologies, I believe we are finally within reach of concretely achieving this vision, and I’m hopeful we can make it so.

That said, to be clear, Llama doesn’t yet realize this vision entirely. It’s a young project, with few users yet other than me, and lots of sharp edges and rough corners. However, I think results like my LLVM build performance — where it outperforms some of the largest machines money can buy — serve as strong evidence that the approach is feasible and can succeed if we push on it.

## Beyond builds

I've talked about software builds here because that's a domain I'm deeply familiar with, and because that's the one I've chosen to tackle with `llamacc`. However, I think the vision here can go well beyond building software. I imagine a world where _most_ compute-intensive tasks that are performed interactively are seamlessly outsourced to the cloud on-demand, while preserving a user experience _as though_ all of your data remained available locally.

As one other concrete example of feasibility, prior to the `gg` paper, the same team demonstrated [ExCamera][excamera], which used Lambda in a similar fashion to perform video editing and transcoding. Between that project, `gg`, and Llama, I feel confident that we've demonstrated that vision has broad feasibility; now we just need to finish building it!

[excamera]: https://www.usenix.org/conference/nsdi17/technical-sessions/presentation/fouladi
