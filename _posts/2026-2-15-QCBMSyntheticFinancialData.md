---
layout: post
title: "Quantum Circuit Born Machines for Synthetic Financial Data"
---

Between 2023 and early 2026, our team at Anzaetek built a synthetic financial data pipeline combining Quantum Circuit Born Machines (QCBM) with Conditional TimeGAN. This post describes the motivation and approach at a high level — the implementation is closed source.

## The Problem

If you backtest trading strategies on historical data, you have exactly one sample path. You can do walk-forward validation, you can bootstrap, but fundamentally you're fitting to a single realization of the market. The conclusions you draw — "this strategy has a Sharpe of 1.8" — are conditioned on that specific history.

Synthetic data generation tries to solve this by producing realistic alternative market histories. The challenge is that financial time series have properties that are hard to reproduce: fat tails, volatility clustering, cross-asset correlations, regime switches, and microstructure effects. Standard methods (GBM, GARCH) capture some of these. GANs capture more. We wanted to see if quantum-inspired generative models could do better on the distributional properties that matter for strategy evaluation.

## Our Approach

We combined two generative models in a pipeline:

**Quantum Circuit Born Machines (QCBM)** are generative models where a parameterized quantum circuit defines a probability distribution over bitstrings. You train the circuit parameters to match an empirical distribution — in our case, discretized distributions of returns, volatility, and regime indicators. QCBMs can represent certain multimodal distributions more efficiently than classical models of comparable size, which is useful when your target distribution has the multi-peaked structure you see in regime-switching markets.

We used QCBMs as the noise source for a **Conditional TimeGAN** — a GAN architecture designed for time series, conditioned on regime labels. The standard TimeGAN uses Gaussian noise; we replaced it with samples from a trained QCBM. The hypothesis was that structured noise from the QCBM would help the generator produce samples with better tail behavior and regime-transition dynamics.

**Market regime labeling** was done upstream using Hidden Markov Models and XGBoost on features extracted from price and volume data. We used Optuna for hyperparameter tuning across the full pipeline.

## What We Learned

The QCBM noise source improved distributional fidelity on several metrics compared to Gaussian noise, particularly on tail statistics and cross-regime transition probabilities. The improvement was modest but consistent across equities, FX, and commodities.

The practical bottleneck was training time. QCBM parameter optimization using PennyLane was significantly slower than the TimeGAN training — even with adjoint differentiation for gradients. For production use, we ended up using a classical approximation of the trained QCBM distribution for actual data generation, with the QCBM serving as a way to find good noise distributions during the research phase.

The regime-conditional architecture was the bigger win. Conditioning on regime labels (regardless of noise source) produced synthetic data where strategy backtests gave much more realistic performance distributions than unconditional generation.

## Tools

PennyLane and Qiskit for quantum circuit models, PyTorch for TimeGAN, and our Rust-based backtesting infrastructure for evaluation. The regime labeling pipeline was Python (scikit-learn, hmmlearn) with Optuna for tuning.

This work was done as part of Anzaetek's quantum ML research between 2023 and February 2026.
