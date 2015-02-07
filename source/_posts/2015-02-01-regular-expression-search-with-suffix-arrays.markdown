---
layout: post
title: "Regular Expression Search with Suffix Arrays"
date: 2015-02-01 15:52:43 -0800
comments: true
extra_css:
 - /stylesheets/diagram.css
categories: 
---

Back in January of 2012, Russ Cox posted an [excellent blog
post][trigram] detailing how Google Code Search had worked, using a
trigram index.

By that point, I'd already implemented early versions of my own
[livegrep][livegrep] source-code search engine, using a different
indexing approach that I developed independently, with input from a
few friends. This post is my long-overdue writeup of how it works.

[trigram]: http://swtch.com/~rsc/regexp/regexp4.html
[livegrep]: http://livegrep.com/

# Suffix Arrays

A [suffix array][suffix-array] is a data structure used for full-text
search and other applications, primarily these days in the field of
bioinformatics.

A suffix array is pretty simple, conceptually: It's a sorted array of
all of the suffixes of a string. So, given the string "hither and
thither", we could construct a suffix array like so:


<table class='diagram'>
<tr class='index'>
<td>0</td><td>1</td><td>2</td><td>3</td><td>4</td><td>5</td><td>6</td><td>7</td><td>8</td><td>9</td><td>10</td><td>11</td><td>12</td><td>13</td><td>14</td><td>15</td><td>16</td><td>17</td>
</tr>
<tr>
<td>h</td><td>i</td><td>t</td><td>h</td><td>e</td><td>r</td><td> </td><td>a</td><td>n</td><td>d</td><td> </td><td>t</td><td>h</td><td>i</td><td>t</td><td>h</td><td>e</td><td>r</td>
</tr>
</table>

<table class='diagram'>
<tr> <th>Index</th> <th colspan='5'>Suffix</th> </tr>
<tr>
<td class='index'>6</td>
<td>&nbsp;</td><td>a</td><td>n</td><td>d</td><td>&nbsp;</td><td>t</td><td>h</td><td>i</td><td>t</td><td>h</td><td>e</td><td>r</td>
</tr>
<tr>
<td class='index'>10</td>
<td>&nbsp;</td><td>t</td><td>h</td><td>i</td><td>t</td><td>h</td><td>e</td><td>r</td>
</tr>
<tr>
<td class='index'>7</td>
<td>a</td><td>n</td><td>d</td><td>&nbsp;</td><td>t</td><td>h</td><td>i</td><td>t</td><td>h</td><td>e</td><td>r</td>
</tr>
<tr>
<td class='index'>9</td>
<td>d</td><td>&nbsp;</td><td>t</td><td>h</td><td>i</td><td>t</td><td>h</td><td>e</td><td>r</td>
</tr>
<tr>
<td class='index'>16</td>
<td>e</td><td>r</td>
</tr>
<tr>
<td class='index'>4</td>
<td>e</td><td>r</td><td>&nbsp;</td><td>a</td><td>n</td><td>d</td><td>&nbsp;</td><td>t</td><td>h</td><td>i</td><td>t</td><td>h</td><td>e</td><td>r</td>
</tr>
<tr>
<td class='index'>15</td>
<td>h</td><td>e</td><td>r</td>
</tr>
<tr>
<td class='index'>3</td>
<td>h</td><td>e</td><td>r</td><td>&nbsp;</td><td>a</td><td>n</td><td>d</td><td>&nbsp;</td><td>t</td><td>h</td><td>i</td><td>t</td><td>h</td><td>e</td><td>r</td>
</tr>
<tr>
<td class='index'>12</td>
<td>h</td><td>i</td><td>t</td><td>h</td><td>e</td><td>r</td>
</tr>
<tr>
<td class='index'>0</td>
<td>h</td><td>i</td><td>t</td><td>h</td><td>e</td><td>r</td><td>&nbsp;</td><td>a</td><td>n</td><td>d</td><td>&nbsp;</td><td>t</td><td>h</td><td>i</td><td>t</td><td>h</td><td>e</td><td>r</td>
</tr>
<tr>
<td class='index'>13</td>
<td>i</td><td>t</td><td>h</td><td>e</td><td>r</td>
</tr>
<tr>
<td class='index'>1</td>
<td>i</td><td>t</td><td>h</td><td>e</td><td>r</td><td>&nbsp;</td><td>a</td><td>n</td><td>d</td><td>&nbsp;</td><td>t</td><td>h</td><td>i</td><td>t</td><td>h</td><td>e</td><td>r</td>
</tr>
<tr>
<td class='index'>8</td>
<td>n</td><td>d</td><td>&nbsp;</td><td>t</td><td>h</td><td>i</td><td>t</td><td>h</td><td>e</td><td>r</td>
</tr>
<tr>
<td class='index'>17</td>
<td>r</td>
</tr>
<tr>
<td class='index'>5</td>
<td>r</td><td>&nbsp;</td><td>a</td><td>n</td><td>d</td><td>&nbsp;</td><td>t</td><td>h</td><td>i</td><td>t</td><td>h</td><td>e</td><td>r</td>
</tr>
<tr>
<td class='index'>14</td>
<td>t</td><td>h</td><td>e</td><td>r</td>
</tr>
<tr>
<td class='index'>2</td>
<td>t</td><td>h</td><td>e</td><td>r</td><td>&nbsp;</td><td>a</td><td>n</td><td>d</td><td>&nbsp;</td><td>t</td><td>h</td><td>i</td><td>t</td><td>h</td><td>e</td><td>r</td>
</tr>
<tr>
<td class='index'>11</td>
<td>t</td><td>h</td><td>i</td><td>t</td><td>h</td><td>e</td><td>r</td>
</tr>
</table>

