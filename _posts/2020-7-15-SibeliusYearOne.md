---
layout: post
title: "Sibelius — a one-maintainer C++ pricing library, year one"
---

A note on [Sibelius](https://github.com/zeta1999/Sibelius), a multi-asset derivatives pricing library I started in late 2019 and pushed hard through 2020 (~158 commits across the year). Closed source beyond the header examples; this is the shape, not the internals.

## What it is

C++ library for derivatives pricing with a scripting layer ("Aria") for payoff definition, plus Greeks, PDE solvers, Heston, barrier analytics, AAD via operator overloading, Monte Carlo with antithetic variates, batch deployment primitives, and a calibration layer.

The Aria language is the same name — and it turns out to be part of the same design instinct — as the DSLs that later show up in [gpu-backtest]({% post_url 2026-4-12-AriaStrategyDSL %}) and [Aria-HDL]({% post_url 2026-4-12-FPGAMetaCompiler %}). They aren't the same compiler; they share a surface shape (`|>` composition, small vocabulary of built-ins, bytecode-back-end friendly).

## One maintainer, three CIs

CI across Linux, macOS, Windows. The discipline of three CIs when one person is writing the code is worth more than it sounds — you don't get away with "works on my box" because the Windows runner will catch the MSVC-vs-GCC template quirks the same night you wrote them.

## What year one taught me

- A pricing library is 20% pricing and 80% plumbing: calendars, calibration harnesses, AAD tape management, Monte Carlo variance reduction, AST caching.
- The scripting language is worth it. Payoffs written in C++ are unreadable two weeks later; payoffs in a small DSL read like the term sheet. The cost is a compiler.
- The closed-source / open-source split matters less than the "what I can demo" split. I kept public the headers, the Aria benchmarks doc, and the CI shape — enough for someone to see the outline without having the full codebase.

## Where it went from there

Sibelius didn't stop in 2020; it's been evolving through 2026 (with quieter years in between). Later posts on Aria-shaped DSLs are lateral descendants of this codebase.

Code skeleton @ [github.com/zeta1999/Sibelius](https://github.com/zeta1999/Sibelius) (public CI only; internals closed).
