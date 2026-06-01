---
layout: post
title: "Flow Master examples: a binary-only quant library you call from C, Python, Go — or the Aria DSL"
---

Two public example repos are up — [flow-master-examples](https://github.com/zeta1999/flow-master-examples) and [price-master-examples](https://github.com/zeta1999/price-master-examples) — showing how to actually consume the [Quant Masters]({% post_url 2026-5-6-PaganiniFlowMaster %}) products. Both ship under a clear licensing model: each product is available **binary-only** or with **full source**, and these repos demonstrate the binary-only seam. This post is mostly about Flow Master.

> **Serious quant analytics, under US$100k/yr per seat** (for smaller desks; it scales with usage). Flow Master gives you the microstructure, market-impact and market-making primitives as a single compiled library you call from C, C++, Python, Go, the shell, or a strategy DSL — a far cheaper seam than a six-figure data-platform license if what you need is the *analytics*, not a tick database.

## The contract: examples, not source

The important design choice, straight from `flow-master-examples/README.md`:

> it contains *no Paganini source and no Paganini binaries*. Every example links against the compiled, stable C ABI (`libpaganini`) or drives the `paganini` CLI. You bring a binary build of Paganini; these examples show how to call it.

That's the whole point of a binary-only license: the library stays proprietary, but it's trivially consumable from C, C++, Python, Go, and the shell. The example code itself is CC0 — copy it into your stack freely. Paganini's source is what the full-source license is for.

## What the binary seam exposes

Flow Master's source-free consumption seam is a stable **C ABI** — shipped as `libpaganini.{a,dylib,so}` plus `paganini.h` — and the `paganini` CLI. The ABI exports a small, versioned set of functions, each wrapping a core Paganini algorithm. A couple of them have Lean correctness specs alongside (microprice and Welford are `sorry`-free; the Avellaneda–Stoikov spec rests on one explicit `axiom`), while the rest — BOCPD, MASS, Kyle's λ — are tested but not formally specified. See the [numerical-algorithms note]({% post_url 2026-5-30-NumericalAlgorithmsJL %}) on what "verified" actually means here:

| C function | Algorithm |
|---|---|
| `paganini_abi_version()` | ABI probe (currently `1`) |
| `paganini_as_quote(...)` | Avellaneda–Stoikov optimal maker quote |
| `paganini_microprice(...)` | size-weighted fair value, provably within `[bid, ask]` |
| `paganini_sample_variance(...)` | Welford online (Bessel-corrected) variance |
| `paganini_bocpd_changepoints(...)` | Bayesian online change-point detection (Adams–MacKay) |
| `paganini_mass_profile(...)` / `…_best_match(...)` | MASS z-normalised distance profile / nearest subsequence |
| `paganini_kyle_lambda(...)` | Kyle's λ price-impact calibration (RLS) |

## What it looks like from C

No Rust, no library headers required — you re-declare the entry points and link `libpaganini` (`examples/c/consumer.c`):

```c
extern int32_t paganini_as_quote(double mid, double inventory, double gamma,
                                 double k, double sigma, double time_left,
                                 double *out_bid, double *out_ask);
extern double  paganini_microprice(double bid, double ask,
                                   double bid_qty, double ask_qty);

int main(void) {
    printf("paganini ABI version: %u\n", paganini_abi_version());

    /* Avellaneda-Stoikov: optimal symmetric quote around a 100.00 mid. */
    double bid = 0.0, ask = 0.0;
    paganini_as_quote(100.0, 0.0, 0.1, 1.5, 0.2, 1.0, &bid, &ask);
    printf("AS quote bid=%.4f ask=%.4f spread=%.4f\n", bid, ask, ask - bid);

    /* Microprice: heavier ask size pulls fair value toward the bid. */
    double mp = paganini_microprice(99.0, 101.0, /*bid_qty*/5.0, /*ask_qty*/15.0);
    printf("microprice=%.4f\n", mp);
}
```

The same call is walked through in C++, Python (ctypes), Go (cgo), and the CLI, with a `scripts/run_all.sh` gate that builds and runs every example and asserts its output.

## Worked examples, in whatever language you bring

The repo isn't just "hello, ABI" — each primitive gets a runnable case in the language that fits it:

- **Market-impact estimate** (`examples/impact/`, Go + cgo) — calibrate **Kyle's λ**, the linear price-impact coefficient, from a trade tape by online recursive least squares. The demo synthesises 10 trades with a known `λ = 0.05` and recovers `0.0499`.
- **Regime detection** (`examples/regime/`, Python + ctypes) — **BOCPD** (Bayesian Online Change-Point Detection, Adams–MacKay 2007) over a return series; the change-point mass spikes at the level shift (`peak_index=24` on a series that breaks at 24).
- **Time-series similarity** (`examples/tss/`, plain C) — **MASS** (Mueen's Algorithm for Similarity Search), an FFT-accelerated z-normalised distance profile, to find the nearest subsequence of a query pattern in a longer series. (This is what the `paganini-tss` crate exposes — "TSS" = time-series similarity, not storage.)
- **MM/LP strategy skeleton** (`examples/strategy/`, C) — a minimal market-making loop built on the C ABI (Avellaneda–Stoikov quoting around a microprice fair value, inventory tracked), the skeleton you'd extend into a real strategy. Its PnL is negative by construction — it's a wiring demo, not a backtest result, and `run_all.sh` only asserts the tick/fill shape.

So the exposed primitives already cover the basics of a market-making / liquidity-provision stack: fair value (microprice), quoting (Avellaneda–Stoikov), impact (Kyle's λ), regime (BOCPD), and analog search (MASS) — each callable from any of the five languages.

## Into the backtester: the Aria DSL and typed-registry plugins

Beyond linking the ABI directly, Flow Master plugs into the [gpu-backtest]({% post_url 2026-3-25-LOBReconstructionGPU %}) engine two ways — both still binary-only, both over the *same* `libpaganini` C ABI (this is documented in Paganini's `docs/ARIA_PAGANINI_PLUGIN.md`):

- **Path A — the Aria strategy DSL** (`examples/plugin-aria-dsl/`). A `.aria` strategy calls Paganini *by name*: `signal fair = pag_microprice()`, `pag_bs_price(...)`, `pag_sabr_iv(...)`, `pag_iv_schadner(...)`. With `bt-dsl --features paganini` the compiler/VM resolve those `pag_*` calls to ABI entry points; with the feature off, nothing Paganini links and gpu-backtest's own test suite is untouched. (This is gpu-backtest's [Aria *strategy* DSL]({% post_url 2026-4-12-AriaStrategyDSL %}) — not to be confused with Price Master's Aria *payoff* language.)
- **Path B — a typed-registry plugin** (`examples/plugin-typed-real/`). Real Paganini quants register in gpu-backtest's `TypedRegistry` as a `QuantPlugin` whose `predict()` calls `paganini_bridge_run_quant("paganini::bs_price", …)` — the whole bridge registry is reachable by name over the ABI. A third example wires the *same* mechanism with your own model and no Paganini at all.

The invariant across every path: Paganini's source never leaves its repo. The consumer declares the C ABI `extern "C"` and links the compiled library — which is exactly what makes a binary-only license practical.

## Honest about the surface

The binary seam is deliberately narrower than the library. A much larger algorithm surface — the vol toolkit, the surface parameterizations, the options-MM quoters from the [Flow Master post]({% post_url 2026-5-6-PaganiniFlowMaster %}) — exists inside Paganini but isn't reachable through the C ABI yet; `NOT_YET_EXPOSED.md` is the backlog, and examples land as the wrappers do. So treat these repos as a growing demonstration of the consumption model, not the full feature list.

## Price Master, briefly

[price-master-examples](https://github.com/zeta1999/price-master-examples) works differently because [Price Master]({% post_url 2026-5-13-PriceMasterAria %}) is driven by Aria payoff scripts rather than a C ABI. It pairs `.aria` payoff files (Korean autocall, snowball, TARN, cliquet, CDS, range accrual) with JSON inputs across categories — exotics, models (Heston closed-form, rough-vol smile), and time-series — each a runnable case with expected output. Same licensing: binary-only or full source.

If you want a build to run these against, or the full-source terms: [renoir42@renoir42.com](mailto:renoir42@renoir42.com?subject=Quant%20Masters).
