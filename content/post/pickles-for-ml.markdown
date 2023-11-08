---
title: "What's with ML software and pickles?"
date: 2023-11-07T21:00:00-08:00
slug: pickles-and-ml
---

I have spent many years as an software engineer who was a total outsider to machine-learning, but with some curiosity and occasional peripheral interactions with it. During this time, a recurring theme for me was horror (and, to be honest, disdain) every time I encountered the widespread usage of Python `pickle` in the Python ML ecosystem.

In addition to their major security issues[^fn-exploit], the use of `pickle` for serialization tends to be very brittle, leading to all kinds of nightmares as you evolve your code and upgrade libraries and Python versions. In my career as a software engineer, I tended to believe that -- to a good approximation -- there were [no valid use cases][alex-talk] for the `pickle` module, and it existed solely as an attractive nuisance; a mistake of a younger, more exuberant, and more naive ecosystem.

[alex-talk]: https://www.youtube.com/watch?v=7KnfGDajDQw

[^fn-exploit]: Over a decade ago, I published one of the first [public writeups][pickle-exploit] (that I know of) of an exploit attacking a `pickle`-based system; but the problem had been well-known for years, and these days there exist [tools for push-button exploitation][sour-pickles].

[sour-pickles]: https://github.com/sensepost/anapickle
[pickle-exploit]: /2011/03/exploiting-pickle/

For the last few years now, though, I've [worked professionally in ML][anthropic], giving me a new perspective on the ecosystem. I've developed a hands-on understanding of why ML software looks the way it does, and some of the problems `pickle` is solving for its users. I still don't like it, but I have a lot more understanding and empathy for the problem space, and I no longer believe the problem is "trivial" or merely one of ignorance or laziness. This post is an attempt to convey some sense of what I've learned, and bridge the gap a bit between these two worlds.

[anthropic]: https://anthropic.com/

# Doing research, not software

At root, the key perspective I want to convey is that most software in the machine-learning ecosystem is designed to support researchers, first and foremost, and not software engineers. Furthermore, research (in this context, at least) is characterized (perhaps even defined) by a few related properties:

- The primary output of research is *knowledge*, not software artifacts. Research teams write software to answer research questions and improve their/their team's/their field's understanding of a domain, more so than they write software in order to have software tools or solutions.
- Researchers tend not to know, _a priori_, which ideas will be successful or useful, and so they prioritize trying lots of ideas efficiently, and they assume most of them won't work out.

As a corollary of the above, researchers (especially in a domain like ML!) may well write many lines of code, but it is also completely normal for most of those lines to be thrown out in short order. Failed experiments will commonly be discarded, and even successful ones are often discarded after the paper has been written. Even when building on a successful idea in future work, it's common that the code itself will be rewritten for the new project, now that the problem is better understood, and with the new problem or research question in mind.

The tradeoffs and affordances in this environment lean much more heavily towards `pickle` (vs other serialization strategies), when compared to a typical software-engineering context:

- Most code is only ever written and run by the same few people (the researcher and their team), and in a single environment; there is often little or no distinction between "development" and "production," and no external users or data sources or interactions. These factors collectively make the _in-practice_ security risks of `pickle` serialization much lower (but still not zero!)
- There is enormous benefit to being able to just take a datastructure -- or even pieces of code -- and serialize them, without pausing to think about schemas or formats or data evolution; we care about time-to-results, and effort spent thinking about schemas or serialization formats or translating between on-disk and in-memory formats is entirely inessential work.
- Since most code is thrown away fairly quickly, the brittleness of `pickle`, and the challenges of handling code changes and version upgrades, are much less salient.

## A research vignette

To make the above tradeoffs a bit more concrete, I want to write up a stylized description of a researcher doing ML research, and try to show some of the concrete use cases she will find for `pickle`, alongside some of the tradeoffs I see, informed by my own experience doing ML research and supporting (far more experienced!) ML researchers.

