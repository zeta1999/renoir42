---
layout: post
title: "gpu-backtest: LOB reconstruction, DSL VM, early numbers (updated)"
---

[gpu-backtest-public](https://github.com/zeta1999/gpu-backtest-public) is a Rust backtesting engine I've been building out for HFT-style strategies. Two commits in, still early, but the shape is there: nine crates, a DSL VM, an LOB engine, a DuckDB data layer, and GPU kernels (CUDA / OpenCL / Metal) for the VM and LOB replay. This post covers what's actually working and what's not, against the current tree.

## What's in

- `bt-core`: `FastDecimal` (i128, 10^18 scale), `LobState` using `BTreeMap<FastDecimal, FastDecimal>` for each side, order/fill/trade types.
- `bt-tick-emulator`: snapshot+delta LOB engine, an execution heuristic (`volume_multiplier`, `lob_fill_fraction`), linear/sqrt market-impact models, and a timing model with four modes: `FullSpeed`, `Realistic`, `LatencySensitivity`, `MinimumLatency`. These control when the *strategy* sees an event, not how fills match.
- `bt-dsl`: Aria — lexer, parser, 44-opcode bytecode, register-machine VM. Built-ins: `sma`, `ema`, `cross_up/down`, `stdev`, `zscore`, `percentile`, `run_min/max`, `exp`, `log`, `prev`.
- `bt-optimizer`: grid / random / Bayesian search, walk-forward scenarios, GPU batch dispatch, Pareto aggregation.
- `bt-data`: DuckDB-backed loader, columnar LRU chunk cache (`bt-data/src/columnar.rs`), ingesters for Bybit LOB JSON, BTC CSV, NYSE TAQ, Yahoo OHLC, XAUUSD MT5, plus a synthetic generator.
- `gpu/`: CUDA, OpenCL, Metal kernels for the VM, LOB replay, execution match, reductions (PnL, Sharpe, drawdown). Loaded at runtime via `libloading`, no GPU SDK required at compile time.

## Numbers

From `cargo bench` on Apple M4 (see `results/` and the README for the full table):

- LOB delta (1 update): 9.3 ns
- LOB best bid/ask: 18 ns
- LOB snapshot (200 levels): 6.1 μs
- LOB mixed stream, 1 snapshot + 100 deltas per cycle: 10.9 μs
- DSL VM step, full momentum strategy: 129 ns
- End-to-end HFT backtest, 573K LOB events + 54K trades: 12.7 s (≈ 49K events/sec)

The BTreeMap-based LOB amortizes well on deltas — updates dominate snapshots in a real feed, which is what you want.

## What isn't there yet

A few things the README (or the previous draft of this post) implies are stronger than they are:

- **Queue-position modeling is not implemented.** The execution heuristic caps fills at a fraction of top-of-book size (`lob_fill_fraction`) and a fraction of observed volume (`volume_multiplier`). That is fine for taker strategies; it isn't a market-making fill model.
- **Fill latency and no partial fills modulo size cap.** The timing model delays when the *strategy* sees a quote. It does not model the exchange-side latency of an order reaching the book, so there is no adverse-selection mechanic in the fills.
- **Data layout is DuckDB, not a Hive lake.** I described it that way before; there is no Hive partitioning in the ingesters, just DuckDB tables plus the LRU columnar cache on top.
- **GPU LOB kernels compile but the hot-path LOB numbers above are CPU.** Metal conformance has a test suite (`metal_conformance`, ignored by default). I haven't cross-checked the GPU LOB path against the CPU path on a large feed yet.

## What I'd add next, in rough order

1. A queue-position estimator inside `LobState` (track arrival order per level), wired through `execution.rs` so a limit order at the best price reports a realistic time-to-fill.
2. A second latency knob for order submission (strategy → book), separate from the current strategy-side `computation_offset_ns`.
3. Replace `BTreeMap<FastDecimal, FastDecimal>` with a flat tick-grid array keyed by an integer tick offset. The ticker metadata already has a tick size; this gets both sides to O(1) delta and contiguous memory for the price ladder.
4. A GPU-vs-CPU LOB parity test in CI that replays the same Bybit stream through both and diffs per-tick snapshots.
5. Turning `strategies/bollinger_wf.aria` plus `walk_forward_bollinger.toml` into an actual wired walk-forward example end-to-end with results checked in — the building blocks are there, the happy path is not demonstrated.

Code @ [github.com/zeta1999/gpu-backtest-public](https://github.com/zeta1999/gpu-backtest-public).
