---
layout: post
title: "Seal DAO: what landed in the private tree this week (not yet pushed public)"
---

Quick field note on [Seal DAO](https://github.com/SealProjectDAO/seal-dao-public). The public mirror was last synced on 2026-04-13. The week since has had most of its work in the private tree; below is what landed there, with a note on what is and isn't yet in the public repo. I'll push the batch after it settles.

Separately — not everything I work on is Seal. I've also been contributing to [moetsi/Sensor-Stream-Pipe](https://github.com/moetsi/Sensor-Stream-Pipe) around the same time; that work sits outside the Seal tree and is its own codebase with its own review flow.

## Ringtail threshold: beyond the placeholder

The [formal post]({% post_url 2026-4-5-FormalVerificationBlockchain %}) and the [architecture post]({% post_url 2026-4-12-SealDAOArchitecture %}) both called out that `SimpleThreshold` is not a real threshold scheme (it just collects individual ML-DSA signatures and a participation bitfield), with `RingtailThreshold` marked future-TODO in `seal-threshold/src/lib.rs`.

This week in the private tree:

- Full Ringtail protocol path, including Lagrange coefficient derivation and rounding primitives — the 2-round aggregate-signature shape from ePrint 2024/1113 rather than the signature-list stand-in.
- A `seal-ringtail-verify` crate: `no_std` verifier that compiles for Solana BPF, with a Soroban Stellar counterpart. The bridge side uses it feature-gated on both chains.
- A cost projection for Solana BPF in compute-units-per-instruction, so we can size the verifier's on-chain footprint before anyone pays for the first cross-chain signature verification.

Status: in the private tree, not yet in `seal-dao-public`. The blog-post caveat from the architecture post ("what's running today is collect individual signatures + bitfield") still describes the public mirror.

## Bridge: observers, MAC verify, e2e

Also private until the next sync:

- Real Solana and Stellar observer RPC clients (`bridges/*` — replacing the earlier stub clients that were enough for TLA+ invariant checks but not for driving real transactions).
- Committee-MAC signature verify implemented as an Anchor program on Solana and as a Soroban contract on Stellar.
- Real SAC (Stellar Asset Contract) transfers inside the Soroban bridge contract.
- `docker-compose` + a `bridge-e2e.sh` that spins up a local Seal node, a local Solana validator, and a Soroban sandbox, and walks a full bridged-asset round-trip through them.
- Per-chain bridge emergency pause with a 2/3 Technical Council supermajority gate; `seal_getBridgeStatus` now surfaces `paused_chains` in its response.

In the public repo, the bridge surface is still the TLA+-proven invariant set and the Kani harnesses on `BridgeManager` — the invariants are proved; the operational tooling above is not.

## DEX: trades in the state root

`TxType::DexMatch` is now emitted per block so matched trades land in the state root and in the ZK proof, instead of existing only in the engine's in-memory orderbook. The per-block batch-auction match documented in the architecture post is otherwise unchanged; this makes its outputs cryptographically anchored to the block they happened in.

## ADR-001: SQL procs default, WASM opt-in

Architecture decision recorded: smart-contract execution defaults to SQL stored procedures (CALL / PL/pgSQL-shaped) on the native SQL state, with WASM as an opt-in path for contracts that need it. The `.seal` demo tree (see below) exercises the SQL-procs side; the WASM side has a validator but no demo contracts yet.

## Demo apps

Three new demo crates landed under `apps/`:

- `copy-trading.seal` — follow-the-leader trades on the on-chain DEX.
- `kyc.seal` — identity attestation with selective disclosure (uses the MPC + ZK stack).
- `kindle.seal` — a reader / content-ledger demo.
- `forms.seal` — a fourth one, a form-collection app that exercises MPC, ZK, AEAD, and the browser wallet surface end-to-end.

These aren't production apps; they're the thing we point at when someone asks "what does the SQL-native chain actually let you do?" that a token transfer doesn't answer.

## Governance and RPC surface

- Gov RPC endpoints for proposal submission / voting / tally.
- Namespace-scoped row-level security in SQL state: the RLS policies formalised in Rocq (`RLS.v`, per the formal post) now apply inside namespaces as well as at the global layer.
- `seal_setTransferFee` / `seal_getTransferFee` RPCs for fee-market experiments.
- A browser extension wallet on top of the existing Electron / Android apps.

## Verification counters

- Kani: 66 of 66 harnesses passing (the remaining 6 took a BTreeMap swap and a harness refactor to close out).
- Test count: 820 → 887 over the week. Mostly driven by the bridge observer, Ringtail verifier, and demo-app crates.

## When this lands public

Soon. I prefer pushing a week of work as a single coherent batch once the Ringtail + bridge e2e scripts stop finding their own bugs, rather than interleaving mid-bug states. If you were checking the public mirror against this post and wondering where half the features were — that's why. The public README and the architecture posts on this blog will get bumped at the same time.

Code @ [github.com/SealProjectDAO/seal-dao-public](https://github.com/SealProjectDAO/seal-dao-public).
