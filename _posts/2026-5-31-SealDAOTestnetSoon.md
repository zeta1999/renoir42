---
layout: post
title: "Seal DAO is close to testnet — help build it before mainnet locks in"
---

Short version: [Seal DAO]({% post_url 2026-4-12-SealDAOArchitecture %}) is approaching a public testnet, and the protocol isn't frozen yet — what you do on the testnet shapes what mainnet commits to. The architectural blockers are in and tested; what's left is the kind of work that goes faster with more hands and sharper eyes on it. So if any of this is your thing — running a validator, partnering on the bridge, building a SQL-native dapp, reviewing the cryptography, or backing the project ahead of testnet — now is the time to reach out. The rest of this post is what landed to get here, and what's honestly still open.

Seal DAO is a post-quantum-secure L1 with a native, consensus-replicated SQL state model: deploy SQL schemas as decentralized apps and query them in PostgreSQL syntax, with ML-DSA-65 / ML-KEM-768 / SHA3 underneath and ZK proving via RISC Zero / SP1. If you've shipped a PHP+MySQL app, the development model will feel familiar — it's roughly "PHP+MySQL, but on-chain." It's Apache-2.0 and fully open: 996 tests, plus Kani proof harnesses and cargo-fuzz targets, all runnable from the repo.

## What landed since the last update

The work since the [weekly update]({% post_url 2026-4-21-SealDAOWeeklyUpdate %}) has been the unglamorous testnet-blocking list — the CRITICAL and HIGH items in `PROBLEMS.md`, now checked off:

- **Post-quantum networking.** Node-to-node and worker links are mTLS/TLS with a **hybrid ML-KEM + X25519** handshake layered on top (`pq_handshake.rs`, `pq_transport.rs`); the session key is `SHA3(domain ‖ ss1 ‖ ss2)`, with a pure-ML-KEM transport as fallback. This replaced an earlier WireGuard/Tailscale plan.
- **Decentralized KMS.** A new `seal-kms-sidecar` crate — Unix-socket + TCP API (auth-token gated), a crash-safe trust store (write-temp-then-rename) with SHA3-384 integrity, secure memory, and reproducible Ed25519 binary signing. The model is local authority + ephemeral cloud workers with a Shamir-split decryption key. The `KmsClient` got a 10 s timeout, 3-retry exponential backoff, and a circuit breaker.
- **Verifiable secret sharing.** `seal-threshold/vss.rs` adds SHA3-256 commitments over Shamir shares with per-share and full-set verification.
- **Bridge.** A single-call `seal_bridgeWithdrawAndClaim` RPC (burn wrapped tokens → signed withdrawal → synchronous unlock), plus a real `seal-relayer` binary submitting to Solana/Stellar with cursor persistence and Prometheus metrics.
- **ZK.** `seal_submitProof` RPC + a `ZkVerifier` wired into the consensus runner, taking a hex proof and the state-transition public inputs (pre/post state roots, height, tx count/hash).
- **Governance & SQL.** Conviction voting now verifies voter balances; `CREATE POLICY` / `DROP POLICY` row-level-security DDL is wired into the SQL engine with PostgreSQL-compatible syntax (`HAS_TOKEN()`, `CURRENT_USER()` predicates).
- **Worker lifecycle.** A `seal-provisioner` crate manages worker lifecycle (Docker backend; validator / bridge-observer / KMS-sidecar kinds), and `seal_getGenesis` makes genesis config queryable instead of hardcoded.
- **Operator runbooks.** The most recent push is the paperwork you write right before you hand a network to other people: node-failure recovery (consensus-liveness guarantees, validator crash / partition / slashing, observer and KMS failure modes, disaster recovery with stake transfer), backup/restore (chain state via Docker volumes + protocol snapshots, plus validator / committee / KMS / council key backup, multi-region), a `QUICKSTART` covering three deploy paths (laptop-local bridge e2e, public Solana-devnet/Stellar-testnet, single-node dev), and a drain-before-migrate `bridge-lifecycle.sh` for redeploying bridge contracts without putting wrapped tokens at risk.

A dev build is already runnable: you can stand up a local node, hit a faucet (`seal_faucet`, 1000 SEAL / 24 h cap), and drive it from the CLI (`seal transfer / faucet / balance`, plus a generic signed-`rpc` passthrough). We also fixed a real address-derivation bug where the node and wallet disagreed on the `bech32m` address. The public testnet is the next milestone.

## Honest about what's left

"Close to testnet" is not "testnet is live." The biggest single remaining piece is the **proof backend**: the ZK path ships with a `StubVerifier` by default, so the chain does not yet cryptographically verify proofs end-to-end. The `seal_submitProof` RPC and the consensus-side verifier wiring are real and tested against the stub — swapping in a real RISC Zero / SP1 backend is a well-scoped task, and if you work on proving systems it's the highest-leverage thing you could pick up here. A security pass is also ongoing. But the architectural blockers — PQC transport, decentralized key management, the bridge round-trip, RLS, governance — are in and tested, not on a roadmap slide. See the [formal-verification post]({% post_url 2026-4-5-FormalVerificationBlockchain %}) for what's machine-checked.

Mainnet is the milestone after this one, and deliberately so: the **mainnet feature set will be refined as the testnet runs**. Economic parameters, governance thresholds, the bridge's chain coverage, which ZK backend, RLS policy ergonomics — these get decided against real usage on the testnet rather than frozen up front. That's a large part of why early testnet participants matter: what you do on it shapes what mainnet commits to.

## Want to help build it?

This is the part where more people genuinely speeds things up. Concretely, we'd like to talk to:

- **Validators / node operators** — the consensus is Algorand-style (VRF + committee voting), no mining, and the operator runbooks above are written. Help us shake them out on real, geographically-spread hardware.
- **Bridge partners** — the Solana + Stellar round-trip works locally and against public devnets; we want partners to harden it against real-world conditions.
- **Dapp builders** — the SQL-native model should feel familiar if you've ever shipped a PHP+MySQL app. Tell us where the abstraction leaks.
- **Cryptography & security reviewers** — the ML-DSA / ML-KEM integration and the Ringtail threshold-signature path are the highest-stakes surface. Independent eyes are welcome before mainnet, not after.

No token has been issued and nothing here is an offer of one — this is an invitation to build, review, and shape, not a sale. (If you're a fund or angel who backs deep-tech / crypto infrastructure at the protocol stage, that conversation is open too.) If any of it is relevant to you, **get in touch**: [build@seal-dao.network](mailto:build@seal-dao.network?subject=Seal%20DAO) or via [seal-dao.network](https://seal-dao.network).
