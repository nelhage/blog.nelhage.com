---
date: 2017-02-19T12:48:34-08:00
slug: kubernetes
title: Thoughts On Kubernetes
---

I spent a while the last week porting [livegrep.com][livegrep] from
running directly AWS to running on [Kubernetes][k8s] on
[Google's Cloud Platform][gcp] (specifically, the
[google container engine][gke], which provisions and manages the
cluster for me).

I left this experience profoundly enthusiastic about the future of
Kubernetes. I think that if Google can execute properly, it's clearly
the future for how we build distributed applications. That said, it
also feels like it has a ways to go yet. This post is some of my
thoughts and experiences from this project.

# The Good

## The Right Abstractions

Kubernetes has a very strong set of abstractions for deploying
applications to a cloud environment, and they're very well thought
out. I can create a "deployment" that ensures some code is running in
a cluster, and then a "service" that handles routing traffic to that
code within the cluster with [a few lines of yaml][backend]. And these
primitives are very well thought-out and robustly implemented;
"deployments" natively support incremental no-downtime deployments,
including healthchecking of the underlying service and so on. And
since the abstraction level is so high -- I'm really describing the
logical units my application is built on, rather than the physical
details -- the implementation will hopefully continue to improve and
evolve without requiring extensive modification by users.

The abstractions are also all very well-integrated with each other
based on a rich, shared, schema; healthchecks are (rightly) a property
of a container, but the "deployment" (ensures K copies of a container
are always present), "service" (within-kubernetes routing), and
"ingress" (external routing) primitives all automatically know to look
at that healthcheck definition and take it into account for their own
purposes.

This is a much higher level of abstraction than my previous deployment
on raw AWS, which had to manage everything from firewall rules to user
accounts up to Amazon autoscaling groups. And it shows in size, too:
My [kubernetes configuration][kubeconf] comes in under 500 lines,
compared with [600+ lines of ansible][ansibleconf] and
[700+ lines of terraform][tfconf] for the ec2 deployment.

And the new version is *better* in many important ways: I have
automated zero-downtime deploys with a single command, and
fully-automated letsencrypt certificates, both of which I never fully
built out on ec2.

## Extensible

Having the right abstractions is a strong start, but Kubernetes'
potential would be limited if you were limited to use cases that had
been thought of and supported by Google's engineers. One of the key
features of configuration management systems like Puppet and Terraform
is the creation of new custom resource types, which can then be
reused, either within a team or organization, or shared and used more
broadly.

Kubernetes also supports user-defined resource types, known as
[third-party resources][thirdparty]. While still beta and somewhat
experimental, they provide a tantalizing vision of what kubernetes'
extensibility will look like once mature.

