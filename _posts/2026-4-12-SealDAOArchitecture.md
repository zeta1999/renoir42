---
layout: post
title: "Seal DAO Architecture: Building a Post-Quantum L1 from Scratch (WIP)"
---

In a [previous post]({% post_url 2026-4-5-FormalVerificationBlockchain %}) I covered the formal verification approach for [Seal DAO](https://github.com/SealProjectDAO/seal-dao-public). This post is about the architecture itself — why we made the design choices we did, and what building an L1 blockchain from scratch in Rust actually involves.

## Why From Scratch

The standard approach to building a blockchain is to fork an existing one — fork Ethereum, fork Cosmos SDK, fork Substrate. You get consensus, networking, storage, and an ecosystem for free. The trade-offs: you inherit their design decisions, their technical debt, and their security assumptions.

We wanted three things that existing L1s don't provide together:
1. **Post-quantum cryptography from day one** — not bolted on later
2. **A native SQL database** — not key-value storage with SQL layered on top
3. **Formal verification** of the consensus and state machine — not just tests

So we built from scratch. 2.14 million lines of Rust, 17-crate workspace, 804 tests.

## Consensus: Algorand VRF

Seal DAO uses a consensus protocol based on Algorand's Verifiable Random Function (VRF). The VRF provides cryptographic sortition — each node can privately determine whether it's been selected to propose or vote in a given round, without revealing this to other nodes until it acts. This gives us:

- **Byzantine fault tolerance** up to 1/3 malicious stake
- **Fast finality** — blocks are final once committed, no rollbacks
- **Energy efficiency** — no proof-of-work

The VRF keys are post-quantum (ML-DSA-65), so the sortition is secure against quantum adversaries. This was a key design requirement — if the VRF can be forged, an attacker can predict and control block production.

## Native SQL Database

Most blockchains store state in a Merkle trie backed by LevelDB or RocksDB — essentially a key-value store. This works for simple token balances but makes complex queries expensive. Want to find all accounts with balance > X that interacted with contract Y in the last N blocks? That's a full state scan.

Seal DAO stores chain state in a native SQL database. Blocks, transactions, accounts, and contract state are all queryable via SQL. The 27 RPC methods expose this directly — you can run analytical queries against the live chain state without an external indexer.

The trade-off is complexity. Maintaining SQL consistency across consensus rounds — where a block might be proposed, partially applied, and then rolled back — requires careful transaction management. The state database uses write-ahead logging with savepoints that align with consensus rounds.

## On-Chain DEX

The on-chain decentralized exchange is built into the L1, not deployed as a smart contract. This means order matching runs at consensus speed with no gas overhead. The DEX supports limit orders, market orders, and an AMM pool for liquidity bootstrapping.

Having the DEX at the protocol level also means the consensus can enforce ordering fairness — the current implementation uses a batch auction mechanism that prevents frontrunning by committing all orders in a round before matching.

## Multi-Chain Bridges

Seal DAO bridges to Solana and Stellar. The bridge design uses MPC (multi-party computation) aggregation — a committee of bridge validators collectively signs cross-chain messages without any single validator holding the full signing key.

The MPC protocol is post-quantum (ML-DSA-65 threshold signatures), which means the bridge security doesn't degrade when quantum computers arrive. This is important because bridges are the highest-value attack target in cross-chain infrastructure — a compromised bridge key means unlimited minting on the target chain.

## Wallet Suite

The wallet comes in three variants:
- **TUI** — terminal-based wallet for power users and server environments
- **Electron/WASM** — desktop wallet with a web-based UI
- **Android** — mobile wallet via JNI bindings to the Rust core

All three share the same Rust cryptographic core — key generation (ML-DSA-65), transaction signing, BIP-39 mnemonic management. The UI layer is different; the crypto never leaves Rust.

## The 17-Crate Workspace

The codebase is split into 17 crates for modularity and compile-time isolation:

The separation means you can build and test the consensus layer without pulling in the DEX, or work on the wallet without compiling the full node. CI runs all 804 tests across the full workspace.

## What We'd Do Differently

**Start with the bridge, not the consensus.** The consensus layer took the longest to build and verify, but the bridge is what creates value — it connects your chain to the rest of the ecosystem. If I did it again, I'd build a bridge-first architecture where the L1 consensus is simpler and the bridge is the primary engineering investment.

**More property-based testing earlier.** We added Kani harnesses and TLA+ models midway through development. Some of the bugs they caught would have been easier to fix if caught during initial design.

The repo is at [github.com/SealProjectDAO/seal-dao-public](https://github.com/SealProjectDAO/seal-dao-public).
