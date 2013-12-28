---
layout: post
status: publish
published: true
title: Tracking down a memory leak in Ruby's EventMachine
author: nelhage
author_login: nelhage
author_email: nelhage@mit.edu
author_url: http://nelhage.com
wordpress_id: 1863
wordpress_url: http://blog.nelhage.com/?p=1863
date: 2013-03-07 13:13:37.000000000 +01:00
tags: []
---
At [Stripe][stripe], we rely heavily on [ruby][ruby] and
[EventMachine][EM] to power various internal and external
services. Over the last several months, we've known that one such
service suffered from a gradual memory leak, that would cause its
memory usage to gradually balloon from a normal ~50MB to multiple gigabytes.

It was easy enough to work around the leak by adding monitoring and
restarting the process whenever memory usage grew too large, but we
were determined to track down the root cause. Our exploration is a
tour through a number of different debugging tools and techniques, so
I thought I would share it here.

Checking for ruby-level leaks
-----------------------------

One powerful technique for tracking down tough memory leaks is
post-mortem analysis. If our program's normal memory footprint is
50MB, and we let it leak until it's using, say, 2GB, 
1950/2000 =
97.5% of the program's memory is leaked objects! If we look at a core
file (or, even better, a running image in `gdb`), signs of the leak
will be all over the place.

So, we let the program leak, and, when its memory usage got large
enough, failed active users over to a secondary server, and attached
gdb to the bloated image.

Our first instinct in a situation like this is that our Ruby code is
leaking somehow, such as by accidentally keeping a list of every
connection it has ever seen. It's easy to investigate this possibility
by using gdb, the Ruby C API, and the Ruby internal GC hooks:

    (gdb) p rb_eval_string("GC.start")
    $1 = 4
    (gdb) p rb_eval_string("$gdb_objs = Hash.new 0")
    $2 = 991401552
    (gdb) p rb_eval_string("ObjectSpace.each_object {|o| $gdb_objs[o.class] += 1}")
    $3 = 84435
    (gdb) p rb_eval_string("$stderr.puts($gdb_objs.inspect)")
    $4 = 4

Calling `rb_eval_string` lets us inject arbitrary ruby code into the
running ruby process, from within gdb. Using that, we first trigger a
GC -- making sure that any unreferenced objects are cleaned up -- and
then walk the Ruby ObjectSpace, building up a census of which Ruby
objects exist. Looking at the output and filtering for the top
objects, we find:

    String => 26399
    Array  => 8402
    Hash   => 2161
    Proc   => 608

Just looking at those numbers, my gut instinct is that nothing looks
too out of whack. Running some back-of-the-envelope numbers confirms
this instinct:

  - In total, that's about 40,000 objects for those object types.
  - We're looking for a gradual leak, so we expect lots of small
    objects. Let's guess that "small" is around 1 kilobyte.
  - 40,000 1k objects is only 40MB. Our process is multiple GB at
    this point, so we are nowhere near explaining our memory usage.

It's possible, of course, that *one* of those Strings has been growing
without bound, and is now billions of characters long, but that feels
unlikely. A quick survey of String lengths using `ObjectSpace` would
confirm that, but we didn't even bother at this point.

Searching for C object leaks
----------------------------

So, we've mostly ruled out a Ruby-level leak. What now?

Well, as mentioned, 95+% of our program's memory
footprint is leaked objects. So if we just take a random sample of
bits of memory, we will find leaked objects with very good
probability. We generate a core file in gdb:

    (gdb) gcore leak.core
    Saved corefile leak.core

And then look at a random page (4k block) of the core file:

    $ off=$(($RANDOM % ($(stat -c "%s" leak.core)/4096)))
    $ dd if=leak.core bs=4096 skip=$off count=1 | xxd
    0000000: 0000 0000 0000 0000 4590 c191 3a71 b2aa  ........E...:q..
    ...