Note that there is no need for each element of the array to store the
entire suffix: We can just store the indexes (the left-hand column),
along with the original string, and look up the suffixes as needed. So
the above array would be stored just as


    [6 10 7 9 16 4 15 3 12 0 13 1 8 17 5 14 2 11]


There [exist][construction] algorithms to efficiently construct suffix
arrays (O(n) time), and in-place (constant additional memory outside
of the array itself).

[suffix-array]: http://en.wikipedia.org/wiki/Suffix_array
[construction]: http://en.wikipedia.org/wiki/Suffix_array#Construction_Algorithms

## Full-text substring search

Once we have a suffix array, one basic application is full-text
search.

Given a string to search, any substring of that text is a prefix of
some suffix. Thus, given a suffix array, any substring present in the
input must be the start of some entry in the suffix array. For
example, searching for "her" in "hither and thither", we find that it
is present in two places in the string, and it begins two lines of the
suffix array:

<table class='diagram'>
<tr>
<td class='index'>15</td>
<td class='match'>h</td><td class='match'>e</td><td class='match'>r</td>
</tr>
<tr>
<td class='index'>3</td>
<td class='match'>h</td><td class='match'>e</td><td class='match'>r</td><td>&nbsp;</td><td>a</td><td>n</td><td>d</td><td>&nbsp;</td><td>t</td><td>h</td><td>i</td><td>t</td><td>h</td><td>e</td><td>r</td>
</tr>
</table>

But because the suffix array is sorted, we can find those entries
efficiently by binary searching in the suffix array. The indexes then
give us the location of the search string in the larger text.

# Towards Regular Expression Search

So, we have a large corpus of source code we want to do regular
expression search over.

