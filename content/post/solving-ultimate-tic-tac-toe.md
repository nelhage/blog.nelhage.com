---
title: "Towards solving Ultimate Tic Tac Toe"
slug: solving-ultimate-ttt
date: 2020-07-15T10:15:21-07:00
---

Summary: [Read about my efforts to solve the game of Ultimate Tic Tac Toe][solving]. It's been a fun journey into interesting algorithms and high-performance parallel programming in Rust.

## Backstory

Starting around the beginning of the COVID-19 lockdown, I've gotten myself deeply nerdsniped by an attempt to solve the game of [Ultimate Tic Tac Toe][uttt], a two-level Tic Tac Toe variant which is (unlike Tic Tac Toe) nontrivial and contains some interesting strategic elements.

I was recently introduced to Ultimate Tic Tac Toe by my partner, Kate, and, as is a bit of habit for me, I decided to explore it by writing a minimax AI. (I previously used this strategy on the game of [Tak][tak], resulting in my AI, [Taktician][taktician], still the [highest-ranked][tak-rankings] Tak-playing entity on the internet). I also intended to use this project as a platform to gain practice and experience with Rust, which I'd long aspired to get more familiar with.

After dabbling in an minimax AI for a little while, I decided to explore whether it might be feasible to actually _solve_ the game (that is, to prove whether the first player has a winning strategy and to find an implementation of such strategy). Solving games is a branch of combinatorial game theory I hadn't had much experience with, and was curious to learn more about. That lead me off onto an adventurous chase through the literature, reading papers about solving Checkers and other games, understanding whole new families of algorithms, and implementing and tweaking and refining them for myself.

After a few months of intermittent work, I have not yet solved the game, but I found it to be a great opportunity to learn Rust, learn about and implement new fun new algorithms, and experiment more with high-performance CPU-instensive parallel programming.

In order to share that journey with you all and anyone who might be interested, I've thrown up a [new website with detailed writeups][solving] of the algorithms I've had the opportunity to research and learn about, and some of the high-performance techniques I've been able to explore, and more. I've found this a really fun and educational project and hope that others might enjoy my writeups or learn something from them as well.

[tak]: https://en.wikipedia.org/wiki/Tak_(game)
[taktician]: https://github.com/nelhage/taktician
[tak-rankings]: https://www.reddit.com/r/Tak/wiki/unofficial_playtak_ranking
[uttt]: https://en.wikipedia.org/wiki/Ultimate_tic-tac-toe
[solving]: https://www.minimax.dev/docs/ultimate/
