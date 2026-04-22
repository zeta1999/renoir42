---
layout: post
title: "qml_model_02 — a build pipeline for quantum-augmented classifiers"
---

Note on `qml_model_02`, private 2025 project (Jul 2025 to early 2026). The interesting part isn't the classifier — it's the delivery shape.

## The shape

- A family of small quantum-augmented classification models, each with its own directory under `models_binary/<model>` and its own `build.sh`.
- An Anaconda / Miniforge runtime wrapped by `runtime-romulus2.sh` (and a fast-path installer `scratch_build.sh`). The runtime is not the conda-env-du-jour — it's a pinned dependency set that produces reproducible builds.
- CI with webhook notifications so a green build posted to wherever the team was watching.
- Each model's `build.sh` produces a binary artefact with all dependencies baked in, runnable from a command line. Not a notebook, not a Python script you need to `pip install` into — a binary.

## Why "binary form"

Most quantum-ML code I see in the wild assumes the reader has a working PennyLane / Qiskit / CUDA environment exactly matching the author's. That's fine for research; it's a maintenance problem for anyone else. A binary artefact says "here is the thing, it runs." The cost is a longer build pipeline; the benefit is that "does this demo work on your machine" has a yes/no answer instead of a half-hour of dependency-hell debugging.

This isn't novel. Scientific software has been shipping this way for decades. Quantum-ML stack just hasn't picked up the discipline yet.

## Status, honestly

- The runtime and build pipeline work. Models under `models_binary/<model>` each build and run.
- The classifiers themselves are research-grade. Some worked well on their target problems; some didn't. The interesting ML comparisons are in [QuetzalcoatlProto]({% post_url 2024-3-10-QuetzalcoatlYearTwo %}) and the 2026 QCBM post.
- The pipeline-hygiene part is what I'd reuse — it's the shape I'd want any small research lab to default to.

Private repo.
