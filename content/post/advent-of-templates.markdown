---
title: "Advent of Code in C++ Template Metaprogramming"
slug: advent-of-templates
date: 2023-12-08T07:30:00-08:00
---
This December, the imp of the perverse struck me, and I decided to see how many days of Advent of Code I could do purely in compile-time C++ metaprogramming.

As of this writing, I’ve done two days, and I’m not sure I’ll make it any further. However, that’s one more day than I planned to do as of yesterday, which is in turn further than I thought I’d make it after my first attempt. So we’ll see how it goes.

That said, Day 1 was interesting enough to post a short writeup. You can [find the code on GitHub](https://github.com/nelhage/aoc2023), but I’ll also do an annotated walkthrough of day 1!


## Basic types

We’ll start by defining a basic compile-time `list` type, and we’ll also use a `literal<v>` in order to lift values into the type system I decided that we’d eschew `constexpr`, since that makes things too easy, but permit ourselves to use basic arithmetic and compile-time integers and booleans.


```c++
template <typename... Elts>
struct list {};

template <auto v>
struct literal {
    constexpr static auto value = v;
};
```

## Preamble: Reading input

Ideally, I would read input using the new [C++23](https://thephd.dev/finally-embed-in-c23) [`#embed`](https://thephd.dev/finally-embed-in-c23) [directive](https://thephd.dev/finally-embed-in-c23), for maximum purity; but as of this writing it seems hard to find a compiler that supports one. Thus, I fell back to a polyfill using `xxd -i`. We can run `xxd -i < input.txt > input.i`, and then include our input using `#include` and a variadic template. We’ll also use a bit of preprocessor trickery to pass the input file in using a `-D` compiler flag.


```c++
template <auto... Elt>
struct read_input {
    using type = list<literal<char(Elt)>...>;
};

#define _QUOTED(x) #x
#define QUOTED(x) _QUOTED(x)

#ifndef INPUT
// Default value to keep my IDE environment happy
#define INPUT /dev/null
#endif

using problem = read_input<
#include QUOTED(INPUT)
>::type;
```

## Folds

C++ template metaprogramming is very much a functional language, and so it’s natural that the main operation we’ll define over lists will be `fold`.  This is a relatively standard construction:


```c++
template <template<typename, typename> typename Fn,
    typename Init, typename List>
struct fold {};

template <template<typename, typename> typename Fn,
    typename Init>
struct fold<Fn, Init, list<>> {
    using type = Init;
};

template <template<typename, typename> typename Fn,
          typename Init, typename El1, typename... Elts>
struct fold<Fn, Init, list<El1, Elts...>> {
    using type = fold<
        Fn,
        typename Fn<Init, El1>::type,
        list<Elts...>>::type;
};
```


## Some more helpers

We’ll introduce a few more helpers, and then we’re almost ready for Part 1. We’ll add a `nil` sentinel, a basic `if` conditional, and an `or_else` function that takes the first non-nil of its arguments:


```c++
struct nil;

template <typename T>
struct is_nil { using type = literal<false>; };

template <>
struct is_nil<nil> { using type = literal<true>; };

template <typename Cond, typename Then, typename Else>
struct if_else {};

template <typename Then, typename Else>
struct if_else<literal<true>, Then, Else> { using type = Then; };

template <typename Then, typename Else>
struct if_else<literal<false>, Then, Else> { using type = Else; };

template <typename L, typename R>
struct or_else { using type = L; };

template <typename R>
struct or_else<nil, R> { using type = R; };
```


# Part 1

If you're not doing Advent of Code, a quick refresher for the problem statement: For each line in the input file, we need to find the **first** and **last** digit in that line, and then concatenate them into a two-digit number (the "calibration number"), and output the sum of every calibration number in the file.

We'll implement this using a single `fold` over the input for efficiency. As we go, we'll track state consistent of a tuple of `(sum so far, first digit in current line, last digit seen so far in line)`:


```c++
template <typename Accum, typename FirstDigit, typename LastDigit>
struct State {
    using accum = Accum;
    using first = FirstDigit;
    using last = LastDigit;
};

using empty_state = State<literal<0>, nil, nil>;
```

The fold function advances the state using some simple rules:

- If the character is a newline,  we compute this line’s calibration value, add it in to the accumulator, and reset the per-line state.
- If the character is a digit, we update the “last digit” to this digit unconditionally
    - And also update the first digit, if we haven’t already seen a digit this line.

These are all fairly straightforward applications of existing helpers:

```c++
template <typename T>
using as_digit = if_else<
    literal<T::value >= '0' && T::value <= '9'>,
    literal<T::value - '0'>,
    nil>;

template <typename In, typename Char>
struct Fn {
    using digit = as_digit<Char>::type;

    struct IfNewline {
        using type = State<
            literal<In::accum::value + 10 * In::first::value + In::last::value>,
            nil,
            nil>;
    };

    using type = if_else<
        literal<Char::value == '\n'>,
        IfNewline,
        if_else<
            typename is_nil<digit>::type,
            In,
            typename if_else<
                typename is_nil<typename In::first>::type,
                State<typename In::accum, digit, digit>,
                State<typename In::accum, typename In::first, digit>
                >::type
            >>::type::type;
};
```

One bit of trickiness is that, in order to make the outer `if_else` lazy, we evaluate `::type` on its **result**, instead of on its arguments. This results in the compiler evaluating whichever of the consequent or the alternate is actually selected.


# Trying it out

For convenience, we’ll allow a tiny bit of runtime behavior to print the result:

```c++
int main() {
    using answer = solve<problem>::type;
    printf("%d\n", answer::value);
}
```

Armed with that, we can now verify the initial test case included in the problem body:

```console
$ xxd -i < test.txt > test.i && \
  c++ -DINPUT=test.i -std=c++20 part1.cc -o out/part1 && \
  out/part1
142
```

But if we try the real input, we run into issues:


```console
$ xxd -i < input.txt > input.i && \
  c++ -DINPUT=input.i -std=c++20 part1.cc -o out/part1 && \
  out/part1

part1.cc:8:1: fatal error: recursive template instantiation exceeded maximum depth of 1024
using as_digit = if_else<

 [cut C++ template spew]

part1.cc:8:1: note: use -ftemplate-depth=N to increase recursive template instantiation depth
```

The basic problem here is that our `fold` needs to recurse for each character in the input, and the provided test case is over 20,000 characters long! We can try increasing `-ftemplate-depth`, per the provided suggestion; On my machine we **can** make the program compile with a large-enough value, but clang gives scary warnings about stack capacity and is very slow -- compiling takes 90s or more.

We’ll need a different approach.

## Optimizing `fold`

The problem here is fairly fundamental; recursive template expansion is basically **the** game in town for template metaprogramming. However, it turns out that there is one option we have up our sleeve.

 `list<>` is a variadic template, which means we can manipulate its entire contents as a C++ [parameter pack](https://en.cppreference.com/w/cpp/language/parameter_pack). As long as we can keep the arguments as a parameter pack, we can use relatively fast paths inside the compiler, and work much more efficiently.

Poring over the above cppreference page, we notice that C++17 added “fold expressions” — a way of turning a parameter pack into an expression that folds a user-chosen operator over the pack! This is suspiciously similar to what we’re trying to do, but it operates on the **value** level, using value-level operators.

However, it turns out that with some judicious use of `auto`, `decltype`, and a wrapper struct, this poses no problem — we can construct types that perform our type-level computation in the process of typechecking the fold expression.

The compiler will still have to deal with a massively nested **expression**, but in my testing that’s much more efficient than a 20,000-deep template:


```c++
template<template<typename, typename> typename Fn>
struct fold_helper {
    template <typename T>
    struct F {
        using type = T;

        template <typename R>
        auto operator<<(F<R>) {
            return F<typename Fn<T, R>::type>{};
        };
    };
};

template <template<typename, typename> typename Fn,
          typename Init, typename... Elts>
struct fold<Fn, Init, list<Elts...>> {
    template <typename T>
    using F = fold_helper<Fn>::template F<T>;

    using type = decltype((F<Init>{} << ... << F<Elts>{}))::type;
};
```


The struct `fold_helper<Fn>::F` translates our type-level function into one that can be evaluated via an expression-level operator. The expression `fold_helper<Fn>::F<A>{} << fold_helper<Fn>::F<B>{}` will evaluate to a value of type `Fn<A, B>::type`. Once we have that, we expand the list elements into a fold expression using that translation, and use `decltype` to access the type of the output. At no point will any code actually be emitted or — horrors! — evaluated.

And this works! With this `fold` implementation, the part 1 solution above compiles iin about a second on my M1 Air, and gives the correct answer!


```console
$ xxd -i < input.txt > input.i && \
  time c++ -fbracket-depth=25000 -DINPUT=input.i -std=c++20 \
    part1.cc -o out/part1 && \
  out/part1
c++ -fbracket-depth=25000 -DINPUT=input.i -std=c++20 part1.cc -o out/part1  1.10s user 0.06s system 101% cpu 1.141 total
54390
```


# Part 2

For part 2, digits may now be spelled out -- `seven82683` has a calibration value of "73."

We’ll start by introducing a short abstraction. Handling newlines in our fold worked, but was a bit annoying. We can write a helper that abstracts the splitting on newlines and lets us worry about individual lines.

The interface will take the form of a `fold` over **lines,** instead of, say, materializing a list-of-lists; keeping our processing streaming in this fashion is very important for performance. The helper is, itself, a fairly straightforward application of `fold`:


```c++
template <typename Head, typename Tail>
struct pair {
    using head = Head;
    using tail = Tail;
};

template <template<typename, typename> typename Fn, typename Delim>
struct fold_lines_f {
    template <typename In, typename Elt>
    struct F{
        using type = pair<
            typename append<typename In::head, Elt>::type,
            typename In::tail
            >;
    };

    template <typename A>
    struct F<A, Delim>{
        using type = pair<
            list<>,
            typename Fn<typename A::tail, typename A::head>::type
            >;
    };
};

template<template<typename, typename> typename Fn,
         typename Init, typename L,
         typename Delim = literal<'\n'>>
struct fold_lines {
    using type = fold<
        fold_lines_f<Fn, Delim>::template F,
        pair<list<>, Init>,
        L>::type::tail;
};
```

Now that we have that, we need to compute calibration values within a single line…


## Manually compiling a state machine

We’re going to once again implement calibration using a single `fold` over each line.

In order to match each of the possible digits, we’ll maintain a finite-state-machine matcher, and write transition rules for each <state, character> pair — this is essentially the DFA that [some regular expression implementations](https://swtch.com/~rsc/regexp/regexp1.html) would produce under the hood. We’ll name states by the “relevant” substring we’ve seen so far; so, for instance, `Sseve` means we just saw “seve”; in this case, if we encounter an `n` then we’ve found the digit `7`.

Note that many digits contain letters which can start another digit, and we’ll need to handle these in our transitions; for instance, if we’re in state `Sseve` and we see an `i`, we need to move into `Sei` because we might be looking at the string “seveight” and need to handle the `eight`. Fortunately, no two digits have more-complex overlaps, so the state machine remains fairly simple.

We’ll start with state definitions, using `S0` to mean “no relevant prefix”:


```c++
struct S0 {};

// one
struct So {};
struct Son {};
// two
struct St {};
struct Stw {};
// three
struct Sth {};
struct Sthr {};
struct Sthre {};
// four
struct Sf {};
struct Sfo {};
struct Sfou {};
// five
struct Sfi {};
struct Sfiv {};
// six
struct Ss {};
struct Ssi {};
// seven
struct Sse {};
struct Ssev {};
struct Sseve {};
// eight
struct Se {};
struct Sei {};
struct Seig {};
struct Seigh {};
// nine
struct Sn {};
struct Sni {};
struct Snin {};
```

And we can define the transition rules. We’ll make liberal use of wildcards, and rely on relative specificity of the partial specializations to pick the correct rule.

If no other rule matches, we go back to `S0`:

```c++
template <typename St, typename El>
struct next_state { using type = S0; };
```

And if we see the first letter for any digit — and no more-specific rule matched — we can move directly into that state.


```c++
// Initial letters
template<typename S> struct next_state<S, literal<'o'>> { using type = So; };
template<typename S> struct next_state<S, literal<'t'>> { using type = St; };
template<typename S> struct next_state<S, literal<'f'>> { using type = Sf; };
template<typename S> struct next_state<S, literal<'s'>> { using type = Ss; };
template<typename S> struct next_state<S, literal<'e'>> { using type = Se; };
template<typename S> struct next_state<S, literal<'n'>> { using type = Sn; };
```

Then we define the transitions within and between rules. Most of these are straightforward — e.g. `(So,` `'``n``'``) → Son`, but we have to be careful of the ones where we might jump “midway” into matching another digit.


```c++
// one
template<> struct next_state<So,    literal<'n'>> { using type = Son; };
template<> struct next_state<Son,   literal<'e'>> { using type = Se; };
template<> struct next_state<Son,   literal<'i'>> { using type = Sni; };
// two
template<> struct next_state<St,    literal<'w'>> { using type = Stw; };
// three
template<> struct next_state<St,    literal<'h'>> { using type = Sth; };
template<> struct next_state<Sth,   literal<'r'>> { using type = Sthr; };
template<> struct next_state<Sthr,  literal<'e'>> { using type = Sthre; };
template<> struct next_state<Sthre, literal<'i'>> { using type = Sei; };
// four
template<> struct next_state<Sf,    literal<'o'>> { using type = Sfo; };
template<> struct next_state<Sfo,   literal<'u'>> { using type = Sfou; };
template<> struct next_state<Sfo,   literal<'n'>> { using type = Son; };
// five
template<> struct next_state<Sf,    literal<'i'>> { using type = Sfi; };
template<> struct next_state<Sfi,   literal<'v'>> { using type = Sfiv; };
// six
template<> struct next_state<Ss,    literal<'i'>> { using type = Ssi; };
// seven
template<> struct next_state<Ss,    literal<'e'>> { using type = Sse; };
template<> struct next_state<Sse,   literal<'v'>> { using type = Ssev; };
template<> struct next_state<Sse,   literal<'i'>> { using type = Sei; };
template<> struct next_state<Ssev,  literal<'e'>> { using type = Sseve; };
template<> struct next_state<Sseve, literal<'i'>> { using type = Sei; };
// eight
template<> struct next_state<Se,    literal<'i'>> { using type = Sei; };
template<> struct next_state<Sei,   literal<'g'>> { using type = Seig; };
template<> struct next_state<Seig,  literal<'h'>> { using type = Seigh; };
// nine
template<> struct next_state<Sn,    literal<'i'>> { using type = Sni; };
template<> struct next_state<Sni,   literal<'n'>> { using type = Snin; };
```

We haven’t yet defined when we “match” a rule, or what to do when we do; It turns out, for this problem, that defining “successful match” as a separate function from our state transition makes both functions much simpler. If you like to be technical, we're essentially implementing what EEs would call [a Mealy state machine](https://en.wikipedia.org/wiki/Mealy_machine) for this matcher.

The `match_digit` function will will have a similar signature to `next_state` — `state, input → digit | nil` -- and tells us which digit (if any) matched at any particular position:


```c++
template <typename State, typename El>
struct match_digit { using type = nil; };

template <> struct match_digit<Son,   literal<'e'>> { using type = literal<1>; };
template <> struct match_digit<Stw,   literal<'o'>> { using type = literal<2>; };
template <> struct match_digit<Sthre, literal<'e'>> { using type = literal<3>; };
template <> struct match_digit<Sfou,  literal<'r'>> { using type = literal<4>; };
template <> struct match_digit<Sfiv,  literal<'e'>> { using type = literal<5>; };
template <> struct match_digit<Ssi,   literal<'x'>> { using type = literal<6>; };
template <> struct match_digit<Sseve, literal<'n'>> { using type = literal<7>; };
template <> struct match_digit<Seigh, literal<'t'>> { using type = literal<8>; };
template <> struct match_digit<Snin,  literal<'e'>> { using type = literal<9>; };

template<typename S> struct match_digit<S, literal<'0'>> { using type = literal<0>; };
template<typename S> struct match_digit<S, literal<'1'>> { using type = literal<1>; };
template<typename S> struct match_digit<S, literal<'2'>> { using type = literal<2>; };
template<typename S> struct match_digit<S, literal<'3'>> { using type = literal<3>; };
template<typename S> struct match_digit<S, literal<'4'>> { using type = literal<4>; };
template<typename S> struct match_digit<S, literal<'5'>> { using type = literal<5>; };
template<typename S> struct match_digit<S, literal<'6'>> { using type = literal<6>; };
template<typename S> struct match_digit<S, literal<'7'>> { using type = literal<7>; };
template<typename S> struct match_digit<S, literal<'8'>> { using type = literal<8>; };
template<typename S> struct match_digit<S, literal<'9'>> { using type = literal<9>; };
```

(We could replace the last 10 specializations with a conditional inside the primary definition, but I find this version somewhat more straightforward, if a bit verbose).

With these types in place, we can now do a very similar fold as in part 1 to compute the calibration value for a single line. Compared to that one, we no longer need to track the overall accumulator — we’ll use `fold_lines` to push that to an outer loop — but we do need to keep the matcher state, and so our state class looks very similar:


```c++
template <typename MatchState, typename First, typename Last>
struct CalibrationState {
    using state = MatchState;
    using first = First;
    using last = Last;
};
```

To advance one element, we compute the output state, and the matched digit (if any). We’ll also use some of our helpers to simplify the “first / last seen digit” computation:


```c++
template <typename State, typename El>
struct LineF {
    using new_state = next_state<typename State::state, El>::type;
    using digit = match_digit<typename State::state, El>::type;

    using type = CalibrationState<
        new_state,
        typename or_else<typename State::first, digit>::type,
        typename or_else<digit, typename State::last>::type
        >;
};
```

With the fold function in place, computing calibration values is easy:


```c++
template<typename Line>
struct calibration {
    using acc = fold<LineF, CalibrationState<S0, nil, nil>, Line>::type;
    using type = literal<acc::first::value*10 + acc::last::value>;
};
```

We can even write some quick test cases — C++ template metaprogramming has excellent support for in-line tests.


```c++
static_assert(is_same<
              typename calibration<typename read_input<
              's', 'e', 'v', 'e', 'n', '8', '2', '6', '8', '3'
              >::type>::type,
              literal<73>>::value);
```


## Putting it all together

The fold over lines is a simple sum of calibration values — almost trivial, at this point.


```c++
template<typename State, typename Line>
struct Fn {
    using lineval = calibration<Line>::type;

    using type = literal<State::value + lineval::value>;
};

using answer = fold_lines<Fn, literal<0>, problem>::type;
```

It works! And, on my laptop, it’s “only” ~1.5s to compile!

# Conclusion

I’ve done a bit of C++ template metaprogramming here and there in the past — for instance, I wrote [a simple x86 assembler for C++, once](https://github.com/nelhage/bemu/blob/master/x86.h) — but this was my first really encounter with C++17 TMP and with truly going down the deep end.

And, I’ve heard this sentiment before, but after working through this problem … honestly I have to admit that modern C++ templates are a pretty passable functional programming language, even with some nice features! And I was pleasantly surprised to discover the fold-expression trick, which even allows for relatively-efficient list processing.

I’m … still not sure how many more days I will do, or how much more C++ I will have cause to write, but I ended up quite enjoying the exercise!
