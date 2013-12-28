---
layout: post
status: publish
published: true
title: ! 'Dear Twitter: Stop screwing over your developers. '
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 314
wordpress_url: http://blog.nelhage.com/?p=314
date: 2010-09-12 23:48:28.000000000 +02:00
tags:
- security
- twitter
- oauth
- stupidity
- lies
- barnowl
---
I really like Twitter. I think it's a great, fun, service, that helps
enable interesting online communities, and is a surprisingly effective
way to spread news and information to lots of people online. One of
the things that I've loved about Twitter is their API, and how open
and welcoming they've been to developers. I even use Twitter from an
IM [client][barnowl] that I develop, using protocol support that I
wrote myself.

[barnowl]: http://barnowl.mit.edu

However, as part of the recent transition to OAuth authentication,
Twitter has made a number of moves that, in my opinion, show that they
just don't care about the developers that write clients for their
service any more. They're evidently too busy pandering to their large
corporate partners and scrambling for sources of income, and they
think they're too cool to have to care about the individual developers
who make their service available to such a wide range of users and use
styles.

Background: OAuth and consumer secrets
--------------------------------------

This all started when Twitter transitioned their API's authorization
scheme to OAuth, an open standard for online authentication, that
provides a much richer, more powerful, and all-around safer security
model than simply handing around passwords in the clear, like Twitter
used to require.

As part of the transition, Twitter requires that all applications to
obtain a "consumer key" and "consumer secret" with Twitter, which are
essentially analogous to a username and password used to identify the
application making the API requests.

This is perfectly reasonable for web applications, where the code runs
server-side, and so can actually keep and use secrets
effectively. However, it's hopeless on the desktop. We have at this
point many years of precedent for the problem of a desktop application
having and using a secret key without revealing that secret key to a
user willing to dig for it: That's exactly the problem [DRM][drm]
systems have tried to solve time and time again, and repeatedly
failed.

The OAuth specification [recognizes][oauth-consumer] this fact, and states,

> In many cases, the client application will be under the control of
> potentially untrusted parties.  For example, if the client is a
> desktop application with freely available source code or an
> executable binary, an attacker may be able to download a copy for
> analysis.  In such cases, attackers will be able to recover the
> client credentials.
> 
> Accordingly, servers should not use the client credentials alone to
> verify the identity of the client.
   
Accordingly, most sane implementations of OAuth, such as Google's
Buzz, don't require secrets from desktop applications, although they
may allow a User-Agent string or similar for apps to voluntarily
identify themselves (which may be reflected in UI, for instance).

Twitter, however, blatantly ignores that statement from the OAuth
spec, and insists that all applications, even desktop applications,
(impossibly) keep their consumer key secret, so that Twitter retains
the power to identify which applications are posting what, and the
ability to disable applications that are spamming or broken.

When confronted about the impossibility of this requirement, Twitter
refused to change their stance, insisting that users instead make a
["best effort"][twitter-ml] attempt to keep their keys secret.

Twitter can (and will) turn off your app
----------------------------------------

This is, of course, impossible requirement for open-source
applications, where the source is to be shared freely. If I want to
distribute an open-source application, Twitter [requires][twitter-oss]
that I do not distribute a consumer secret, but rather require every
user who downloads my source to register their own app (a process that
is optimized, in terms of user experience, for developers, and not the
average user), and plug it into configuration or into the
application. They have held to this position despite the fact that
this is an awful user experience for developers or users who want to
use an open-source application. I can only conclude that Twitter has
decided that small, open-source applications are not worth their
worry, and they don't care about forcing their users to jump through
absurd hoops.

Like I said, I develop an open-source Twitter application. I was
vaguely aware of Twitter's policy on consumer secrets, but concluded
that their documentation must just be confused, since their
requirements seemed utterly unreasonable. Accordingly, when I
implemented OAuth support, I just [embedded][barnowl-key] my
credentials in my source, to make life easy for all of my users.

Recently, however, after I mentioned Twitter's broken OAuth policies
on Twitter, I received an email from Twitter Support:

> It has recently come to our attention that an application registered to you,
> BarnOwl, has had its Consumer Secret and Key pair posted publicly online
>
> […]
>
> and as such we've reset your API keys and ask that in the future you ensure
> these are not posted publicly.

And, just like that, my app suddenly appeared broken to every one of my users
(they began receiving unhelpful "401 Unauthorized" errors).

Now, in this case, it was my "fault" for posting the key publicly. But
suppose I did package my application and made a "best effort" attempt
to hide my keys, but some hacker came along, extracted my keys, and
posted them online? According to the policy described in that email
above, I can only believe that Twitter would also find it necessary to
revoke my credentials, again breaking my app for all of my users. As a
developer, and as the one who will get blamed if that happens, I'm
suddenly a lot less thrilled about writing a Twitter client.

Twitter (and their buddies) are exempt from the rules
-----------------------------------------------------

