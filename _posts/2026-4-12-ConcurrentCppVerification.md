---
layout: post
title: "Thetis26: which tool caught which bug (updated)"
---

Follow-on to the [Thetis26 overview post]({% post_url 2026-3-15-LockFreeDataStructures %}). This one is concrete: the `models/` tree in the repo is organised as a bug taxonomy (`BUG-1` through `BUG-13+`), each tagged to one or more specs, and it's a better way to talk about formal methods than abstract "what CBMC is good at" framings.

## What's actually in `models/`

- **TLA+** — 18 specs under `models/tla/`. Algorithm level. Hashmap probe/insert/resize/migrate, ring-buffer mode transition, skip-list linearizability, EBR allocator, STM retry/composability/OneFile, TLS stream FSM, WebSocket send-serialisation, io_uring partial write, epoll EPOLLOUT handling.
- **SPIN (Promela)** — 13 models. Liveness and progress properties: hash-map `insert_progress`, `bounded_patience`, `erase_migrate_race`, wait-free helping, TLS handshake, TLS BIO ordering, WS reconnect / send-serialisation, skip-list EBR unlink.
- **CBMC** — 14 bounded-model-check harnesses. Split between data-structure invariants (`size_invariant`, `slot_fsm`, `hash_bounds`) and protocol / crypto bounds (`aead_bounds`, `kem_bounds`, `sign_bounds`, `tls_record_bounds`, `ws_frame_bounds`, `http_upgrade_bounds`, `tcp_partial_write`, `iouring_partial_write`, `epoll_epollout_inv`, `secure_buffer_bounds`).
- **GenMC (RC11)** — 12 C11 memory-model stress tests covering CAS2 replacement, EBR safety and thread-id reuse, skip-list EBR retire, STM commit/retry ordering, ring-buffer v4 transition, hash-map insert/find/erase ordering, migration races.
- **Nidhugg (SC / x86-TSO / ARM)** — 4 files: `insert_arm`, `find_arm`, `erase_arm`, `cas2_replacement`. The specific purpose is catching bugs that x86-TSO hides and ARM exposes.
- **Lean 4** — 2 files: `SlotFsm.lean` (hashmap slot state machine), `StmOpacity.lean` (STM opacity).

## Bug taxonomy, with which tool caught which

Pulled directly from `models/README.md` — each `BUG-n` is a real entry with a fix commit behind it, not a retrospective narrative.

- **BUG-1** — hashmap insert probe livelock (two threads racing for an empty slot). Caught at spec level by `hashmap_insert.tla`. Fix cross-checked by `insert_ordering.c` under GenMC.
- **BUG-2** — hashmap counter non-linearizability. `hashmap_size_buggy.tla` exhibits the drift; `size_invariant.c` under CBMC confirms it on the actual C++.
- **BUG-3** — erase/resize deadlock (`hashmap_resize_v5_buggy.tla`).
- **BUG-10** — EBR bitmap thread-ID reuse. TLA+ at `ebr_thread_id.tla`; GenMC under C11 at `ebr_thread_id_reuse.c`. This is a case where the spec is fine but the C11 memory model allows an execution TLA+ can't see.
- **BUG-11** — skip-list EBR use-after-free. TLA+ for unlink and a separate linearizability spec, GenMC for `skiplist_ebr_retire.c`.
- **BUG-12** — `ring_buffer_v4` UNKNOWN→SPSC mode transition. A second producer interleaving during mode detection corrupts data if SPSC fast-path is already taken. TLA+ (`ring_buffer_v4_spsc_mpmc.tla`) checks `ModeMonotonic` and `NoDataRace`; SPIN (`ring_buffer_v4_mode.pml`) checks progress; GenMC (`ring_buffer_v4_transition.c`) checks the C11 realisation.
- **BUG-13** — `help_migrate` swings the current table when batches are *claimed* rather than *completed*, leaving ghost entries. TLA+ (`hashmap_migrate_done.tla`) catches it at spec level; **BUG-13 Race 3** specifically is the ARM64 IRIW-shape race where the fence is load-bearing, modelled by `hashmap_migrate_fence.tla`.

This is what "multiple tools" is really for: some bugs only exist in the C11 memory model (GenMC), some only exist on weak memory (Nidhugg), some are algorithmic and hardware-independent (TLA+), and some are implementation-level (CBMC).

## CBMC isn't just for data structures

The CBMC set is roughly half data-structure harnesses (`size_invariant`, `slot_fsm`, `hash_bounds`, `secure_buffer_bounds`) and half protocol/crypto bounds. `tls_record_bounds`, `ws_frame_bounds`, `http_upgrade_bounds`, `tcp_partial_write` all check input-length and state-machine invariants on the network path. `aead_bounds`, `kem_bounds`, `sign_bounds` cover the ML-KEM-768 / ML-DSA-65 / XChaCha20-Poly1305 wrapper layer — parameter-length preconditions, buffer sizes, and the places where a bad input could silently truncate. Not verified crypto — just defended edges around it.

## Specs-first, honestly

For the first few data structures I wrote the C++ then the TLA+. For the later ones I flipped the order. The later ones went faster. That's the lesson and it's not a novel one — it's re-learning that thinking before typing is cheap.

I'd avoid putting a number on "what % of time went to verification" — I don't have that number honestly, and retrofitting it makes the post dishonest.

## What I'd change

- Promote the two Lean files into a proper spine: pull `SlotFsm` up to match `hashmap_insert.tla` line for line, and extend `StmOpacity` to cover the Kogan-Petrank-style helping.
- A per-data-structure `bugs.md` that cross-references the code diff, the spec diff, and the CI job that would now catch a regression. Today that's implicit in the `BUG-n` tags; making it explicit would help a reviewer land a PR.
- Nidhugg coverage is four files; extend to cover the ring-buffer v4 transition and STM commit, which are currently only GenMC-checked.

Code @ [github.com/zeta1999/Thetis26-public](https://github.com/zeta1999/Thetis26-public).
