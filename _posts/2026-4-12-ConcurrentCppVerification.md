---
layout: post
title: "Concurrent C++ in 2026: What TLA+ and CBMC Actually Catch"
---

In a [previous post]({% post_url 2026-3-15-LockFreeDataStructures %}) I described the overall architecture of [Thetis26](https://github.com/zeta1999/Thetis26-public) and why formal verification matters for lock-free code. This post gets concrete — specific bugs caught by specific tools, and why you need multiple verification approaches.

## The Bug That Testing Missed

Early in the development of the SPSC ring buffer, I had a version that passed 50 million iterations of a concurrent stress test on x86_64. Every run, every time. Ship it, right?

GenMC found a bug on the first run. The issue was a `memory_order_relaxed` store to the write index that could be reordered past the data write on ARM64's weak memory model. On x86 (which has Total Store Order), the hardware enforces the ordering for free. On ARM64, it doesn't.

The fix was one line: change `memory_order_relaxed` to `memory_order_release` on the write index store. But finding which line needed changing — that's what GenMC is for.

## Four Tools, Four Strengths

Each tool in the Thetis26 verification suite catches a different class of bug:

### TLA+ — Algorithm Correctness

TLA+ operates on an abstraction of the algorithm, not the code. A TLA+ spec for the SPSC ring buffer models producers and consumers as processes, the buffer as a sequence, and the read/write indices as variables. The model checker (TLC) explores all possible interleavings and verifies invariants:

- **Safety**: the buffer never returns data that wasn't written
- **FIFO ordering**: elements come out in the order they were put in
- **No lost updates**: every write is eventually readable
- **Bounded**: the buffer never holds more than N elements

TLA+ catches design-level bugs — the algorithm itself is wrong, independent of implementation language or memory model.

### CBMC — Implementation Verification

CBMC (C Bounded Model Checker) takes the actual C++ source code and exhaustively checks it up to a bounded depth. It catches:

- Integer overflow in index arithmetic
- Out-of-bounds array access
- Assertion violations in the actual implementation
- Divergence between the code and the TLA+ spec

CBMC works on concrete C++ — templates, pointers, bit manipulation and all. The bound is a trade-off: deeper bounds catch more bugs but take exponentially longer.

### SPIN — Protocol Properties

SPIN checks liveness properties and deadlock freedom via Promela models. For Thetis26, SPIN verifies:

- **Deadlock freedom**: no interleaving where all threads are stuck
- **Liveness**: if a producer writes, a consumer eventually reads
- **Starvation freedom** (for wait-free variants): every thread completes within bounded steps

These are properties that TLA+ can also check, but SPIN's explicit-state model checking is sometimes more efficient for protocol-level properties.

### GenMC — Memory Model Bugs

GenMC is the specialist. It explores all executions allowed by the C/C++ memory model (including relaxed, acquire/release, and seq_cst) and checks for:

- Data races
- Memory ordering violations
- Incorrect use of `memory_order_relaxed` vs `memory_order_acquire`/`release`
- Bugs that only manifest on weak memory model architectures (ARM64, RISC-V)

This is the tool that caught the SPSC ring buffer bug described above. Testing on x86 would never have found it.

## The Verification Workflow

The lesson from building Thetis26: **specs first, code second**.

For the first few data structures, I wrote the C++ first and verified after. Every time, verification caught bugs that required design changes — not just code fixes. For the later data structures, I wrote the TLA+ spec first, verified the algorithm, then implemented in C++ and verified the implementation with CBMC and GenMC.

The specs-first approach is slower upfront but dramatically faster overall. A design bug caught in TLA+ is a 10-minute fix. The same bug caught in C++ after building the wait-free helping mechanism is a week of rework.

## Cost

Twenty TLA+ and Promela models, CBMC harnesses for every data structure, GenMC configurations for every lock-free variant. It's a significant investment — probably 30-40% of the total development time went into verification.

Is it worth it? For a library that other systems depend on for correctness under concurrency — yes, unambiguously. A subtle memory ordering bug in a lock-free queue can corrupt data silently for months before anyone notices. The verification cost is paid once; the confidence compounds.

The repo is at [github.com/zeta1999/Thetis26-public](https://github.com/zeta1999/Thetis26-public).