Repeating a few times, we notice that most of the samples include what
looks to be a repeating pattern:

    00000f0: b05e 9b0a 0000 0000 0000 0000 0000 0000  .^..............
    0000100: 0000 0000 0000 0000 0100 0000 dcfa 1939  ...............9
    0000110: 0000 0000 0000 0000 0a05 0000 0000 0000  ................
    0000120: 0000 0000 0000 0000 d03b a51f 0000 0000  .........;......
    0000130: 8000 0000 0000 0000 8100 0000 0000 0000  ................
    0000140: 00a8 5853 1b7f 0000 0000 0000 0000 0000  ..XS............
    0000150: 0000 0000 0000 0000 0100 0000 0100 0000  ................
    0000160: 0000 0000 0000 0000 ffff ffff 0000 0000  ................
    0000170: 40ef e145 0000 0000 0000 0000 0000 0000  @..E............
    0000180: 0000 0000 0000 0000 0100 0000 0000 0000  ................
    0000190: 0000 0000 0000 0000 0a05 0000 0000 0000  ................
    00001a0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
    00001b0: 8000 0000 0000 0000 8100 0000 0000 0000  ................
    00001c0: 00a8 5853 1b7f 0000 0000 0000 0000 0000  ..XS............
    00001d0: 0000 0000 0000 0000 0100 0000 0100 0000  ................
    00001e0: 0000 0000 0000 0000 ffff ffff 0000 0000  ................
    00001f0: 4062 1047 0000 0000 0000 0000 0000 0000  @b.G............
    0000200: 0000 0000 0000 0000 0100 0000 1b7f 0000  ................
    0000210: 0000 0000 0000 0000 0a05 0000 0000 0000  ................
    0000220: 0000 0000 0000 0000 e103 0000 0000 0000  ................
    0000230: 8000 0000 0000 0000 9100 0000 0000 0000  ................
    0000240: 00a8 5853 1b7f 0000 0000 0000 0000 0000  ..XS............
    0000250: 0000 0000 0000 0000 0100 0000 0100 0000  ................
    0000260: 0000 0000 0000 0000 ffff ffff 0000 0000  ................
    0000270: 50b0 350b 0000 0000 0000 0000 0000 0000  P.5.............
    0000280: 0000 0000 0000 0000 0100 0000 0000 0000  ................
    0000290: 0000 0000 0000 0000 0a05 0000 0000 0000  ................
    00002a0: 0000 0000 0000 0000 30de 4027 0000 0000  ........0.@'....
    00002b0: 0077 7108 0000 0000 0060 7b97 b4d2 f111  .wq......`{.....
    00002c0: 9000 0000 0000 0000 8100 0000 0000 0000  ................
    00002d0: 00a8 5853 1b7f 0000 0000 0000 0000 0000  ..XS............
    00002e0: 0000 0000 0000 0000 0100 0000 0100 0000  ................

Those `ffff ffff` blocks, repeated every 128 bytes, leap out at me,
and 4 out of 5 samples of the core file reveal a similar pattern. It seems
probable that we're leaking 128-byte objects of some sort, some field
of which is `-1` as a signed 32-bit integer, i.e. `ffff ffff` in hex.

Looking further, we also notice a repeated `00a8 5853 1b7f 0000`, two lines before each `ffff ffff`. If you've stared at too many Linux coredumps, as I
have, that number looks suspicious. Interpreted in little-endian, that
is `0x00007f1b5358a800`, which points near the top of the userspace
portion of the address space on an amd64 Linux machine.

In fewer words: It's most likely a pointer.

The presence of an identical pointer in every leaked object suggests
that the pointer most likely points to some kind of "type" object or
tag, containing information about type of the leaked object. For
instance, if we were leaking Ruby String objects, every one would have
an identical pointer to the Ruby object that represents the `String'
class. So, let's take a look:

    (gdb) x/16gx 0x00007f1b5358a800
    0x7f1b5358a800:	0x0000000000000401	0x00007f1b53340f24
    0x7f1b5358a810:	0x00007f1b532c7780	0x00007f1b532c7690
    0x7f1b5358a820:	0x00007f1b532c7880	0x00007f1b532c78c0
    0x7f1b5358a830:	0x00007f1b532c74f0	0x00007f1b532c74c0
    0x7f1b5358a840:	0x00007f1b532c7470	0x0000000000000000
    0x7f1b5358a850:	0x0000000000000000	0x0000000000000000
    0x7f1b5358a860:	0x0000000000000406	0x00007f1b5332a549
    0x7f1b5358a870:	0x00007f1b532c7a40	0x00007f1b532c7a30

