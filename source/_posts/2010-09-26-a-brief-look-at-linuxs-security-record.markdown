---
layout: post
status: publish
published: true
title: A brief look at Linux's security record
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 343
wordpress_url: http://blog.nelhage.com/?p=343
date: 2010-09-26 23:16:19.000000000 +02:00
tags:
- linux
- security
- kernel
---
<p>After the fuss of the last two weeks because of <a href="http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3081">CVE-2010-3081</a> and <a href="http://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2010-3301">CVE-2010-3301</a>, I decided to take a look at a handful of the high-profile privilege escalation vulnerabilities in Linux from the last few years.
</p>
<p>
So, here's a summary of the ones I picked out. There are also a large number of smaller ones, like an <a href="http://sota.gen.nz/af_can/"><code>AF\_CAN</code></a> exploit, or the <a href="http://cve.mitre.org/cgi-bin/cvename.cgi?name=2010-1084">l2cap</a> overflow in the Bluetooth subsystem, that didn't get as much publicity, because they were found more quickly or didn't affect as many default configurations.
</p>
<table border="2" cellspacing="0" cellpadding="6" rules="groups" frame="hsides">
<col align="left"></col><col align="left"></col><col align="right"></col><col align="right"></col><col align="left"></col>
<thead>
<tr><th>CVE name</th><th>Nickname</th><th>introduced</th><th>fixed</th><th>notes</th></tr>
</thead>
<tbody>
<tr><td>CVE-2006-2451</td><td><code>prctl</code></td><td>2.6.13</td><td>2.6.17.4</td><td></td></tr>
<tr><td>CVE-2007-4573</td><td><code>ptrace</code></td><td>2.4.x</td><td>2.6.22.7</td><td>64-bit only</td></tr>
<tr><td>CVE-2008-0009</td><td><code>vmsplice</code> (1)</td><td>2.6.22</td><td>2.6.24.1</td><td></td></tr>
<tr><td>CVE-2008-0600</td><td><code>vmsplice</code> (2)</td><td>2.6.17</td><td>2.6.24.2</td><td></td></tr>
<tr><td>CVE-2009-2692</td><td><code>sock\_sendpage</code></td><td>2.4.x</td><td>2.6.31</td><td><code>mmap\_min\_addr</code> helped <sup><a class="footref" name="fnr.1" href="#fn.1">1</a></sup></td></tr>
<tr><td>CVE-2010-3081</td><td><code>compat\_alloc\_user\_space</code></td><td>2.6.26<sup><a class="footref" name="fnr.2" href="#fn.2">2</a></sup></td><td>2.6.36</td><td></td></tr>
<tr><td>CVE-2010-3301</td><td><code>ptrace</code> (redux)</td><td>2.6.27</td><td>2.6.36</td><td>64-bit only</td></tr>
</tbody>
</table>


<p>
I'll probably have some more to say about these bugs in the future, but here's a few thoughts:
</p>
<ul>
<li>
At least two of these bugs existed since the 2.4 days. So no matter what kernel you've been running, you had privilege escalation bugs you didn't know about for as long as you were running that kernel. We don't know whether or not the blackhats knew about them, but are you feeling lucky?
</li>
<li>
I bet there are at least a few more privesc bugs dating back to 2.4 we haven't found yet.
</li>
<li>
If you run a Linux machine with untrusted local users, or with services that are at risk of being compromised (e.g. your favorite shitty PHP webapp), you'd better have a story for how you're dealing with these bugs. Including the fact that some of these were privately known for years before they were announced.
</li>
<li>
It's not clear from this sample that the kernel is getting more secure over time. I suspect we're getting better at finding bugs, particularly now that companies like Google are paying researchers to audit the kernel, but it's not obvious we're getting better at not introducing them in the first place. Certainly CVE-2010-3301 is pretty embarrassing, being a reintroduction of a bug that had been fixed seven months previously.
</li>
</ul>



<div id="footnotes">
<h2 class="footnotes">Footnotes: </h2>
<div id="text-footnotes">
<p class="footnote"><sup><a class="footnum" name="fn.1" href="#fnr.1">1</a></sup> <code>mmap_min_addr</code> mitigated this bug to a DoS, but several bugs that allowed attackers to get around that restriction were announced at the same time.
</p>
<p class="footnote"><sup><a class="footnum" name="fn.2" href="#fnr.2">2</a></sup> The public exploit relies on a call path introduced in 2.6.26, but observers have pointed out <a href="http://www.webhostingtalk.com/showpost.php?p=7026467&postcount=192">the possibility</a> of exploit vectors affecting older kernels.
</p>
</div>
</div>
