---
title: "Explaining the Ackermann function"
slug: coq-ackermann
date: 2024-07-24T09:59:08-07:00
math: true
description: |
 TK TK describe me daddy
---

I've been learning [Coq][coq] recently, out of curiosity. At its core, Coq is built around a functional programming language that is recognizably similar to ML-family languages like OCaml and Haskell.

TKTK better intro

## Recursion and termination

[coq]: https://coq.inria.fr/

Unlike most ML languages, though, Coq requires (for soundness reasons) that every defined function is **total**: they must terminate[^theory] with a result for every input. In particular, infinite loops of any sort must be disallowed. To implement that restriction, Coq requires that a recursive function have a "decreasing argument" that **syntactically** shrinks on each recursion[^proof].

[^theory]: Of course, by this we mean "terminate in the underlying mathematical model"; we can still write functions that crash due to out-of-memory errors, or kill programs with `^C` and thus never return in that sense.

[^proof]: Coq does actually let you implement more complex recursive schemes, as long as you can provide an appropriate proof of their termination. This detail isn't relevant to this post.

What does it mean to "syntactically" shrink? It means that we're only allowed to make a recursive call if we:
- Perform a pattern-match on the "decreasing argument"
- Make the recursive call passing a *sub-object* of the argument, which we extracted during the pattern match.

I'll give an example by way of a brief detour into datatypes and language:

For this post, I'm going to restrict myself to talking about functions operating over the [Peano encoding][peano]. In addition to being Coq's default number representation, I think the Peanos give us a desirable balance between "as simple as possible" while admitting deep complexity. They feel to me like the "simplest infinite datatype" in some sense (question to readers: Is there an appropriate way to formalize that statement? Is my intuition correct in any such formalism?).

While this post is inspired by learning Coq, I'm going to use Haskell for my example code; I expect more readers to have (some) familiarity with it, and I find its syntax more readable here. We can define the Peano integers ourselves like so:

```haskell
data Nat = O | S Nat

zero :: Nat = O
one :: Nat = S O
```

Over the peano integers, Coq's "decreasing argument" restriction essentially means we must pattern-match some argument agains `S p`, and then recurse by passing `p` to the recursive case. For example, here is how we might multiply a number by two:

<a name="double"></a>
```haskell
double :: Nat -> Nat
double O     = O
double (S p) = S (S (double p))
```

We're allowed to decrease **faster** (e.g. we may match `S (S p)` and recurse on `p`), but we cannot recurse on any other expression that did not directly result from a pattern match of `(S ...)`.

## How long can you go?

If you have the right sort of curious and/or slightly-devious mind, when you encounter this "decreasing argument" limit, it raises a natural question: Just **how** many times can such a function recurse -- as a function of its input -- before it does ultimately terminate?

This questions turns out to be a great way to understand the infamous [Ackermann function][ackermann]! In particular, I claim that the Ackermann function, in its standard two-argument form, is essentially a natural answer to that challenge!

Let's dig in.

[ackermann]: https://en.wikipedia.org/wiki/Ackermann_function

## Framing the question

The question of "what's the longest-recursing function" is, of course, not well-defined; it will turn out, for instance, that given some function `g` that recurses \\(N_g(n)\\) times, we can always construct a `g'` that recurses \\(N_g(n)+1\\) times, or \\(2*N_g(n)\\), and so on.

However -- in the same spirit as the [big number duel][duel] -- we can choose to exclude "cheap tricks" and focus on the key **ideas** that let us recurse ever-more times, ignoring approachs or complications that don't add fundamental power to our approach.

I've already mentioned that we're operating over the Peano integers. In a similar spirit of minimalism and of getting at the essential question, we're going to focus on a **single** recursive function definition, which uses no other definitions or functions than the Peano datatype we defined above. Notably, this means we won't allow ourselves to define or use (e.g.) an addition primitive, and more-or-less our only "primitive" operations will be addition by 1 (`S`) and subtraction by 1 (pattern-match on `S p`).

## Warmup: the unary case

We'll start by considering the unary case: a function of a single (natural number) argument.

For this case, we're pretty limited in what we can do. Given `f x`, we have two cases:

- If `x` is zero (`O`), there are no smaller numbers, so we cannot recurse.
- If `x` is nonzero, we detect this by pattern matching on `(S p)` for some `p`, and we're allowed to recurse with `f p`, essentially subtracting one.

In addition, we can invoke the `S` constructor, but we cannot do so **inside** the recursive argument.

We can pattern-match and strip more `S` steps, but doing so will only make our argument shrink faster, and thus reduce our number of recursive calls.

