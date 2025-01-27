---
author: nelhage
author_email: nelhage@mit.edu
author_login: nelhage
author_url: http://nelhage.com
date: 2008-09-18T12:07:49Z
status: publish
tags:
- wifi
- linux
- wpa_supplicant
- ubuntu
title: 'wpa_supplicant: GUI and wpa_action'
url: /2008/09/wpa_supplicant-gui-and-wpa_action/
wordpress_id: 13
wordpress_url: http://blog.nelhage.com/?p=13
---

I've made two new interesting discoveries about `wpa_supplicant` since
writing my last blog post on the subject. (Actually, I pretty much
made both of them while reading documentation in order to write it,
and have been lame about writing them up).

Using `wpa_gui`
--------------

It turns out that `wpa_gui` not only allows you to select existing
networks, but also to scan for and add new networks to your
configuration file. In addition, you can run it as yourself, without
needing to `sudo` it. In order to do so, you need to add two lines to
`/etc/wpa_supplicant/wpa_supplicant.conf`:

    ctrl_interface_group=netdev
    update_config=1

`ctrl_interface_group` selects a UNIX group that will be given
permission to read/write the control socket. I chose `netdev` because
it seems like it's supposed to be networking-related, and my login
user was already in it on my Ubuntu machine.

`update_config` allows `wpa_supplicant` to write back to its conf file
if instructed to configure new networks by a UI (`wpa_cli` or
`wpa_gui`). Note that this will squash any comments you have in the
file.

`wpa_action` — a mostly-baked roaming solution
----------------------------------------------

The setup I described in the previous post causes `wpa_supplicant` to
manage associating with access points, while Debian's `ifupdown`
request DHCP independently. There's no communication between the
layer, so if you switch networks, or associate sometime _after_ we
bring up the interface, nothing tells `dhclient` to request a new
lease. It turns out we can turn this picture inside-out, and make
`wpa_supplicant` responsible for bringing up and down a virtual
interface, whenever it associates or loses association.

To make this work, we're going to need to edit
`/etc/network/interface` again. Our `wpa_supplicant.conf` can stay
unchanged; Debian's wrapper scripts do all the magic. Replace your
`ath0` block and add a virtual `default` interface as follows:

    iface ath0 inet manual
      wpa-driver wext
      wpa-roam /etc/wpa_supplicant/wpa_supplicant.conf

    iface default inet dhcp

The way this is going to work is that, whenever `wpa_supplicant`
associates to a network, it will bring up the virtual `default`
interface, causing `ifupdown` to spawn `dhclient` and request
DHCP. When it loses association, it brings it down, killing the DHCP
daemon.

Furthermore, we can associate different virtual interfaces with
different networks. Suppose that I usually want DHCP, but at home
(essid `nelhage`) I don't run a DHCP server, and just want my laptop
to always grab `10.0.1.100`. I can add an interface to
`wpa_supplicant.conf`:

    network={
        ssid="nelhage"
        id_str="nelhage"
        key_mgmt=NONE
    }

And then I add a new virtual interface to `interfaces`, corresponding
to the `id_str`:

    iface nelhage inet static
            address 10.0.1.100
            netmask 255.255.255.0
            network 10.0.1.0
            gateway 10.0.1.1

Now, if `wpa_supplicant` associates to the `nelhage` network, it will
bring up the `nelhage` interface, binding `ath0` to the static
configuration there listed.

For documentation, check out the third section of
`/usr/share/doc/wpasupplicant/README.modes.gz` on your Debian or
Ubuntu machine.

In conclusion...
----------------

This setup actually seems pretty close to the correct design for a
roaming wifi architecture, to me. Unfortunately, my experience is that
it hasn't worked well for me; For some reason, when I put it in
roaming mode, it fails to associate with networks that it otherwise
works fine with. I suspect that this is related to `madwifi` suckage
as much as `wpa_supplicant` suck, though, so I'd encourage everyone
else who's been fighting with wifi to try it out and report back if it
works for them.
