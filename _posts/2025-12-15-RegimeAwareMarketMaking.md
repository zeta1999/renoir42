---
layout: post
title: "Regime-Aware Market Making with HMM + HJB"
---

I came across a project by [pellera9](https://github.com/pellera9) that caught my attention: [regime-aware-optimal-quoting](https://github.com/pellera9/regime-aware-optimal-quoting). It's a clean implementation of something I've spent a lot of time thinking about — optimal market making under regime switching — and I think it deserves more visibility.

## What It Does

The classic Avellaneda-Stoikov (2008) model gives you optimal bid/ask quotes for a market maker with CARA utility and inventory risk. It's elegant, but it assumes the market behaves the same way all the time — same volatility, same order arrival rates, same spread dynamics. Anyone who has actually traded knows that's not true.

This project extends Avellaneda-Stoikov to handle regime switching:

1. **Feature extraction from LOB data** — microstructure features from the limit order book (spread, depth imbalance, trade flow)
2. **Gaussian HMM with sticky priors** — detect latent market regimes without state flickering
3. **Regime-dependent HJB solver** — solve the Hamilton-Jacobi-Bellman optimal control problem separately per regime
4. **Online Bayesian filtering** — maintain a posterior over the current regime and blend optimal quotes at runtime

## Why This Is Interesting

Most academic market making papers stop at the HJB solution under a single set of parameters. The gap between that and something deployable is enormous. This project bridges several of those gaps.

**The HMM is the right tool here.** Regime detection for market making is a filtering problem, not a classification problem. HMMs give you a posterior distribution over states, and the forward algorithm is fast enough for real-time use. The sticky prior (vectorized EM with Dirichlet self-transition bias) prevents the state flickering that would make downstream quoting unstable.

**The HJB solver is regime-coupled.** This isn't just "run Avellaneda-Stoikov with different vol estimates." The backward ODE integration accounts for the possibility of regime transitions during the trading horizon. If you're in a low-vol regime but there's a 15% chance of transitioning to high-vol in the next 30 minutes, your optimal quotes should already reflect that.

**Walk-forward validation.** The project includes a proper walk-forward test harness with parameter grid search — the right way to evaluate trading strategies.

**The LOB simulator models fills realistically** — queue position, partial fills, adverse selection. Not just "if price crosses your quote, you're filled." This matters because fill modeling is where most backtesters silently lie.

## What I'd Add

- **Non-Gaussian emissions** — heavy-tailed distributions for the observation model
- **GPU acceleration of the HJB solver** — the backward ODE is embarrassingly parallel across the inventory grid
- **Online HMM parameter updates** — adapt to structural changes rather than relying on a fixed historical fit

Solid work. The repo is at [github.com/pellera9/regime-aware-optimal-quoting](https://github.com/pellera9/regime-aware-optimal-quoting).
