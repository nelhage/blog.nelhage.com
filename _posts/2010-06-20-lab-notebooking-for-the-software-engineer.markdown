---
layout: post
status: publish
published: true
title: Lab Notebooking for the Software Engineer
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 254
wordpress_url: http://blog.nelhage.com/?p=254
date: 2010-06-20 22:53:07.000000000 +02:00
categories:
- Software Engineering
tags:
- software
- lab notebooks
- writing
---
A few weeks ago, I [wrote][prev] that software engineers should keep
lab notebooks as they work, in addition to just documenting things
after the fact. Today, I'm going to share the techniques that I've
found useful to try to get in the habit of lab-notebooking my work,
even though I still feel like I could be better at writing things
down.

[prev]: http://blog.nelhage.com/2010/05/software-and-lab-notebooks/

Here's my advice for keeping a lab notebook as a computer scientist:

<dl>
<dt>Use a text file</dt>

<dd> <p>There's no need to use anything fancier. And if you're writing
code all day, you can just keep your log file open in your editor of
choice, so that you don't need to leave your hacking environment to
make a note of something. I'm an emacs junkie and use
<a href="http://orgmode.org/">org-mode</a> to keep all my notes, but use whatever you're
comfortable with.</p>


<p>You could use a physical notebook, but unless you expect to want
documentation of what you were working on for some legal reason, I
don't see the point. And most programmers I know hate writing things
out by hand.</p>
</dd>

<dt>Treat your file as append-only</dt>

<dd>One of the downsides of just using a text file is that you need to
enforce some discipline on yourself. It's tempting to go back and fix
up things that have changed, or to keep editing a desgin description
as it evolved. But creating a perfect finished piece of documentation
is a different task. The point here is to have a log of everything you
do, not a single finished document. You can work on one side-by-side
with the (dated or even timestampped) append-only log.</dd>

<dt>Keep automatic backups.</dt>

<dd>Do this any way you want. I keep my log files in git, and have a
cron job that commits and pushes to a server every five minutes. This
provides at least two benefits. First, it applies an automatic timestamp (to
within five minutes) on anything I type, and second, it serves to make
sure that a recent version is available on my server, in case I switch
machines and want to keep working on a project.
</dd>

<dt>Train yourself to make regular entries.</dt>

<dd>

<p>Keeping this nice text file, regularly backed up, is useless if you
don't record relevant things into it. Getting yourself into the habit
of writing enough down is tricky. One strategy I've found helpful is
to write down a list of "triggers" for situations that you promise to
always record. With an explict list of these that you've decided to
always record, you can get good at making a note whenever you perform
one of these tasks, which gets you into the habit of writing things
down. Here's a partial list of some of the sorts of things I make a
point of writing down:</p>

<ul>

<li>Whenever I take a measurement. That is, any time I run
<tt>time</tt>, or look at <tt>top</tt> to figure out how much memory a
program under development is using, or similar, I'll write down the
number and the circumstances under which it was recorded. I've found
that I almost always would like to have a better record of such things
later.</li>

<li>Whenever I do a non-trivial search and find a result. This can be
anything from googling for a solution to some problem where it takes
me several tries to find the right keywords, or grepping a large
codebase searching for the function that performs some task, or
tracking down some API function in documentation. If I wanted it once,
I'll often want it later, and if it was hard to find, I'll feel stupid
if I didn't write it down.</li>

<li>Whenever I make an explicit design decision. If I'm thinking about
how to design some function, system, process, or whatever, I'll stop
and take a moment to write down the options I was considering, and why
I made the choice I did. This ensures I can go back and remember why I
built a system the way it is.
</li>


<li>Whenever I wish I had written something down last time. This
should be obvious, but if you find yourself thinking "I know I did
this before; Why didn't I take notes?", you should
<strong>always</strong> make a point of writing it down this time, no
matter what excuse you come up with for why you really won't need that
information a third time.
</li>

</ul>

<p>These categories are deliberately broad, but at the same time
well-defined. It's fairly easy to train yourself to notice whenever
any of these points triggers, and remember to stop and write something
down. And that's the first step to getting in the habit of writing
down everything that you'll later wish you had written down.</p>

</dd>

</dl>
