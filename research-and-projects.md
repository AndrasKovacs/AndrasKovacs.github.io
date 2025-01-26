---
layout: archive
title: "Research and projects"
permalink: /research-and-projects
author_profile: true
---

I summarize below my most notable outputs and the recommended up-to-date
versions for them. I don't list all [publications](/publications) here.

### Metatheory of quotient inductive-inductive and higher inductive-inductive types

This is the topic of my PhD thesis, titled [Type-Theoretic Signatures for
Algebraic Theories and Inductive Types](pdfs/phdthesis_compact.pdf). This is the most
up-to-date version of this research. My older papers in the same topic are
mostly superseded. There is one paper which is not superseded, because the
thesis only summarizes it, but does not really improve or revise it:

- [For Finitary Induction-Induction, Induction is Enough](https://drops.dagstuhl.de/opus/volltexte/2020/13070/), with
  Ambroise Lafont and Ambrus Kaposi

These [slides about my thesis](pdfs/wg6stockholm.pdf) might be also intersting.

### Efficient evaluation for cubical type theories

Existing cubical type theory implementations all suffer from efficiency problems.
I have a [WIP repository](https://github.com/AndrasKovacs/cubeval) with a
prototype implementation that has major new optimizations.

### Staged compilation & metaprogramming

- Implementation of [dependently typed runtime code generation](https://github.com/AndrasKovacs/dtt-rtcg).
- ICFP 2024: *Closure-Free Functional Programming in a Two-Level Type Theory*. [Paper](pdfs/2ltt_icfp24.pdf). [Code supplement](https://github.com/AndrasKovacs/staged/tree/main/icfp24paper/supplement).
- ICFP 2022: *Staged Compilation With Two-Level Type Theory*. [Paper](pdfs/2ltt.pdf). [Appendix](pdfs/2ltt_appendix.pdf). [Implementation](https://github.com/AndrasKovacs/staged).
- [Staged push/pull fusion in typed Template Haskell](https://github.com/AndrasKovacs/staged-fusion). It's mostly made obsolete by the ICFP 2024 paper.

### High-performance elaboration with dependent types

- [smalltt](https://github.com/AndrasKovacs/smalltt) is an implementation of
  elaboration for a minimal dependently typed language, which demonstrates a
  collection of performance optimization techniques.
- [normalization-bench](https://github.com/AndrasKovacs/normalization-bench) is
  a benchmark suite for conversion checking and normalization for pure lambda
  terms in a number of different languages and runtime systems, with an emphasis
  on machine-code-compiled HOAS (higher-order abstract syntax). It's a bit outdated now
  (implementations and runtime options could be improved), but I think it's
  still informative.

### Elaboration zoo

[This is](https://github.com/AndrasKovacs/elaboration-zoo) intended as a
pedagogical demonstration of best practice implementations of various
elaboration features, in settings with dependent types. It's far from complete,
and pull requests are welcome.

### Inference and unification

I've done some research on inference for first-class polymorphism. This is related to
what is called "impredicative" type inference in the older ML/GHC literature (I
explain in the following sources why "impredicative" is a misnomer). I produced
the following paper in 2020:

- [Elaboration With First-Class Implicit Function Types](https://dl.acm.org/doi/10.1145/3408983)

However, shortly after this was published, I found that there is a practically much more compelling
solution:

- [First-class polymorphism with dynamic order
  elaboration](https://github.com/AndrasKovacs/elaboration-zoo/tree/master/06-first-class-poly)

My stance now is that the older source is overkill: it could be in principle
scaled to more advanced inference, but in practice barely anyone would notice
the difference, and the new approach is much simpler. As far as I'm aware,
"dynamic order elaboration" is the best practice solution for first-class
polymorphism in practice.

I have also developed an extension to Miller's pattern unification algorithm. I
haven't yet made any formal publication about this; see this [Stack Exchange
answer](https://cstheory.stackexchange.com/questions/50914/swapping-arguments-of-variables-in-higher-order-pattern-unification/50918#50918)
for some explanation and links to more material. See also these WITS 23
[slides](pdfs/wits24prez.pdf).

### Universe features

This is a small but fun piece which shows that fancy predicative universe features (polymorphism, first-class levels, transfinite levels, level bounds) can be modeled with induction-recursion:

- [Generalized Universe Hierarchies and First-Class Universe Levels](https://drops.dagstuhl.de/opus/volltexte/2022/15748/pdf/LIPIcs-CSL-2022-28.pdf)
