---
author: nelhage
author_email: nelhage@mit.edu
author_login: nelhage
author_url: http://nelhage.com
date: 2011-08-08T13:32:29Z
published: true
status: publish
tags:
- linux
- DEFCON
- security
- exploits
- kvm
- blackhat
title: 'BlackHat/DEFCON 2011 talk: Breaking out of KVM'
url: /2011/08/breaking-out-of-kvm/
wordpress_id: 474
wordpress_url: http://blog.nelhage.com/?p=474
---

I've posted <a href="https://nelhage.com/talks/kvm-defcon-2011.pdf">the final slides</a> from my talk this year at <a href="http://defcon.org/">DEFCON</a> and <a href="http://blackhat.com/">Black Hat</a>, on breaking out of the <a href="http://www.linux-kvm.org/page/Main_Page">KVM</a> Kernel Virtual Machine on Linux.

<iframe src="https://www.slideshare.net/slideshow/embed_code/8908773" width="427" height="356" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC;border-width:1px 1px 0;margin-bottom:5px" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="https://www.slideshare.net/NelsonElhage/virtunoid-breaking-out-of-kvm" title="Virtunoid: Breaking out of KVM" target="_blank">Virtunoid: Breaking out of KVM</a> </strong> from <strong><a href="https://www.slideshare.net/NelsonElhage" target="_blank">Nelson Elhage</a></strong> </div>

<b>[Edited 2011-08-11]</b> The <a href="https://github.com/nelhage/virtunoid">code is now available</a>. It should be fairly well-commented, and include links to everything you'll need to get the exploit up and running in a local test environment, if you're so inclined.

In addition, as I mentioned, this bug was found by a simple KVM fuzzer I wrote. I'm also going to clean that up and release it, but don't expect it too soon.

I had a great time meeting lots of interesting people at BlackHat and DEFCON, some that I'd met online and others I hadn't. If any of you are ever in Boston, drop me a note and we can grab a beer or something.
