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
categories:
- Computer Security
- linux
tags:
- linux
- security
- kernel
---
<p>After the fuss of the last two weeks because of <a href="http:&#47;&#47;cve.mitre.org&#47;cgi-bin&#47;cvename.cgi?name=CVE-2010-3081">CVE-2010-3081<&#47;a> and <a href="http:&#47;&#47;web.nvd.nist.gov&#47;view&#47;vuln&#47;detail?vulnId=CVE-2010-3301">CVE-2010-3301<&#47;a>, I decided to take a look at a handful of the high-profile privilege escalation vulnerabilities in Linux from the last few years.
<&#47;p>
<p>
So, here's a summary of the ones I picked out. There are also a large number of smaller ones, like an <a href="http:&#47;&#47;sota.gen.nz&#47;af_can&#47;"><code>AF\_CAN<&#47;code><&#47;a> exploit, or the <a href="http:&#47;&#47;cve.mitre.org&#47;cgi-bin&#47;cvename.cgi?name=2010-1084">l2cap<&#47;a> overflow in the Bluetooth subsystem, that didn't get as much publicity, because they were found more quickly or didn't affect as many default configurations.
<&#47;p>
<table border="2" cellspacing="0" cellpadding="6" rules="groups" frame="hsides">
<col align="left"><&#47;col><col align="left"><&#47;col><col align="right"><&#47;col><col align="right"><&#47;col><col align="left"><&#47;col>
<thead>
<tr><th>CVE name<&#47;th><th>Nickname<&#47;th><th>introduced<&#47;th><th>fixed<&#47;th><th>notes<&#47;th><&#47;tr>
<&#47;thead>
<tbody>
<tr><td>CVE-2006-2451<&#47;td><td><code>prctl<&#47;code><&#47;td><td>2.6.13<&#47;td><td>2.6.17.4<&#47;td><td><&#47;td><&#47;tr>
<tr><td>CVE-2007-4573<&#47;td><td><code>ptrace<&#47;code><&#47;td><td>2.4.x<&#47;td><td>2.6.22.7<&#47;td><td>64-bit only<&#47;td><&#47;tr>
<tr><td>CVE-2008-0009<&#47;td><td><code>vmsplice<&#47;code> (1)<&#47;td><td>2.6.22<&#47;td><td>2.6.24.1<&#47;td><td><&#47;td><&#47;tr>
<tr><td>CVE-2008-0600<&#47;td><td><code>vmsplice<&#47;code> (2)<&#47;td><td>2.6.17<&#47;td><td>2.6.24.2<&#47;td><td><&#47;td><&#47;tr>
<tr><td>CVE-2009-2692<&#47;td><td><code>sock\_sendpage<&#47;code><&#47;td><td>2.4.x<&#47;td><td>2.6.31<&#47;td><td><code>mmap\_min\_addr<&#47;code> helped <sup><a class="footref" name="fnr.1" href="#fn.1">1<&#47;a><&#47;sup><&#47;td><&#47;tr>
<tr><td>CVE-2010-3081<&#47;td><td><code>compat\_alloc\_user\_space<&#47;code><&#47;td><td>2.6.26<sup><a class="footref" name="fnr.2" href="#fn.2">2<&#47;a><&#47;sup><&#47;td><td>2.6.36<&#47;td><td><&#47;td><&#47;tr>
<tr><td>CVE-2010-3301<&#47;td><td><code>ptrace<&#47;code> (redux)<&#47;td><td>2.6.27<&#47;td><td>2.6.36<&#47;td><td>64-bit only<&#47;td><&#47;tr>
<&#47;tbody>
<&#47;table>


<p>
I'll probably have some more to say about these bugs in the future, but here's a few thoughts:
<&#47;p>
<ul>
<li>
At least two of these bugs existed since the 2.4 days. So no matter what kernel you've been running, you had privilege escalation bugs you didn't know about for as long as you were running that kernel. We don't know whether or not the blackhats knew about them, but are you feeling lucky?
<&#47;li>
<li>
I bet there are at least a few more privesc bugs dating back to 2.4 we haven't found yet.
<&#47;li>
<li>
If you run a Linux machine with untrusted local users, or with services that are at risk of being compromised (e.g. your favorite shitty PHP webapp), you'd better have a story for how you're dealing with these bugs. Including the fact that some of these were privately known for years before they were announced.
<&#47;li>
<li>
It's not clear from this sample that the kernel is getting more secure over time. I suspect we're getting better at finding bugs, particularly now that companies like Google are paying researchers to audit the kernel, but it's not obvious we're getting better at not introducing them in the first place. Certainly CVE-2010-3301 is pretty embarrassing, being a reintroduction of a bug that had been fixed seven months previously.
<&#47;li>
<&#47;ul>



<div id="footnotes">
<h2 class="footnotes">Footnotes: <&#47;h2>
<div id="text-footnotes">
<p class="footnote"><sup><a class="footnum" name="fn.1" href="#fnr.1">1<&#47;a><&#47;sup> <code>mmap_min_addr<&#47;code> mitigated this bug to a DoS, but several bugs that allowed attackers to get around that restriction were announced at the same time.
<&#47;p>
<p class="footnote"><sup><a class="footnum" name="fn.2" href="#fnr.2">2<&#47;a><&#47;sup> The public exploit relies on a call path introduced in 2.6.26, but observers have pointed out <a href="http:&#47;&#47;www.webhostingtalk.com&#47;showpost.php?p=7026467&postcount=192">the possibility<&#47;a> of exploit vectors affecting older kernels.
<&#47;p>
<&#47;div>
<&#47;div>
