---
title: "TKTK The Ackerman function"
slug: ackerman
date: 2024-07-24T09:59:08-07:00
math: true
---

I've been spending some time learning [Coq][coq] recently. At its core, Coq is built around a functional programming language that is recognizably similar to ML-family languages like OCaml and Haskell.

[coq]: https://coq.inria.fr/

Unlike most ML languages, though, Coq requires (for soundness reasons) that every defined function is **total**: they must terminate (at least in theory) with a result for every input. In particular, infinite loops of any sort must be disallowed. To implement that restriction, Coq requires that a recursive function recurse only on an argument that is **syntactically** smaller than the original argument.

"Syntactically smaller" essentially means that we are allowed to pattern-match on a recursive data type and recurse on the inner value, and nothing else. For instance, given a classic Lisp-style `cons`-cell list, we can pattern match on `head :: tail` and recurse on the `tail`. We also can't call some other function and recurse on its output, even if it will always-in-practice be smaller[^proof].

If you have the right sort of curious and/or slightly-devious mind, this restriction raises an obvious question: Just how many times **can** we manage to recurse (as a function of our input), given this limitation?

[^proof]: Coq does actually let you implement more complex recursive schemes, as long as you can provide an appropriate proof of their termination. I'm going to ignore that feature here.

## Framing the question

This question is not really well-defined; it will turn out, for instance, that given a function `g` that recurses \\(N_g(n)\\) times, we can always define one that results in \\(N_g(n)+1\\) recursive calls, or \\(2*N_g(n)\\), or many other such expressions.

However -- in the same spirit as the [big number duel][duel] -- we can choose to exclude "cheap tricks" and focus on the key **ideas** that let us recurse ever-more times, ignoring approachs or complications that don't add fundamental power to our approach. It turns out this will lead to an interesting answer!

In that spirit, I'll adopt a few more restrictions, in the same of focusing on the fundamental idea:

First, we'll restrict ourselves to operating with the natural numbers as our only datatype -- and specifically, we'll use the [Peano encoding][peano] (not-coincidentally, this is Coq's default representation of integers). This feels like a natural "minimal and yet rich" choice to me.

I'll use Haskell for running examples; I expect more readers to have passing familiarity with it than with Coq, and I also find its syntax easier for this problem. We'll define the Peano integers ourselves like so:

```haskell
data Nat = O | S Nat

zero :: Nat = O
one :: Nat = S O
```

Second, I'll also limit our study to a **single** self-contained recursive function (possibly with multiple arguments); no helper functions or external definitions. Notably, this means our only primitive operations will be to add one (by invoking `S`), or to subtract one (by pattern-matching on `(S x)`).

## The unary case

We'll start with a warmup, and study the case of functions of a single argument.

For that case, we're pretty limited. Given `f x`, we have two cases:

- If `x` is zero (`O`), there are no smaller numbers, so we cannot recurse.
- If `x` is nonzero, we detect this by pattern matching on `(S p)` for some `p`, and we're allowed to recurse with `f p`, essentially subtracting one.

We could pattern-match and strip more `S` steps, but doing so will only make our argument shrink faster, and thus reduce our number of recursive calls.

Thus, for a function of one argument, Coq's "strictly decreasing" rule means we can only ever recurse ≤ `n` times, given an input value of `n`.

# The n-ary case: lexicographic order

What about functions of more than one argument?

First, let's examine what the "strictly decreasing" rule means in the context of multiple arguments. It turns out that the natural extension of the rule is that the argument list as a whole must be strictly decreasing on every call, according to the [lexicographic order][lexicographic].

Concretely, if we're defining a two argument function:

```haskell
f m n = ...
```

Then we are allowed to recursively call ourselves, so long as either:

- We shrink `(S m)` to `m` in our first arugment, OR
- We leave `m` unchanged, and shrink `(S n)` to `n` in our second argument

[lexicographic]: https://en.wikipedia.org/wiki/Lexicographic_order

## Intentionally quadratic

This is enough for interesting behavior! Note that, as long as we shrink the first argument, we can do anything we want with the second -- in particular, we are allowed to increase it arbitrarily. We can use that freedom to easily get more-than-linear recursive calls:

```haskell
loop m n =
  case (m, n) of
    (O   , O   ) -> O
    (S m', O   ) -> S (loop m' m) -- `n` increases from 0 to `m`!
    (S _ , S n') -> S (loop m n')
```

In the second case, the one I've labeled, we **decrease** `m`, which means we're allowed to do anything we like with `n`; in particular, we can increase it! In this case, we just increase it to match `m`, which ends up generating a triangular pattern and a quadratic (in the top-level `m`) number of recursive calls.

If we **don't** decrease `m` -- as in the last case -- then our behavior is similar to the one-argument case, and the best we can do is to decrease our second argument as slowly as possible: one step each loop.

## Going big

This suggests that our main goal is thus: every time we decrease `m` in a recursive call, we have to also make our second argument -- `n` -- as large as possible. Thus, we've reduced our problem, in some sense, into generating very large numbers as soon as possible.

How do we generate large numbers? Remember, I've sworn off helper functions,


# Appendix: Why lexicographic order?

In Coq, as in most ML-family languages, functions of multiple arguments are handled by [currying][currying] -- a function of two arguments is treated as a function of one argument, which returns a second function, which returns the result. That is, defining a two-argument function like so:

```haskell
add x y =
  case x of
    O     -> y
    (S p) -> S (add p y)
```

is essentially sugar for the nested:

```haskell
add' x =
  let f y =
        case x of
          O     -> y
          (S x) -> S (add' x y)
  in f
```




[currying]: https://en.wikipedia.org/wiki/Currying
[peano]: https://en.wikipedia.org/wiki/Peano_axioms
[duel]: https://web.mit.edu/arayo/www/bignums.html
