---
layout: post
status: publish
published: true
title: Followup to "A Very Subtle Bug"
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 160
wordpress_url: http://blog.nelhage.com/?p=160
date: 2010-03-03 13:45:11.000000000 +01:00
tags:
- unix
- python
- tar
- followup
---
After my [previous post][0] got posted to [reddit][1], there was a bunch of
interesting discussion there about some details I'd handwaved
over. This is a quick followup on some the investigation that various
people carried out, and the conclusions they reached.

[0]: /2010/02/a-very-subtle-bug

In the reddit thread, [lacos/lbzip2][2] [objected][6] that in his
experiments, he didn't see `tar` closing the input pipe before it was
done reading the file, and so questioned where the `SIGPIPE`/`EPIPE`
was coming from in the first place. I had actually done similar
experiments with similar results, but I was still seeing the `EPIPE`,
so I knew it could happen, but I couldn't totally explain why.

A friend of mine, David Benjamin, was curious enough to source-dive
`tar`, and [posted his results][3] on his own blog. He discovered that
by default, `tar` does not close the pipe after finding all the files
it needs, because the `tar` archive format allows for later copies of
the same file, which would supercede the previous ones. This explains
why `lacos` and I saw `tar` reading to the end of a `linux-2.6`
tarball, even if we only asked for the first file.

He also discovered, however, that a typical tar file ends with a
number of `NUL` blocks, which `tar` treats as end-of-file. And so
`tar` will close the pipe after reading the first of these, which
opens a narrow race condition whereby tar can potentially do so before
`gzip` has written the remaining `NUL` blocks, resulting in a
`SIGPIPE`.

Finally, the discussion inspired `lacos` to post a [query][4] clarifying `tar`'s behavior with respect to SIGPIPE and closing the pipe early to the `help-tar`
mailing list, which resulted in a brief thread that, among other
things, revealed that the bug I posted about has been [fixed][5] in
GNU tar as of last summer, by having `tar` reset the disposition of
`SIGPIPE` to `SIG_DFL` before spawning a child. It was also pointed out that tar checks whether a filter subprocess is killed by `SIGPIPE`, and treats that as a success -- so it's not actually necessary for a `tar` filter to handle `SIGPIPE` and exit cleanly, like `gzip` does.

[1]: http://www.reddit.com/r/programming/comments/b7djd/stuff_like_this_makes_me_hate_python_subtle_bugs/
[2]: http://lacos.hu/
[3]: http://davidben.scripts.mit.edu/blog/2010/02/28/tar-filled-pipes/
[4]: http://lists.gnu.org/archive/html/help-tar/2010-03/msg00000.html
[5]: http://lists.gnu.org/archive/html/bug-tar/2009-06/msg00009.html
[6]: http://www.reddit.com/r/programming/comments/b7djd/stuff_like_this_makes_me_hate_python_subtle_bugs/c0lc0dy
