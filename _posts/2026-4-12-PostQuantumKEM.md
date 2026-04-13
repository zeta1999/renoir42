---
layout: post
title: "Post-Quantum Key Exchange in Practice: ML-KEM-768"
---

In a [previous post]({% post_url 2026-3-30-PostQuantumSecureMemory %}) I covered the secure memory side of [rust-secure-memory](https://github.com/zeta1999/rust-secure-memory-public) — guard pages, mlock, zeroization. This post goes deeper into the cryptographic layer: the ML-KEM-768 post-quantum key encapsulation mechanism and how it integrates with XChaCha20-Poly1305 for encrypted memory enclaves.

## Why Post-Quantum Now

NIST finalized ML-KEM (formerly CRYSTALS-Kyber) as a standard in 2024. The "harvest now, decrypt later" threat is real — adversaries recording encrypted traffic today can break it once quantum computers reach sufficient scale. For anything that stores long-lived secrets (key material, credentials, session tokens), migrating to PQ crypto is not premature — it's overdue.

## ML-KEM-768: The Basics

ML-KEM is a lattice-based key encapsulation mechanism. The "768" refers to the security parameter — ML-KEM-768 targets roughly 192-bit classical security and 128-bit quantum security.

The protocol is simple:
1. **KeyGen**: generate a public/private keypair
2. **Encapsulate**: using the public key, produce a ciphertext and a shared secret
3. **Decapsulate**: using the private key and the ciphertext, recover the shared secret

The shared secret is then used as the key for a symmetric cipher — in our case, XChaCha20-Poly1305.

## The Practical Concerns

**Key sizes**: ML-KEM-768 public keys are 1,184 bytes, ciphertexts are 1,088 bytes. That's roughly 10x larger than X25519. For network protocols this matters; for encrypting memory enclaves on the same machine, it's irrelevant.

**Performance**: encapsulation and decapsulation are fast — sub-millisecond on modern hardware. The bottleneck in rust-secure-memory is never the KEM; it's the symmetric encryption of the enclave contents.

**Hybrid approach**: in production, you might want ML-KEM-768 + X25519 in a hybrid construction, so you don't lose security if either scheme is broken. rust-secure-memory currently uses ML-KEM-768 standalone, but the architecture supports pluggable KEM backends.

## Encrypted Enclaves

The flow in rust-secure-memory:

1. Allocate a memory region with `mlock` (prevent swapping to disk) and guard pages (detect overflow/underflow)
2. Generate an ML-KEM-768 keypair
3. When the enclave is "sealed", encapsulate to produce a shared secret
4. Encrypt the enclave contents with XChaCha20-Poly1305 using the shared secret
5. Zeroize the plaintext and the shared secret
6. To unseal, decapsulate and decrypt

The enclave can be sealed and unsealed multiple times. Each seal operation generates a fresh shared secret via a new encapsulation, so there's no key reuse.

## Formal Verification

The KEM integration is covered by Kani harnesses and cargo-fuzz campaigns. The Kani harnesses verify that encapsulate/decapsulate round-trips correctly for all reachable states. cargo-fuzz tests the parsing of malformed ciphertexts and public keys to ensure no panics or memory corruption.

This doesn't prove the cryptographic security of ML-KEM itself — that's a mathematical property of the underlying lattice problem. What it proves is that the Rust code correctly implements the protocol and handles edge cases safely.

The repo is at [github.com/zeta1999/rust-secure-memory-public](https://github.com/zeta1999/rust-secure-memory-public).
