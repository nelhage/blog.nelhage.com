---
layout: post
status: publish
published: true
title: Using wpa_supplicant on Debian/Ubuntu
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 10
wordpress_url: http://nelhage.scripts.mit.edu/madeofbugs/?p=10
date: 2008-08-22 14:06:00.000000000 +02:00
tags:
- debian
- wifi
- linux
- wpa_supplicant
---
I've been using `wpa_supplicant` to manage wifi on my Ubuntu laptop
for a while, and have found that it's pretty close to what I want for
managing wireless — closer than anything else I've found, at least. I
figured I should document my setup and experiences.

Some Background
----------------

You probably all know just how much wireless on Linux can be a pain to
get working right. Getting drivers and so forth working is usually
fine these days, especially if you're using Ubuntu, but managing
connecting to multiple networks and dealing with WPA and WEP is a
serious pain in the ass. Debian's solution the `ifupdown`
infrastructure lets you specify a single network or `any`, and doesn't
have an answer for encryption, as far as I can tell. Ubuntu (and
Fedora)'s NetworkManager works great when it works, but it wants to
own your entire networking stack, isn't very transparent or debuggable
when networking isn't working, and the only interface is a dock
applet, which is problematic for my minimalist [XMonad][xmonad]-based
desktop.

Enter `wpa_supplicant`
----------------------

Despite its name, `wpa_supplicant` isn't just about WPA. It's actually
a general management system for your wireless in disguise. You give it
a config file of networks you want to connect to if they're available,
optionally with priorities, and settings about the kind of encryption
and a password or key if needed. You then tell it "go", and it will go
scan for networks and connect to the appropriate ones as needed. If
you need to override it, there's a command line client (`wpa_cli`) to
connect to the running ndaemon and tell it connect to a specific
network or AP (I think — I haven't actually had occasion to use it
much at all)

My configuration
----------------

I have an Atheros wifi card, so my wifi device is `ath0`. Adjust this
as appropriate (it'll probably be `eth1` with most other drivers)

First, install the necessary packages:

    $ sudo apt-get install wpasupplicant

Then set up your configuration:

* `/etc/network/interfaces` — We're still going to use `ifupdown` to
manage getting DHCP, but just not for wireless. So add a stanza to
`interfaces` that looks something like:

        auto ath0
        iface ath0 inet dhcp
        wpa-driver wext
        wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf

* `/etc/wpa_supplicant/wpa_supplicant.conf` — This is the file where
you're going to specify what networks you want to connect
to. `/usr/share/doc/wpasupplicant/examples/` can explain the full
range of options better than I can, but there are some examples
below. For now, you can just put a

        ctrl_interface=/var/run/wpa_supplicant

at the start of the file.

Now, configure your networks in `wpa_supplicant.conf`. Some examples:

* MIT's network -- open, no encryption

        network={
            ssid="MIT"
            key_mgmt=NONE
        }

* WEP, hex key

        network={
            ssid="langtonlabs"
            key_mgmt=NONE
            wep_key0=deadbeef
        }

* WPA1, password

        network={
            ssid="wireless-is-a-lie"
            psk="passw0rd"
        }

Now if you bring up the interface with `ifup ath0`, `wpa_supplicant`
will start scanning for networks and associate as needed. The crappy
thing about this solution is that there's no communication between
`wpa_supplicant` and `dhclient`, so you won't automatically try to get
a new lease if you switch networks. I solve this with a `ifup --force
ath0` when I move my laptop between access points. I don't do this too
often without suspending, though, so it's not a huge deal. Browsing
documentation points me at something called `wpa_action` that's
supposed to fix this... If I figure it out I'll post again.

This works quite well for me, better than any other solution I've
found for moving my laptop between multiple access points, and handles
WEP, WPA, and WPA2 just fine. Hopefully it'll be helpful for someone
else.

[xmonad]: http://xmonad.org
