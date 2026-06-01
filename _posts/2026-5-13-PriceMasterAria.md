---
layout: post
title: "Price Master: the Aria payoff language and deterministic GPU Monte Carlo (Sibelius)"
---

First, a name collision to clear up. There are **two** things called Aria in this family. The one in [gpu-backtest]({% post_url 2026-4-12-AriaStrategyDSL %}) is a *strategy* DSL — signals and `when ... -> buy/sell` rules. The one here, in Sibelius, is a *contract / payoff* description language. Same naming instinct, different jobs. Sibelius is the sell-side core behind **Price Master**.

## Aria, the payoff language

It's a combinator-based contract language in the SPJ/Peyton-Jones/Eber "Composing Contracts" lineage (the same idea behind Bloomberg's DLIB/BLAN, built on LexiFi's MLFi algebra). You describe a payoff out of primitives and the engine prices it. The combinator set, from `AriaSpec.md`:

| Combinator | Description |
|---|---|
| `zero` / `one(ccy)` | null contract / pay 1 unit of `ccy` now |
| `give(c)` | flip to the counterparty view |
| `and` / `or` | both contracts / holder takes the better one |
| `scale(e, c)` | scale payments by expression `e` |
| `when(d, c)` / `until(d, c)` | acquire on / active until date `d` |
| `anytime(d1, d2, c)` | American exercise in `[d1, d2]` |
| `on(dates, c)` | Bermudan exercise on specific dates |
| `cond(b, c1, c2)` | branch on a boolean |
| `barrier(dir, level, asset, hit, miss)` | `up_in` / `up_out` / `down_in` / `down_out` |

## What a script looks like

Scripts are `.aria` files. Comments are `--`, payoffs are built from `let` bindings and the combinators above, and `|> when(date)` pins a cashflow to a settlement date. Here's a plain FX call (`TestData/payoffs/fx_call.aria`):

```haskell
-- FX call: payoff = notional * max(USDKRW(T) - K, 0)
contract FXCall {
    let fx = fxrate("USDKRW")
    let K = 1300.0
    let notional = 1000.0
    notional * max(fx - K, 0.0) * one(KRW) |> when(2026-01-01)
}
```

The point of a payoff language is the exotics, where `var` + `:=` carry path state across observation dates. A trimmed Korean autocall (ELS) — the running minimum `pmin`, a `tf` "already terminated" flag, and a `smooth_step` barrier at each semi-annual date (`korean_autocall_3y.aria`, periods 3–6 elided):

```haskell
contract KoreanAutocall3Y {
    let basket = min(spot("KOSPI2") / 280.0, spot("SAMSUNG") / 100.0)

    var pmin = 1000000.0          -- running minimum of the basket
    var tf   = 0.0                -- terminated-flag, 0..1
    pmin := min(pmin, basket)

    let d1 = when(2025-06-01, one(KRW))
    let d2 = when(2025-12-01, one(KRW))

    -- Period 1: autocall barrier at 90%
    let t1  = smooth_step(0.9 - pmin, 10.0, 0.0)
    let et1 = min(1.0 - tf, t1)
    tf   := tf * (1.0 - d1) + min(1.0, tf + et1) * d1
    pmin := pmin * (1.0 - d1) + 1000000.0 * d1

    -- Period 2: barrier steps down to 90% -> 85% -> ... over the schedule
    let t2  = smooth_step(0.9 - pmin, 10.0, 0.0)
    let et2 = min(1.0 - tf, t2)
    tf   := tf * (1.0 - d2) + min(1.0, tf + et2) * d2

    -- Coupons on early termination (5% per period), accreting
      0.05 * et1 * d1
    + 0.10 * et2 * d2
    -- ... periods 3-6 ...
    -- Knock-in put at maturity if never autocalled and basket < 60%
    + (1.0 - tf) * min(basket - 0.60, 0.0) * d2
}
```

That's the whole pitch: the same file describes a vanilla and a path-dependent autocall, and the engine prices either one. The `smooth_step` instead of a hard indicator is deliberate — it keeps the adjoint Greeks well-behaved through the barrier. Other scripts in `TestData/payoffs/` cover snowballs, TARNs, cliquets, range accruals, quantos, CDS, and dual-leg IR swaps.

A contract compiles to flat bytecode and is evaluated either across Monte Carlo paths (SIMD, antithetic, QMC) or via a PDE solver (1D/2D, American via tridiagonal + projection). Models behind it: Black-Scholes, Heston, Dupire local vol, rough vol. Greeks come out by adjoint algorithmic differentiation. XVA (CVA/KVA) and SIMM counterparty risk sit on top.

## The unglamorous GPU work

The recent commits aren't features — they're the determinism plumbing that makes a GPU pricer trustworthy. CUDA Monte Carlo had an RNG-state writeback race; fixing it meant a deterministic reduction and a **portable RNG mode** so the CPU and GPU paths produce the *same* numbers. CI now keeps **dual goldens split by platform** and auto-enables CUDA so the build matches the goldens it's checked against. If your GPU pricer can't reproduce its own outputs, none of the headline throughput matters.

## Deep hedging — what exists, what doesn't

`DEEPHEDGE_TODO.md` is honest about the gap between the demo and a portfolio-grade hedger:

| Surface | Status |
|---|---|
| `RLHedgeTrainer::trainPolicy` — REINFORCE, short call, BS dynamics, single asset | ✅ |
| `evaluatePolicy` — score vs BS-delta over N paths | ✅ |
| Multi-asset variant (`AriaMultiAssetHedge.h`) | ✅ |
| `AriaDeepHedgeTrain` / `Evaluate` JSON endpoints | ✅ |
| Train on an arbitrary Aria contract | ❌ |
| Train on a **portfolio** of contracts | ❌ |
| Per-trade notionals, fixings, multi-currency netting | ❌ |
| Production reward (mean-CVaR, exponential utility) | ❌ |
| GPU training | ❌ |

So: a working single-instrument RL hedger with JSON train/evaluate endpoints, and a clear list of what portfolio-level hedging still needs. There's also a `dnn-plugin` plan for a stable C ABI with memory-domain awareness, to keep the model boundary clean.

## Numbers

MC throughput ~579M path-steps/sec at 65,536 paths × 252 steps (x86-64, 32 threads, GCC `-O2` auto-vectorized); PDE1D European under 5 ms on a 501×365 grid (larger grids scale up — ~50 ms at 801×365). Self-benchmarked — the GPU MC numbers are currently transfer-bound, which is exactly why the determinism work came before any speedup claim.

The buy-side counterpart — order flow, microstructure, market-making quoters — is [Paganini / Flow Master]({% post_url 2026-5-6-PaganiniFlowMaster %}).
