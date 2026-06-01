---
layout: post
title: "Paganini: the buy-side vol toolkit behind Flow Master"
---

Paganini is the Rust quantlib I use for market making, microstructure, and options dealer flow. **Flow Master** is the commercial packaging of it — the buy-side half of Quant Masters. This post is about what's actually in the library, not the product page.

The shape, straight from the README:

```
ticks ─► LOB ─► features ─► quoter ─► OMS ─► fills ─► PnL
           │       │            │
           ▼       ▼            ▼
        Microprice  AS / GLFT    Avellaneda-Stoikov
        OFI         skew         (per-strategy)
        Welford RV  GPU dot
        Hawkes
        VPIN / Lee-Ready / Kyle's λ
```

## Microstructure features

Eight streaming features over a price-level LOB (O(log K) update, O(1) best): microprice, order-flow imbalance, depth imbalance, Welford realized variance, Hawkes intensity, VPIN, Lee-Ready, Kyle's λ. Streaming and batch APIs share one algorithm set.

## The pricing crate

`paganini-pricing` is pure Rust: forward Black-Scholes price plus a 12-Greek set, **three IV inverters** —

- Schadner variance-space Halley (3 iterations ≈ 78 ns, ~2.2× a Jäckel inverter),
- safeguarded-Newton Jäckel,
- Corrado-Miller direct,

and **five surface parameterizations** (SABR / Wing / SVI / SSVI / eSSVI), each with Nelder-Mead, BFGS, and ridge-regularized calibration variants. On top of the standard no-arb checks there's **PAV isotonic + iterative-local butterfly repair** for the cases where the raw surface isn't monotone.

## The vol-trading toolkit

The post-v0.3 work is the variance/skew layer:

- **Variance swaps** — Demeterfi-Derman-Kamal-Zou fair-strike, forward variance with a term-structure walker, and a VIX-style 30-day constant-maturity vol.
- **Five OHLC realized-vol estimators** — close-to-close, Parkinson, Garman-Klass, Rogers-Satchell, Yang-Zhang.
- **Delta-quoted skew** — 25-Δ risk reversal / butterfly with a strike-at-delta inverter.
- **Per-Greek P&L attribution** — decompose realized P&L back onto the Greeks.

There's also a position-aggregating `Portfolio` with per-strike and per-tenor Greek buckets, eight `OptionStrategy` templates (verticals, straddle, strangle, iron condor, risk reversal, butterfly, calendar / diagonal), an `IncrementalGreeks` Taylor-update pricer, and inverse-quote (Deribit-style coin-margined) converters.

## Market making

For the perp / linear desk: Avellaneda-Stoikov, GLFT, and inventory + OFI skew quotes. For the options desk (`paganini-mm::options_mm`): a vol-AS quoter with vega-bucket caps and a delta-band auto-hedge.

## Numbers, with the caveats attached

From the Criterion suite, single core: `bs_price` ~13 ns, `iv_corrado_miller` ~5 ns, `quote_chain(5 strikes)` ~210 ns. SABR calibration round-trips to <1 bp RMSE. Broad test + doctest coverage across the workspace, with cargo-fuzz targets running clean.

These are self-benchmarked microbenchmarks, single core — useful for relative comparison, not a throughput SLA. The GPU backends (Metal, OpenCL, CUDA) are real kernels validated against the CPU reference. The Lean4 specs are `sorry`-free; the one non-theorem is an explicit `axiom` in the Avellaneda–Stoikov spec (`first_order_optimal`) standing in for the not-yet-formalized HJB optimality — the rest (Welford, microprice, OFI, Almgren–Chriss schedule properties) are closed by tactic.

What this library deliberately *isn't*: a live trading system. Quoting, routing, and holding inventory against real venues is [Trade Master / Vivaldi]({% post_url 2026-5-20-TradeMasterVivaldi %}); the sell-side pricing core is [Price Master / Sibelius]({% post_url 2026-5-13-PriceMasterAria %}).
