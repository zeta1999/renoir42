---
layout: post
title: "Regime-Aware Market Making with HMM + HJB"
---

A repo worth a look: [ahmetsbilgin/regime-aware-optimal-quoting](https://github.com/ahmetsbilgin/regime-aware-optimal-quoting), a Birkbeck MSc dissertation on extending Avellaneda-Stoikov to regime switching.

## What it does

Four pieces:

1. Rolling-window LOB features: log-return, realized vol, relative spread, depth imbalance.
2. Gaussian HMM with sticky self-transition priors (EM, log-space forward-backward).
3. Regime-coupled HJB solved backward by ODE integration, one value function per regime with transitions via a generator matrix Q.
4. Online Bayesian filter for the current regime posterior, used to blend quotes at runtime.

## What I found interesting

The HMM fits the problem: regime detection here is filtering, not classification. Sticky priors keep the posterior from flickering, which would otherwise destabilize the quotes downstream.

The HJB is regime-coupled rather than "A-S run per regime": the backward ODE accounts for the chance of regime transitions within the horizon. If you are in a low-vol regime but likely to transition to high-vol, your current quotes already reflect it.

## Caveats

The backtest simulator (`src/backtest/simulator.py`, `src/execution/order_manager.py`) uses naive crossing fills at top-of-book, unit size, no queue position, no partial fills. That is the main gap for a market-making backtest — adverse selection is modeled in the control problem but not in the fill simulation.

The README advertises walk-forward validation and grid search, but I did not find the harness in `examples/` — there is a single-run `run_backtest.py`, no walk-forward loop. Possibly in a branch or not yet committed.

## Things I would add

- Heavy-tailed emissions for the HMM.
- GPU port of the backward ODE: embarrassingly parallel across the inventory grid.
- Online HMM updates rather than a fixed historical fit.

Code @ [github.com/ahmetsbilgin/regime-aware-optimal-quoting](https://github.com/ahmetsbilgin/regime-aware-optimal-quoting).