Suppose a researcher is experimenting with a new deep-learning model architecture, or a variation on an existing one. Her architecture is going to have a whole bunch of configuration options and hyperparameters: the number of layers, the types of each layers, the dimensionality of various vectors, where and how to normalize activations, which nonlinearity(ies) to use, and so on. Many of the model components will be standard layers provided by the ML framework, but the researcher will be inserting bits and pieces of novel logic as well.

Our researcher needs a way to describe a particular concrete model -- a specific combination of these settings -- which can be serialized and then reloaded later. She needs this for a few related reasons:

- She likely has access to a compute cluster containing GPUs or other accelerators she can use to run jobs. She needs a way to submit a model description to code running on that cluster so it can run _her_ model on the cluster.
- While those models are training, she needs to save snapshots of their progress in such a way that they can be reloaded and resumed, in case the hardware fails or the job is preempted.
- Once models are trained, the researcher will want to load them again (potentially both a final snapshot, and some of the partially-trained checkpoints) in order to run evaluations and experiments on them.

We can imagine many different solutions here! We could write a protobuf or JSON description of the model, and code to convert that into an actual model instance. We could even imagine expressing the model as "an array of command-line arguments" and parsing those into a model instance.

However, here's one solution that solves all of these problems, and requires virtually no marginal work up front:
- The definition of "a specific model" is just "an object that implements the model" (likely a [`torch.nn.Module`][nn.module] subclass)
- The reearcher constructs these objects in some ad-hoc way by a research script, which she edits and evolves as she runs new experiments
- She then serializes the entire object using `pickle`, and uses that as the stored version for later analysis or to load and execute on a remote machine.

This approach requires no fuss, no particular discipline or pattern for how the research code or implementation is structured, and virtually no cognitive overhead compared to "just constructing a running a model." It supports standard components and custom components with equal ease: the researcher can make small tweaks at random points deep in the class hierarchy, or replace the entire model architecture, and neither one requires much thinking about or editing the serialization system.

At this point, our researcher likely writes some scripts to instantiate various Cartesian grids of hyperparameters, schedules them all onto the cluster, and gets lunch (or leaves for the weekend) while they train.

[nn.module]: https://pytorch.org/docs/stable/generated/torch.nn.Module.html

These pickle files are definitely brittle! They serialize the entire implementation detail of the model, and it's very easy to break old experiments and models as the researcher adds new settings and parameters and refactors their experiments. But again, most of the experiments yield uninteresting results, and there's no need to ever look at them again. For the few that are, it's easy enough to keep track of the git commit associated with each experiment, and roll your checkout back. And if our researcher needs to preserve a few specific models more durably, it's usually possible to fix them up with a few tactical fixes in a `__setstate__` method somewhere.

That said, if the experiments are collectively a success and we need to continue down this line of study, and try even more variants, and preserve models even longer for comparisons, the brittleness does starts to hurt, and so our researcher may eventually consider a refactor. One straightforward approach is to separate out the model "description" and implementation: the model configuration/settings/hyperparameters are grouped into a `class ModelConfig:`; this class is what is serializes (still via `pickle`), and the researcher hand-maintains a `build(cfg: ModelConfig) -> Model` function that knows how to instantiate the full model given a configuration.

The researcher now only needs to worry about the forwards- and backwards-compatibility of this class, and can fairly free refactor any implementation code by also refactoring the code that constructs the `Model` object from a `ModelConfig`.

But it's worth noting that our resercher (and/or her team) are *still experimenting* and trying new things and iteration and "time to experiment" and perhaps more importantly "cognitive cost to experiment" rule all. And so, while many of the fields on this class will be nice "normal" integers and booleans and perhaps enumerations, it's a bit tedious to have to add support for each further experiment.

And so, perhaps our researcher leaves a number of "escape hatches."  As a trivial example: it's still moderately trendy in this field to try replacing the nonlinearity or "activation function" that we apply to a tensor after a fully-connected layer (I'm [guilty here myself][solu]).

