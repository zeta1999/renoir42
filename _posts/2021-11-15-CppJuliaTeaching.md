---
layout: post
title: "Teaching C99 and Julia side by side"
---

A note on [C99_and_julia_introduction](https://github.com/zeta1999/C99_and_julia_introduction), a short course repo I put together in Feb–Mar 2021.

## The premise

If you teach C99 and Julia in parallel, at the numerics layer, the languages agree more than students expect. Both give you column-major arrays (Julia by default, C if you decide to), both let you reason in terms of strided memory, both expose the actual machine representation. The things that differ — GC, dispatch, the notebook-friendly tooling — are not the things that matter when you're teaching what a `for` loop over a matrix actually does.

## What the course covered

- Arrays, indexing, strides. Drawing the same memory diagram for both languages.
- Loops vs broadcast. Showing that Julia's `.` broadcast lowers to a loop you could have written in C.
- A small numerical problem (something along a Cholesky or a Jacobi iteration) written both ways, benchmarked both ways, profiled both ways.
- Where Julia's zero-cost-abstraction promise turns out to cost runtime — specifically when you hit a type-unstable path and the JIT falls back to boxed values.

## What I'd revise

The course predates a lot of the modern C++ / Rust / Julia interop story. If I were teaching this today, I'd probably add a Rust column and put them side by side in three. Also the Julia compile-time-vs-first-run-time story has changed enough with native-code caching that the "just wait for compilation" slide needs rewriting.

Code / slides scaffolding @ [github.com/zeta1999/C99_and_julia_introduction](https://github.com/zeta1999/C99_and_julia_introduction).
