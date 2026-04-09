---
layout: post
title: "LOB Reconstruction in Xμs"
---

[gpu-backtest](https://github.com/zeta1999/gpu-backtest-public) is a backtesting engine for HFT strategies. Still very much a work in progress, but the part worth talking about already is the limit order book (LOB) reconstruction — taking raw exchange feed data and maintaining a live order book state that a strategy can query — because getting this fast and correct is where most backtesters fall short.

## Why LOB Reconstruction Is Hard

An exchange market data feed gives you two kinds of messages: **snapshots** (the full book at a point in time) and **deltas** (individual changes — add, modify, cancel, trade). A snapshot might come once every few seconds. Between snapshots, you need to apply deltas in sequence to maintain the current book state.

The problems:
- **Volume**: A liquid instrument on Binance or CME can produce thousands of deltas per second
- **Correctness**: Missing or misordering a single delta corrupts the entire book state until the next snapshot
- **Latency**: If your backtest needs to evaluate a strategy at every book update, the LOB engine is on the hot path

Most Python-based backtesters sidestep this by working with aggregated OHLC bars. That's fine for swing strategies but useless if you want to test market making, stat arb, or anything that depends on queue position and spread dynamics.

## The Approach

The gpu-backtest LOB engine maintains a 200-level order book using a snapshot+delta reconstruction approach:

1. **Snapshot ingestion**: Parse the initial book state into a sorted price-level array
2. **Delta application**: Apply each delta (add/modify/cancel/trade) to the in-memory book
3. **Execution matching**: When a strategy generates an order, simulate fills against the reconstructed book with four timing modes (immediate, next-tick, time-priority, pro-rata)

Early benchmarks are looking promising — the reconstruction sits at around **5 microseconds per snapshot+delta cycle** for a 200-level book, and the full pipeline handles tens of thousands of events per second. Plenty of room to improve, but already fast enough to iterate on strategies without waiting around.

## The Aria DSL

Strategies are written in Aria, a small domain-specific language that compiles to a bytecode VM. The point of Aria is fast iteration — you can write, compile, and test a strategy variant without recompiling Rust code. Compilation is cheap enough that parameter sweeps and walk-forward optimization are practical at scale.

## Data Layer

Underneath, there's a DuckDB-backed data lake with columnar LRU buffers. Historical tick data goes in as Hive-partitioned JSONL (by exchange, instrument, date), and the LOB engine pulls from the columnar cache during backtests. This means you can run backtests against months of tick data without holding it all in memory.

The repo is at [github.com/zeta1999/gpu-backtest-public](https://github.com/zeta1999/gpu-backtest-public). WIP — contributions and feedback welcome.
