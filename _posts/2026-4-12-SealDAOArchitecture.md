---
layout: post
title: "Seal DAO architecture: what the 17 crates actually do today (updated, WIP)"
---

Companion to the [Seal DAO formal-verification post]({% post_url 2026-4-5-FormalVerificationBlockchain %}). That one covered the `formal/` tree. This one covers the `crates/` tree — which parts are wired up, which parts are placeholders with a TODO header, and which parts are honestly aspirational. The earlier draft of this post inflated a few of those categories; rewriting against the code.

## 17 crates, roughly by maturity

From `crates/` (alphabetised): `seal-app`, `seal-bridge`, `seal-cli`, `seal-consensus`, `seal-crypto`, `seal-merkle`, `seal-mpc`, `seal-node`, `seal-p2p`, `seal-sql`, `seal-storage`, `seal-tee`, `seal-threshold`, `seal-token`, `seal-vrf`, `seal-wallet`, `seal-zk`.

The Rust under `crates/` (excluding vendored deps and `target/`) is about 45K lines, not the 2.1M figure the earlier draft quoted — that was counting the vendor tree.

Tests: 816 `#[test]` / `#[tokio::test]` entries across the workspace. The "804" in the earlier draft was approximately right.

## Consensus: not Algorand

The earlier draft said "Algorand VRF." That's wrong and it's the second time this particular claim has crept in. From `seal-vrf/src/lib.rs`, verbatim:

- `HmacVrf` — "HMAC-SHA3 stub (NOT PQ, for testing only)".
- `PqVrf` — "ML-DSA + SHA3 construction (PQ-secure, practical). **Use `PqVrf` for production.**"
- `LatticeVrf` — "LB-VRF placeholder (Module-LWE/SIS, future)".
- `LavVrf` — the Lattice-based many-time VRF referenced in the Lean formal work.

So production sortition is ML-DSA + SHA3, with a ~3.3 KB proof (one ML-DSA-65 signature). No EC-VRF anywhere in the stack.

The committee voting layer on top uses a 2/3 threshold. `SealConsensus.tla` proves Agreement, NoEquivocation, MonotonicHeight, and Progress on that shape.

## Threshold signatures: current is placeholder, real one is future

`seal-threshold/src/lib.rs` is explicit about this, and I'm going to quote it because the earlier draft of this post was not:

> `SimpleThreshold` (current) — Collects individual ML-DSA signatures and a participation bitfield. **NOT a real threshold scheme (signatures are not aggregated).** Used for protocol development.
>
> `RingtailThreshold` (future, TODO) — Ringtail lattice-based threshold signatures (ePrint 2024/1113). 2-round interactive protocol producing a single ~13.4 KB signature from 100 committee members.

So: "MPC-aggregated threshold signatures at the bridge" is what the design points at. What's actually running today is "collect individual signatures + bitfield, verify each." The checklist on `RingtailThreshold` in the source file lists porting the reference implementation, wiring round-1 preprocessing, Lean 4 proof of security properties, Kani/Miri/fuzz verification, and a 100-member WAN signing benchmark as still-to-do.

## Native SQL, with proofs at the operation layer

`seal-sql/` is real and the Rocq modules in the companion post prove the state-operation semantics (`SqlState.v`: `insert_lookup`, `insert_duplicate_fails`, `update_preserves_count`, `delete_missing_fails`, `sql_ops_deterministic`). Row-level security policy composition is in `RLS.v` (`default_deny`, `non_bypassable_pair`, `adding_policy_restricts`).

The tradeoff the earlier draft mentioned is real: keeping the SQL state consistent with consensus rounds needs careful transaction semantics. I called this "fast finality, no rollbacks" and then two paragraphs later described rollbacks — that was a contradiction. What's actually true: the state layer uses WAL-style savepoints aligned with consensus rounds so that a proposed-but-not-committed block can be reverted cheaply. Once a block commits, it is final.

## DEX: batch auction, real

One of the parts that checks out cleanly. `seal-token/src/orderbook.rs:1` opens with "On-chain order book DEX with per-block batch auction matching" using price-time priority: all orders submitted during a block interval are matched at block-production time. Lean proofs in `DEX.lean` cover `conservation_of_quantity`, `trade_price_bounded`, `no_trade_when_no_crossing`, `partial_fill_nonneg`, `time_priority`, `matching_reduces_orders`.

## Bridges: scaffolded to two chains

`bridges/solana/` and `bridges/stellar/` exist. `SealBridge.tla` proves the accounting invariants (`MintedLeqLocked`, `NoDoubleMint`, `NoMintWithoutLock`, `BurnedLeqMinted`) that map to `bridge.rs` in-code checks. What I would **not** claim is that either bridge is production-validated end-to-end against a live Solana or Stellar validator; the TLA+ side is proved, the Rust side has Kani harnesses on the invariants, but I haven't watched a cross-chain asset move and arrive in an independent explorer.

## Wallet

Three client shapes under `apps/`:

- `seal-cli` — terminal / server-side driver. This is what the earlier draft meant by "TUI."
- `apps/seal-wallet/` — Electron shell over WASM-compiled Rust core, with Tauri also wired.
- `apps/seal-wallet-android/` — JNI bindings over the same Rust core.

Same cryptographic core (ML-DSA-65, BIP-39) behind all three. The UI layer differs; the crypto stays in Rust.

## What I'd actually change next

Dropping the "start with the bridge, not the consensus" framing from the previous draft — that's revisionist and not a defensible lesson given the bridge itself is currently more scaffold than live infrastructure. Concrete next steps, grounded in TODOs already in the source:

- Land `RingtailThreshold` as the real threshold backend, following the checklist in `seal-threshold/src/lib.rs` (port → wire preprocessing → Lean proof → benchmarks).
- Extend Aeneas extraction (currently targeting `seal-merkle`, per the formal post) to `seal-storage` and at least one consensus path, so the Lean proofs are about MIR-extracted definitions rather than hand-written models.
- Drive one full bridged-asset roundtrip (Seal ↔ Solana) end-to-end in a dev environment with observability on both sides, before claiming the bridge "works."
- Take Miri off the soft-skip path in `scripts/ci-formal.sh` for the crypto crates.

Code @ [github.com/SealProjectDAO/seal-dao-public](https://github.com/SealProjectDAO/seal-dao-public).
