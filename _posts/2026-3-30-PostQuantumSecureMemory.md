---
layout: post
title: "Post-Quantum Secure Memory in Rust"
---

[rust-secure-memory](https://github.com/zeta1999/rust-secure-memory-public) is a Rust library for handling sensitive data in memory — cryptographic keys, tokens, passphrases, anything that shouldn't linger in RAM longer than necessary or be readable by adjacent memory.

## The Problem

When you store a secret in a normal `Vec<u8>` or `String`:
- It can be swapped to disk by the OS
- It stays in memory after you're done with it (until the allocator reuses that page)
- A neighboring buffer overflow can read it
- A core dump includes it in plaintext

In languages with garbage collection, you don't even control when the memory is freed. Rust gives you deterministic drop, which helps, but doesn't solve the other problems.

## What rust-secure-memory Does

**LockedBuffer** — a heap allocation with:
- `mlock` to prevent the OS from swapping it to disk
- Guard pages on both sides (unmapped pages that trigger a segfault on access — catches overflows and underflows)
- Canary values to detect overwrites
- Secure wipe on drop (zeroes the memory before freeing)

**Enclave** — an encrypted-at-rest container. Data is encrypted with XChaCha20-Poly1305 when not actively in use. You open the enclave, get a reference to the plaintext in a LockedBuffer, work with it, and close it — the plaintext is wiped and only the ciphertext remains.

**Key derivation** — Argon2id (memory-hard) combined with a sequential VDF (SHA3-256 chain, time-hard). The Argon2id handles brute-force resistance; the VDF adds a wall-clock time cost that can't be parallelized.

**Post-quantum key encapsulation** — ML-KEM-768 (FIPS 203). For cases where you're exchanging keys between processes or machines, the KEM provides protection against harvest-now-decrypt-later attacks. NIST finalized ML-KEM in 2024; the 768 parameter set targets 192-bit security.

## Cross-Platform

The library works on Unix (mmap/mlock/mprotect) and should work on Windows too (VirtualAlloc/VirtualLock/VirtualProtect) — haven't tested Windows yet, but the API is the same and the platform-specific code is behind a clean abstraction layer.

## Verification

- 38 unit tests, 2 integration tests
- Property tests (proptest) for the crypto primitives
- Kani harnesses for bounded model checking of the memory management code
- 4 cargo-fuzz targets for the encryption and KDF paths

The repo also includes **sedit**, a TUI encrypted text editor that demonstrates the library — encrypted files on disk, plaintext only in LockedBuffers during editing.

The repo is at [github.com/zeta1999/rust-secure-memory-public](https://github.com/zeta1999/rust-secure-memory-public).