For livegrep.com, I wanted automatic certificate management using
[letsencrypt][letsencrypt]-provisioned certificates, so I adopted the
[kube-cert-manager][kcm] project, based on
[Kelsey Hightower's original project][kcm-kelsey].

Enabling it was fairly easy, although it involved a few manual steps;
I created a GCP volume and service account, added the service account
to a kubernetes secret, and then grabbed
[two](https://github.com/PalmStoneGames/kube-cert-manager/blob/master/k8s/certificate-type.yaml)
[files](https://github.com/PalmStoneGames/kube-cert-manager/blob/master/k8s/deployment.yaml)
from their repository, edited one slightly, and ran `kubectl create
-f` to create the configuration in my kubernetes cluster.

Once I'd done so, things got almost magically simple. I created a
[certificate object][cert] describing the certificate I wanted, waited
a few seconds for the certificate manager to notice it and negotiate
with the letsencrypt ACME server, and then I had a valid TLS
certificate available as a Kubernetes secret:

    $ kubectl describe secret beta.livegrep.com
    Name:           beta.livegrep.com
    Namespace:      default
    Labels:         domain=beta.livegrep.com
    Annotations:    stable.k8s.psg.io/kcm=true

    Type:   kubernetes.io/tls

    Data
    ====
    tls.crt:        3448 bytes
    tls.key:        1675 bytes

But in fact, I didn't actually even need to create the `certificate`
object. `kube-cert-manager` integrates with the existing kubernetes
`ingress` type: I added [few lines of "annotations"][ann] to my
ingress definition, and the certificate controller automatically
provisioned a certificate, which was automatically picked up by the
ingress controller, registered with google's cloud load balancer, and
available to serve https traffic into my application.

Third-party extensions in general, and `kube-cert-manager` in
particular, are new, beta, and somewhat experimental, and I ran into a
few rough edges that I'll mention later. However, even so, they still
worked impressively well, and give a tantalizing vision of the future
in which kubernetes will have a rich, extensible ecosystem of
third-party resources for controlling not only your kubernetes cluster
but any external dependencies you might want, in a uniform and
integrated way.

# The Bad

## Docker

Kubernetes, by default[^docker], runs code in the form of docker
containers distributed through a docker image repository.

While I'm incredibly enthusiastic about the future of container images
as a deployment format, docker continues to feel like a hot mess every
time I interact with it, and it's entirely unclear to me how I'm
"supposed" to build docker images for my application; I ended up
stitching together a lot of pieces together by hand in a way that felt
like I was working against the grain at every step. Here's a few of
the concrete issues and open questions I ran into:

### Tagging images

Every time I try to understand docker repositories, I'm left a little
baffled by how I'm supposed to use "tags". The standard convention
seems to be to rely heavily on mutable tags, tagging based on major
releases or similar.

However, as an operator, this feels like a nightmare, making tracking
the provenance of my images, or building reproducible images, an
absolute nightmare. I'd much rather tag builds with a hyper-specific
version number, so I have a more-or-less complete and readable
provenance chain, and then bump those dependencies explicitly as
needed. Similarly, kubernetes appears to implicitly expect immutable
tags:

- The default
  [`ImagePullPolicy`](https://kubernetes.io/docs/user-guide/images/#updating-images)
  of `IfNotPresent` will cache a given image locally ~forever.

- All the doc examples for
  [deploying a new version](https://kubernetes.io/docs/user-guide/deployments/#updating-a-deployment)
  assume you do so by switching over to a new image tag (which will
  automatically trigger an incremental rollout); I experimented
  initially with always using the `latest` tag, and I was unable to
  find a way to force kubernetes to do a "no-op" redeploy just to get
  it to re-pull a tag.

However, none of the Docker tooling seems to support immutable
per-version tags well. For example, I can't parameterize the `FROM`
line, so if I want to depend on specific versions of an upstream
image, I'm forced to rewrite my `Dockerfile` every time I want to
update. And [docker hub](https://hub.docker.com)'s automatic builds
seem to only support constant tags for a given branch, and not a
per-build tag.

### Building images

Livegrep is divided into two services, a C++ backend that manages the
index and processes the actual queries, and a Go web tier. In
addition, I run an `nginx` frontend to serve static files. All three
components (the C++ binary, the Go binary, and the
Javascript/HTML/CSS) build from the
[same source repository](https://github.com/livegrep/livegrep).

I have no idea how I'm "supposed" to build `Dockerfile`s for this. I
could build a single image that installs both livegrep and `nginx` and
the configuration for all three components, and invoke it in different
ways; That'd be easiest but feels kludgy and wouldn't scale past a
certain point [^build]. What I wanted to do was build four images: A
base image containing the `livegrep` build artifacts, and then three
specialized images for each service:


    ubuntu  <-- livegrep/base  <-- livegrep/backend
                              ^--- livegrep/frontend
                              ^--- livegrep/nginx

However, I have no idea how to build such trees of images using
`Dockerfile`s. I could write four `Dockerfile`s and run `docker build`
four times, but how do I specify the appropriate base image in the
child images? In particular, it's very important that a given build
reference a *specific* version of the `base` image, since I want to
produce three leaf images from the same repository. But as described
above, Docker seems to want me to use fixed tags, which makes keeping
track of this tree challenging; If my leaf images start with `FROM
livegrep/base:latest`, how do I ensure that they actually build
against the right version?

I ended up building out a moderately complex build system of my own,
based on a Debian-esque version number
(`[livegrep git sha]-[packaging revision]`), with
[some scripts](https://github.com/livegrep/livegrep.com/tree/995765a1a4c44f8d8df2e744925877661eee64a1/bin/)
to handle the builds. It works and I'm quite happy with it, but I wish
there had been an obvious approach.

### Deploying configuration

The final question I don't have a satisfying answer for is how to
deploy configuration data into my docker images. I ended up baking
hardcoded configuration into my images, which feels very unsatisfying.

To pick a trivial example, my
[nginx configuration](https://github.com/livegrep/livegrep.com/blob/995765a1a4c44f8d8df2e744925877661eee64a1/docker/nginx/nginx.conf#L47-L47)
hardcodes the "livegrep.com" domain. This works fine for me, but means
that no one else can reuse this setup to deploy their own internal
livegrep instance. But I'm not at all clear what the "right" option
is. It seems plausible I could store the configuration in a kubernetes
[ConfigMap](https://kubernetes.io/docs/user-guide/configmap/), but how
do I get it from there into the `nginx.conf`? Do I need to templatize
my configuration at container startup? It seems that today that's
something I would need to build myself.

## The declarative/convergence model

Kubernetes follows a declarative resource model, in which an operator
configures a resource by creating the object definition, and then a
Kubernetes "controller" asynchronously comes by and attempts to update
reality to conform to the described state.

Like all such systems, problems can arise when reality and the desired
state are irreconcilable without destructive action, or when the
change requires a transition that the controller doesn't know how to
make. Concretely, what this tends to mean is that certain properties
can only be set at initial object creation time, and can't be changed
later. However, because the controller acts asynchronously from the
Kubernetes API, such changes result in no visible errors, just silent
divergence.

I ran into a few such cases while working on livegrep, where I had to
destroy and re-create a resource in order for a change to take
effect. Sometimes, once I dug, this behavior was documented somewhere;
Sometimes I couldn't find any mention of it.

This was mostly a minor annoyance, because I was developing a new
deployment with no SLA, and so as soon as I realized what was going on
I could just delete everything and start over. If I were building an
application with more severe uptime requirements, I could imagine
getting very frustrated trying to figure out how to orchestrate
complex operational changes in a zero-downtime manner.

## It's very young

Kubernetes definitely feels very young and still a
work-in-progress. Many of the features I relied on, like the
third-party extensions for letsencrypt, and the GCP ingress controller
that automatically configured Google's Cloud Load Balancer from within
Kubernetes, are marked as beta and subject to change. Kubernetes is
changing rapidly, and documentation from a month ago may already be
out of date, and there are very few public examples of nontrivial
setups to learn from.

The third-party resource ecosystem is still very new and evolving; I
had to switch from Kelsey Hightower's `kube-cert-manager` to the new
`PalmStoneGames` implementation, and that one is still
[missing features](https://github.com/PalmStoneGames/kube-cert-manager/issues/37)
that I could really use. I'm completely sold on the potential of this
ecosystem, but it remains mostly potential at this point.

Security is another area where it seems like there's work to be
done. Kubernetes secrets
[are stored in plaintext in etcd](https://kubernetes.io/docs/user-guide/secrets/#security-properties),
which is fine for many applications but would probably scare me at a
certain scale. Kubernetes
[supports](https://kubernetes.io/docs/admin/authorization/)
fine-grained access control to the kubernetes API (essential for
running a truly multi-tenant cluster), but still defaults to "always
allow" and has three(!) different access-control mechanisms, and I'm
unclear which one is considered "the future".

Assuming Google plays their cards right, and community enthusiasm
continues to build and mindshare continues to center, this will all
obviously sort itself out in time. However, based on this experience,
while I remain very excited for Kubernetes' potential, I'd be wary of
adopting it or recommending it for a site with strict availability,
security, or stability requirements.

[livegrep]: https://livegrep.com/
[k8s]: https://kubernetes.io/
[gcp]: https://cloud.google.com/
[gke]: https://cloud.google.com/container-engine/
[backend]: https://github.com/livegrep/livegrep.com/blob/1fdd0b5fddd31c8a3fcd641d888f67fcd122a745/kubernetes/backend-linux.yaml
[thirdparty]: https://kubernetes.io/docs/user-guide/thirdpartyresources/
[letsencrypt]: https://letsencrypt.org/
[kcm]: https://github.com/PalmStoneGames/kube-cert-manager
[kcm-kelsey]: https://github.com/kelseyhightower/kube-cert-manager

[kubeconf]: https://github.com/livegrep/livegrep.com/tree/c0b96d4d082183a0166a8c40f25ab243d3232986
[ansibleconf]: https://github.com/livegrep/livegrep.com/tree/233bb93f1097225d75775b5440774132808b11df/ansible
[tfconf]: https://github.com/livegrep/livegrep.com/tree/233bb93f1097225d75775b5440774132808b11df/terraform
[cert]: https://github.com/livegrep/livegrep.com/blob/233bb93f1097225d75775b5440774132808b11df/gcloud/kubernetes/cert-beta.livegrep.com.yaml
[ann]: https://github.com/livegrep/livegrep.com/blob/c0b96d4d082183a0166a8c40f25ab243d3232986/kubernetes/frontend.yaml#L71-L74

[^build]: In retrospect, for my application, this would almost
    certainly have been the right solution; But I wanted to do the
    more-general thing out of stubborn curiosity. And even a single
    image wouldn't have obviously my question of how to version
    images.

[^docker]: Kubernetes nominally supports other container backends,
    like rkt; My impression is that docker is currently very much the
    "blessed" runtime. It's also the only one supported by GKE, which
    was how I deployed my cluster.
