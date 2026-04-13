---
layout: post
title: "From DSL to Silicon: Building an FPGA Meta-Compiler in Rust"
---

[fpga-meta-compiler](https://github.com/zeta1999/fpga-meta-compiler-public) is a hardware description language and compiler that generates synthesizable RTL, formal verification proofs, and GPU emulation kernels from a single source. The language is called Aria-HDL, and it shares DNA with the Aria DSL used in [gpu-backtest](https://github.com/zeta1999/gpu-backtest-public) and the Pricer Master derivatives pricing library — the same pipe operator, the same composability principles, but targeting silicon instead of software.

## Why Another HDL

Verilog and VHDL are fine for writing hardware, but they give you no help with verification, no path to GPU emulation for fast iteration, and no machine-readable resource estimates. If you want all three, you end up maintaining separate codebases that inevitably drift apart.

Aria-HDL compiles a single `.ahdl` source into 10 different backends:

```
                        +---> Verilog ---> Yosys / Vivado ---> bitstream
                        +---> VHDL ---> NVC / GHDL ---> simulation
Aria-HDL source (.ahdl) +---> Verilator bundle ---> C++ co-simulation
         |              +---> SVA / PSL + .sby ---> SymbiYosys ---> proofs
         v              +---> Lean 4 ---> proof obligations
     Compiler IR        +---> CUDA kernel ---> GPU semantic emulation
         |              +---> Metal shader ---> Apple Silicon emulation
         |              +---> OpenCL kernel ---> cross-vendor emulation
         |              +---> JSON resource report ---> design space exploration
         |
         +---> Optimization: retiming, C-slowing, CDC enforcement
```

Write once, verify formally, emulate on GPU, synthesize to FPGA. That's the pitch.

## The Language

Aria-HDL is declarative — you describe what the hardware does, not how to wire gates. A few key features:

- **`let` / `var`**: immutable bindings for combinational logic, `var` for registers (state)
- **Pipe operator (`|>`)**: natural left-to-right dataflow for pipelines
- **Rich type system**: arbitrary-width integers (`int<N>`, `uint<N>`), fixed-point (`fix<W,I>`), IEEE floats (fp32 down to fp8), microscaling blocks (`mx<T,N>`), structs, tagged unions
- **Hardware primitives**: `module`, `pipeline` with stage boundaries, `systolic` arrays with PE grids, `stream<T>` with flow control, `fifo<depth=N>` with CDC support
- **Inline formal verification**: `assert`, `assume`, `cover`, `always`, `never`, `eventually` as first-class constructs — not bolted on after the fact

## Formal Verification

This is where it gets interesting. The compiler emits two kinds of verification output:

1. **SVA/PSL assertions** with SymbiYosys configuration files — bounded model checking of the synthesizable RTL
2. **Lean 4 proof obligations** — type-checked proof skeletons with struct and enum definitions matching the hardware types

The formal backend isn't an afterthought. CDC (clock domain crossing) violations are hard errors — the compiler refuses to emit RTL if you have an unregistered clock domain crossing. Pipeline latency mismatches are caught at compile time.

## GPU Emulation

Before committing to a synthesis run (which can take hours on a complex design), you can emulate the design semantically on CUDA, Metal, or OpenCL. The GPU kernels mirror the hardware behavior cycle-by-cycle, so you can validate functional correctness at GPU speed rather than RTL simulation speed.

This is particularly useful for systolic array designs (DNN inference, matrix multiply) where the design space is large and you want to sweep parameters before committing to a bitstream.

## Compiler Passes

The compiler IR goes through several optimization passes:
- **Retiming**: move registers across combinational logic to balance pipeline stages
- **C-slowing**: trade latency for throughput by inserting pipeline stages
- **CDC enforcement**: hard error on unregistered domain crossings
- **Pipeline balancing**: ensure all pipeline paths have equal latency
- **Resource estimation**: JSON output with LUT/register/DSP/BRAM counts and Pareto-optimal alternatives for `approx()` blocks

## Use Cases

The examples in the repo cover:
- **Systolic arrays** for DNN inference (INT8 MACs)
- **Cryptographic accelerators** (modular arithmetic with formal proofs)
- **Networking offload** (TCP checksum with state machines)
- **Activation functions** (piecewise linear approximation via `approx()`)

The repo is at [github.com/zeta1999/fpga-meta-compiler-public](https://github.com/zeta1999/fpga-meta-compiler-public). Still WIP but the core pipeline works end-to-end.
