---
layout: post
title: "gpu-backtest: Monte Carlo scenarios, FX cross-rate costs, a few DSL ops (updated)"
---

Short update on [gpu-backtest-public](https://github.com/zeta1999/gpu-backtest-public). The optimizer grew a Monte Carlo scenario mode, FX cross-rate costs landed in the cost model, and the Aria DSL picked up a few opcodes.

## Monte Carlo scenarios in the optimizer

Walk-forward validation gives one realised Sharpe per parameter set, from one draw of history. Monte Carlo scenario search adds a distribution: each candidate parameter set runs against N synthetic paths plus the historical one, and the score is the distribution of Sharpe across scenarios.

Two pieces:

- A `synth-data` CLI subcommand on `bt-engine` that generates a synthetic universe into DuckDB. The model is a regime-switching Gaussian with fat-tail overrides — deliberately simple. `bt-engine synth-data <output.duckdb> [--tickers T1,T2] [--bars N] [--seed S]`.
- Monte Carlo runs wired through the optimizer. `demo_monte_carlo.rs` under `bt-engine/tests/` exercises the full meta-parameter-across-scenarios path; the config flags sit on the engine config (`num_scenarios`, etc.). Each (parameter set, scenario) pair is one GPU thread, so 10K parameters × 100 scenarios is one grid dispatch.

This is not a substitute for validated out-of-sample testing on real data. The point is to reject parameter sets that look great on the single historical draw but collapse under plausible resamplings of it.

## FX cross-rate and hedging cost

`bt-core/src/fx.rs` is new: a rate table with cross-rate derivation via a base currency and a hedging cost model. The previous cost model treated non-USD assets as if they lived in USD directly. With the cross-rate path wired through, a strategy that trades EUR-denominated assets and reports PnL in USD now pays the actual spread plus a configurable hedging overhead. Strategies that were long EUR through a USD-strength regime get scored honestly instead of picking up free FX carry.

## Aria DSL opcodes

The DSL bytecode grew `LinRegSlope` and `HurstExponent` (rolling linear-regression slope and Hurst exponent) as statistical ops, and new input slots `LastSide`, `BidVol`, `AskVol` for microstructure features. These are the ones the optimizer was asking for — the earlier DSL could express momentum and breakout but couldn't cleanly express persistence or order-flow-biased signals.

A library of TA-shaped strategies now ships under `strategies/`: bollinger (three variants), donchian, dual_thrust, ema_crossover, keltner, ma_ribbon, macd, mean_reversion, momentum, naive_vwap, rsi, trend_regime, volume_breakout, zscore_reversion, atr_breakout. Classical baselines — the point is having something to benchmark against before any ML variant shows up.

One real fix that landed alongside this: OHLC-based PnL was double-counting the close-to-next-open gap on overnight positions. The synthetic-data path surfaced it — on synthetic paths the bug produced Sharpe values that were too good, which is exactly the kind of cross-check Monte Carlo is for.

## OpenCL on Apple Silicon

`gpu/src/opencl/backend.cpp` now detects missing `cl_khr_fp64` and falls back to fp32 (`backend.cpp:231`). That's the state of the OpenCL runtime on macOS. Doesn't make the backend double-precision where the hardware doesn't support it; it just means the same OpenCL kernel source compiles and runs on M-series as well as on CUDA / Metal. I haven't independently benchmarked CUDA ↔ OpenCL ↔ Metal parity on a large backtest; treat parity as "the same Aria program compiles and runs" rather than "the outputs are bit-identical across vendors."

Code @ [github.com/zeta1999/gpu-backtest-public](https://github.com/zeta1999/gpu-backtest-public).
