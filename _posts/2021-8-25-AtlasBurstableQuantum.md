---
layout: post
title: "Atlas — bursting quantum-optimisation jobs to AWS Braket"
---

Short note on Atlas, a 2021 private service I built for bursting quantum-optimisation jobs from a small control plane to AWS Braket (D-Wave and gate-model backends). Jun–Nov 2021, ~59 commits. Closed source.

## What it did

A minimal API + auth layer + Docker-on-EC2 executor:

- **Auth and credit**: `cmdAddUser`, `cmdAddCredit`, `cmdGetToken`. Token-based API; credits metered against run cost.
- **Burst**: `build.burst.portfolio.sh`, `burst-monitor`, `burstbuild.sh` / `burstbuild.expect`. A client submits a job; the controller packages it into a Docker image configured via `ATLAS_SECRET`, ships it to EC2, and streams logs back.
- **Quantum backends**: `BraketDwave.py`, `BraketQAOA.py`. D-Wave annealer path for QUBO problems; QAOA path for gate-model simulators.
- **QUBO / Ising plumbing**: `ContinuousToBinary.py`, `cov_to_qubo.py`, `VQE_to_ising.py`, multi-bit resolution encoding so a continuous-valued portfolio weight could land in a binary-variable QUBO cleanly.
- **RL demo**: an early reinforcement-learning portfolio demo (Jul 2021, "better RL demo") that depended on LaplaceKorea/Portfolio-Optimization-and-Goal-Based-Investment-with-Reinforcement-Learning.
- **Ising experiments**: `ising` commit in Jul 2021 — classical Ising ground-state checks against the quantum/annealer results.

## Why burstable

At the time the question was whether a quantum backend was worth the latency for a specific class of portfolio problems. The only honest way to answer that was to run the same QUBO through an annealer, through QAOA on a simulator, and through classical solvers, and compare. Atlas was the plumbing that made "run the same problem on all three" one command.

The answer, for the problem sizes I could run, was "not yet, for this family of problems." That's the kind of answer you want to be able to produce, even if it's negative.

## What I'd do differently

- Don't build an auth system. A pre-shared key on top of SSH port-forwarding would have been fine for the audience.
- The multi-bit-resolution conversion is the interesting piece. Promoting that into a standalone library — "continuous-to-binary for QUBO" with a clear error bound per bit — would have had more reuse value than the whole burst apparatus.

Closed source. The `BraketDwave.py` / `BraketQAOA.py` shapes later informed the [QuetzalcoatlProto]({% post_url 2023-10-15-QuetzalcoatlAnsatzBenchmarks %}) ansatz-benchmark harness.
