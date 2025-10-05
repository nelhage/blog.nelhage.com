---
title: "Solving Regex Crosswords with Z3: Multiple approaches"
slug: regex-crosswords-z3
date: 2025-08-29T15:04:58-07:00
---

I've been meaning for a while now to spend more time getting to know [z3][z3] and [SMT solving][smt-lib] more generally. While out on part 2 of my pat leave, a friend happened to remind me of the existence of [regex crossword puzzles][regexle], and I got sniped. I've now implemented several different approaches to solving regex crosswords using z3. In this post I'm going to walk through a number of approaches and share some of what I learned.

- [ ] TK TK maybe show an example crossword?

## Regexes as DFAs

Thinking about this problem, my immediate thought was to rely on the fact that a regular expression describes a [regular language][regular-lang][^regular], and thus can be recognized using a deterministic finite automaton. Thus, I can encode a regular expression into a DFA, and then express regex matching in Z3 in terms of its transition function[^z3-re].


[^regular]: I include here the obligatory disclaimer that many modern "regular expression" libraries support non-regular features like backreferences. My strategy won't work for regex crosswords that use those, but I choose to be okay with that.
[^z3-re]: Some readers may be thinking "But wait! Doesn't Z3 natively support regular expressions??" When I started this project I wasn't aware of that feature, but see below!

That is, if we have a three-character string `S = [c0, c1, c2]`, and we have the transition table `T : State × Char → State` for our regex, we can express "the regex matches S" as something like:

```python
state_0 = init_state
state_1 = T(state_0, c0)
state_2 = T(state_1, c1)
state_3 = T(state_2, c2)
assert(state_3 in accept_states)
```

[regular-lang]: https://en.wikipedia.org/wiki/Regular_language

## Turning a regular expression into a DFA

With a bit of searching, I discovered [qntm]'s [greenery library][greenery], which was pleasingly exactly what I was looking for -- a relatively straightforward Python library for manipulating regular expressions, including transforming back and forth between regex syntax and DFAs (which it calls [FSMs][greenery-fsm]). It will build an FSM for us, and in general it is pleasingly straightforward, optimized for simplicity, ease-of-use, and experimentation instead of raw performance, which is just fine for us -- we're going to make z3 do all the heavy lifting.

[qntm]: https://qntm.org/
[greenery]: https://github.com/qntm/greenery
[greenery-fsm]: https://github.com/qntm/greenery?tab=readme-ov-file#fsm

One complication is that `greenery` handles regular expressions over arbitrary Unicode characters; in order to avoid enormous lookup tables, it represents transitions in terms of character ranges, which I found slightly more annoying to work with. For this project, I'm willing to assume an alphabet of English letters (A-Z), so I wrote a helper to translate `greenery`'s structure into a flat `numpy` transition table:

```python
ALPHABET = string.ascii_uppercase

def flatten_fsm(fsm):
    nstate = len(fsm.states)
    nvocab = len(ALPHABET)

    assert fsm.states == set(range(nstate))
    state_map = np.full((nstate, nvocab), -1, dtype=np.int32)
    for st, map in fsm.map.items():
        for i, c in enumerate(ALPHABET):
            for cc, dst in map.items():
                if cc.accepts(c):
                    state_map[st, i] = dst
                    break
    return state_map
```

I also defined a small dataclass to bundle together various representations and analysis for a single regular expression:

```python
@dataclass
class Regex:
    pattern: str
    parsed: greenery.Pattern

    # 2d: (state, vocab) -> new state
    transition: np.ndarray
    # 1d: state -> bool
    accept: np.ndarray

    @classmethod
    def from_pattern(cls, pattern: str):
        parsed = greenery.parse(pattern)
        fsm = parsed.to_fsm().reduce()

        transition = flatten_fsm(fsm)
        nstate = transition.shape[0]
        accept = np.zeros((nstate,), bool)
        for st in fsm.finals:
            accept[st] = True

        return cls(
            pattern=pattern,
            parsed=parsed,
            transition=transition,
            accept=accept,
        )
```

## Mapping into Z3

In order to put the pieces together, we need to teach Z3 about the transition functions for each clue, define Z3 variables for each character in our grid, and then add appropriate constraints on the characters using the relevant transition functions.

