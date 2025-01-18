---
author: nelhage
author_email: nelhage@mit.edu
author_login: nelhage
author_url: http://nelhage.com
date: 2010-06-06T20:21:11Z
status: publish
tags:
- software
- code review
- programming
- ksplice
- development
title: 'Confessions of a programmer: I hate code review'
url: /2010/06/i-hate-code-review/
wordpress_id: 241
wordpress_url: http://blog.nelhage.com/?p=241
---

<div id="text-1">

<p>
Most of the projects I've been working on today have fairly strict
code review policies. My <a href="http://www.ksplice.com">work</a> requires code review on most of our
code, and as we bring on <a href="http://blog.ksplice.com/2010/03/quadruple-productivity-with-an-intern-army/">an army of interns</a> for the summer, I've been
responsible for reviewing lots of code. Additionally, about five
months ago <a href="http://barnowl.mit.edu/">BarnOwl</a>, the console-based IM client I develop, adopted an
official pre-commit review policy. And I have a confession to make: I
hate mandatory code review.
</p>
<p>

I should be clear that I think I still <b>believe</b> in code review
as a useful tool for developers working together on a project. A
reviewer will flag as bad style or inefficient or ugly things that you
let slide working for yourself. A reviewer comes at code without the
assumptions of how it's supposed to work that you made, and can often
spot errors you made because mixed up a mental model of your intent
with the code you actually wrote.
</p>

<p> In addition, code review is a great way to ensure that at least
two people are familiar with each piece of your infrastructure. There
is often a natural tendency for different developers to take ownership
of specific pieces of code or infrastructure, and be the only ones who
touch or maintain it. But then it breaks while they're on vacation,
and everyone else is left trying to figure out how this code was ever
supposed to work. A mandatory code review policy is often a great way
to mitigate that class of problem.
</p>

<p> But, theoretical and observed benefits of code review aside,
speaking personally, as both a developer and as a reviewer, I've been
finding it a really frustrating process.  </p>

</div>

<div id="outline-container-1.1" class="outline-3">
<h3 id="sec-1.1">As an author </h3>
<div id="text-1.1">


<p>
As a developer, I hate that code review adds unpredictably long
latencies into my development workflow. Once I've finished and tested
a feature to my satisfaction, and sent it off for code review, I have
to wait for a potentially long time before I can actually mark it as
done. This means that, when deciding what to do next, I have
essentially three choices:
</p>
<ol>
<li>
Busy-wait. Get coffee, read reddit, and check my mail until the
review request comes back.

</li>
<li>
Continue development on top of the code I just sent for code
review.

</li>
<li>
Work on something completely different.

</li>
</ol>

<p>All three of these options suck. (1) is convenient if I can expect the
review to come back shortly, since doing something idle like reading
reddit or checking mail lets me keep the code in the back of my mind,
so I don't have to page it back in when the review response comes
back. But obviously it's inefficient, wasted time, and unacceptable if
I don't expect a reply within an hour at most.
</p>
<p>
(2) is often what I want to be doing. I'll often be working on a
project with several logically distinct but related stages. It often
makes sense to send out a review request for each, since they can be
deployed and reviewed separately.
</p>
<p>
However, if a reviewer comes back with significant comments on the
code I sent out, I now not only need to update that code, but I also
need to rebase the work I've done since then on top of the result,
which may be a real pain if I'm working on something closely related
(e.g. if my work used APIs I built previously, and the reviewer asks
for changes in those APIs in some way).
</p>
<p>
Option (3) avoids both problems, but means that I'm continually
swapping different projects in and out of focus. This slows me down,
since I have to constantly re-remember where I was in each project,
which APIs I was using, and so on. Any developer can tell you that
they hate switching between different projects too frequently.
</p>
<p>
Option (3) might be more tolerable with large projects, where
development/review cycles are on the order of weeks. But those aren't
the projects I'm working on.
</p>
</div>

</div>

<div id="outline-container-1.2" class="outline-3">
<h3 id="sec-1.2">As a reviewer </h3>
<div id="text-1.2">


<p>
Reviewing code is one of those things that I would enjoy if I had
infinite time, but that I find a nuisance when I don't. I do enjoy
reading other peoples' code and figuring out how it could be better,
and giving feedback to help get it there.
</p>
<p>
But doing so well takes a lot of time, and a lot of time is something
I rarely have these days. In addition, because of my above complaints
about dealing with code review as a developer, I'm always acutely
aware that someone is probably waiting on my reply in order to get
work done. So I always feel rushed to reply to code review requests,
as well.
</p>
<p>
So, even though I always feel like code review should be something I'm
enjoying, I tend to find it a frustrating experience where it just
feels like a task I have to do before I can get back to real work.
</p>
<p>

This is, of course, in part just a symptom of being too busy, but code
review makes it a task that bugs me more than other time drains. If a
customer reports a critical bug that I have to drop everything to
investigate, I'll be annoyed, but I at least feel like I'm doing
something important that fixes a real problem and hopefully ends with
a happy customer. If I spend a day doing nothing but reading code
reviews, I'll end up feeling unsatisfied and unproductive. Because
code review feels fundamentally optional -- even though I believe it's
beneficial, it's something we've chosen to do, not something that we
absolutely have to do in order for the project or business to keep
operating -- it's more frustrating to find myself spending a large
amount of time on.

</p>
</div>

</div>

<div id="outline-container-1.3" class="outline-3">
<h3 id="sec-1.3">In conclusion </h3>
<div id="text-1.3">


<p>
I believe in code review as a powerful tool. But in practice, I've
found it frustrating to work with. I'd like to hear your thoughts:
Does code review work for you? Am I doing it wrong, in some ways? Is
it just a question of changing my attitude in some way?
</p>
<p>
I know that a lot has been written about doing code review
effectively. I'd appreciate pointers to anything you've found
particularly compelling.
</p></div>
</div>
