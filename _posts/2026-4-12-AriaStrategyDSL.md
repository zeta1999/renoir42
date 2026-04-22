---
layout: post
title: "Aria: a strategy DSL that compiles for CPU and GPU from the same bytecode (updated)"
---

Follow-on to the [gpu-backtest LOB post]({% post_url 2026-3-25-LOBReconstructionGPU %}). This one is about the DSL side — `bt-dsl`, with Aria programs compiled to a bytecode that the same VM runs on CPU, CUDA, OpenCL, and Metal.

## Why a DSL and not a Rust callback

The real reason is the GPU. A parameter sweep with 10K combinations wants 10K independent strategy executions in parallel. You can't run a Rust closure on a CUDA/Metal shader. You can run a fixed-size instruction dispatch loop over `Vec<Instruction>` with per-scenario register overrides, which is what the VM does.

A secondary reason is iteration speed — strategies are data files you swap in and out, not Rust code you recompile. But the parameter-sweep framing I used in an earlier draft was a strawman: sweeping a parameter does *not* recompile a host-language backtest either; it just calls the same function with a different argument. The real gain is running the sweep on a GPU, and for that you need a shader-executable representation.

## What Aria actually looks like

From `strategies/momentum.aria` in the repo:

```
signal sma_fast = sma(close, 10)
signal sma_slow = sma(close, 50)
signal trend_up   = cross_up(sma_fast, sma_slow)
signal trend_down = cross_down(sma_fast, sma_slow)

let position_size = 1.0

strategy momentum_crossover {
    when trend_up   -> buy(position_size)
    when trend_down -> sell(position_size)
}
```

`signal` is a reactive binding that gets recomputed each tick. `let` is a compile-time constant. `var x = 0.0` plus `x := x + 1.0` gives you mutable state across ticks. The `when cond -> action` arm is the only control construct; there's no `if`/`then`/`else`. That keeps the compiled program a straight line of conditional emits, which maps cleanly onto branchless GPU code.

## Built-ins

From `bt-dsl/src/bytecode.rs`:

- Temporal / rolling: `sma`, `ema`, `run_max`, `run_min`, `cross_up`, `cross_down`, `prev`.
- Statistical: `stdev`, `zscore`, `percentile`, `linreg_slope`, `hurst_exponent`.
- Scalar math: `abs`, `max`, `min`, `exp`, `log`.
- Order actions: `buy`, `sell`, `flatten`, `limit_buy`, `limit_sell`, `vwap`.

The pipe operator `|>` is lexed but is syntactic sugar for left-to-right composition; it doesn't introduce any new semantics.

## Compiler and bytecode

Pipeline under `bt-dsl/src/`: `lexer.rs` → `parser.rs` → `ast.rs` → `compiler.rs` → `bytecode.rs`. Output is a `CompiledProgram` with a `Vec<Instruction>`, a constants pool, and register/state slot counts. Each instruction is 10 bytes, `#[repr(C)]` for direct GPU upload.

The `Opcode` enum has 46 distinct variants grouped as: data loading (8), arithmetic (11), comparison (6), logic (3), branchless `Select` (1), temporal (6), order emission (5), statistical (5), and `Halt`. The repo README says "44 opcodes" — that figure predates `LinRegSlope` and `HurstExponent` being added; take the count from the enum, not the README.

One detail worth calling out: `vwap(qty, duration)` is parsed and type-checked like any other order action, but `compiler.rs:180` lowers it to a plain `EmitBuy`. The bytecode VM is not responsible for time-slicing; the OMS in `bt-oms` does the VWAP scheduling using the duration as a target horizon. At the VM level, VWAP is a buy.

Compilation is fast enough to be a non-issue: `compile_full_strategy` runs in 6.1 μs in `cargo bench` (Apple M4). If you wanted to JIT a new program per walk-forward window, you could.

## VM

`vm_cpu.rs` for the CPU path (native Rust, straight instruction dispatch loop). `vm_gpu.rs` marshals the same `CompiledProgram` into a buffer that CUDA / OpenCL / Metal kernels (`gpu/src/{cuda,opencl,metal}/vm_kernel.*`) execute per thread.

For parameter sweeps, `bt-optimizer/src/batch.rs` packs N parameter sets as a flat SoA buffer indexed `overrides[param_idx * num_scenarios + scenario_idx]`. Each GPU thread reads its scenario's overrides into the VM's input registers before the dispatch loop starts. One bytecode program, N parameter vectors, no recompilation.

## What I'd do next

- A small type-checker pass between parser and compiler. Today you can write `stdev(cross_up(...), 20)` and get a compiled program that produces nonsense; the compiler doesn't distinguish series-valued from boolean-valued signals.
- A bytecode disassembler that prints opcodes alongside the source line that produced them, to make optimisation passes debuggable.
- Constant-folding and dead-signal elimination. Today, an unreferenced signal still emits its bytecode.
- A GPU-vs-CPU VM parity test in CI that runs the same program on both paths over a shared fixture and diffs per-tick register values.

Code @ [github.com/zeta1999/gpu-backtest-public](https://github.com/zeta1999/gpu-backtest-public).