[solu]: https://transformer-circuits.pub/2022/solu/index.html

How do we store an activation function (which is likely literally a Python function of type `Tensor -> Tensor`) on our configuration?

We could register functions in a `dict[str, Callable]` somewhere and store functions by name ... but hey, isn't the Python module namespace such a dict itself, really? So if we're using `pickles`, we can probably just store functions directly on our configuration.

Emboldened by this realization, our resarcher goes further; she can even store an entire `build_xxxx: Callable[[Config], Module | None]` override, which lets the configuration override arbitrary parts of the configuration in arbitary ways!

Such features restore most of the old brittleness problem; but, importantly, they let our research team walk a line between the older, more-stable, core configurations, which are first-class in "normal" parameters, and the frothy churn of experimentation, where (almost) nothing lasts long, and speed-of-iteration rules all.

## Scope creep

In my personal opinion, everything I've described above is "basically fine," if a bit distateful and risky, **as long as the project is truly only run in trusted contexts for internal research use**.

The bigger problem, though, is that once you've started using `pickle` routinely in this way, it becomes a habit to use them for all of your serialization problems, even ones where pickles are much more problematic. And not only do researchers do so, but this pattern tends to leak out into the tools and ecosystem.

PyTorch, for instance, uses `pickle` files as part of its default serialization format (via [`torch.load`][torch.load] and [`torch.save`][torch.save]), even though in most cases they really only need a few "vanilla" data structures -- lists and dicts of strings and Tensors. PyTorch even has a custom C++ `pickle` loader that only supports a few built-in types, which can be enabled using the `weights_only` argument to the loader; but for backwrds compatibility this flag defaults to `False`, and (in the absense of an attacker) enabling it can only break things; and so it is rarely used, and every PyTorch weight file is an open invitation for users to use the dangerous interface and leave themselves vulnerable.

[torch.load]: https://pytorch.org/docs/stable/generated/torch.load.html
[torch.save]: https://pytorch.org/docs/stable/generated/torch.save.html

# What's to be done?

My main goal in writing this piece is to try to highlight "the view from inside," as it were, of why I see so much `pickle` usage inside of the ML ecosystem and some of the problems it's solving for its users. That said, as machine learning grows in importance and gains users and deployments in varied contexts, I forsee the problems caused by unsafe `pickle` usage to grow in importance and urgency, and I want to offer a few musings on some paths forward.

## Better data formats and interfaces

First, I think that we (and here I mean primarily those of us with software engineering backgrounds and skills, and/or who work in or near ML in primarily-SWE roles) should continue investing in high-quality alternatives to `pickle`, in order to give researchers more and better alternatives. If done well, we can even offer researchers **better** experiences than `pickles` in many cases, with better security properties.

I would be remiss here not to point that the Google machine learning ecosystem built around XLA -- `Tensorflow` and `jax` and its various libraries and frameworks -- have largely avoided the problems described here; they tend to use protocol-buffer and [numpy-format][numpy-format] based serialization formats, and pickles are much less rampant. On the one hand, I view them as something of an existence proof that such an ecosystem is possible; on the other hand, I tend to believe that PyTorch has become so dominant because it generally offers researchers a superior usage experience, especially around onboarding and ease of modification/exploration, and so I am hesitant to point to the Google libraries as an ideal solution.

[numpy-format]: https://numpy.org/doc/stable/reference/generated/numpy.lib.format.html#module-numpy.lib.format

The Huggingface [safetensors library][safetensors] is an excellent example of progress in this space, offering a solid foundation for serializing tensor data in a secure and performant way. In my mind it's best viewed as a slightly low-level primitive; it supports serializing and restoring objects of the form `dict[str, Tensor]`; very often researchers want nested dictionaries, or to store metadata alongside weights in a single bundle; I am excited to see the development of more-ergonomic researcher-friendly formats and APIs on top of `safetensors`.

