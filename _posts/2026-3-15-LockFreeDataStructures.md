---
layout: post
title: "Building Lock-Free Data Structures That Actually Work"
---

[Thetis26](https://github.com/zeta1999/Thetis26-public) is a C++20 library of lock-free and wait-free concurrent data structures. It started as a personal challenge — could I build concurrent containers that are both fast and provably correct? — and grew into something I use as the foundation for several other projects.

## What's In It

The library includes SPSC and MPMC ring buffers, concurrent dense hashmaps, skip lists, stacks, and priority queues. Each data structure comes in three variants where it makes sense: locked (mutex baseline), lock-free (CAS-based progress guarantee), and wait-free (bounded-step completion for every thread).

The current benchmarks on x86_64:
- **SPSC ring buffer**: 379M ops/sec
- **Dense hashmap (wait-free)**: 117M ops/sec

It cross-compiles to x86_64, ARM64, RISC-V 32, FreeBSD, and ESP32, with CI running on all targets.

## Why Formal Verification Matters Here

Testing concurrent code is necessary but deeply insufficient. You can run millions of iterations of a multithreaded test and miss a bug that only manifests under a specific interleaving that your scheduler happens to never produce. The classic bugs — ABA problems, missed wakeups, memory ordering violations — hide in the exponential space of thread interleavings.

Thetis26 uses five verification tools across 20 models:

- **TLA+** for high-level algorithm correctness — does the ring buffer actually behave like a FIFO under all interleavings?
- **CBMC** (C Bounded Model Checker) for implementation-level verification — does *this specific C++ code* match the TLA+ spec?
- **SPIN** for protocol-level properties — liveness, deadlock freedom
- **GenMC** for memory model verification — are the `std::memory_order` annotations correct under the C++20 memory model?

The combination matters. TLA+ can tell you your algorithm is correct, but it operates on an abstraction. CBMC checks the actual code but has bounded depth. GenMC specifically targets weak memory model bugs — the kind where `memory_order_relaxed` vs `memory_order_acquire` is the difference between correct and subtly broken.

## The Hard Parts

**Memory orderings are the real enemy.** The algorithm for a lock-free hashmap is well-known. Getting the memory orderings right so it works on ARM64 (which has a weaker memory model than x86) is where the months go. x86's TSO memory model hides bugs that ARM64 will happily expose in production.

**Wait-free is genuinely harder than lock-free.** Lock-free guarantees that *some* thread makes progress. Wait-free guarantees that *every* thread completes in bounded steps. The difference sounds academic until you have a real-time constraint — if you need guaranteed worst-case latency per operation, lock-free isn't enough. The wait-free variants use helping mechanisms where faster threads assist slower ones, which adds complexity but gives you that bounded-step guarantee.

**Cross-platform atomics on embedded targets** (ESP32, RISC-V 32) require careful fallback strategies. Not every target supports every atomic width natively — you need to know where the compiler is inserting lock-based fallbacks for your "lock-free" code.

## What I'd Do Differently

If I started over, I'd write the TLA+ specs first, before any C++. I did it the other way around for the first few data structures — write the code, then verify — and the verification step caught real bugs that would have been design-level catches with specs-first.

The repo is at [github.com/zeta1999/Thetis26-public](https://github.com/zeta1999/Thetis26-public).
