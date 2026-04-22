---
layout: post
title: "Aria-HDL update: Leiserson-Saxe retiming, constraint annotations, PCIe BAR test (updated)"
---

Follow-up to the [Aria-HDL overview]({% post_url 2026-4-12-FPGAMetaCompiler %}) on [fpga-meta-compiler-public](https://github.com/zeta1999/fpga-meta-compiler-public). Short post on what actually landed since that one. Repo still WIP; so are several of these pieces.

## Leiserson-Saxe retiming

`src/passes/leiserson_saxe.rs` now implements FEAS + OPT against a proper W / D matrix build (modified Floyd-Warshall). FEAS checks whether a target clock period is achievable and returns the retiming; OPT binary-searches over the distinct values in D.

There's a physical rewrite step that takes the retiming solution and produces a plain-IR program, rather than annotating nodes and making every downstream backend re-interpret the annotations. Composes cleanly with the rest of the passes that way.

The default retimer is still the simpler `retiming.rs`. Switching LS in as the default is a follow-up item.

## PCIe BAR integration test

`tests/integration.rs` has a `test_pcie_mac_roundtrip` that compiles `examples/pcie_mac.ahdl`, instantiates `pcie_emu::PcieEmulator`, writes operands through the emulated BAR, ticks the IR emulator, and reads the MAC result back. Same integration file also has a `test_verilator_vs_emulator_differential` that compiles a module to Verilog, runs it under real Verilator, runs it through the IR emulator, and asserts the outputs match cycle-for-cycle (skipped when Verilator isn't on PATH).

This is useful but I want to be clear it is a test, not a replacement for real FPGA bringup. It does prove the compiler's own IR emulator matches Verilator on at least one mixed combinational + sequential design, which is what caught a width-inference bug where the compiler had been sign-extending on unsigned binary operators.

## Resource annotations

The parser grew a general `@annotation(args)` syntax. The resource pass recognises `@max_luts(N)`, `@max_regs(N)`, `@max_dsp(N)`, `@max_bram(N)`, `@max_error(e)`, and an `@impl` hint for target-board selection. `.arsc` files overlay the same caps per-board; the merge picks the tighter value when the same cap appears in more than one place.

`@max_error` interacts with the approx engine: if you put it on a sigmoid or tanh, the engine picks between piecewise-linear / LUT / polynomial alternatives to fit both the resource budget and the error bound. Before this, the choice was manual.

## ONNX import — partial on purpose

This is the bit I want to be most careful about, because the earlier draft of this post oversold it.

`src/onnx_import.rs` covers a narrow subset of ONNX ops — MatMul, Conv (1D, approximated as MAC), Relu, Sigmoid, Tanh, Linear, MaxPool, Quantize. Not a general ONNX-to-FPGA pipeline. There are two input paths: a hand-written protobuf reader that ingests a real `.onnx` binary for the supported op types, and a simplified textual descriptor (`.onnx-text`) used by the MNIST example. Anything outside the supported op set is rejected rather than silently miscompiled.

So: useful for specific MNIST-class demos today, not a drop-in for arbitrary models. The claim in the earlier draft that "an ONNX model becomes a candidate for FPGA synthesis with a single CLI invocation" only holds if the model is in the supported op subset.

## VHDL export — sort of fixed

The VHDL emitter was the least mature of the emitters for a while. It now covers the IR expression set closely enough that the same examples that emit Verilog also round-trip through the VHDL path. "Sort of fixed" rather than "solid" — I've not driven it through NVC or GHDL to a completed simulation of a non-trivial design, so the claim is shape-correctness, not tool-validated.

## Width inference

`a + b` where `a : uint<8>` and `b : uint<4>` now returns `uint<9>` without needing an explicit cast. Unsigned add widens by one; signed follows the IEEE-style adapted rules. The compile-time error only fires when the inferred width conflicts with a user annotation. This replaces several `#[allow(width_mismatch)]`-style internal suppressions that were masking the sign/zero-extension bug above.

## Tests

`TESTING.md` now covers CLI invocations with dependency notes for macOS and Linux, plus a set of manual tests that walk through the external-tool integrations (Yosys, Verilator, SymbiYosys) — the automated suite can't exercise those without the binaries present.

Code @ [github.com/zeta1999/fpga-meta-compiler-public](https://github.com/zeta1999/fpga-meta-compiler-public).