Of course, if I'm a big enough player that Twitter *cares* about my
user base, I'm immune. Twitter's own Android app's credentials were
compromised in a high-profile [article][ars] on arstechnica, and
subsequently
[tweeted](http://twitter.com/artanis00/status/22866583578), but I have
yet to see Twitter break their own app and lose users because someone
else knew how to use `strings`.

Similarly, [gwibber][gwibber], the microblogging application that
ships enabled by default in Ubuntu Lucid, ships its credentials in
plain-text in the file `/usr/share/gwibber/data/twitter` on every
recent Ubuntu Lucid machine. But, according to the author of the same
Ars article, apparently they're immune from retribution because
Canonical "negotiated a compromise" to allow the app to continue
working. Whatever this compromise was, it apparently didn't involve
Canonical making any effort to keep their keys secret.

The message to me here couldn't be clearer. If you're a big player, or
Twitter itself, your app is immune from takedown by Twitter, because
Twitter doesn't want to offend you or lose your users. But if you're a
little guy, any rogue hacker who knows how to run `strings` has the
ability to cause Twitter to suddenly and mysteriously break your
application for everyone who uses it.

Twitter is lying to us about their policies
-------------------------------------------

Twitter has [promised][ratelimit] that they "don't discriminate or
prioritize anyone on the API, even ourselves". But, it turns out,
Twitter does discriminate. Even as Twitter
[claims](http://dev.twitter.com/announcements) that Basic Auth is
deprecated, and "All applications must now use OAuth", they built a
back-door into their API, allowing old versions of their Android
client -- or anyone pretending to be it -- to continue using basic
authentication, simply by appending `?source=twitterandroid` to their
URLs:

    $ curl -u nelhage http://api.twitter.com/1/account/rate_limit_status.json
    Enter host password for user 'nelhage':
    {"remaining_hits":0,
     "hourly_limit":0,
     "reset_time":"Mon Sep 13 00:29:37 +0000 2010",
     "reset_time_in_seconds":1284337777}
    $ curl -u nelhage http://api.twitter.com/1/account/rate_limit_status.json?source=twitterandroid
    Enter host password for user 'nelhage':
    {"remaining_hits":150,
     "reset_time":"Mon Sep 13 00:29:44 +0000 2010",
     "hourly_limit":150,
     "reset_time_in_seconds":1284337784}

When August 31 of this year rolled around, Twitter disabled Basic
Authentication for all clients but their own. Even though this flag
day was well-publicized to developers, it still caught many developers
and users unaware, and users running old versions found their
applications suddenly breaking, through no fault of their own --
unless they were running Twitter's own app. Twitter gave itself a free
pass to all the pain and frustrated users that everyone else had to
cope with.

Aside: What counts as "best-effort"?
------------------------------------

I decided to dig into some Twitter applications I or friends use to
determine just how well they protect their credentials. How much work
do I have to do to be safe from Twitter's consumer-secret patrol?

The answer, apparently, is "not much":

<dl>
 <dt>Twitter for Android</dt>

 <dd>As mentioned, their keys are in the app's <tt>classes.dex</tt>
 file in plain text. Their keys have been <a
 href="http://twitter.com/artanis00/status/22866583578">published
 online</a>, but apparently Twitter doesn't care.</dd>

 <dt>Seesmic for Android</dt>

 <dd>Same story as Twitter's app. Plain-text in the <tt>.dex</tt>,
 although they haven't been published online as far as I know. </dd>

 <dt>Gwibber</dt> <dd>As mentioned, shipped in plain text with
 Lucid. Apparently Canonical is a big enough player that they don't
 have to follow the rules.</dd>

 <dt>Twirssi</dt>

 <dd><p>Twirssi embeds Twitter in irssi, a popular IRC client. This is another
 "small fry", on a vaguely comparable scale to mine, so I was especially curious
 what they did. The answer, apparently ... is <a href="http://github.com/zigdon/twirssi/blob/master/twirssi.pl#L544">rot13:</a></p>

 <code><pre>
 $twit = Net::Twitter->new(
     traits => [ 'API::REST', 'OAuth', 'API::Search' ],
     ( grep tr/a-zA-Z/n-za-mN-ZA-M/, map $_,
       pbafhzre_xrl => 'OMINiOzn4TkqvEjKVioaj',
       pbafhzre_frperg => '0G5xnujYlo34ipvTMftxN9yfwgTPD05ikIR2NCKZ',
     ),
     source => "twirssi",
     ssl    => !Irssi::settings_get_bool("twirssi_avoid_ssl"),
 );
 </code></pre>

 <p>Do you feel secure yet?</p></dd>
</dl>


[oauth]: http://oauth.net/
[oauth-consumer]: http://tools.ietf.org/html/rfc5849#page-30
[twitter-ml]: http://groups.google.com/group/twitter-development-talk/msg/965d471cb39fac6d
[twitter-oauth]: http://dev.twitter.com/pages/basic_to_oauth
[twitter-oss]: http://groups.google.com/group/twitter-development-talk/browse_thread/thread/c16c72e691c77312?pli=1
[twirssi]: http://github.com/zigdon/twirssi/blob/master/twirssi.pl#L529
[ars]: http://arstechnica.com/security/guides/2010/09/twitter-a-case-study-on-how-to-do-oauth-wrong.ars
[unwarranted]: http://benlog.com/articles/2010/09/02/an-unwarranted-bashing-of-twitters-oauth/
[defending]: http://benlog.com/articles/2010/09/07/defending-against-your-own-stupidity/
[ratelimit]: http://dev.twitter.com/pages/rate_limiting_faq#who
[drm]: http://en.wikipedia.org/wiki/Digital_rights_management
[barnowl-key]: http://github.com/barnowl/barnowl/blob/1eafdfaa641575186dfaf8f9310b6105b08f8d92/lib/BarnOwl/Module/Twitter/Handle.pm#L33
[gwibber]: http://gwibber.com/
[android-key]: http://twitter.com/artanis00/status/22866583578
