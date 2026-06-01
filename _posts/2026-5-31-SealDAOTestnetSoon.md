---
layout: post
title: "Seal DAO: closing the testnet checklist — PQC networking, decentralized KMS, bridge, ZK (we're close — get in touch)"
---

Short version: [Seal DAO]({% post_url 2026-4-12-SealDAOArchitecture %}) is close to a public testnet, and if you're interested in it — as a validator, a bridge partner, someone building SQL dapps on it, or an investor — now is the time to reach out. The rest of this post is what landed to get here.

Seal DAO is a post-quantum-secure blockchain with a native distributed SQL database — "PHP+MySQL but on-chain": deploy SQL schemas as decentralized apps, query them in PostgreSQL syntax, with ML-DSA-65 / ML-KEM-768 / SHA3 underneath and ZK proving via RISC Zero / SP1. 996 tests, Apache-2.0.

## What landed since the last update

The work since the [weekly update]({% post_url 2026-4-21-SealDAOWeeklyUpdate %}) has been the unglamorous testnet-blocking list — the CRITICAL and HIGH items in `PROBLEMS.md`, now checked off:

- **Post-quantum networking.** Node-to-node and worker links are mTLS/TLS with a **hybrid ML-KEM + X25519** handshake layered on top (`pq_handshake.rs`, `pq_transport.rs`); the session key is `SHA3(domain ‖ ss1 ‖ ss2)`, with a pure-ML-KEM transport as fallback. This replaced an earlier WireGuard/Tailscale plan.
- **Decentralized KMS.** A new `seal-kms-sidecar` crate — Unix-socket + TCP API (auth-token gated), a crash-safe trust store (write-temp-then-rename) with SHA3-384 integrity, secure memory, and reproducible Ed25519 binary signing. The model is local authority + ephemeral cloud workers with a Shamir-split decryption key. The `KmsClient` got a 10 s timeout, 3-retry exponential backoff, and a circuit breaker.
- **Verifiable secret sharing.** `seal-threshold/vss.rs` adds SHA3-256 commitments over Shamir shares with per-share and full-set verification.
- **Bridge.** A single-call `seal_bridgeWithdrawAndClaim` RPC (burn wrapped tokens → signed withdrawal → synchronous unlock), plus a real `seal-relayer` crate submitting to Solana/Stellar with cursor persistence and Prometheus metrics.
- **ZK.** `seal_submitProof` RPC + a `ZkVerifier` wired into the consensus runner, taking a hex proof and the state-transition public inputs (pre/post state roots, height, tx count/hash).
- **Governance & SQL.** Conviction voting now verifies voter balances; `CREATE POLICY` / `DROP POLICY` row-level-security DDL is wired into the SQL engine with PostgreSQL-compatible syntax (`HAS_TOKEN()`, `CURRENT_USER()` predicates).
- **Ops.** A `seal-provisioner` crate manages worker lifecycle (Docker backend; validator / bridge-observer / KMS-sidecar kinds), and `seal_getGenesis` makes genesis config queryable instead of hardcoded.

A dev build is already drivable: a faucet (`seal_faucet`, 1000 SEAL / 24 h cap), flat one-shot CLI commands (`seal transfer / faucet / balance`, plus a generic signed-`rpc` passthrough), and a fix to a real address-derivation bug where the node and wallet disagreed on the `bech32m` address. The public testnet is the next milestone.

## Honest about what's left

"Close to testnet" is not "testnet is live." The ZK path ships with a `StubVerifier` by default — the RPC and consensus wiring are real, swapping in the RISC Zero / SP1 backend is the remaining step. A security pass is ongoing. But the architectural blockers — PQC transport, decentralized key management, the bridge round-trip, RLS, governance — are in and tested, not on a roadmap slide. See the [formal-verification post]({% post_url 2026-4-5-FormalVerificationBlockchain %}) for what's machine-checked.

Mainnet is the milestone after this one, and deliberately so: the **mainnet feature set will be refined as we play with the testnet**. Economic parameters, governance thresholds, the bridge's chain coverage, which ZK backend, RLS policy ergonomics — these get decided against real usage on the testnet rather than frozen up front. That's a large part of why early testnet participants matter: what you do on it shapes what mainnet commits to.

## Interested? Now's the time

If any of this is relevant to you — running a validator, bridging assets, building a SQL-native dapp, a security review, or investing ahead of testnet — **get in touch**: [renoir42@renoir42.com](mailto:renoir42@renoir42.com?subject=Seal%20DAO) or via [seal-dao.network](https://seal-dao.network). Early conversations are the ones that shape what testnet, and then mainnet, look like.
