---
title: "Approximate  deduplication with Jaccard similarity and MinHash"
slug: fuzzy-dedup
date: 2024-05-26T12:07:28-07:00
math: true
---
Suppose we have a large collection of documents, and we wish you identify which documents are **approximately** identical to each other. How would we go about that operation? We’re particularly interested in the case where the collection is very large — perhaps a terabyte or more — and so efficiency is paramount.

In this post I want to explore the method of approximate deduplication via Jaccard similarity and the MinHash approximation trick. This is a commonly-used approach to this problem (e.g. the [GPT-3 paper](https://arxiv.org/pdf/2005.14165) describes using it as part of their dataset preparation pipeline), but one I found a bit hard to understand from available resources.

![Excerpt from the GPT-3 describing how they removed approximate duplicates form the training dataset](gpt-3-dedup.png)

# Jaccard similarity

A natural approach to approximate deduplication is to define some notion of “similarity” between any pair of documents, and group together any two documents whose similarity value is above some threshold. So if we have some universe of possible documents \\(U\\), we might want a similarity measure $$S: U \times U \rightarrow [0,1]$$
and we’ll consider two documents “approximately the same” if \\(S(A,B) \geq S_\textrm{crit}\\).[^transitive]

[^transitive]: Mostly an aside for now: note that this definition is not necessarily transitive — we may have documents \\(A, B, C\\) such that \\(S(A,B)\geq{}S_\textrm{crit}\\) and \\(S(B,C) \geq{} S_\textrm{crit}\\) but \\(S(A,B) < S_\textrm{crit}\\). We’ll have to decide what to do in such circumstances — do we group together all three documents? If we are deduplicating in an online fashion, and we throw out \\(B\\) when we first encounter it, does that result in us keeping \\(C\\) even thought we would not have if it preceded \\(B\\)?

One measure of similarity widely used across several domains, including large-scale text processing is the [Jaccard index](https://en.wikipedia.org/wiki/Jaccard_index), also known as the Jaccard similarity coefficient.

The Jaccard index is a function on **sets**, that characterizes the similarity of two finite sets by comparing the size of their overlap with the size of their union:

$$J(A,B) = \frac{|A\cap{}B|}{|A\cup{}B|}$$

We can render this graphically as so:

> TKTK figure: Jaccard similarity

This calculation should make some intuitive sense; if two sets are similar, they should have **mostly** the same elements (and thus have a relatively large intersection), with relatively few elements unique to each set (and thus the union should not be much larger than the intersection).

It has two very natural limit points, which define its range: For two disjoint sets, the numerator \\(|A\cap{}B|\\) is zero, and the index goes to zero. But if the sets are identical, \\(A\cap{}B = A\cup{}B = A = B\\), and the Jaccard similarity is 1.

If we want to deduplicate document using the Jaccard index, we'll have to tackle two problems, which I'll take in turn: First, we have to turn our documents into sets, so that we can apply Jaccard, and then we'll have to figure out how to make it efficient.

# Turning documents into sets

In order to apply the Jaccard index to textual documents, we need to convert them into sets of some sort. There are a number of ways to do this, but I'll touch on two fairly standard ones.

Before applying either of these approaches, we may also want to apply one or more forms of text normalization. For instance, we likely want to convert to a standard [Unicode normalization form][unicode-norm], and we may also wish to case-fold, collapse runs of whitespace, or perform similar translations.

[unicode-norm]: https://www.unicode.org/reports/tr15/

## n-grams or "shingles"

One approach is to just represent each document as a set of all the n-grams that appear in it, for some value of `n`. In the text mining field, these are often referred to as "shingles". We can pick any value of `n`, with the primary tradeoff that smaller values will tend to compare documents more coarsely, and larger ones generating more distinct shingles and thus larger sets that are more expensive to process. According to one source[^fn-n], values of n between 5 and 9 appear to be common, depending on the application.

[^fn-n]: [Mining of Massive Datasets][book] §3.2.2

[book]: http://infolab.stanford.edu/~ullman/mmds/booka.pdf

## Tokenization

We can instead tokenize the input text using some sort of tokenizer, and use the set of unique tokens in each document. This is the approach taken in the GPT-3 paper, which mentions "Spark’s standard tokenizer," which seems to be [this class][spark-token], which simply lowercases the input and then splits on whitespace.

We could also use a hybrid approach, applying a tokenizer that produces numeric token IDs, and then looking at all n-grams of token IDs. In this case we would likely use a smaller value of `n`, since individual tokens should be higher-entropy than bytes or characters.

[spark-token]: https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.ml.feature.Tokenizer.html

# Performance

We now have a **definition** of "approximate similarity" -- the Jaccard index computed over the feature-set of each document -- that we wish to apply.

For very small corpora, we could potentially apply that definition directly. However, the definition we've given depends on computing \\(J(D_0, D_1)\\) for **every** pair of documents in our corpus, which entails \\(O(n^2)\\) Jaccard computations. Morever, each similarity computation isn't even all that cheap -- it involves computing both the union and intersection of the feature sets, which will tend to be **around** the same size as the input documents themselves.

If we have \\(N\\) documents, each of average length \\(D\\), the performance of the naive approach is something like \\(O(N^2D)\\). That's not feasible; we generally need something that scales **most** with \\(N\log{}N\\) to be acceptable at massive scale, and \\(O(N)\\) would be even better.

It turns out, if we're willing to accept a few more approximations, we can achieve that! Let's dig in.

## Approximating the Jaccard similarity

We'll turn first to the problem of approximating the Jaccard similarity of two sets. Recall that we're computing the ratios of two sizes: the intersection and the union of our two input sets.

> TKTK figure: Jaccard similarity

The naive approach here requires fully computing the union and the intersection, which in turn requires looking at every member of each set.

One common trick to approximate such a ratio is **sampling**. If we have a way to select points uniformly at random from the denominator, and test whether or not each point is present in the numerator of our ratio, we can approximate the ratio in that way.

> TKTK figure: sampling




## Notes
- Discussion of efficiency?
- Jaccard similarity
- Converting documents to sets
    - “shingles” aka n-grams
    - “words”
- Approximating Jaccard similarity
    - Sampling approximation
    - Using a random ordering for efficiency
    - Using several random orderings
- Scaling up to the corpus
    - Just use the min-hash signature
    - Splitting
