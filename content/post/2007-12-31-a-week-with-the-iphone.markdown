---
author: nelhage
author_email: nelhage@mit.edu
author_login: nelhage
author_url: http://nelhage.com
date: 2007-12-31T01:41:00Z
published: true
status: publish
tags:
- iPhone
- hardware
- sync
title: A week with the iPhone
url: /2007/12/a-week-with-the-iphone/
wordpress_id: 6
wordpress_url: http://nelhage.scripts.mit.edu/madeofbugs/?p=6
---

I've had a new iPhone for about a week now, so I figure it's time to
write up some thoughts about it.

First, the little things. It is, in typical Apple fashion, an
incredibly slick piece of work. Scrolling and zooming images or
webpages is simple, easy, and, well, just fun to do and watch. Mobile
Safari does a great job of making full webpages usable on the tiny
screen.

The keyboard is totally fine after a little practice. I don't think
it'd work nearly as well for e.g. working at a shell; the predictive
text is key to using it well, and getting at symbols is a bit of a
pain. Also, I've found that using it (or just using the phone heavily)
with one hand (phone in palm, thumb on keys) is _horrible_ for my
hand. Doing so for more than a very short time leaves the back of my
hand and/or thumb hurting for the rest of the day.

I haven't hacked the phone at all -- Firmare 1.1.2 patched the hole
jailbreakme used to get in, and my attempts to downgrade the firmware
left the phone nonfunctional until I flashed it up again. I may try
again later, but I'll probably just wait and see what the official SDK
looks like in a month or two.

My first major complaint about the iPhone is that it seems to be, for
all it's supposedly running a nearly full-blown OS X, a single-tasking
device. There's no way that get Mail to download your email in the
background. Switch away from Safari, and the only option is to pause
what it's doing, not keep loading or running JS in the background. So,
if I want to be logged into AIM via a Web 2.0 javascript client,
that's all I'm doing. No checking mail or making notes in the
background, or even browsing the web in abother window! Leave two
windows open long enough, and Safari will eventually decide to
entirely forget about the contents of the inactive one, presumably to
save memory. I haven't checked, but I bet even an incoming call will
completely pause whatever's running, so stay on a call for more than
30s and you'll get bumped. I understand the desire to keep resource
usage down, but this is pretty annoying.

**Edit**: Apparently Mail is fetching in the background. That doesn't
change the fact that Safari, and hence every "supported" custom "app"
(by which I mean webapps), can't run in the background.

The second issue is more fundamental. The iPhone seems to be basically
a dumb internet client. It expects to be connected to the web all the
time. Take away the web, and it becomes more of an iPod than a
PDA. And EDGE, while it's not horrible, just doesn't quite cut it for
this purpose. Hiveminder is practically unusable from the thing over
EDGE, due to the server roundtrips for every operation. And while a
local client might be able to hide that in the background, we don't
(currently) get the ability to write such a thing even if someone
wanted. The Javascript AIM client I've been using is decent, but it's
definitely not as smooth as a local one could be. (And Safari doesn't
save passwords, so I get to type my password, mixed caps and symbols
and all, on the soft keyboard every time. It's the little things.)
You also can't save content from the web on the phone itself; You can
download images or calendars from a computer to the phone, but not
from the phone itself, which is pretty annoying.

I've spent most of the last four days outside of wifi, so I've been
using the phone via EDGE a lot. I'm starting to buy into Jesse's
vision of a disconnected syncable future more and more. I really want
my data local, not 1s latency away, or completely inaccessible because
I happened to step inside the wrong building.

I think the summary is: No, cute little JS webapps are not in fact
nearly sufficient as a development platform for this thing. It's got
great potential, but Apple, please give us a real SDK. When you
release your announced SDK in a month or so, it had better let us
write apps that are first-class in every way compared to the built-in
apps. Otherwise, I will never be able to take the (unhacked) iPhone
seriously.
