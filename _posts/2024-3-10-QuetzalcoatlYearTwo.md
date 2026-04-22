---
layout: post
title: "QuetzalcoatlProto, year two: portfolio-via-QUBO and the API plumbing that outlived it"
---

Follow-on to the [year-one QuetzalcoatlProto post]({% post_url 2023-10-15-QuetzalcoatlAnsatzBenchmarks %}). 2024 was a more focused year on the same codebase — ~1,153 R-B commits from Jan to May 2024, concentrated on portfolio optimisation via QUBO and on getting the API layer production-shaped.

## What the focus was

- **Portfolio-to-QUBO**, end to end. Covariance matrix in, QUBO out via multi-bit-resolution encoding, solve on a quantum simulator or annealer, decode back to continuous weights. `portfolio_for_qubo.py`, `cov_to_qubo.py`, `ContinuousToBinary.py`. A lot of the 2024 work was about making the decoding loss cheap to reason about — the bit-width vs rounding-error tradeoff is the knob that actually matters.
- **VQE-to-Ising cross-checks**. Same problem, two solver paths (VQE on a simulator, classical Ising ground-state search), so a discrepancy is a bug in one of them and not "the quantum part is noisy." `VQE_to_ising.py`.
- **API plumbing** — `LaplaceAPI`, `LaplaceWSAPI`, `JSONAPIs`, plus the `run_api_tests_almost_prod_*` shell scripts. Stress-testing the service with and without QRNG, with and without GPU backends, inside and outside Docker.
- **XGBoost integration** — `xgboost.onnx`, `testing_set.csv`, `training_set.csv`. Classical ML baselines for the same problems the quantum side was trying to solve, so we could quote both numbers next to each other.

## What changed from 2023

- 2023 was "can we make this work at all." 2024 was "can we make it reproducible." The second question is the one customers actually care about.
- The API scripts became honest about what "almost_prod" meant — with QRNG, without QRNG, with docker-in-docker, with GPU, without GPU. You can tell from the filenames that the deployment matrix was expanding faster than the tests.
- The ansatz benchmarking slowed down as the tooling stabilised. Once you've compared TwoLocal, StronglyEntangling, and a handful of custom circuits on the same objective, you have your answer; adding a twentieth ansatz doesn't help.

## What I'd single out

- The multi-bit-resolution encoding is the one piece of code I'd lift out today and release on its own. "Continuous → binary → QUBO → back" with a clear per-bit error bound is reusable beyond any specific quantum backend.
- The `run_api_tests_*` script family is the other reusable thing — a matrix of deployment modes that you run before every release. Less glamorous than the ansatz work; more load-bearing.

## What the year didn't produce

- A quantum-advantage result for any of the problem sizes we could run. Same honest answer as 2023.
- A cleanly factored core library. The codebase is still R&D-shaped; the good bits are load-bearing but embedded.

Private repo, not public.

After 2024 I stepped back from Quetzalcoatl — the pipeline work in 2025-2026 moved into separate repos (`time_series_base`, `finance_lab`, `qml_model_02`) rather than continuing to grow this one tree.
