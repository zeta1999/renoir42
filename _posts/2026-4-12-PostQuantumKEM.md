---
layout: post
title: "ML-KEM-768 in rust-secure-memory: what's wired, what isn't (updated)"
---

Follow-on to the [rust-secure-memory overview]({% post_url 2026-3-30-PostQuantumSecureMemory %}). An earlier draft of this post described the KEM as the mechanism behind Enclave sealing. It isn't. Rewriting against what `kem.rs`, `enclave.rs`, and `crypto.rs` actually do.

## What's actually implemented

`secure-memory/src/kem.rs` wraps [`ml-kem`](https://crates.io/crates/ml-kem) (FIPS 203 / ML-KEM-768) as a standalone primitive:

- `KemKeyPair::generate()` — returns a pair where the secret key (2400 bytes) is stored in a `LockedBuffer` and the public key (1184 bytes) is a plain `Vec<u8>`.
- `encapsulate(public_key)` — returns `(Ciphertext [1088 bytes], SharedSecret [32 bytes in a LockedBuffer])`.
- `KemKeyPair::decapsulate(ciphertext)` — returns the 32-byte shared secret as a `LockedBuffer`.

Sizes pulled directly from the constants at the top of `kem.rs`: `EK_SIZE = 1184`, `DK_SIZE = 2400`, `CT_SIZE = 1088`, `SS_SIZE = 32`. ML-KEM-768 targets ~192-bit classical / 128-bit quantum security (NIST Category 3).

## What the Enclave actually uses

Here's the important correction. An `Enclave` is not encrypted with a per-seal KEM shared secret. Reading `enclave.rs`:

- `Enclave::new(data)` calls `crypto::session_encrypt(data)`.
- `crypto::session_encrypt` looks up the **per-process session key** — a random 32-byte key lazily initialised in a `LockedBuffer` (`crypto.rs:19-28`).
- Encryption is XChaCha20-Poly1305 with that session key.
- `Enclave::open()` does the reverse with `session_decrypt`.

So Enclave sealing is symmetric-only. ML-KEM is exposed alongside it as a separate primitive, but there is no current code path where an `encapsulate()` result becomes the key to an `Enclave`. The previous draft of this post described that integration in step-by-step detail. It doesn't exist.

The practical consequence: Enclaves work *within* a single process (the session key is wiped on `destroy_session_key()` / `purge()`). If you wanted to ship a sealed Enclave to another process and let them unseal it with their own long-lived keypair, you'd need to wire ML-KEM into the sealing path yourself. That wiring is not there today.

## "Pluggable KEM backends" — also not real

The earlier draft claimed "the architecture supports pluggable KEM backends." Grep for `trait` or `dyn` in `kem.rs` returns nothing. The module hard-wires `MlKem768` from the `ml-kem` crate. A hybrid construction (X25519 ⊕ ML-KEM-768) is something I'd *want* and listed in the previous post's "what I'd add" — it's not something the code currently supports via a swap-in interface.

## Size tradeoff, honestly

ML-KEM-768's public key at 1184 bytes and ciphertext at 1088 bytes are roughly 35× the size of X25519's 32-byte public and 32-byte ciphertext, not 10× as the previous draft said. For on-machine use it's irrelevant. For network protocols it's the dominant handshake cost — `notbbg`'s PQC transport layer pays exactly this price on every connection.

## Verification of the KEM wrapper

- **3 Kani harnesses** in `kem.rs` (lines 235, 244, 253) — bounded checks on input validation for encapsulate/decapsulate. Not "all reachable states" as the previous draft implied; Kani is depth-bounded, and that matters.
- **`fuzz_kem.rs`** — a `cargo-fuzz` target that feeds arbitrary-length buffers into both `encapsulate` and `decapsulate`, asserting wrong-sized inputs are rejected and that CT_SIZE-byte inputs trigger ML-KEM's implicit-rejection path (a silent different-secret return, per FIPS 203) without panicking.

Neither of these proves anything about ML-KEM's cryptographic soundness — that's a property of the underlying MLWE/MLWR problems, argued in the FIPS 203 standard. They prove that this Rust wrapper rejects malformed inputs and preserves buffer sizing discipline.

## What I'd do next, and honestly

- Actually wire ML-KEM into Enclave sealing as an *alternative* path: `Enclave::seal_for(public_key)` that encapsulates, derives a symmetric key via HKDF, encrypts under that, stores the ciphertext alongside the KEM ciphertext. Unsealer holds a `KemKeyPair`.
- Add the hybrid X25519 ⊕ ML-KEM-768 construction before anyone depends on ML-KEM alone for an adversarial setting. The defensive shape is trivial: run both KEMs, concat shared secrets, HKDF into the session key.
- Extend the Lean 4 proofs in `proofs/lean4/SecureMemory/` with a spec for the seal/unseal composition if the above wiring ships, so the top-level security argument has a machine-checked skeleton rather than just composition lemmas.

Code @ [github.com/zeta1999/rust-secure-memory-public](https://github.com/zeta1999/rust-secure-memory-public).
