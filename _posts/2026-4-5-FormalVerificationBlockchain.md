---
layout: post
title: "Formal verification on Seal DAO: TLA+, Lean 4 + Aeneas, Rocq, Kani (updated)"
---

[Seal DAO](https://github.com/SealProjectDAO/seal-dao-public) is a post-quantum L1 in Rust with on-chain SQL. The `formal/` tree pulls five tool families against different layers. This post is about that tree, not the chain.

## What's actually there

Counted against the current `formal/` directory:

- **TLA+** — 3 specs + 2 model-check drivers (5 `.tla` files total).
- **Lean 4** — `SealVerify/` with Aeneas extraction wiring plus hand-written proofs for Hash, MerkleTree, VRF (LaV), and DEX matching. 29 theorems/lemmas.
- **Rocq** — `seal_verify/` with Balance, StateMachine, SqlState, RLS. 29 theorems/lemmas.
- **Kani** — 66 `#[kani::proof]` harnesses across 19 files under `crates/seal-{token,consensus,merkle,crypto,threshold,node,bridge}/`.
- **Miri** — wired into `scripts/ci-formal.sh`; runs if the nightly Miri component is installed, else skipped gracefully.
- **cargo-fuzz** — 9 targets under `fuzz/fuzz_targets/`.

## TLA+

Two specs do the heavy lifting, plus a composite:

- `SealConsensus.tla` — 2/3-threshold committee voting. Proves Agreement (no two honest nodes finalize conflicting blocks), NoEquivocation, MonotonicHeight, and Progress (liveness). SQL execution is explicitly called out as orthogonal to consensus in the spec header; the TLA+ doesn't model SQL isolation.
- `SealBridge.tla` — lock/mint/burn bridge invariants: `MintedLeqLocked`, `NoDoubleMint`, `NoMintWithoutLock`, `BurnedLeqMinted`. Each one maps to a concrete check in `bridge.rs` via a comment header. This is the part I'd point a bridge reviewer at first.
- `SealCompositeProof.tla` — composition of the two.

## Lean 4 with Aeneas

The most interesting thing on the Lean side isn't a theorem, it's the plumbing. `formal/lean/SealVerify/Aeneas.lean` sets up a [charon](https://github.com/AeneasVerif/charon) → [aeneas](https://github.com/AeneasVerif/aeneas) pipeline that extracts Rust MIR for `seal-merkle` into Lean 4 definitions, so the proofs are about a model generated from the real code rather than a hand-written re-implementation that can drift. Hand-written proofs then compose on top.

What actually gets proved:

- **MerkleTree.lean** — `insert_lookup`, `insert_lookup_other`, `rootHash_deterministic`, `rootHash_injective`, `delete_lookup`, `delete_idempotent`, `delete_then_insert`, `delete_changes_root`. This is what I'd frame as "Merkle operations preserve the map abstraction and the root hash is a function of the contents," not "inclusion proofs are sound and complete" — that latter phrasing was in the earlier draft of this post and is tighter than what's proved.
- **VRF.lean** — `LaV.uniqueness`, `LaV.eval_deterministic`, `LaV.satisfies_uniqueness`, `election_integrity`. The VRF is LaV (Lattice-based many-time VRF), a post-quantum construction. It is not the Algorand VRF.
- **Hash.lean** — injectivity and incremental hashing correctness.
- **DEX.lean** — on-chain matching: `conservation_of_quantity`, `trade_price_bounded`, `no_trade_when_no_crossing`, `partial_fill_nonneg`, `time_priority`, `matching_reduces_orders`.

## Rocq

Four modules, all under `seal_verify/`:

- **Balance.v** — token arithmetic. `credit_debit_roundtrip`, `transfer_conserves`, `stake_unstake_roundtrip`, and the supporting well-formedness lemmas.
- **StateMachine.v** — chain transition function. `transition_deterministic`, `transfer_conserves_pair`, `invalid_transfer_no_change`, `mint_increases_supply`, `burn_decreases_supply`.
- **SqlState.v** — on-chain SQL operations on the state KV: `insert_increases_count`, `insert_lookup`, `insert_duplicate_fails`, `update_preserves_count`, `delete_missing_fails`, `sql_ops_deterministic`.
- **RLS.v** — row-level-security policy composition: `default_deny`, `policy_deterministic`, `adding_policy_restricts`, `non_bypassable_single`, `non_bypassable_pair`, `policy_commutative_pair`, `single_deny_blocks`.

Reward/slashing isn't in Rocq; it's covered at the implementation level by Kani harnesses in `seal-consensus/src/slashing.rs`.

## Kani at implementation level

Where the 66 harnesses live, grouped:

- `seal-token/` — balance, emission, orderbook, params, treasury (the density is highest here).
- `seal-consensus/` — config, genesis, slashing, validator.
- `seal-merkle/`, `seal-crypto/`, `seal-threshold/` (NTT, Ringtail), `seal-node/` (committee, delegation, fees, governance), `seal-bridge/`.

## Fuzzing

9 targets: `fuzz_address_parse`, `fuzz_block_deserialize`, `fuzz_committee_vote`, `fuzz_merkle_ops`, `fuzz_pqvrf_verify`, `fuzz_ringtail_verify`, `fuzz_sql_parser`, `fuzz_tx_deserialize`, `fuzz_vrf_verify`. Corpus and artifacts are checked in.

## What this doesn't get you

- **TLA+ is an abstraction.** A gap between the spec and the Rust is not caught by TLA+. The Aeneas pipeline closes that for `seal-merkle` by generating the Lean model from MIR; other crates are still at the hand-written level.
- **Kani is bounded.** Harnesses check up to a configured depth; deeper interleavings are out of reach.
- **Performance isn't a property.** Correct but slow consensus is still a DoS vector.
- **The spec can be wrong.** A false conclusion from a mis-stated invariant is worse than no proof, because it's load-bearing.
- **Miri is opt-in** in the CI script — if a contributor runs without the nightly Miri component installed, the step is skipped rather than failing. Worth knowing before assuming it catches every UB on every PR.

## What I'd add next

- Extend the Aeneas extraction beyond `seal-merkle` — the storage and consensus crates are the next candidates.
- Actually formalise sortition distribution, which is currently claimed at the README level but not modelled. A probabilistic TLA+ or a separate Lean probability sketch would both be reasonable shapes.
- A TLA+ model for SQL transaction isolation on top of consensus, rather than leaving the two layers disjoint.
- Make Miri a hard CI requirement on the crypto crates, not a soft skip.

Code @ [github.com/SealProjectDAO/seal-dao-public](https://github.com/SealProjectDAO/seal-dao-public).
