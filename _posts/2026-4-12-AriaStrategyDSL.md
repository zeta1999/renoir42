---
layout: post
title: "Designing a Strategy DSL: the Aria Language in gpu-backtest"
---

In a [previous post]({% post_url 2026-3-25-LOBReconstructionGPU %}) I covered the LOB reconstruction side of [gpu-backtest](https://github.com/zeta1999/gpu-backtest-public). This post is about the other half — the Aria strategy language and why I built a bytecode compiler for it.

## The Problem With Hardcoded Strategies

Most backtesting engines make you write strategies in the host language (Python, C++, Rust). This means every strategy variant requires a recompile. If you want to sweep 10,000 parameter combinations across 5 strategy variants with walk-forward optimization, you're recompiling 50,000 times. That's not practical.

The alternative is an interpreted DSL, but interpretation is too slow for GPU execution and adds unpredictable latency on the hot path.

Aria splits the difference: a compiled DSL that produces bytecode, executed by a register-based virtual machine that runs identically on CPU, CUDA, OpenCL, and Metal.

## The Language

Aria is small by design. A momentum strategy looks like:

```
let fast = sma(close, 10)
let slow = sma(close, 50)

if cross_up(fast, slow) then buy(1.0)
if cross_down(fast, slow) then sell(1.0)
```

Built-in functions: `sma`, `ema`, `cross_up`, `cross_down`, `run_max`, `run_min`, `stdev`, `zscore`, `percentile`. Order actions: `buy(qty)`, `sell(qty)`, `flatten()`, `limit_buy()`, `limit_sell()`, `vwap()`. The pipe operator (`|>`) composes transformations left-to-right. Variables are mutable with assignment semantics for state across ticks.

## The Compiler

The compilation pipeline:

1. **Lexer** — tokenizes Aria source
2. **Parser** — produces an AST
3. **Compiler** — emits a 44-opcode bytecode program

The 44 opcodes cover arithmetic, comparison, logical operators, built-in function calls, order actions, control flow, and state management. The bytecode is compact — a typical strategy compiles in microseconds.

## The VM

The register machine executes bytecode against live LOB state. Each tick, the VM receives the current book snapshot (bid/ask/mid/spread/volume) and evaluates the strategy program. If the program emits an order action, it's passed to the OMS (order management system) for simulated execution against the reconstructed order book.

The same VM code compiles for CPU (native Rust), CUDA, OpenCL, and Metal. On GPU, each thread runs an independent VM instance with different parameters — this is how you parallelize parameter sweeps. A grid search across 10,000 parameter combinations runs as 10,000 concurrent VM instances on the GPU.

## Why Bytecode Instead of Interpretation

Three reasons:

1. **GPU compatibility**: you can't interpret on a GPU shader. Bytecode maps directly to a fixed instruction dispatch loop that GPU compilers can optimize.
2. **Determinism**: the VM executes the same number of instructions for the same input regardless of platform. No JIT warmup, no branch prediction variation.
3. **Compilation speed**: 164K programs/sec on an M4. Cheap enough that Bayesian optimization can recompile strategies on every iteration.

## Walk-Forward Optimization

The optimizer supports grid search, random search, and Bayesian parameter optimization with walk-forward validation. Each scenario runs the full pipeline: data loading, LOB reconstruction, strategy execution, PnL computation. Results are reduced on GPU (Sharpe ratio, max drawdown, total PnL) and collected on the host.

The Aria DSL makes this practical because strategy variants are data (bytecode programs), not code (compiled binaries). The optimizer generates, compiles, and evaluates thousands of variants without touching the Rust build system.

The repo is at [github.com/zeta1999/gpu-backtest-public](https://github.com/zeta1999/gpu-backtest-public).
