---
date: 2019-11-12T14:00:00-08:00
slug: declarative-configuration-management
title: The architecture of declarative configuration management
---

With the ongoing move towards “infrastructure-as-code” and similar notions, there’s been an ongoing increase in the number and popularity of declarative configuration management tools. This post attempts to lay out my mental model of the conceptual architecture and internal layering of such tools, and some wishes I have for how they might work differently, based on this model.


# Background: declarative configuration management

Declarative configuration management refers to the class of tools that allow operators to **declare** a desired state of some system (be it a physical machine, an EC2 VPC, an entire Google Cloud account, or anything else), and then allow the system to automatically **compare** that desired state to the present state, and then automatically **update** the managed system to match the declared state.

Declarative configuration management was largely pioneered by [Puppet](https://puppet.com/), but many other tools fall into this category. Some of the ones I’ll mention by way of example in this post include:

- [Puppet](https://puppet.com/) - an early configuration management tool primarily focused on managing individual hosts
- [Chef](https://www.chef.io/) - Another early tool, making use of a Ruby DSL, which also focuses on managing single hosts
- [Terraform](http://terraform.io/) - Hashicorp’s tool for managing entire cloud environments on almost any provider and of almost any sort
- [Kubernetes](https://kubernetes.io/) - The popular container orchestration tool. I class it in this category because the primary paradigm for configuring services on Kubernetes involves pure-data declarations of desired states, and one or more [controllers](https://kubernetes.io/docs/concepts/architecture/controller/) responsible for updating reality to match.

# The layers

I conceptualize any of these systems into three distinct layers (or perhaps, two layers and an interface boundary between them). We’ll start with the middle layer / the interface boundary, since that’s the layer in terms of which everything else is defined:

## Resource schema

At the core of any such model is what I will call the “resource schema.” This schema is the definition of which types of resources can be managed by the tool, and what properties they have (as well as their semantics). For instance, for Puppet, the resource schema includes types like [`file`](https://puppet.com/docs/puppet/5.5/types/file.html) or [`mount`](https://puppet.com/docs/puppet/5.5/types/mount.html), as well as the definition of all the properties those types support.

The bulk of the user documentation for most declarative configuration management tools is dedicated to documenting this schema, explaining the various resources you can define and configure using the tool.

This schema is often extensible via plugins or other extensions; Puppet supports [defining custom resources](https://puppet.com/docs/puppet/6.0/custom_resources.html) in Ruby, and Terraform supports plugins that define new resource types.

Kubernetes has perhaps the more explicit resource schema of all our examples: They explicitly publish the schema of all core resource types as a [schema definition](https://github.com/kubernetes/api/blob/master/core/v1/generated.proto) in Protocol Buffer format, and custom resource types may provide an [OpenAPIv3 validation schema](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/) to document and enforce their shape.

## Concrete resource syntax

The resource schema tells us what types of resources our tool supports, but in order to use the tool we need to declare a concrete catalog containing a specific set of resources that the tool is responsible for maintaining. In order to do that, we need some concrete syntax we can type into our editor, check into version control, and then pass to the tool to reify this set of resource descriptions.

Importantly, while ultimately applying a configuration to a system operates on a fully-realized concrete set of resources, we often don’t want to **express** our configuration in this form. We may want to **parameterize** our definition (e.g. to use different credentials in production vs staging environments), or make use of **abstraction**, (e.g. to define a custom “service” type that configures logging and supervision in a common way). Thus, we desire a concrete syntax that can both


1. Express specific concrete resources from the schema with specific parameters
2. Express some degree of abstraction, parameterization, iteration, or other concepts, to simplify the problem of defining human-managed configuration files.

In some systems, like Puppet and Terraform, this problem is addressed via a custom language which supports defining resources, and has some set of usually ad-hoc “module” and/or “configuration” primitives, which allow a level of abstraction within the language. While this combination is convenient for many purposes, it often results in the slow and ad-hoc growth of features in the language, as it grows more and more “real language”-like features, such as general [loops](https://www.hashicorp.com/blog/hashicorp-terraform-0-12-preview-for-and-for-each/).

Chef takes a different approach: By embedding their syntax in an embedded Ruby DSL, Chef makes the whole of the Ruby language available for expressing abstraction, iteration and composition. Confusingly to me, it also has its own abstraction notions (of resources and resource inclusion) that run entirely along-side and independent of Ruby’s own mechanisms for extension and abstraction; you could imagine a “purer” version of this approach that relied solely on Ruby-language features like modules and functions to provide abstraction and reusable components.

Kubernetes takes a radically different approach: the only concrete syntax the tool itself accepts is raw JSON or YAML defining fully-concrete resources, and the core tool itself pushes the problem of generating these to another layer. Users are welcome to define configuration directly for simple cases, or to build their own templating or abstraction systems which produce these resources. The open-source community has produced several such models, including [Helm](https://helm.sh/), [skycfg](https://github.com/stripe/skycfg), and [Jsonnet](https://jsonnet.org/articles/kubernetes.html).

This approach is in some ways more cumbersome, and risks a more-fragmented ecosystem, but also has the benefit of an extremely simple and clear abstraction boundary, and of allowing (with a tool like `skycfg`) the use of a “real” programming language (of the user’s choice, even) and usual abstractions and tools to produce configurations.


## The convergence engine

On the far side of the tool from the concrete syntax, we have the convergence engine, whose job is to take a concrete resource catalog, and update the managed system to match the desired catalog.

Declarative configuration management derives both its power, and many of its pitfalls, from a tool’s engine. The whole point of declarative configuration management is that the configuration specifies only the desired state, and it is up to the engine to figure out **how to get there**, which drastically simplifies configurations and makes them less path-dependent, or reliant on the previous state of the system.

However, this flexibility is also be the source of operational pitfalls: For many systems in production, the question of “how you get there from here” **matters**, and when this is the case, declarative configuration management can be fraught.

As a simple example, imagine a tool that supports being configured by reading a configuration file on startup, or via an online API (e.g. an HTTP administration API). Further, let’s imagine that restarting the service involves a period of downtime, but online configuration changes are seamless. [HAproxy](http://www.haproxy.org/), for example, may have this characteristic depending on how it is deployed.

A pure declarative model only lets us specify the desired “end state” configuration; the question of whether to restart the service and incur downtime, or whether to issue online configuration updates, is left to the convergence engine. But if we care about availability, this distinction may be important to use, and using the tool safely may require us to work around the convergence engine, and/or rely on implementation details of a specific version.


Of the systems I’m aware, Kubernetes has perhaps the most explicit notion of this layer as distinct from the other layers, [under the name of “controllers”.](https://kubernetes.io/docs/concepts/architecture/controller/)


# Implications of this model

I find this conceptual layering quite useful when thinking about and evaluating configuration management tools; even if a specific tool does not have a layering as strictly defined as the taxonomy I’ve defined here, reasoning about tools in terms of the layering helps me understand their features and distinctions between them, and I find it clarifies my understanding of some of the benefits and challenges of various systems. Here’s a few concrete thoughts I have based on this model for directions I’d like to see explored (or that are being explored and I think should be more widely-understood) in these tools:


## Concrete syntax and abstraction

Thinking in this model has really made me appreciate the strict layering that Kubernetes takes, where resource definitions are accepted and stored only in fully-reified terms (typically as JSON or YAML), and abstraction is provided external to the system.

The problem of “defining the shape of a Kubernetes pod” is core to what Kubernetes does, and it makes sense for that be defined by the Kubernetes API. The problem, however, of “defining 10 pods with similar shapes but some parameters filled in,” is a problem entirely like those encountered in other programming domains, and one more-than-adequately solved by primitives in any programming language (functions, variables, etc). Separating these concerns (rather than conflating them like Puppet does) lets both systems do what they do best, and lets developers write abstractions in tools and styles they already understand, instead of trying to wrap their head around how to write a loop in Puppet’s bespoke DSL.

This boundary also makes configuration more testable — if you can run the configuration generation entirely separately from the convergence, it’s easy to write tests that generate a catalog and make assertions about it, or to run a diff against an older version to understand the implications of your change. Pure refactors can also be pursued with exceptional confidence: as long as a new version produces the same catalog, you can be confident your change had no functional effect!

[Pulumi](https://www.pulumi.com/) is an interesting new approach in this direction. It embraces the “use a pre-existing, real language to write your abstractions” philosophy, but under the hood it [has a notion of resources and and convergence engine](https://www.pulumi.com/docs/intro/concepts/how-pulumi-works/) (which they call the “deployment engine”) that is fairly similar to my layering.


## Pluggable convergence engines

One direction which this model suggests to me, but which I haven’t seen explored yet, is the idea of having pluggable convergence engines built on top of the same resource schema. This direction feels promising for alleviating the challenges of using declarative configuration management in an environment where the details of operations are important.

To get a sense of why this might be useful, imagine a terraform AWS configuration that is commonly used for two different purposes:

- A production environment receiving live traffic is maintained and evolved using a terraform configuration, to ensure that the state of production is expressed in code and matches a checked-in configuration.
- Temporary staging or development environments are regularly created and destroyed off the same configuration, to test versions of the application or configuration changes, prior to production.

We desire these environments (production and staging) to look as much alike as possible (modulo, perhaps, credentials or a few other details), which is why we generate them from the same configuration. However, the **operations** we are willing to perform on both are very different! When creating or updating a staging environment, we are happy to tolerate downtime, and in fact routinely completely-destroy and recreate-from-scratch environments. In production, though, uptime is paramount, and it is critical that we never destroy data or perform operations that will take services offline (at least without doing so deliberately and in a planned way).

We could imagine resolving this tension if Terraform had two different convergence engines (or, perhaps and equivalently, one engine with two policies):

- The “create a new environment” engine, which always creates from scratch every resource it was given. This would excel at spinning up fresh environments as quickly as possible, since it would have to perform a minimum of introspection or logic and would just issue a series of “Create()” calls. It would also be comparatively simple and guarantees that any valid state expressed in configuration can be realized in the managed environment.
- The “production operations” engine, which behaves much like the existing engine, and compares reality against the requested state, and designs a plan to get there. Importantly, however, it by design will **never issue a destructive operation**, and will **error out** on changes that cannot be executed non-disruptively. Unlike the “create new environment” engine, this engine may not always be able to apply valid configurations (e.g. it may never delete certain types of resources), but will **fail safe** for production usage, allowing operators to apply changes in production with greater confidence and without requiring deep operator expertise about which operations are safe.
