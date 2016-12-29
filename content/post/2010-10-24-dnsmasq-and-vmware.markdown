---
author: nelhage
author_email: nelhage@mit.edu
author_login: nelhage
author_url: http://nelhage.com
date: 2010-10-24T23:15:23Z
published: true
status: publish
tags:
- testing
- vmware
- dns
- dhcp
- sysadmin
title: Configuring dnsmasq with VMware Workstation
url: /2010/10/24/dnsmasq-and-vmware/
wordpress_id: 383
wordpress_url: http://blog.nelhage.com/?p=383
---

I love VMware workstation. I keep VMs around for basically every
version of every major Linux distribution, and use them heavily for
all kinds of kernel testing and development.

This post is a quick writeup of my networking setup with VMware
Workstation, using dnsmasq to assign my VMs addresses and provide a
DNS server to resolve VM addresses.

The objective
-------------

I want to be able to resolve my VM's hostnames so that I can ssh to
them, or run other network services and access them from the host. I
could just assign static addresses and put them in `/etc/hosts`, but
that's totally lame, and liable to be a source of error and
frustration, because I have dozens of VMs, and add and remove them
frequently.

We're going to set things up so that when VMs get addresses from DHCP,
their hostnames automatically become resolvable, using the `.vmware`
domain. To do this, we're going to set up a piece of software called
`dnsmasq`, which is a flexible DNS and DHCP server, designed for
basically exactly this purpose.


The setup
---------

Because I use my VMs for local testing, I just keep most of them on a
local NAT on my machine. I configure that virtual network inside
VMware as follows (run `vmware-netconfig`, or follow the appropriate
menus):

<a href="/images/posts/2010/10/vmw.png"><img src="/images/posts/2010/10/vmw.png" alt="" title="VMware workstation network configuration" width="512" height="576" class="aligncenter size-full wp-image-388" /></a>

Note how I **disable** "Use local DHCP service to distribute IP
addresses to VMs" -- we're going to set up dnsmasq to prove DHCP, so
we don't want it fighting with VMware's.

Notice that the subnet I'm using here is `172.16.37.*` -- if you
choose a different one, you'll need to adjust accordingly later.

Configuring `dnsmasq`
---------------------

Then, I install `dnsmasq`, and configure `/etc/dnsmasq.conf` as
follows:

    listen-address=172.16.37.1
    listen-address=127.0.0.1
    no-dhcp-interface=lo

    server=192.168.1.1
    local=/vmware/

    no-hosts
    no-resolv

    domain=vmware
    dhcp-fqdn

    dhcp-range=172.16.37.3,172.16.37.200,12h
    dhcp-authoritative
    dhcp-option=option:router,172.16.37.2

Here's what each of those lines mean, in order:

    listen-address=172.16.37.1
    listen-address=127.0.0.1
    no-dhcp-interface=lo

We don't want `dnsmasq` serving DHCP or DNS to the outside world or
other virtual networks, so we only tell it to listen on the local
interface -- so that we can talk to it from the host -- and to the
virtual network we set up in the previous step. We don't want it
serving DHCP to `localhost`, though, so we tell it not to.

    server=192.168.1.1
    local=/vmware/

Here we tell `dnsmasq` how to forward DNS requests to the outside
world. We're going to be using `dnsmasq` as our primary nameserver,
and having it forward requests for things it doesn't understand to a
real DNS server. In my case, that's my LAN's router, at
`192.168.1.1`. The `local` line tells `dnsmasq` that the `.vmware`
domain is local, and it should never forward requests to resolve
things in that domain.

If I needed something more complicated, it might be possible to use
the `resolv-file` option or similar, but I don't, personally.

    no-hosts
    no-resolv

These options tell `dnsmasq` not to look at `resolv.conf` or
`/etc/hosts` when resolving names -- we want it only to resolve VMs
itself, and to forward everything else.

    domain=vmware
    dhcp-fqdn

This tells dnsmasq to assign the `.vmware` domain to hosts it hands
out DHCP to, so that we can resolve VMs in the `.vmware` domain.

    dhcp-range=172.16.37.3,172.16.37.200,12h
    dhcp-authoritative

And finally, we configure the DHCP server. We give it a range of
addresses to assign on the subnet we created earlier. I stop at
`.200`, so that I can leave the last few open for static IPs if I need
for some reason, and we start at `.3` -- `.1` is the host, and `.2` is
the address of VMware's router. `dhcp-authoritative` enables some
optimizations when `dnsmasq` knows it is the only DHCP server around.

    dhcp-option=option:router,172.16.37.2

Finally, we need `dhcp-option` to tell DHCP clients to use the
VMware-provided router at `.2` as their gateway, instead of using the
host, at `.1`. We could configure the host to be a NAT server using
Linux's NAT, but that's outside the scope of this document.

Configuring the host
--------------------

Now, we need to configure the host to use dnsmasq as our DNS
server. This is a simple matter of telling the host to use `127.0.0.1`
as our DNS server, and to add `.vmware` to our search path. If we're
editing `resolv.conf` directly, it would look like:

    search vmware
    nameserver 127.0.0.1


Configuring guests
------------------

We need to configure our guests to send a hostname along with their
DHCP requests, so that `dnsmasq` can add them to its address
table. How to do this varies by OS, but most modern OSes do it
automatically. If they don't, here are a few hints:

For RHEL-based distros, edit `/etc/sysconfig/network-scripts/ifcfg-INTERFACE`, and add a line like

     DHCP_HOSTNAME=centos-5-amd64

For most other Linux distributions, you can often edit `dhclient.conf`
(usually in `/etc/` or `/etc/dhclient/`) to include:

     send host-name "centos-5-amd64";

Or, with a recent `dhclient`,

     send host-name "<hostname>";

will make it look up the machine's actual hostname.

Conclusions
-----------

That's all there is to it. This is a pretty simple setup, but
hopefully someone else will find this useful. If you need `dnsmasq` to
do something more subtle, the [documentation][dnsmasq] is mostly quite
good.

[dnsmasq]: http://www.thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html

