---
layout: post
title: "Aria-HDL: a single .ahdl source through ten backends (updated, WIP)"
---

[fpga-meta-compiler-public](https://github.com/zeta1999/fpga-meta-compiler-public) is an HDL and Rust compiler called Aria-HDL. One `.ahdl` source, ten backends: Verilog, VHDL, Verilator bundle, SymbiYosys/SVA, Lean 4 proof obligations, CUDA, Metal, OpenCL, a JSON resource report, and an IR dump. Marked WIP in the README and here, honestly.

The name overlap with the [gpu-backtest Aria DSL]({% post_url 2026-4-12-AriaStrategyDSL %}) is the `|>` pipe token and the `let` / `var` binding shape. They're separate compilers with separate frontends; don't read "same language."

## Backends

From `src/main.rs` CLI flags and the per-backend files in `src/`:

| Backend | Flag | File |
|---|---|---|
| Verilog | `--emit-verilog` | `verilog.rs` (+ `verilog_lint.rs`) |
| VHDL | `--emit-vhdl` | `vhdl.rs` |
| Verilator bundle | `--emit-verilator` | `verilator.rs`, `verilator_driver.rs` |
| SVA/PSL + SymbiYosys | `--emit-sby` | `formal.rs`, `formal_aware.rs` |
| Lean 4 | `--emit-lean4` | `lean4.rs` |
| CUDA | `--emit-cuda` | `gpu.rs` |
| Metal | `--emit-metal` | `metal.rs` |
| OpenCL | `--emit-opencl` | `opencl.rs` |
| Resource report (JSON) | `--resource-report` | `resource.rs` |
| IR dump | `--emit-ir` | `ir/pretty.rs` |

## DSL surface (verified in `token.rs` / `parser.rs`)

- Hardware primitives: `module`, `pipeline` (with `|>` stage boundaries), `systolic`, `stream<T>`, `fifo<depth=N>`, `approx()`.
- Types: `int<N>`, `uint<N>`, `fix<W,I>`, `fp32/fp16/bf16/fp8e4m3`, `bits<N>`, `struct`, `enum` (tagged unions), `mx<T,N>` microscaling blocks.
- Bindings: `let` (combinational) and `var` (register).
- Inline formal: `assert`, `assume`, `cover`, `always`, `never`, `eventually` are keywords in the token table — not comments or external annotations.

Most of that compiles through `lowering.rs` into the IR under `src/ir/` (`expr`, `module`, `types`, `formal`) before any backend sees it.

## Passes

Under `src/passes/`:

- `retiming.rs` — move registers across combinational logic to balance stages. The scaffolding for Leiserson-Saxe is in `leiserson_saxe.rs` but most pass calls go through the plainer retiming for now.
- `c_slow.rs` — trade latency for throughput by stage insertion.
- `cdc.rs` — clock-domain-crossing enforcement. This one actually returns a hard `Err` (see `cdc.rs:102`, "errors" vector pushed and surfaced) rather than a warning; unregistered crossings fail the compile.
- `balance.rs` — equalise latency across pipeline paths.

## Examples that currently parse and emit

`examples/` contains: `alu.ahdl`, `blinker.ahdl`, `constrained_mac.ahdl`, `cpu8051.ahdl`, `crypto.ahdl`, `dnn_inference.ahdl`, `dsl_showcase.ahdl`, `pcie_mac.ahdl`, `pipeline_demo.ahdl`, `resource_demo.ahdl`, `sigmoid_lut.ahdl`, `tcp_ip.ahdl`, plus `ecp5_85k.arsc` (device resource file) and an ONNX-text MNIST classifier used by `onnx_import.rs`. So the use-case families — systolic MAC, networking offload, small CPU core, activation LUT, crypto unit, ONNX import — have at least one worked example each.

## GPU semantic emulation, honestly

The CUDA / Metal / OpenCL emitters are faster-than-RTL functional emulation of the compiled IR. It's not a cycle-accurate replacement for Verilator — it's a "does the semantics still do what I meant" check before you start a multi-hour Yosys or Vivado run. Useful for design-space sweeps (e.g. varying a PE grid) where you're burning iterations on the wrong thing.

## What's real vs thin vs aspirational — being honest

The repo is WIP and so are several of the features. Ranking roughly by maturity:

- **Solid**: frontend (lexer, parser, typecheck, lowering), the IR under `src/ir/`, the CDC pass returning real `Err`, the example `.ahdl` files parsing and type-checking.
- **Working but not tooling-validated**: the Verilog and VHDL emitters produce output, and the Verilator bundle wires up a build flow — but I have **not** driven an example all the way through Yosys / Vivado / NVC to a bitstream or a finished sim run, so "synthesizable" is a claim about shape, not a claim I've watched a toolchain accept.
- **Thin**: `formal.rs` is a light SVA/PSL + SymbiYosys scaffold, not a fleshed-out proof backend. The Lean 4 emitter produces type and obligation skeletons; closed proofs are not shipped. The default retiming pass is the plain one; the Leiserson-Saxe implementation is in the tree but not on the default path.
- **Aspirational / skeletal**: `pcie_emu.rs`, `host_cosim.rs`, and the ONNX import path (which covers a partial op subset via a hand-written protobuf reader and a companion textual format) are starting points, not production pipelines. Don't read them as working end-to-end.

The resource-report JSON (`resource.rs`) is the one feature I'd describe as "substantial and probably accurate for the design shapes currently exercised" — LUT/register/DSP/BRAM estimates plus an approx-unit Pareto front. But I haven't cross-checked its numbers against a real Vivado `report_utilization` output for the same module, so treat the figures as self-consistent rather than vendor-validated.

## What I'd do next

- Drive one example all the way: `.ahdl` → Verilog → Yosys → bitstream on an ECP5, with a Verilator co-sim parity test in CI. That's the first honest "end-to-end" claim. Today the compiler emits all ten outputs but I haven't pinned a full toolchain journey.
- Promote a Lean 4 emitted obligation into a closed proof for one non-trivial module (e.g. the systolic MAC from `constrained_mac.ahdl`), to demonstrate the backend does more than scaffold types.
- Replace the default retimer with Leiserson-Saxe properly and benchmark against the current pass on the DNN example.
- Add a GPU-vs-Verilator parity test on `sigmoid_lut.ahdl` so the "semantic emulation" claim has teeth.

Code @ [github.com/zeta1999/fpga-meta-compiler-public](https://github.com/zeta1999/fpga-meta-compiler-public).