Thus, the [`double`](#double) function from earlier seems to be about the most we can do, where we recurse a number of times equal to our input.

# The n-ary case: lexicographic order

What about functions of more than one argument?

First, we need to consider what our "strictly decreasing" rule allows. Skipping over some details, it turns out that the natural extension of the rule to multiple arguments is to look at the [lexicographic order][lexicographic][^why-lex] on the argument list.

Concretely, if we're defining a two argument function:

```haskell
f m n = ...
```

Then the allowed recursive calls look like:

- We pattern match `m` to `(S m')`, and recurse on `f m' <anything>`, OR
- We leave `m` unchanged, pattern-match `n` to  `(S n')`, and recurse with `f m n'`.

[lexicographic]: https://en.wikipedia.org/wiki/Lexicographic_order

## Intentionally quadratic

That turns out to allow a **lot** more interesting behavior! In particular, notice that as long as we're shrinking our first argument, we can can do anything we want with the second -- in particular, we are allowed to increase it arbitrarily!

We can use that freedom to easily get more-than-linear recursive calls; here's a function that recurses quadratically many times (in its first argument):

```haskell
loop m n =
  case (m, n) of
    (O   , O   ) -> O
    (S m', O   ) -> loop m' m -- `n` increases from 0 to `m`!
    (S _ , S n') -> loop m n'
```

In the second case, the one I've labeled, we decrease `m` to `m'`, which means we're allowed to increase the second argument. Here, we just increase to the most convenient number we have lying around (`m`), which results in a triangular pattern of calls which totalling O(m^2).

Note also that in the case where we **don't** decrease `m` -- as in the last line -- our behavior is similar to the one-argument case, and essentially our only interesting behavior is to decrease `n` by one step each time.

## Going big

This observation makes it clear that "recursing many times" and "generating very large numbers" are intimitately related problems. In order to recurse many times, we need to generate large numbers, so that we can grow `n` (our second argument) rapidly for each decrease in `m` (our first argument). However, the problems are also connected in the opposite direction: Since we're working with a Peano representation of the integers, the only way to create very large numbers is to apply the successor function many, many times; the natural way to do this is to recurse many times, applying the `S` constructor as we recurse.

This gives us a sketch of our strategy:

- In addition to recursing many times, we'll make sure to apply the `S` function in some form as we do, in order to also return large numbers.
- Then, each time we decrease `m` in a recursive call, we'll generate a large number for the new value of `n` -- using an appropriate **second** recursive call.

Here's a warmup. In our `loop` function in the last section, the key recursive case increased `n` to match `m`:

```haskell
    (S m', O   ) -> loop m' m
```

Instead of `m`, we can **nest** our recursive call, like so:

```haskell
loop_exp m n =
  case (m, n) of
    (O   ,  O) -> O
    (S m',  O) -> (loop_exp m' (loop_exp m' m)) -- nested calls here!
    (_', S n') -> S (loop_exp m n')             -- Apply `S` here
```

We also add an `S` constructor call into the final case, in order to ensure that we're counting up our eventual result.

Our previous `loop` was quadratic in `m`; I won't do the analysis to prove it, but we've now managed to recurse a number of times **exponential** in `m` -- a huge increase!



[^why-lex]: It turns out this isn't a special case or even a separate rule; the lexicographic ordering rule falls out directly from the "strictly decreasing" rule and [currying][currying]. Recall that we can treat two-argument functions as nested single-argument functions, where we pass one argument, and receive back a function which accepts our second argument. e.g this two-argument function:

    ```haskell
    f x y = BODY
    ```

    is just sugar for the nested:

    ```haskell
    f x =
      let inner y = BODY
      in inner
    ```

    That, in turn, lets us analyze a two-argument function only in terms of two single-argument definitions! If we want to recurse, we can choose whether we call `f`, or whether we call `inner`:

    - If we call `f`, we're subject to its decreasing requirement, and we have to decrease `x`. But `f` doesn't know about `y`, so we can do whatever we want with it.
    - On the other hand, if we call `inner`, it inherits the same captured value of `x`, and so we can't change that. But we're also subject to `inner`'s decreasing requirement, and so we must shrink `y`.

    Which gives us precisely the "lexicographic ordering" behavior!

    In fact, Coq requires functions have **one** decreasing argument, and thus does not allow you to define the Ackermann function in standard two-argument form; but it accepts it just fine if you rewrite it to explicitly curry in this style.


[currying]: https://en.wikipedia.org/wiki/Currying
[peano]: https://en.wikipedia.org/wiki/Peano_axioms
[duel]: https://web.mit.edu/arayo/www/bignums.html


<!--

OLD DEAD DRAFT

{-

What's the "simplest function" that:
- Grows unreasonably quickly, but
- Can be "trivially" shown to terminate

?

I want to unpack those goals, and show that the answer is interesting.

# "Simple"

"simplicity" is always a tricky notion, and I will accept I may make
some arbitary-seeming choices here. But I'll adopt some rules that I
think feel like a reasonable definition of "as small as possible but
still interesting."

Namely:

- Pure functional setting

  We're going to use a purely-functional paradigm (I'll use Haskell
  because I'm familiar with it, but everything here is trivially
  portable to any ML-inspired language, including Rust).

  Avoiding mutation and using a strictly functional construction makes
  analysis much simpler.

- We're going to operate over Peano numbers.

  The Peano numnbers are a classic encoding of the natural numbers,
  still fairly widely used, and are pretty much perfect when you want
  a structure that is as barebones as possible, but which admits a
  *ton* of interesting emergent complexity. I think they're probably
  "the simplest infinite inductive data type" in a meaningful sense.

  We'll bring our own definition, for the sake of being self-
  contained, as well as some distinguished constants:
-}

data Nat = O | S Nat

zero :: Nat = O
one :: Nat = S O

{-

- Only one function

  We will define one (recursive) function, and one function only.

  We won't appeal to any helper functions or define any helpers
  inline. This also feels "clearly simpler" than defining multiple
  functions.

  I *will* permit it to be n-ary, with more than one argument, because
  this turns out to be necessary for nontrivial behavior, given my
  other rules.

  Coupled with operating over the Peanos, this means we can't just
  invoke addition or exponentiation or anything else; our main
  operations will be successor (by invoking `S`) and predecessor, by
  pattern-matching against (S m).

# Termination

What's the simplest way to ensure a recursive function terminates?

Many of us are familiar with the struggle of messing up a base case
and seeing a recursive definition call itself forever (or until the
stack runs out), so I think this isn't just a theoretical question.

We could of course ban recursive calls, but that bans most interesting
behavior in a functional context. I think the next-requirement is
probably to ensure that each recursive call constructs an argument
list that is *strictly smaller than* the argument list to the current
function. Moreover, it should be strictly smaller in a syntactically-
obvious way.

By "smaller," here, I'm going to refer to a lexicographical order of
the function arguments. Since we're operating over the Peano integers,
this ordering has no infinite decreasing sequences, and so, as long as
we decrease at every step, we're guaranteed to eventually bottom out
at (0, 0, 0, ...), even if it may take a while.

(This whole post, in fact, is inspired by [Coq's requirement][coq] for
fixpoint functions of a decreasing argument; I was learning Coq on a
lark this past week.)

[coq]: https://coq.inria.fr/doc/V8.4pl6/refman/Reference-Manual003.html#Fixpoint

By way of example, I allow this function:

-}

legal O O      = one
legal O (S n)  = legal O n      -- Okay: (O, n) < (O, S n)
legal (S m) n  = S (legal m n)  -- Okay: (m, n) < (S m, n)

{-

But not this one (even though it does also terminate in all cases):

-}

bad O n          = n
bad (S m) O      = (bad O (S m))      -- Okay: O < (S m), decreasing in first argument
bad (S m) (S n)  = (bad (S (S m)) n)  -- BAD: (S m) -> (S (S m)) is increasing in first argument!

{-

# Warmup: single-argument function

Let's define a single-argument function subject to our rules so
far. We'll have two cases, one for O, and one for (S n):

f :: Nat -> Nat
f O     = ...
f (S n) = ...

The O case isn't allowed to recurse, so it's limited it what it can do
and how much effect it can have. We'll do the simplest behavior that
isn't the identity, and just return one (aka (S O)):

-}

f O = one

{-

What about the `f (S n)` case? Remember, our goal -- modulo our
"simplicity" goals -- is to grow as quickly as possible. That
suggests:

- (a) We should recurse any time we can, and
- (b) We should recurse on the *largest* allowed argument

Since those rules will, together, make us run "as long as possible"
subject to our restrictions.

Thus, we should definitely generate a recursive call to `f n` --
that's the largest case smaller than our `(S n)` argument.

What should we do once we've recursed?

We don't really *have* much we can do with a natural number, absent
any other functions or helpers. We can't even do something like `(f n)
+ (f n)`, since we don't have a `+`.

Once again, we'll do the simplest nontrivial thing, and just apply `S`
to the recursive output:

-}

f (S n) = S (f n)

{-

With a little bit of work, we can easily show that this function is
not very interesting; our `f` ends up ultimately just applying `S` to
its input. We could just have written

f = S

and saved our keystrokes.

On the other hand, it does suggest that our "simplicity" heuristics
have real bite: they've ruled out almost all intereting behavior.

## Going further

So, what happens if we're allowed to have **two** arguments? We now
have four cases:

g :: Nat -> Nat -> Nat
g O O         = ...
g O     (S n) = ...
g (S m) O     = ...
g (S m) (S n) = ...

For the first two cases, we can't decrease the first argument any
further, and so we only have the second to play with. Thus, we
basically reduce to the single-arugment `f` case, and we can write:

-}

g O n = S n

{-

2/4 cases down!

What about `g (S m) O`? We can't decrease the second argument, so we
need to decrease the first, so we'll want to recurse on `g m [something]`.

Since we're decreasing in the second argument, we're allowed to put
any value at all into the second argument.

-}





main = do putStrLn "hello"

-->
