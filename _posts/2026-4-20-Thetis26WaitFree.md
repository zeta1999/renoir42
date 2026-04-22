---
layout: post
title: "Thetis26: a round of BSD / embedded / Docker fixes (updated)"
---

[Thetis26](https://github.com/zeta1999/Thetis26-public) is a C/C++ library built around wait-free data structures and a thin event loop on top — targeting Linux (epoll), macOS / BSD (kqueue), DPDK, ESP32 and a couple of FreeRTOS boards. The wait-free label applies to the data-structure primitives (ring buffers, hashmap, skip list, STM), not the event loop itself.

Field report on a round of fixes that moved the library from "compiles everywhere" to "passes the test suite on more of those targets."

## kqueue + UDP: don't register EVFILT_WRITE

The macOS UDP benchmark would stop receiving under load. Cause: the socket had both `EVFILT_READ` and `EVFILT_WRITE` registered, and kqueue's level-triggered write events fire continuously on a writable UDP socket — they starve the read filter in the event loop's drain loop.

The comment in `include/thetis/net/detail/kqueue_backend.hpp:60-64` now records exactly this: register `EVFILT_READ` only on kqueue, add `EVFILT_WRITE` on demand via `enable_write()`. UDP doesn't meaningfully backpressure at the kernel anyway, so `sendto` direct and never wait on writability.

## FreeBSD loopback: `connect()` returning 0 immediately

On Linux and macOS the async connect path returns `EINPROGRESS` and the event loop arms a completion on writable. On FreeBSD loopback, `connect()` can return `0` directly — the socket is already connected and no writability notification will arrive. The old code registered, waited forever, and timed out.

Fix in `event_loop.hpp` (the "Connected immediately (common on FreeBSD loopback)" branch): register the fd as a live connection and fire the completion synchronously. While patching that, also filled in a `post_connect_rearm` stub on backends that didn't define it — a linker error on the ESP32 build was the first sign.

## ESP32 in QEMU vs real hardware

The ESP32 port runs through `idf.py` on real hardware and through the Espressif QEMU fork in CI. QEMU has a few quirks that fail tests that pass on silicon:

- 64-bit atomics on a 32-bit target trip a FreeRTOS assert under QEMU. Real hardware is fine. QEMU-skip.
- `dense_hashmap_v6` runtime tests — same FreeRTOS assert shape. QEMU-skip.
- `ordered_dict_scaling` on sanitizer builds blows the test deadline — divided iteration count by a `kSanitizerDiv` (5, 10, or 1 depending on build).

The test harness now distinguishes "backend genuinely doesn't support this" (explicit capability check) from "QEMU lies" (tagged skip with a reason). The latter shouldn't silently translate to "feature broken on ESP32" in CI output.

## Ring-buffer scaling under QEMU

Same shape as the sanitizer issue — the ring-buffer scaling benchmark pushes enough ops to starve a QEMU scheduler. `kRingBufferBSDDiv` (and `kSanitizerDiv` in `test_ring_buffer_scaling.cpp`) drops the iteration count for those environments. The test still exercises the contention paths; it stops trying to emulate a 10M-op/sec firehose inside QEMU.

## Docker on Apple Silicon

The AFL++ and DPDK sanitizer images are x86_64-only. Running them without a platform flag on an M4 gives the unhelpful `exec format error`. Both dockerfiles now pin `FROM --platform=linux/amd64 …` (see `docker/Dockerfile.fuzz` and `docker/Dockerfile.dpdk`); Rosetta picks them up and they run — slow but correct.

## Takeaway

Wait-free data structures make the hot path fast. The boring work is making the event loop behave identically across kernels that look superficially similar. kqueue isn't epoll isn't io_uring isn't LwIP — each has a specific shape of "what it delivers, in what order, under what conditions" you can't read out of the man page.

Code @ [github.com/zeta1999/Thetis26-public](https://github.com/zeta1999/Thetis26-public).
