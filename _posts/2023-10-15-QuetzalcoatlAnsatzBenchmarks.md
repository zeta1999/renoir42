---
layout: post
title: "QuetzalcoatlProto — quantum ansatz benchmarks, year one"
---

A long overdue note on QuetzalcoatlProto (private, [LeifKingston/QuetzalcoatlProto](https://github.com/LeifKingston/QuetzalcoatlProto)), the quantum R&D codebase at the core of the Anzaetek / Laplace quantum-finance work during 2023. Multi-person project; I'm the majority contributor but Amarou Keita, Antoine Mf, aurel10, Maxime ROP, and Keitamara all committed. My 2023 footprint was ~1,662 commits across May to December.

## What the repo is

A kitchen-sink harness for running, comparing, and benchmarking quantum ansätze on concrete optimisation and ML problems:

- **Ansatz benchmarks** — `ansatz_benchmarks.py`, `_multi.py`, `_multi_filters.py`, plus a `benchmark_ansatz` directory. Sweep different circuit ansätze (TwoLocal, StronglyEntangling, custom) against the same objective and measure convergence, fidelity, resource cost.
- **VQE / QAOA** — `vqe.py`, `VQEChecker.py`, `VQEHelper.py`, `VQEQiskit2.py`, `VQESimulator.py`, `vqe_qubo.py`, `VQE_to_ising.py`. Multiple VQE implementations against the same reference, so "is this bug in my code or in the backend" has a way to be answered.
- **QUBO pipelines** — `clientQubo.py`, `cov_to_qubo.py`, `ContinuousToBinary.py`, `portfolio_for_qubo.py`, `QUBOSimulator`. Portfolio problems lowered into QUBO form, multi-bit-resolution encoding (the Atlas 2021 legacy cleaned up).
- **QRNG** — quantum-random-number generation with a classical testing harness, including production-readiness scripts.
- **Synthetic data + dataset generators** — `ansatz_benchmarks_generate_dataset.py`, `_synthetic_data.py`, plus a family of `generator_ansatz_benchmarks_*` scripts for training-set creation.
- **Web APIs** — `LaplaceAPI.py`, `LaplaceWSAPI.py`, client counterparts, plus the start/stop/debug shell scripts. The quantum work was exposed as a REST + WebSocket service behind token auth.

## What the year was actually about

The product-facing framing was "quantum-inspired and quantum-real optimisation, exposed as APIs." The research-facing framing was "figure out which ansatz is least bad for which problem family, and benchmark honestly rather than picking the ansatz that demos best."

Those two goals aren't always aligned. The repo reflects the tension — a lot of the ansatz-benchmark infrastructure exists because early demos used whichever circuit converged on the training example, and we needed a way to stop doing that.

## What I'd claim

- The benchmarking harness is the reusable piece. "Run circuit A, circuit B, circuit C against the same problem with the same seed and report all three" is mundane software engineering that a lot of quantum papers don't do.
- The API layer (Laplace / WebSocket) was never the point. It let the research keep moving without the pipeline work cluttering the quantum core.
- QRNG and VQE got most of the polish; QAOA stayed more experimental.

## What I wouldn't claim

- This was not a quantum-advantage result. For the problem sizes we could run, classical solvers were routinely competitive or better. The honest value of the work was producing that comparison carefully.
- The codebase is big and evolved quickly. Commit messages are largely `fix` — that's normal for an R&D tree but not a reviewable history.

Private repo; not pushed publicly. The 2024 follow-on shows up in a separate post.
