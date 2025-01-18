---
author: nelhage
author_email: nelhage@mit.edu
author_login: nelhage
author_url: http://nelhage.com
date: 2010-09-26T23:16:19Z
status: publish
tags:
- linux
- security
- kernel
title: A brief look at Linux's security record
url: /2010/09/a-brief-look-at-linuxs-security-record/
wordpress_id: 343
wordpress_url: http://blog.nelhage.com/?p=343
---

<p>After the fuss of the last two weeks because of <a href="http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-3081">CVE-2010-3081</a> and <a href="http://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2010-3301">CVE-2010-3301</a>, I decided to take a look at a handful of the high-profile privilege escalation vulnerabilities in Linux from the last few years.
</p>
<p>
So, here's a summary of the ones I picked out. There are also a large number of smaller ones, like an <a href="http://sota.gen.nz/af_can/"><code>AF\_CAN</code></a> exploit, or the <a href="http://cve.mitre.org/cgi-bin/cvename.cgi?name=2010-1084">l2cap</a> overflow in the Bluetooth subsystem, that didn't get as much publicity, because they were found more quickly or didn't affect as many default configurations.
</p>

<style>
th, td {
    padding: 0 10px;
}

thead tr {
    border-bottom: 1px solid #aaa;
}

table {
    border: 1px solid black;
    margin: 1em 0;
}
</style>

| CVE name      | Nickname                                | Introduced      | Fixed    | Notes                                      |
|---------------|-----------------------------------------|-----------------|----------|--------------------------------------------|
| CVE-2006-2451 | `prctl`                      | 2.6.13          | 2.6.17.4 |                                            |
| CVE-2007-4573 | `ptrace`                     | 2.4.x           | 2.6.22.7 | 64-bit only                                |
| CVE-2008-0009 | `vmsplice` (1)               | 2.6.22          | 2.6.24.1 |                                            |
| CVE-2008-0600 | `vmsplice` (2)               | 2.6.17          | 2.6.24.2 |                                            |
| CVE-2009-2692 | `sock_sendpage`             | 2.4.x           | 2.6.31   | `mmap_min_addr` [^mmap_min_addr] helped. |
| CVE-2010-3081 | `compat_alloc_user_space` | 2.6.26[^compat] | 2.6.36   |                                            |
| CVE-2010-3301 | `ptrace` (redux)             | 2.6.27          | 2.6.36   | 64-bit only                                |



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

[^mmap_min_addr]: `mmap_min_addr` mitigated this bug to a DoS, but several bugs that allowed attackers to get around that restriction were announced at the same time.

[^compat]:  The public exploit relies on a call path introduced in 2.6.26, but observers have pointed out <a href="http://www.webhostingtalk.com/showpost.php?p=7026467&postcount=192">the possibility</a> of exploit vectors affecting older kernels.
