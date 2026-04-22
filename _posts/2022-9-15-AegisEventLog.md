---
layout: post
title: "Aegis — a signed, hash-chained event log"
---

Short note on Aegis, a small event-logging service I built Jul–Sep 2022 (private repo, under the LaplaceKorea org). ~2 months, ~24 R-B commits. Closed source.

## What it did

Append-only event log with cryptographic integrity guarantees:

- **Signed events**. Each event carried an Ed25519 signature over `(prev_hash, payload, timestamp)`. `signevent` in the commit log.
- **Hash chain**. `cryptohashlogevent` — each entry's hash fed into the next entry's signature input, so tampering with any historical event invalidated every subsequent signature. Standard "certificate transparency"-shaped design at the log level.
- **Timestamps with external proof**. The `get timestamp per hash` commit is about producing a signed-timestamp artefact that could be re-verified by a third party — useful for compliance evidence that an event existed at a given time.
- **Per-event encryption**. AES-GCM payload encryption with a key wrapped under the client's public key. `encrypt,decrypt, yet to test` — the "yet to test" honesty captured the state at that point.
- **Client init**. `InitClient`, `test client config` — the boring on-boarding plumbing (key generation, initial chain anchor, ACL seed).

## What I'd redo now

Three years and several crypto-library projects later:

- Drop the bespoke encryption layer. The wrapper I wrote around AES-GCM is exactly the sort of thing [rust-secure-memory]({% post_url 2026-3-30-PostQuantumSecureMemory %}) later got right by not reinventing. XChaCha20-Poly1305 with a session key, or a proper envelope with HPKE, is the modern shape.
- Add a post-quantum signature option. In 2022 ML-DSA wasn't finalised; in 2024 it was. A design that slots ML-DSA in as a second signature over the same hash chain would cost roughly one afternoon and buy harvest-now-forge-later resistance.
- The hash-chain-and-signed-timestamp design itself has aged well. That part I'd keep.

Closed source.