First, livegrep reads all of the source and flattens them together
into a single enormous buffer (remembering which regions of the buffer
correspond to which source files), and building a suffix array over
that buffer. (This is something of a simplification; future blog posts
will discuss the specifics in more detail, but it's close enough).

But now how do we use that array to match regular expressions?

As a first pass, we can find literal substrings of the regular
expression, find any occurrences of those strings using the suffix
array, and then only search those locations in the corpus.

For example, given `/hello.*world/`, we can easily see that any
mathches must contain the string "hello", and so we could look up
"hello", find all lines containing it, and then check each line
against the full regular expression.

## More complex searching

But it turns out we can do better. Because of the structure of the
suffix array, we can perform at least two basic queries beyond
substring search:

- Range search: We can efficiently find all instances of a character
  range, by binary-searching each end. If we have the range `[A-F]`,
  we binary-search for the first suffix starting with `A`, and the
  last suffix starting with `F`, and we know that every index in
  between (in the suffix array) starts with something between A and F.

- Chained searches: If we have a chunk of the suffix array that we
  know has a common prefix, we can narrow down the search by doing
  additional searches and looking at the *next* character, within that
  range. For example, if we want to find `/hi(s|m)/`, we can
  binary-search to find all positions starting with `hi`, which gives
  us a contiguous block of the suffix array. But since that block is
  sorted, we can do *another* pair of binary searches within that
  range, looking only at the third character, one looking for `s` and
  one for `m`, which will give us two smaller chunks, once for "his"
  and one for "him".

And we can combine these strategies. We can search for `/[a-f][0-9]/`
by:

1. Binary searching to find `a-f`
2. Splitting that into 6 ranges, one for `a`, `b`, `c`, `d`, `e`, and
   `f`
3. For each of those ranges, doing a binary search over the *second*
   character of the suffix, to find the 0-9 range


<div>
<div class='searchdemo'>
  1.
  <table class='diagram' id='d1'>
  <tr><td style='height: 50px'>…</td></tr>
  <tr><td class='match thick down'>AAAAAA</td></tr>
  <tr><td class='match thick up down'>&nbsp;</td></tr>
  <tr><td class='match thick up down'>&nbsp;</td></tr>
  <tr><td class='match thick up down'>…</td></tr>
  <tr><td class='match thick up down'>&nbsp;</td></tr>
  <tr><td class='match thick up'>FZZZZZ</td></tr>
  <tr><td style='height: 50px'>…</td></tr>
  </table>
</div>

<div class='searchdemo'>
  2.
  <table class='diagram'>
  <tr><td style='height: 50px'>…</td></tr>
  <tr><td class='match thick greydown'>A…</td></tr>
  <tr><td class='match thick greyup greydown'>B…</td></tr>
  <tr><td class='match thick greyup greydown'>C…</td></tr>
  <tr><td class='match thick greyup greydown'>D…</td></tr>
  <tr><td class='match thick greyup greydown'>E…</td></tr>
  <tr><td class='match thick greyup'>F…</td></tr>
  <tr><td style='height: 50px'>…</td></tr>
  </table>
</div>

<div class='searchdemo'>
  3.

  <table class='diagram'>
  <tr><td style='height: 50px' colspan='2'>…</td></tr>
  <tr>
    <td rowspan='3' class='match thick greydown'>A…</td>
    <td class='up down no thin'></td>
  </tr>
  <tr><td class='greyup greydown match'>A[0-9]…</td></tr>
  <tr><td class='up down no thin'></td></tr>
  <tr>
    <td rowspan='3' class='match thick greydown'>B…</td>
    <td class='up down no thin'></td>
  </tr>
  <tr><td class='greyup greydown match'>B[0-9]…</td></tr>
  <tr><td class='up down no thin'></td></tr>
  <tr>
    <td rowspan='3' class='match thick greydown'>C…</td>
    <td class='up down no thin'></td>
  </tr>
  <tr><td class='greyup greydown match'>C[0-9]…</td></tr>
  <tr><td class='up down no thin'></td></tr>
  <tr>
    <td rowspan='3' class='match thick greydown'>D…</td>
    <td class='up down no thin'></td>
  </tr>
  <tr><td class='greyup greydown match'>D[0-9]…</td></tr>
  <tr><td class='up down no thin'></td></tr>
  <tr>
    <td rowspan='3' class='match thick greydown'>E…</td>
    <td class='up down no thin'></td>
  </tr>
  <tr><td class='greyup greydown match'>E[0-9]…</td></tr>
  <tr><td class='up down no thin'></td></tr>
  <tr>
    <td rowspan='3' class='match thick greydown'>F…</td>
    <td class='up down no thin'></td>
  </tr>
  <tr><td class='greyup greydown match'>F[0-9]…</td></tr>
  <tr><td class='up down no thin'></td></tr>
  <tr><td style='height: 50px' colspan='2'>…</td></tr>
  </table>
</div>

<div>

When we're done, each of the resulting ranges in the suffix array then
contains only locations matching `/[A-F][0-9]/`.

Approximately, this means we can handle any regular expression
consisting of character classes, literal characters, and alternations
(`|`) — we handle alternations and non-contiguous character classes
essentially by matching each side separately and combining the
matches.

A future blog post will give more details about how we walk an
arbitrary regular expression and extract a relevant piece for
indexing, but this is the basic idea. In the meanwhile,
[indexer.cc][indexer.cc] contains the hairy details if you're curious.

[indexer.cc]: https://github.com/livegrep/livegrep/blob/master/src/indexer.cc

## Using the Results

Walking the suffix array as above produces a (potentially-large)
number of positions in the input corpus, which we want to
search. Rather than searching each individually, livegrep takes all of
the matches and sorts them in memory. We then walk the list in order,
and if several matches are close together, we just feed the entire
range of text to `RE2` at once, which reduces the number of RE2 calls
and prevents us from duplicating work if a single line has multiple
matches for from the index lookup.

Once we've run `RE2` over all of the match positions, we have a list
of positions in the corpus that we know to match. We look up which
files those correspond to (using, essentially, just a table that maps
character ranges to source file names), and return the matches.

If we've also been given a file reference to constrain the search
(e.g. `file:foo\.c`), we walk the file-contents map in parallel with
the list of results from the index walk, and discard positions unless
the file containing them matches the file regular expression.


# More…

That's the basic picture. Next time, I'll discuss in more detail how
we construct index queries and transform regular expressions into an
index query, and then finally I'll talk about how the suffix array and
file-contents data structures actually work in livegrep, versus the
simplified version presented here. Notably, livegrep actually
compresses the input significantly, which reduces the memory needed
and speeds up the searches, at the cost of additional complexity
building the index and interpreting results.