The first field, `0x401`, contains only two bits set, suggesting some
kind of flag field. After that, there are a whole bunch of
pointers. Let's chase the first one:

    (gdb) x/s 0x00007f1b53340f24
    0x00007f1b53340f24:	"memory buffer"

Great. So we are leaking ... memory buffers. Thanks.

But this is actually fantastically informative, especially coupled
with the one other piece of information we have: `/proc/<pid>/maps`
for the target program, which tells us which files are mapped into our
program at which addresses. Searching that for the target address, we
find:

    7f1b53206000-7f1b5336c000 r-xp 00000000 08:01 16697      /lib/libcrypto.so.0.9.8

`0x7f1b53206000` ≤ `0x7f1b5358a800` < `7f1b5336c000`, so this mapping
contains our "type" object. `libcrypto` is the library containing
OpenSSL's cryptographic routines, so we are leaking some sort of
OpenSSL buffer object. This is real progress.

I am not overly familiar with libssl/libcrypto, so let's go to the
source to learn more:

    $ apt-get source libssl0.9.8
    $ cd openssl*
    $ grep -r "memory buffer" .
    ./crypto/err/err_str.c:{ERR_PACK(ERR_LIB_BUF,0,0)		,"memory buffer routines"},
    ./crypto/asn1/asn1.h: * be inserted in the memory buffer
    ./crypto/bio/bss_mem.c:	"memory buffer",
    ./README:        sockets, socket accept, socket connect, memory buffer, buffering, SSL
    ./doc/ssleay.txt:-	BIO_s_mem()  memory buffer - a read/write byte array that
    ./test/times:talks both sides of the SSL protocol via a non-blocking memory buffer

