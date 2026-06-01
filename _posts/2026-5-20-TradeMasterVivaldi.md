---
layout: post
title: "Trade Master (Vivaldi): the plan for a single-binary, multi-venue trading system (WIP)"
---

This one is a plan, not a release. **Trade Master** — codename **Vivaldi**, keeping the composer naming next to Paganini and Sibelius — is the binary that would quote, hedge, route orders, hold inventory, and talk to venues. The pieces around it already exist; the integrating engine is the work in progress.

The tiers, from the architecture plan:

- **Vivaldi** — the live trading binary (this, the WIP part).
- [Paganini / Flow Master]({% post_url 2026-5-6-PaganiniFlowMaster %}) — the buy-side quant library.
- [Price Master / Sibelius]({% post_url 2026-5-13-PriceMasterAria %}) — the derivatives pricing core.
- [notbbg]({% post_url 2026-4-1-MarketDataTerminal %}) — the data + GUI tier (its OSS cut already ships).
- [gpu-backtest]({% post_url 2026-3-25-LOBReconstructionGPU %}) — research / backtest parity.

## What Vivaldi has to do

The scope the plan is written against:

1. Crypto vol / options market making — Deribit first, then Binance Options, Kraken Derivatives, with a watching brief on options-bearing DEXes.
2. Multi-exchange arbitrage — one side passive, the other hedging (and a "both sides passive when possible" mode).
3. A generic OMS for liquidity provision and optimal basket execution.
4. Portfolio optimization — factor models plus news/alert-driven signals.
5. Liquidity provision for new LOB-style venues, including the DEX inside Seal DAO.
6. Smart auto-close on large moves (stop-loss with a regime trigger).
7. Regime-change detection at instrument *and* basket level.
8. Venue toxicity detection — markout, queue stability, fade rate.
9. Eventually, FPGA execution on AWS F2 or similar.

## Phases, locked to Paganini versions

The design rule is that the two repos move in lockstep — every Vivaldi phase names the Paganini version it needs. The one-screen map:

| Vivaldi phase | Paganini | Theme |
|---|---|---|
| V1 | v0.1 | data plane + LOB + paper trading |
| V2 | v0.1 | live Deribit + OMS + kill-switch |
| V3 | v0.2 | crypto perp MM + delta hedger |
| V4 | v0.2 | multi-venue + SOR + queue + toxicity |
| V5 | v0.3 | crypto options MM + vol surface |
| V6 | v0.4 | research-to-live parity (gpu-backtest) |
| V7 | v0.4 | cross-asset stat arb + UDP/multicast fabric |
| V8 | v0.9 | FPGA hot path |
| V9 | v0.7 | portfolio opt + LLM signals |
| V10 | v0.6 | DEX liquidity provision + AMM |

## Where it actually stands

The honest status: Paganini's library is at v0.3, Sibelius prices contracts today, and the notbbg terminal's OSS cut boots on a fresh clone. What doesn't exist yet is the Vivaldi binary that wires quoting → routing → inventory → hedging against live venues. The screenshots on the Trade Master page are the notbbg terminal standing in as the current data/GUI layer; the trading surface is being built on top of it. Conservative architecture first (NAS-friendly local deployment, hard kill-switch, event-driven core), live venues second.
