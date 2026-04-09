---
layout: post
title: "Formal Verification for a Blockchain: Kani, Lean 4, and Rocq"
---

[Seal DAO](https://github.com/SealProjectDAO/seal-dao-public) is a post-quantum blockchain with a native SQL database, built in Rust. One of the things we invested heavily in was formal verification — using multiple tools at different abstraction levels to build confidence that the protocol is correct.

This post covers the verification approach, not the blockchain design itself.

## Why Multiple Tools

No single verification tool covers everything. They operate at different levels of abstraction and catch different classes of bugs:

- **TLA+** works at the protocol/algorithm level. You model the system as a state machine and check invariants across all reachable states. Good for consensus, leader election, message ordering — the distributed systems properties.
- **Lean 4** is a proof assistant. You write mathematical proofs that certain properties hold. Good for core data structure invariants (Merkle tree correctness, VRF output distribution).
- **Rocq/Coq** — same category as Lean 4, used for token conservation proofs and the reward/slashing logic.
- **Kani** is a bounded model checker for Rust. It takes your actual Rust source code and checks assertions up to a bounded depth. Good for implementation bugs — off-by-one errors, integer overflows, panic paths.
- **Miri** — Rust's undefined behavior detector. Catches memory safety violations that safe Rust prevents but unsafe blocks can introduce.
- **cargo-fuzz** — coverage-guided fuzzing for the serialization, networking, and crypto layers.

## What We Verified

### Protocol Level (TLA+)

6 TLA+ models covering:
- **Consensus safety**: No two honest nodes finalize conflicting blocks
- **Liveness**: If enough nodes are honest, new blocks are eventually finalized
- **VRF committee selection**: The sortition process selects committees with the expected size distribution
- **SQL transaction isolation**: Concurrent SQL transactions on-chain don't produce anomalies

### Mathematical Properties (Lean 4, Rocq)

- **Merkle tree correctness**: Inclusion proofs are sound and complete
- **VRF output distribution**: The Algorand VRF produces uniformly distributed outputs
- **Token conservation**: The total token supply is invariant under all state transitions (Rocq)
- **Reward/slashing**: Validator rewards and slashing penalties are correctly computed and bounded (Rocq)

### Implementation (Kani)

16 Kani harnesses covering:
- Serialization round-trips (encode then decode equals original)
- Integer arithmetic in fee calculation (no overflow)
- State transition function edge cases
- SQL query planner correctness for simple queries

### Fuzzing (cargo-fuzz)

9 fuzz targets on:
- Network message parsing (malformed packets)
- SQL parser (adversarial queries)
- Crypto primitives (malformed keys, signatures)
- RPC API inputs

## What Formal Verification Doesn't Catch

It's worth being honest about the limits:

- **TLA+ models an abstraction**, not the code. A bug in the translation from spec to implementation won't be caught.
- **Kani is bounded** — it checks up to N steps. A bug at step N+1 is invisible.
- **Neither tool reasons about performance** — a correct but slow consensus implementation is still a DoS vector.
- **The specification itself might be wrong** — verification proves that the code matches the spec, not that the spec captures what you actually want.

The combination of tools at multiple abstraction levels mitigates these gaps. TLA+ catches algorithm bugs that Kani's bounded depth would miss. Kani catches implementation bugs that TLA+'s abstraction would miss. Fuzzing catches the weird inputs that nobody thought to model.

The repo is at [github.com/SealProjectDAO/seal-dao-public](https://github.com/SealProjectDAO/seal-dao-public).
