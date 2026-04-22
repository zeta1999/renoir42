---
layout: post
title: "QCBM noise in a Conditional TimeGAN, for synthetic financial data"
---

Some notes on one of the experiments we ran at Anzaetek between 2023 and early 2026 inside the Sqetch fintech platform. The work itself is closed source; here I only describe the setup.

## Setup

A Conditional TimeGAN (PyTorch) for financial time series, with the generator noise distribution replaced by samples from a trained Quantum Circuit Born Machine (QCBM). So the pipeline is:

1. Build AR(1) innovations from the data, Gaussianised dimension-by-dimension via the empirical CDF.
2. Train a QCBM (StronglyEntanglingLayers ansatz, PennyLane `default.qubit`) to match the joint copula distribution over a small subset of dims, at a single time step, in $ \\{0,1\\}^{D \cdot B} $ (D dims, B bits per dim).
3. Sample from the QCBM, map bits back to $ (0,1) $, apply $ norm.ppf $, then an AR(1) step in Gaussian space, then $ norm.cdf $ to get uniform $ Z $ for the generator.
4. Feed those $ Z $ into the Conditional TimeGAN generator; other dims stay i.i.d. Gaussian.

Conditioning at the TimeGAN level was done on past log-return windows and, optionally, HMM regime labels.

Datasets: equity prices (Google), energy, and gold. Regime labels came from an HMM + XGBoost pipeline tuned with Optuna.

## Why a QCBM at all

The honest answer: to check whether a parameterised quantum distribution, used as the latent noise of an otherwise standard TimeGAN, moves any of the usual TimeGAN evaluation metrics (discriminative, predictive, PCA/t-SNE) in a noticeable way.

Two points made it worth trying:
- QCBMs can encode multimodal joint distributions with relatively few parameters, which is convenient if you already think the financial copula is multi-peaked (regimes).
- Replacing only the noise leaves the rest of the architecture unchanged, so the comparison is cheap.

## What I can say without overclaiming

The QCBM setup trained and ran end-to-end. On the three datasets and the committed metrics, the QCBM noise was broadly comparable to Gaussian noise — not a clear win, not a clear loss. Conditioning on regime labels mattered more than the choice of noise distribution.

The QCBM training loop is slow: `default.qubit` is a CPU state-vector simulator, and even a small copula (2-3 dims, 2-3 bits each, a few strongly-entangling layers) takes the bulk of the wall-clock time. Changing `diff_method`, moving to `lightning.gpu`, or cutting the ansatz depth are the knobs that actually move training time.

## What I would not claim

- I would not claim QCBM samples "fix fat tails" on these metrics; the committed evaluation does not measure tails directly.
- I would not claim FX was in the dataset mix — it was not.
- I would not claim there is a closed-form classical surrogate being used in place of the QCBM at inference time; sampling is straight from the trained circuit.

## Tools

PennyLane, PyTorch, scikit-learn, hmmlearn, Optuna. All classical; the "quantum" part is a simulator.
