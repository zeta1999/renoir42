---
layout: post
title: "Formalizing Numerical Algorithms — Work in Progress"
---

A short note on an ongoing experiment: [numerical-algorithms](https://github.com/zeta1999/numerical-algorithms).

I've been exploring what it takes to apply formal methods — the kind of tools I use for concurrent data structures (TLA+, CBMC) and blockchain verification (Kani, Lean 4) — to classical numerical algorithms. The question is simple: can we formally verify properties of numerical code beyond just "it doesn't crash"?

Numerical computing has a specific class of bugs that testing alone struggles to catch: catastrophic cancellation, loss of significance, accumulation of rounding errors across iterations, and edge cases where an algorithm that's correct in exact arithmetic silently produces garbage in IEEE 754. These bugs pass thousands of test cases and then bite you on a specific input range.

This repo is a collection of small experiments along those lines. It's early-stage and very much work in progress — more of a research notebook than a library. If this area interests you, the repo is public and I'll update it as things take shape.