I chose to represent both states and characters as integers (adding assertions to ensure that they were in-range), since that seemed simplest. Then, we need to encode the transition function relationship, between states and characters.

Z3 supports [function types][z3-functions], which can be declared and manipulated in expressions and assertions. Thus, I declared a function of the appropriate type, and informed Z3 of the transition table using an assertion for each entry in the transition table:

[z3-function]: https://microsoft.github.io/z3guide/docs/logic/Uninterpreted-functions-and-constants


```python
def build_func(solv: z3.Solver, clue: Clue):
    ctx = solv.ctx
    state_func = z3.Function(
        clue.name + "_trans",
        z3.IntSort(ctx),
        z3.IntSort(ctx),
        z3.IntSort(ctx),
    )
    pat = clue.pattern

    for (state, char), next_state in pat.all_transitions():
        solv.add(state_func(state, char) == next_state)

    s, c = z3.Ints("s c", ctx)
    solv.add(
        z3.ForAll(
            [s, c],
            (state_func(s, c) >= 0) & (state_func(s, c) < pat.nstate),
        )
    )
    return state_func
```

Given that definition, and given a string (represented as a list of characters), we can easily express the "regex matching" constraint:

```python
def assert_matches(solv: z3.Solver, clue: Clue, chars: list[z3.AstRef]):
    state_func = self.build_func(solv, clue)
    pat = clue.pattern

    # Create constants for each state
    states = z3.IntVector(clue.name + "_state", 1 + nchar, ctx=solv.ctx)

    # Express the transition requirement
    for i, ch in enumerate(chars):
        solv.add(state_func(states[i], ch) == states[i + 1])

    # Each state must be valid
    for st in states:
        solv.add(0 <= st)
        solv.add(st < pat.nstate)
    # State zero is the initial state
    solv.add(states[0] == 0)

    # The final state must be accepting
    solv.add(z3.Or([
        states[-1] == i
        for i, ok in enumerate(pat.accept) if ok
    ])
```

## Putting it together

With those primitives, we now just need to define a grid of characters, and add appropriate assertions. I was still using [regexle][regexle] as a source of test cases; it operates on a hexagonal grid, and mapping coordinates between the hex grid and the three different clue axes was honestly one of the hardest parts of this exercise! Glossing over that bit, though, the code is quite simple:

```python
def make_char(solv: z3.Solver, name: str) -> z3.ExprRef:
    ch = z3.Int(name, solv.ctx)
    solv.add(0 <= ch)
    solv.add(ch < len(ALPHABET))
    return ch

grid = [
    [make_char(solv, f"grid_{x}_{y}") for y in range(maxdim)]
    for x in range(maxdim)
]

for clue in all_clues:
    coords = word_coords(clue.axis, clue.index)
    chars = [grid[x][y] for x, y in coords]

    assert_matches(
        solv,
        clue,
        chars,
    )
```

# Solver performance

