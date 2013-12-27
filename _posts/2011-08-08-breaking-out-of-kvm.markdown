---
layout: post
status: publish
published: true
title: ! 'BlackHat/DEFCON 2011 talk: Breaking out of KVM'
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 474
wordpress_url: http://blog.nelhage.com/?p=474
date: 2011-08-08 13:32:29.000000000 +02:00
categories:
- Computer Security
- Low-level hacking
tags:
- linux
- DEFCON
- security
- exploits
- kvm
- blackhat
---
I've posted <a href="http://nelhage.com/talks/kvm-defcon-2011.pdf">the final slides</a> from my talk this year at <a href="http://defcon.org/">DEFCON</a> and <a href="http://blackhat.com/">Black Hat</a>, on breaking out of the <a href="http://www.linux-kvm.org/page/Main_Page">KVM</a> Kernel Virtual Machine on Linux.

<div style="width:425px; margin:auto; padding: 1em" id="__ss_8908773"><strong style="display:block;margin:12px 0 4px"><a href="http://www.slideshare.net/NelsonElhage/virtunoid-breaking-out-of-kvm" title="Virtunoid: Breaking out of KVM">Virtunoid: Breaking out of KVM</a></strong><object id="__sse8908773" width="425" height="355"><param name="movie" value="http://static.slidesharecdn.com/swf/ssplayer2.swf?doc=kvm-defcon-2011-110818165327-phpapp02&stripped_title=virtunoid-breaking-out-of-kvm&userName=NelsonElhage" /><param name="allowFullScreen" value="true"/><param name="allowScriptAccess" value="always"/><embed name="__sse8908773" src="http://static.slidesharecdn.com/swf/ssplayer2.swf?doc=kvm-defcon-2011-110818165327-phpapp02&stripped_title=virtunoid-breaking-out-of-kvm&userName=NelsonElhage" type="application/x-shockwave-flash" allowscriptaccess="always" allowfullscreen="true" width="425" height="355"></embed></object></div>

<b>[Edited 2011-08-11]</b> The <a href="https://github.com/nelhage/virtunoid">code is now available</a>. It should be fairly well-commented, and include links to everything you'll need to get the exploit up and running in a local test environment, if you're so inclined.

In addition, as I mentioned, this bug was found by a simple KVM fuzzer I wrote. I'm also going to clean that up and release it, but don't expect it too soon.

I had a great time meeting lots of interesting people at BlackHat and DEFCON, some that I'd met online and others I hadn't. If any of you are ever in Boston, drop me a note and we can grab a beer or something.

