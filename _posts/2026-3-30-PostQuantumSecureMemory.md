---
layout: post
title: "rust-secure-memory: a memguard port with ML-KEM and Lean proofs (updated)"
---

[rust-secure-memory-public](https://github.com/zeta1999/rust-secure-memory-public) is a Rust port of Go's [memguard](https://github.com/awnumar/memguard) — same LockedBuffer / Enclave shape, with an ML-KEM-768 KEM and a set of Lean4 proofs added. Ships with `sedit`, a TUI encrypted text editor that exercises the library end-to-end.

## Core types

- `LockedBuffer` — heap allocation with guard pages (`PROT_NONE` on both sides), `mlock` / `VirtualLock`, canary sentinels checked constant-time on drop, wipe-on-drop via `zeroize`, `MADV_DONTDUMP` on Linux, and a `freeze` / `melt` toggle that flips the pages read-only via `mprotect` / `VirtualProtect`.
- `Enclave` — XChaCha20-Poly1305 container (256-bit key, 192-bit nonce). `open()` returns plaintext in a LockedBuffer; `seal()` wipes the buffer and keeps only ciphertext.
- `Stream` — chunked encrypted reader/writer for data that doesn't fit in a single buffer.
- `kem::KemKeyPair` — ML-KEM-768 (FIPS 203, finalized Aug 2024). Shared secrets land in a LockedBuffer on both sides.

## KDF

`derive_key_combined` chains Argon2id (memory-hard, default 64 MiB / 3 iterations) into a sequential SHA3-256 stretch. The code is honest about what the stretch is — from `crypto.rs`:

> Not a true VDF (no efficient verification), but provides provable sequential work.

So: it's a sequential hash chain that adds wall-clock cost an attacker can't parallelise per passphrase trial. Calling it a VDF is aspirational.

## Platforms

Unix (`mmap` / `mlock` / `mprotect` / `madvise`) and Windows (`VirtualAlloc` / `VirtualLock` / `VirtualProtect`) are both fully implemented in `platform.rs`. `scripts/build-all.sh` cross-compiles for Linux x86_64/ARM64, macOS ARM64, and Windows x86_64/ARM64. I've exercised Unix regularly; Windows compiles clean but I haven't put the binary through real usage — `MADV_DONTDUMP` has no Windows equivalent, which is the one platform gap I'd call out.

## Verification

- 38 unit tests + 1 editor integration test (encrypted file roundtrip).
- `proptest` covers the crypto and KDF primitives (roundtrip, determinism, size bounds, KEM implicit rejection).
- 7 Kani harnesses across `crypto.rs`, `kem.rs`, `platform.rs` — input validation on encrypt/decrypt, encapsulate/decapsulate, and `round_up`.
- 5 `cargo fuzz` targets: `fuzz_buffer_ops`, `fuzz_enclave`, `fuzz_encrypt_decrypt`, `fuzz_kdf`, `fuzz_kem`.
- Lean4 proofs in `proofs/lean4/SecureMemory/` — Composition, ImplicitRejection, KeySeparation, Buffer, Primitives. The Lean side is about the KEM+AEAD composition and buffer/purge semantics at a spec level, not a line-by-line proof of the Rust.

`scripts/ci.sh` runs fmt/clippy/test; `scripts/miri.sh` wires up Miri for UB detection (nightly).

## sedit

The editor is the honest consumer of the library: decrypt → plaintext only ever lives inside a LockedBuffer while the buffer is edited → re-encrypt on save. File format (v2): 8-byte magic, 16-byte per-file salt, 24-byte nonce, ciphertext, 16-byte Poly1305 tag. Argon2id (64 MiB, 3 iter) then 1000× SHA3-256 stretching. v1 files (fixed salt) are still readable.

## What I'd do next

- Actually exercise the Windows binary on a Windows box — CI builds it, I don't run it.
- Add a Kani harness for the freeze/melt state machine on the buffer.
- Extend the Lean spec with a model of the guard-page invariant, not just the crypto composition.
- A `hybrid` KEM wrapper: X25519 ⊕ ML-KEM-768 so a classical break doesn't sink the shared secret.

Code @ [github.com/zeta1999/rust-secure-memory-public](https://github.com/zeta1999/rust-secure-memory-public).
