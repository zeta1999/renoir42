---
layout: post
title: "Thetis26: lock-free / wait-free data structures, with formal models"
---

A short post on one of my personal C++20 repos: [Thetis26-public](https://github.com/zeta1999/Thetis26-public).

## Scope

The repo is broader than just data structures — it also has low-latency networking (UDP/multicast, TCP, TLS via mbedTLS, WebSocket, HTTP), post-quantum crypto (ML-KEM-768, ML-DSA-65, XChaCha20-Poly1305), a handful of exchange WebSocket connectors, and demo apps (a VDF database, a UDP camera pipeline). In this post I only cover the concurrent data-structure side.

## Data structures

Ring buffer (SPSC, MPMC v1-v4, LRQ), dense hashmap, ordered dict (skip list), stack (Treiber + elimination), priority queue. Each one ships as three variants when it makes sense: lock-based (mutex baseline), lock-free (CAS with explicit `std::memory_order`), wait-free (Kogan-Petrank-style helping, or equivalent per structure).

The primitives below them are worth a mention: RDCSS/MCAS, 128-bit CAS2 on x86_64 (64-bit on RISC-V 32), epoch-based reclamation, a OneFile wait-free STM, a cache-line-aligned pool allocator.

Numbers per structure and per platform are in `docs/benchmark_summary.md` and `docs/performance.md`; I'd rather point you there than quote cherry-picked figures.

## Formal models

`models/` has CBMC, GenMC, SPIN, TLA+, Nidhugg, plus some Lean material — five model checkers in active use, with ~20 models total covering the key data structures. The division of labor:

- TLA+ for algorithm-level invariants (is the ring buffer actually a FIFO under all interleavings?).
- CBMC for bounded checking of the C++ itself.
- SPIN for protocol-level liveness / deadlock freedom.
- GenMC (RC11) and Nidhugg (TSO/PSO/POWER) for memory-model bugs: the cases where `memory_order_relaxed` vs `memory_order_acquire` is the difference.

The reason for the split: TLA+ tells you the abstract algorithm is fine, but on an abstraction. CBMC and the stateless model checkers (GenMC, Nidhugg) operate on the code. For weak memory you really need the last one — x86-TSO hides bugs that ARM64 will happily expose.

## Cross-compile

Builds and tests on x86_64 Linux, ARM64 Linux, macOS (Apple Silicon), RISC-V 32, FreeBSD, and ESP32-C3/S3 (the S3 is compile-only; FreeBSD full-VM tests work on x86_64 Linux with KVM only, not on Apple Silicon — worth being honest about). CI scripts live under `scripts/ci_multiplatform.sh` and the per-target `cross_*.sh` wrappers.

One thing that bit me on embedded targets: not every RISC-V 32 / ESP32 target supports every atomic width natively, so `std::atomic<T>` with a wide `T` can silently fall back to a lock-based shim. Worth checking the generated code rather than assuming the "lock-free" label from `std::atomic::is_lock_free()` means what you hope.

## What I'd change

Mostly: write the TLA+ spec before the C++, not after. I did it in the wrong order for the first few structures, and the model-checker found things that would have been obvious design-time catches. Not a novel lesson, just one I had to re-learn.

Code @ [github.com/zeta1999/Thetis26-public](https://github.com/zeta1999/Thetis26-public).