Only one of those is a string constant, so we go browse
`./crypto/bio/bss_mem.c` and read the docs ([bio(3)][bio] and
[buffer(3)][buffer]) a bit. Sparing you all the details, we learn:

 - OpenSSL uses the `BIO` structure as a generic abstraction around any kind of
   source or sink of data that can be read or written to.
 - A `BIO` has a pointer to a `BIO_METHOD`, which essentially contains
   a small amount of metadata and a
   [vtable](http://en.wikipedia.org/wiki/Virtual_method_table),
   describing what specific kind of `BIO` this is, and how to interact
   with it. The second field in a `BIO_METHOD` is a `char *` pointing
   at a string holding the name of this type.
 - One of the common types of `BIO`s is the `mem` `BIO`, backed
   directly by an in-memory buffer (a `BUF_MEM`). The `BIO_METHOD` for
   memory `BIO`s has the type tag `"memory buffer"`.

So, it appears we are leaking `BIO` objects. Interestingly, we don't
actually seem to be leaking the underlying memory buffers, just the
`BIO` struct that contains the metadata about the buffer.

[bio]: http://www.openssl.org/docs/crypto/bio.html
[buffer]: http://www.openssl.org/docs/crypto/buffer.html

Tracing the source
------------------

This kind of leak has to be in some C code somewhere. Clearly nothing in pure ruby code should be able to do this. The server in question contains no C extensions we wrote, so it's probably in some third-party library we use.

A leak in OpenSSL itself is certainly possible, but OpenSSL is very
widely used and quite mature, so let's assume (hope) that our leak is not
there, for now.

That leaves EventMachine as the most likely culprit. Our program uses
SSL heavily via the EventMachine APIs, and we do know that
EventMachine contains a bunch of C/C++, which, to be frank, does not
have a sterling reputation.

So, we pull up an EventMachine checkout. There are a number of ways to
construct a new `BIO`, but the most basic and common seems to be
`BIO_new`, sensibly enough. So let's look for that:

    $ git grep BIO_new
    ext/rubymain.cpp:		out = BIO_new(BIO_s_mem());
    ext/ssl.cpp:	BIO *bio = BIO_new_mem_buf (PrivateMaterials, -1);
    ext/ssl.cpp:	pbioRead = BIO_new (BIO_s_mem());
    ext/ssl.cpp:	pbioWrite = BIO_new (BIO_s_mem());
    ext/ssl.cpp:	out = BIO_new(BIO_s_mem());

Great: there are some calls (so it could be one of them!), but not too
many (so we can reasonably audit them all).

Starting from the top, we find, in `ext/rubymain.cpp`:

    static VALUE t_get_peer_cert (VALUE self, VALUE signature)
    {
    	VALUE ret = Qnil;
    	X509 *cert = NULL;
    	BUF_MEM *buf;
    	BIO *out;

    	cert = evma_get_peer_cert (NUM2ULONG (signature));

    	if (cert != NULL) {
    		out = BIO_new(BIO_s_mem());
    		PEM_write_bio_X509(out, cert);
    		BIO_get_mem_ptr(out, &buf);
    		ret = rb_str_new(buf->data, buf->length);
    		X509_free(cert);
   		BUF_MEM_free(buf);
    	}

    	return ret;
    }

The OpenSSL APIs are less than perfectly self-descriptive, but it's not too hard to puzzle out what's going on here: 

We first construct a new `BIO` backed by a memory buffer:

    out = BIO_new(BIO_s_mem());

We write the certificate text into that `BIO`:

    PEM_write_bio_X509(out, cert);

Get a pointer to the underlying `BUF_MEM`:

    BIO_get_mem_ptr(out, &buf);

Convert it to a Ruby string:

    ret = rb_str_new(buf->data, buf->length);

And then free the memory:

    BUF_MEM_free(buf);

But we've called the wrong free function! We're freeing `buf`, which
is the underlying `BUF_MEM`, but we've leaked `out`, which is the
`BIO` itself we also allocated. This is exactly the kind of leak we saw in our
core dump!

Continuing our audit, we find the exact same bug in
`ssl_verify_wrapper` in `ssl.cpp`. Reading code, we learn that
`t_get_peer_cert` is called by the Ruby function [`get_peer_cert`][em-getpeercert], used
to retrieve the peer certificate from a TLS connection, and that
`ssl_verify_wrapper` is called if you pass `:verify_peer => true` to
[`start_tls`][em-starttls], to convert the certificate into a Ruby
string for passing to the `ssl_verify_peer` hook.

So, any time you make a `TLS` or `SSL` connection with EventMachine
**and verify the peer certificate**, EventMachine was leaking 128
bytes! Since it's presumably pretty rare to make a large number of SSL
connections from a single Ruby process, it's not totally surprising
that no one had caught this before.

Having found the issue, the
[fix](https://github.com/eventmachine/eventmachine/commit/b2006a6f4893f35ca8b1b5fc283f3b1e2127bc5c)
was simple, and was promptly
[merged](https://github.com/eventmachine/eventmachine/commit/016800f60bd1ec1894fd73ccd0c2634f5fabc1c9)
upstream.

[stripe]: https://stripe.com
[ruby]: http://www.ruby-lang.org/en/
[EM]: http://rubyeventmachine.com/
[em-getpeercert]: http://eventmachine.rubyforge.org/EventMachine/Connection.html#get_peer_cert-instance_method
[em-starttls]: http://eventmachine.rubyforge.org/EventMachine/Connection.html#start_tls-instance_method