Researchers often want to serialize Python class trees as well as tensor data; perhaps the advent and recent dominance of of Python type annotations and APIs like `attrs` and `dataclasses` built on top of them offers the potential for a library that is driven by Python type annotations and a small amount of additional metadata (perhaps building on [`cattrs`](https://catt.rs/en/stable/)) that allows for more-or-less direct serialization and deserialization of [certain] trees of type-annotated Python classes.

[safetensors]: https://huggingface.co/docs/safetensors/index

## Restricting `pickle`

For better or worse, though, we have a lot of `pickle` files and libraries that use `pickle` files today, and the problem is still growing. While we work to provide alternatives, and to encourage projects to migrate to those alterntives, here's a speculative idea for some harm reduction or mitigation:

First, I wonder if it would be useful for Python to support a global "nopickle" mode that disabled the `pickle` module entirely. Such a flag would at least allow operators deploying Python to easily know if they were at some risk from loading untrusted pickles at runtime.

However, if that is too restrictive, what about adding options to limit `pickle` usage in specific ways?

### Pickle manifests

In particular, I'm intrigued by the idea of requiring all usages of `pickle` to declare a list of valid binary objects for decoding, probably named by sha256 or another crytographic hash. We could imagine supporting both specifying expected objects at the callsite of `pickle.load` (or some new API), or out-of-band in some sort of manifest file, potentially which specified legal (call stack, blob ID) tuples. Any pickle load which did not specify this data, or did not match an allowed hash, would be rejected.

The reason this approach intrigues me is that it might allow those deploying ML sotfware to -- once -- collect an inventory of all the model weight files or configuration files being loaded, and then "freeze" that set, ensuring no **new** pickles would be encountered at runtime.

This, in turn, largely reduces the problem of managing `pickle` usage to the problem of managing third-party dependencies; this is not, to be sure, a solved problem, but it's a problem you probably had anyways.

Specifically, in order to deploy a new third-party ML model, you typically need to install both the model implementation (in the form of a Python module or script), **and** the model weights. Both of these pose security risks if they are malicious; but the risk from `pickle` is in some sense worse because even if you review and pin a specific version of the model code, the `pickle` file will often be downloaded at runtime, meaning you are still exposed to RCE from outside of your system.

If we could guarantee that that code only loaded specific pickle files, which we could audit once when we first choose to trust that dependency, then we have made the `pickle` use case "look like" the problem of installing third-party code, where once we fetch and pin a version, there is at least no **additional** runtime attack surface.

We could imagine making these ideas even more complex with a runtime configuration system; in the limit, we might imagine fairly complex APIs like these to control `pickle` usage:

```python
with pickle.no_pickles() as token:
  pickle.loads(...) # NoPickleException

  with pickle.allow_pickles(token):
    pickle.loads(...)

  with pickle.allow_pickles(token, only_objects=[
    "sha256:some sha256",
  ]):
    pickle.loads(buf) # hashes buf, only proceeds if it matches

  # You must pass the same token returned by `no_pickles` to
  # `allow_pickle`; this prevents libraries from trivially blanket
  # calls to renable `pickle`
  with pickle.allow_pickles(bad_token):  # BadTokenException
    ...
```

Such mechanisms will of course never be perfect; it will always be possible for libraries to hack around them. In the limit, a library can always vendor a pure-Python implementation of `pickle` without any of these limitations. However, that's only one of a million ways a library could (deliberately or inadvertently) embed a backdoor; the goal of this proposal is not to solve supply-chain security, but to mitigate the specific risk of `pickle`, and, as above, to shift the risk to package-install time instead of runtime. We would hope that libraries implementing egregious bypasses of a standard security mechanism would be noticed and flagged by the community as untrustworthy[^trust].

Such features could be prototyped out-of-tree in a library that monkeypatches the `pickle` module, but there might be value in eventually baking it into the interpreter, in terms of standardization and adoption.

[^trust]: This sentence reads as naive to me, too, as I write it. But again, I do think it's true that such a solution largely reduces the problem to the existing supply-chain security problem, both for better and for worse.