At this point -- once I put all the pieces together -- I was able to solve crosswords (and cross-check a few hand / with [regexle]'s web interface), but it was ... painfully slow. Some 3x3 puzzles took upwards of **ten minutes** of Z3 time on my M1 Macbook Air! Letting Z3 use multiple cores was worth a bit of a speedup, but fundamentally that seemed way too slow.

- [ ] TK performance charts

It seemed likely I'd done something wrong, but as a Z3 novice it was not immediately clear to me how to investigate, or what to change. I ended up exploring **numerous** variants, with many false starts and red herrings.

I had some sense, before this project, that SMT solvers have notoriously opaque and [unstable][smt-unstable] performance properties, but even so, this project left me quite surprised (and somewhat disheartened) by some of the surprising performance behavior I observed.

[smt-unstable]: https://ceur-ws.org/Vol-4008/SMT_paper21.pdf

For the next part of this post, I'll start by exploring which changes resulted in substantial performance improvements (as best I currently understand things!), and then I'll explore some of the stranger performance surprises I encountered.

## Pruning states

I'm not a Z3 expert, but I understand our specific problem quite well, so my first idea was to give Z3 an easier task by encoding some additional constraints which we can deduce from the structure of the regexes and their FSMs.

As a motivating example, consider the regex `(NR|Q|I)+`, from [today's regexle](https://regexle.com/?side=3&day=458) as I write this. It's immediately obvious that the answer can only contain the letters N, R, Q, and I. However, the code so far uses an alphabet of `A-Z` for all clues. What if we instead detected the actual legal alphabet, and encoded it into our Z3 problem?

As humans we easily detect that property by looking at the regex source. We could write code to do a similar analysis, but in general the problem is not quite as straightforward. I realized, however, that we can instead do a similar analysis on the transition tables we already have. Moreover, because I am now [a scientist][circuits-framework] as well an engineer, I choose to do so in a few scant lines using some `numpy` tricks.

We define a state in our FSM as "dead" if there is no sequence of transitions leading from that state to any accepting state. Because we work on a minimal FSM, courtesy of [a `greenery` analysis pass](https://github.com/qntm/greenery/blob/e55c96712677d56ef14664a1595a47fb7f26bc01/greenery/fsm.py#L166-L172), we know there is at-most one dead state, and all out-edges from that state will be self-loops. We can easily detect the dead state by looking for that condition:

[circuits-framework]: https://transformer-circuits.pub/2021/framework/index.html

```python
# Method on the above class `Regex`
@cached_property
def dead_states(self) -> set[int]:
    looped = self.transition == np.arange(self.nstate)[:, None]
    return set(np.flatnonzero(looped.all(-1) & ~self.accept))
```

(Note that we will only ever have zero or one dead states, but using `set[int]` will let us treat those cases uniformly by iterating over the set, which I found a bit cleaner than using `int | None`)

Given the dead state, we can define a dead **character** as one which always transitions to a dead state, from any initial state:

```python
@cached_property
def dead_vocab(self) -> set[int]:
    dead = set()
    for d in self.dead_states:
        dead |= set(np.flatnonzero((self.transition == d).all(0)))
    return dead
```

We can also ask which characters are dead starting from any given state:

```python
def dead_from(self, state: int) -> set[int]:
    dead = set()
    for d in self.dead_states:
        dead |= set(np.flatnonzero((self.transition == d)[state]))
    return dead
```

Armed with that information, we can give Z3 a few more constraints:

```python
for d in pat.dead_vocab:
    for ch in chars:
        solv.add(ch != d)

for d in pat.dead_from(0):
    solv.add(chars[0] != d)

for d in pat.dead_states:
    for st in states:
        solv.add(st != d)
```

These constraints are, strictly-speaking redundant, but our hope is that our knowledge of the problem structure lets us find these facts more easily than Z3 can, and let Z3 focus faster on the "interesting" part of the search space.

Indeed, I found this pruning exercise an interesting example of how you can use a generic prover/solver like Z3, but nonetheless augment it with domain-specific heuristics to improve performance. My sense is that this hybrid is fairly common in practice; solvers aren't magical and if you can deduce additional structure using domain-specific analysis, it will often give the solver an important boost.

If I wanted, it would be relatively straightforward to additionally ask Z3 to **verify**, on some specific puzzles, that these constraints are redundant, which would be a useful correctness check. For this toy project I didn't bother.

## Defining the transition function explicitly

I described, above, encoding the transition table for each regex as a Z3 function, and then nailing down its behavior using pointwise assertions about each `(state, character)` pair. I eventually discovered that I could get better performance by representing the transition as an explicit expression, instead of an uninterpreted function; I speculate that doing so encouraged Z3 to reason through the relation equationally instead of falling back on search, but I'm honestly unclear.

We'll write a Python function which represents the transition table as an explicit `if-then` ladder analyzing a (state, character) input. We'll start with a helper which effectively does a "match" or "switch" statement on a single variable:

```python
def build_match(
    var: z3.AstRef,
    test: list[z3.AstRef],
    result: list[z3.AstRef],
) -> z3.AstRef:
    """Return a Z3 `if` ladder comparing var against each `test` value.

    If `var == test[i]`, the ladder evaluates to `result[i]`. If no
    `test` matches, evaluate to `result[-1]`; it is anticipated that
    normally the list of tests will be exhaustive.
    """
    expr = result[-1]
    for test_, then_ in zip(test[:-1], result[:-1], strict=True):
        expr = z3.If(var == test_, then_, expr)
    return expr
```

Now, given Z3 variables corresponding to a state and a character, we can build an explicit expression defining the new state:

```python
def build_next_state(
    self, clue: Clue, st: z3.AstRef, ch: z3.AstRef
) -> z3.AstRef:
    pat = clue.pattern
    by_state = [
        build_match(
            ch,
            self.alphabet,
            [self.states[out] for out in pat.transition[i]],
        )
        for i in range(pat.nstate)
    ]

    return build_match(
        st,
        self.states[: pat.nstate],
        by_state,
    )
```

Now, where previously we had written `state_func(states[i], ch) == states[i + 1]`, we can instead embed the entire expression for each transition:

```python
for i, ch in enumerate(chars):
    solv.add(self.build_next_state(clue, states[i], ch) == states[i + 1])
```

This approach resulted in a substantial speedup, and, in particular, made performance much more **consistent**; at larger puzzle sizes, I saw many fewer "slow puzzles" than with the old pointwise-assertion approach.

Unfortunately, this direct approach had a significant downside! Constructing the huge z3 if-then ladders turned out to be quite expensive, to the point where we spent far longer constructing the Z3 expression than we spent solving it!

### Sharing the transition expression

In the end, I was able to find two strategies which allowed me to get the performance advantages of an explicit expression, but which didn't require constructing the expression once per character per clue.

The first approach I found was to use a `z3.Lambda` expression. We can use the same `build_next_state` function, but then wrap the result in a lambda expression, which turns `st` and `ch` into formal parameters of the lambda, and then repeatedly evaluate that lambda for each (state, character) pair:

```python
st = z3.Int("st")
ch = z3.Int("ch")

lambda_ = z3.Lambda([st, ch], self.build_funcexpr(clue, st, ch))

for i, ch in enumerate(chars):
    solv.add(lambda_[states[i], ch] == states[i + 1])
```

While reading [the SMT-LIB specification][smt-lib-spec], I noticed that, first, SMT-LIB allows defining functions with explicit bodies:

```lisp
(define-fun double ((x Int)) Int (* x 2))

;; evaluates to `8`
(simplify (double 4))
```

and then, second, that SMT-LIB defines that syntax in terms of `declare-fun` and a `forall` assertion:

![Screenshot of the SMT-LIB specification, defining "define-fun" as equivalent to declaring the named function using "declare-fun", and asserting its behavior using a "forall" ranging over the input variables.](smt-define-fun.png "SMT-LIB specification v2.7, p66, defining the semantics of `define-fun`")

Thus, I initially tried using a `z3.Function`, as initially, and pinning down its behavior using a forall:

```python
state_expr = self.build_funcexpr(clue, st, ch)
solv.add(z3.ForAll([st, ch], state_func(st, ch) == state_expr))
```

Unfortunately, while that approach did solve puzzles, doing only that resulted in some of the slowest solve times of any approach I tried!

However, after some more digging, I discovered the Z3 [macro-finder "tactic"][macro-finder]. Z3 tactics are transformations which can be applied to goals, which can contain algorithms or approaches above and beyond the core SMT solver. `macro-finder`, at its most basic, finds `forall` assertions like the one we just wrote, and then copy-pastes them into every invocation of the named function, resulting in an expression much like the one we originally constructed using Python in this section. However -- presumably by virtue of being implemented in C++ and closer to the core solver data model -- it does so much more efficiently. I found that all three approaches (the explicit expression, a Z3 `lambda`, and a `forall`+`macro-finder`) resulted in similar solve times in Z3, the latter two had comparable Python setup runtimes.

[macro-finder]: https://microsoft.github.io/z3guide/docs/strategies/summary#tactic-macro-finder


<!-- ORIGINAL DRAFT STARTS HERE -->

## Using z3 enumeration sorts

[regexle.com](https://regexle.com), though, supports puzzles up side-length 32! If we want to go past side-3, we're going to need to do a lot better.

Z3 is [an SMT solver][smt]. What that means, specifically, is that it's a SAT solver augmented with support for various datatypes and rules and and logic on those types, including (e.g.) the rules of integer arithmetic.

We've been using integers to name states and characters, since that's easy and since that's where most Z3 tutorials start. However, our FSM states and our characters don't actually have any meaningful arithmetic structure; any effort Z3 expends searching for or trying to use rules about addition, or range arithmetic, or other solvers or heuristics, is just wasted. Can we find a way to tell Z3 that (e.g.) our FSM states are just N different distinct objects with no structure **other** than what we describe?

I suspect this is a case where asking a Z3 expert would have saved me a ton of time, but I instead spent a long time reading Z3 and SMT-LIB documentation, as well as various Stack Overflow questions and answers, to arrive at what I believe is the correct answer: Z3 enumeration types. We can define our FSM states and our alphabet as Z3 enumeration types, which means essentially what I just said: This type has exactly N distinct values, but no other assumed structure.

```python
nstates = max(c.pattern.nstate for c in all_clues)
state_sort, states = z3.EnumSort(
    "State",
    [f"S{i}" for i in range(nstates)],
)

char_sort, alphabet = z3.EnumSort("Char", list(ALPHABET))
```

The rest of our code is only minimally changed; we need to replace `IntSort` in the declaration of our transition functions, and we need to replace `i` with `states[i]` or `alphabet[i]` any time we are expressing a comparison in the Z3 AST.

This small change produced a **drastic** improvement, both to the mean solve time, but **especially** at the tails:

![Z3 solve time, integer encoding vs EnumSort. EnumSort solves every side-3 puzzles in under a tenth of second, while the integer encoding takes at least 0.2s and as much as 3s](int-v-enum.png "Violin plot of Z3 solve time for the two approaches. Both methods use the pruning strategy from the previous section.")
{.center}

### A false start: `FiniteDomainSort`

While exploring options for representations, I stumbled upon [z3.FiniteDomainSort][finite-domain] in the z3py documentation. A "sort" is the Z3 name for what I'd think of a "type" in other contexts, and so a "finite-domain sort" ought to be a fresh datatype which has exactly `size` distinguished elements -- exactly what I needed. I coded up an implementation ... and my solver promptly started emitting total garbage: completely, obviously, invalid solutions. After a bit of searching, I found [a bug report][finite-domain-bug] and some other mentions that it is a special-case object only intended for the z3 Datalog engine, and just silently misbehaves in other contexts. Welp.

[finite-domain]: https://z3prover.github.io/api/html/class_microsoft_1_1_z3_1_1_finite_domain_sort.html
[finite-domain-bug]: https://github.com/Z3Prover/z3/issues/4842

### So many ways to describe a function

## Using Z3's `RegEx` theory


<!-- However, while skimming the [SMT-LIB language standard][smt-lib-spec], I happened to notice the `define-fun` command, which allows the definition of a named function **with an explicit implementation**. For instance, the following script prints `8` when evaluated with z3 (or another compatible solver):

```scheme
(define-fun double ((x Int)) Int (* x 2))
(simplify (double 4))
```

I could find a way to access `define-fun` using the Z3 Python bindings, but that discovery was enough to set me on a search to find an alternate way to encode the FSM transition tables into Z3, figuring that perhaps an explicit encoding would help Z3 evaluate and search faster.

I ended up trying a **lot** of variations. I'll walk through some of the
-->


[smt-lib-spec]: https://smt-lib.org/papers/smt-lib-reference-v2.7-r2025-07-07.pdf






## Outline
- Regex as DFA
- Getting a DFA
  - Greenery
  - Flattening
- First try: integers, implicitly-defined function

- Exploring solver performance
  - intro, meta
- Accidentally going very slowly
  - extraneous forall
  - The `state` bound
- Pruning
- Function representation
  - pointwise
  - `python`
  - lambda
  - forall
  - forall + macro-finder
- Z3 Sort choice
  - **almost** no change...
  - Helped me find my `forall` footgun
- Z3 Regexes
- Optimizing Z3py

## TODO
- [ ] Rework intro
- [ ] Link to code
- [ ] (maybe?) file and link to issue
- [ ] stylize as "Z3" consistently

[z3]: https://microsoft.github.io/z3guide/
[smt]: https://en.wikipedia.org/wiki/Satisfiability_modulo_theories
[smt-lib]: https://smt-lib.org/
[regexle]: https://regexle.com/
