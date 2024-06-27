---
title: "Finding near-deduplicates with Jaccard similarity and MinHashing"
slug: fuzzy-dedup
date: 2024-05-26T12:07:28-07:00
math: true
extra_css:
 - quantiles.css
---
Suppose we have a large collection of documents, and we wish you identify which documents are **approximately** identical to each other. How would we go about that operation? We’re particularly interested in the case where the collection is very large — perhaps a terabyte or more — and so efficiency is paramount.

In this post I want to explore the method of approximate deduplication via Jaccard similarity and the MinHash approximation trick. This is a commonly-used approach to this problem (e.g. the [GPT-3 paper](https://arxiv.org/pdf/2005.14165) describes using it as part of their dataset preparation pipeline), but one I found a bit hard to understand from available resources.

![To further improve model quality and prevent overfitting (which becomes increasingly important as model capacity increases), we fuzzily deduplicated documents (i.e. removed documents with high overlap with other documents) within each dataset using Spark's MinHashLSH implementation with 10 hashes, using the same features as were used for classification above. We also fuzzily removed WebText from Common Crawl. Overall this decreased dataset size by an average of 10%](gpt-3-dedup.png "Excerpt from the GPT-3 paper describing fuzzy deduplication")

# Jaccard similarity

A natural approach to approximate deduplication is to define some notion of “similarity” between any pair of documents, and group together any two documents whose similarity value is above some threshold. So if we have some universe of possible documents \\(U\\), we might want a similarity measure $$S: U \times U \rightarrow [0,1]$$ and we’ll consider two documents “approximately the same” if \\(S(A,B) \geq S_\textrm{crit}\\).

As a slight aside, it's worth noticing that this definition is not -- in general -- transitive -- we may have three documents \\(A, B, C\\) such that \\(S(A,B)\geq{}S_\textrm{crit}\\) and \\(S(B,C) \geq{} S_\textrm{crit}\\) but \\(S(A,B) < S_\textrm{crit}\\). That means that "approximately identical" is not an [equivalence relation][equivalence], and is part of the reason that approximate deduplication is trickier to scale than finding exact matches.

[equivalence]: https://en.wikipedia.org/wiki/Equivalence_relation

One measure of similarity widely used across several domains, including large-scale text processing is the [Jaccard index](https://en.wikipedia.org/wiki/Jaccard_index), also known as the Jaccard similarity coefficient.

The Jaccard index is a function on **sets**, that characterizes the similarity of two finite sets by comparing the size of their overlap with the size of their union:

$$J(A,B) = \frac{|A\cap{}B|}{|A\cup{}B|}$$

We can render this graphically as so:

> TKTK figure: Jaccard similarity

This calculation should make some intuitive sense; if two sets are similar, they should have **mostly** the same elements (and thus have a relatively large intersection), with relatively few elements unique to each set (and thus the union should not be much larger than the intersection).

It has two very natural limit points, which define its range: For two disjoint sets, the numerator \\(|A\cap{}B|\\) is zero, and the index goes to zero. But if the sets are identical, \\(A\cap{}B = A\cup{}B = A = B\\), and the Jaccard similarity is 1.

If we want to deduplicate document using the Jaccard index, we'll have to tackle two problems, which I'll take in turn: First, we have to turn our documents into sets, so that we can apply Jaccard, and then we'll have to figure out how to make it efficient.

# Representing documents into sets

In order to apply the Jaccard index to textual documents, we need to represent them as sets of some sort. Commonly we'll refer to the elements of such sets, in the abstract, as "feartures," and we'll talk about transforming documents into "feature sets."

There are a number of ways to do this; I'll touch on two fairly common ones.

Before applying either of these approaches, we may wish to apply one or more forms of text normalization. For instance, we likely want to convert to a standard [Unicode normalization form][unicode-norm], and we may also wish to case-fold, collapse runs of whitespace, or perform similar transformations.

[unicode-norm]: https://www.unicode.org/reports/tr15/

## n-grams aka "shingles"

One approach is to just represent each document as a set of all the n-grams that appear in it, for some value of `n`. In the text mining field, these are often referred to as "shingles". We can pick any value of `n`, with the primary tradeoff that smaller values will tend to compare documents more coarsely, and larger ones generating more distinct shingles and thus larger sets that are more expensive to process. According to one source[^fn-n], values of n between 5 and 9 appear to be common, depending on the application.

[^fn-n]: [Mining of Massive Datasets][book] §3.2.2

[book]: http://infolab.stanford.edu/~ullman/mmds/booka.pdf

## Tokenization

We can instead tokenize the input text using some sort of tokenizer, and use the set of unique tokens in each document. This is the approach taken in the GPT-3 paper, which mentions "Spark’s standard tokenizer," which seems to be [this class][spark-token], which simply lowercases the input and then splits on whitespace.

We could also use a hybrid approach, applying a tokenizer that produces numeric token IDs, and then looking at all n-grams of token IDs. In this case we would likely use a smaller value of `n`, since individual tokens should be higher-entropy than bytes or characters.

[spark-token]: https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.ml.feature.Tokenizer.html

# Scaling Jaccard similarity

We now have a **definition** of "approximate similarity": We convert documents into sets of features, and wish to find documents with high Jaccard similarity between their feature sets.

For very small corpora, we could potentially apply that definition directly. However, the definition we've given depends on computing \\(J(D_0, D_1)\\) for **every** pair of documents in our corpus, which entails \\(O(n^2)\\) Jaccard computations. We need to do better than that: if we had a million documents -- still a fairly small number, compared to datasets like "the public web" -- that would be half a **trillion** comparisons.

We can compare this to finding exact duplicates, where we will hash every document, and only compare documents with matching hashes. As long as the rate of hash collision is low enough, we compare each document to \\(O(1)\\) other documents, and only require \\(O(n)\\) work in total.

It turns out, if we're willing to accept a few approximations, we can similarly scale Jaccard similarity! Let's dig in.

## Approximating Jaccard similarity

First, a brief preview of the ground we'll cover:

We'll first consider just the problem of approximating the Jaccard similarity between two documents. We'll find an approximation that avoid examining the entire feature set, and instead only considers a fixed-size (and relatively small) "signature," which suffices to approximate the Jaccard similarity. We'll then scale up to comparing many documents, where we'll exploit the structure of that signature to group candidate documents together, and avoid doing the full all-pairs computation.

### MinHash signatures

Recall that the Jaccard similarity is the ratio of two sizes: the intersection and the union of our two input sets.

$$J(A,B) = \frac{|A\cap{}B|}{|A\cup{}B|}$$

When considering a ratio of areas like this, one classic strategy is to **sample**. If we can select elements from an appropriate distribution, we can query whether those samples are present in the top and bottom halves of the ratio, and use our empirical ratio as an estimate of the true ratio.

How do we sample from \\(A\cup{}B\\)? Here's where the tricks start. We'll take a few observations, and combined them in a way that lets us do nearly all of the work ahead-of-time, as a pre-processing pass over **individual** feature sets.

- First, we'll make the problem apparently more complicated: Let's pick (at random) a mapping assigning each possible feature to some randomly-chosen value. Call this mapping \\(P(x)\\). Now we can sample a random element by selecting the feature in our set that has the smallest random value[^sql]:

$$ x_{\textrm{random}} \leftarrow{} \argmin_{x\in{}A\cup{}B}{P(x)} $$

- Randomly assigning a value to every possible feature is infeasible, but we can approximate it for most purposes using a good hash function (c.f. the "[random oracle][oracle]" model used in cryptographic analysis). If we tolerate the (very small) risk of hash collisions, we can also **only** keep the hash value (and not the relevant feature), which will leave us with a fixed-size value:

$$ x_{\textrm{sig}} \leftarrow{} \min_{x\in{}A\cup{}B}{H(x)} $$

[oracle]: https://en.wikipedia.org/wiki/Random_oracle

- Next, we'll exploit the fact that `min` is associative and so we can rewrite this to pre-process each set individually:

{{<plaintext>}}
\begin{align*}
a_{\textrm{min}} &\leftarrow{} \min_{x\in{}A}H(x) \\
b_{\textrm{min}} &\leftarrow{} \min_{x\in{}B}H(x) \\
x_{\textrm{sig}} &\leftarrow{} \min(a_\textrm{min},b_\textrm{min})
\end{align*}
{{</plaintext>}}

[^sql]: If you've ever written `SELECT ... FROM table ORDER by random() LIMIT 1`, you've used a similar trick!

What has all this manipulation achieved? We can pick an appropriate hash function and pre-process **each set** individually, summarizing it with a single hash value. We can then pick any two sets, take the `min` of their signatures, and we have (the hash of) a uniformly-randomly-selected member of their union. In order to estimate the Jaccard similarity, we want to know if that element is present in **both** sets (in their intersection), or just one.

But this construction makes that trivial! We know that \\(x_\textrm{sig}\\) hash the minimum hash function of any element in either set. Therefore, if it is present in, say, set \\(A\\), it must also be the minimum element in set \\(A\\). And we know the minimum elements of each set -- that's precisely what we stored.

Thus, we don't actually need to consider \\(x_\textrm{sig}\\) -- we just need to ask whether \\(a_\textrm{min} = b_\textrm{min}\\). The probability that those elements match is precisely \\(J(A, B)\\)!

Of course, that probability is taken over the universe of possible assignments of features to integers (aka, with some caveats, "over our choice of hash function"); so far, we've only selected a single representative from each set, and our estimate can only take on the values 0 or 1 -- not a lot of precision.

However, we can improve on that by selecting \\(k\\) different hash functions from some appropriate hash family, and summarizing each feature set into a \\(k\\)-element vector:

{{<plaintext>}}
$$
A_\textrm{sig} =
\begin{pmatrix}
\displaystyle\min_{x\in{}A}H_1(x) &
\displaystyle\min_{x\in{}A}H_2(x) &
\cdots{} &
\displaystyle\min_{x\in{}A}H_k(x)
\end{pmatrix}
$$
{{</plaintext>}}

Given two of these signatures, we can approximate the Jaccard similarity by counting how many hashes match:

$$
J(A,B) \approx{} \frac{1}{k}\sum_{i=1}^{k} (A_\textrm{sig}[i] = B_\textrm{sig}[i])
$$

One caveat to mention: the choice of the hash family function here is a bit subtle. We are attempting to approximate a random permutation over the universe of features, but the number of such permutations grows extremely quickly, and so our hash family will represent a tiny fraction of all **possible** permutations. We need to be sure that members of our hash family are not inappropriately correlated -- formally, the salient property here is referred to as ["min-wise independence"][min-wise]. Fortunately, this problem is reasonably well-studied, and we can select solutions from the literature.

[min-wise]: https://en.wikipedia.org/wiki/MinHash#Practical_min-wise_independent_hash_functions

## Comparing all documents

We've now condensed each document into a \\(k\\)-element fingerprint of hash values, which allows efficient approximation of the similarity between any two documents.

The next problem is to find approximate duplicates throughout our entire corpus -- documents with a high similarity -- **without** considering every pair of documents. Our basic technique will be to group documents according to some combination of MinHash values, and then perform the full MinHash comparison over all pairs of documents within each group.

### Using the full signature

The simplest version is to simply to use all \\(k\\) MinHash values together as a grouping key, and consider two documents "approximate duplicates" iff all of their MinHash values match. When the GPT-3 paper says they "fuzzily deduplicated documents [...] using Spark's MinHashLSH implementation with 10 hashes," I'm pretty sure this is what they mean: They split each document into features, computed 10 MinHash values for each document (using 10 different hashes[^MinHashLSH]), and grouped documents using that 10-vector as a grouping key.

The strongest virtue of this approach is its simplicity and efficiency. "Group by some high-cardinality key" is an efficient operation and easy to scale horizontally, and is offered as a basic primitive in virtually any data-processing toolkit (it's arguably **the** core primitive in [MapReduce][reduce], in the form of the "shuffle" between the map and reduce stages).

How does this approach behave? For a single pair of documents, we expect each MinHash value to be equal with probability \\(J(A,B)\\), so we expect all 10 to match with \\(p=J(A,B)^k\\). For \\(k=10\\), here's what that looks like:

{{<img src="probability-10-hashes.png"
       alt="Probability all 10 MinHashes match, as a function of similarity">}}

And here are some quantiles, which I find helpful for getting a "feel" of the distribution:

| p(all match) |   1% |  10% |  25% |  50% |  75% |  90% |
|:-------------|-----:|-----:|-----:|-----:|-----:|-----:|
| Jaccard      | 0.63 | 0.79 | 0.87 | 0.93 | 0.97 | 0.99 |
{.quantiles}

We can see that similarities below 0.6 or so will almost-never collide, and that the odds of matches become good around 0.9 or so. If we're primarily concerned about documents that are very close siblings, this approach may be perfectly sufficient. Furthermore, I suspect -- but haven't verified -- that in many corpora, we will mostly encounter relatively bimodal Jaccard values. Unrelated documents have similarity close to 0, and we'll have many near-identical documents: the same document fetched at different times, with minor edits or metadata differences.

It's also worth noting that the \\(J^{k}\\) calculation holds for a **single** pair of documents. If we have many documents that are all similar, the pairwise probabilities are not at all independent. In practice, they're likely to end up hashed into two or three buckets, and so we will detect almost all of the duplicates.

[mapreduce]: https://en.wikipedia.org/wiki/MapReduce


[^MinHashLSH]: If you're curious, Spark [documents its choice of hash family][MinHashLSHModel], along with the relevant literature reference.

[MinHashLSHModel]: https://spark.apache.org/docs/3.1.1/api/python/reference/api/pyspark.ml.feature.MinHashLSHModel.html#pyspark.ml.feature.MinHashLSHModel

### Going fuzzier

(Note: this discussion is primarily sourced from ["Mining of Massive Datasets"][book] section 3.4. I have been unable to easily determine how often and in which contexts this approach ends up getting used in practice.)

What if we want to detect "fuzzier" duplicates -- for whatever reason (presumably determined by empirical study), we want to find pairs with a similarity threshold of 0.8 or 0.7 or some other value?

Our first technique will be to use a subset of our \\(k\\) MinHash hashes as a grouping key, and then, within each group, to compare the full signature to evaluate the estimated similarity.

By only using \\(r\\) hashes in our grouping key, can we increase the likelihood that less-similar documents will hash together, but only so far; \\(J^r\\) will always be smaller than \\(J\\), and using too-few will increase the rate of false matches where dissimilar documents end up in the same bucket.

What we can do instead is to do **multiple passes** of grouping documents, using a **different** subset of our \\(k\\) hashes for each time. If we compute \\(k=20\\) hashes as our signature, we might divide them into \\(g=4\\) groups of \\(r=5\\) hashes each, and then perform 4 different grouping operations, one by each group of hashes.

What are the odds that two documents end up hashed together in **at least one** of these rounds?

- The odds that two documents collide in any given group is \\(J^r\\)
- So the odds they **don't** colide in any given group is \\(1-J^r\\)
- So the odds that don't collide in **any** of the groups is \\(1-J^r)^g\\)

Thus, the odds that they collide at-least-once ends up as:

\\[ p = 1 - (1-J^r)^g \\]

For example, the above hypothetical of 4 groups of 5 hashes above ends up with a probability curve like so:

{{<img src="probability-grouped-hashes.png"
       alt="Probability two documents collide in *any* bucket, for 4 groups of 5 hashes">}}

The curve is not as steep as our earlier example, but we've successfully shifted it to the left -- the odds of colliding become 50% at somewhere around \\(J=0.7\\).

It turns out, that for any choice of \\(r\\) and \\(g\\) greater than 1, this will produce a somewhat S-shaped curve of varying steepness and midpoint, allowing us a wide space within which to trade off sensitivity, spurious collisions, and the costs of producing signatures and grouping documents.

# Closing thoughts

I think MinHash signatures are a pretty neat trick! I had not encountered them prior to coming across them in the GPT-3 paper, and found existing writeups moderately confusing; I hope this one helps someone out there grasp the idea or exposes some folks to this neat trick!

# Postscript: MinHash and HyperLogLog

While researching and writing this post, I realized that the core MinHash trick reminds me a bit of my other favorite [sketch][sketch]: [HyperLogLog][hll].

The key idea in HyperLogLog (going back to [a much older algorithm][martin-flajolet]) is to hash each element of a stream, count the number of leading zeros in each hash value, and we store a **running maximum** of "number of leading zeros."

The algorithm is very different in the details, but I think there's a clear conceptual similarity: In both cases, we use a hash function to map an arbitrary unknown distribution into a uniform distribution (in some appropriate sense), and then we compute a running extremum of some function on that distributon, which -- with appropriate calculation -- allows us to estimate some distributional property using only O(1) space.

And on some search, it turns out that the connection in some sense goes deeper; HyperLogLog and MinHash are (sort of) duals, in a sense: HyperLogLog is a cardinality estimator, which means it can estimate the size of the **union** of two sets; Meanwhile, MinHash estimates the (relative) size of the **intersection** of two sets. If you combine those two, you can produce a sketch that lets you ask questions about both intersections and unions of arbitrary sets!

This idea was noticed [at least by 2013][hllminhash], and there's also an [ongoing][hyperminhash] [literature][setsketch] of sketches that combine ideas from the two data structures in interesting ways. I think that's neat!


[martin-flajolet]: https://en.wikipedia.org/wiki/Flajolet%E2%80%93Martin_algorithm
[sketch]: https://www.cs.cornell.edu/content/sketching-algorithms
[hll]: https://en.wikipedia.org/wiki/HyperLogLog
[setsketch]: https://arxiv.org/abs/2101.00314
[hllminhash]: https://tech.nextroll.com/media/hllminhash.pdf
[hyperminhash]: https://arxiv.org/abs/1710.08436
