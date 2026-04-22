---
layout: post
title: "Carving finance_lab out of Sqetch"
---

Short project-hygiene note. In December 2025 the QCBM + TimeGAN + regime-detection work that had been living inside the Sqetch fintech platform got moved into its own repo, `finance_lab`. The [QCBM + TimeGAN post]({% post_url 2026-2-15-QCBMSyntheticFinancialData %}) covers the technical content; this one is just about why the split.

## Why separate

- **Dependency weight**. The QCBM work pulls in PennyLane, PyTorch, scikit-learn, hmmlearn, Optuna. The rest of Sqetch doesn't. Keeping them in the same repo meant every Sqetch build paid for an import of the whole quantum stack.
- **Release cadence**. The research code iterates faster than the platform code. Separate repos let each release on its own cadence without the research commits cluttering the platform's diff.
- **Reviewability**. A PR touching "the research notebook plus the production config plus the TEE attestation flow" is unreviewable. Three PRs across three repos are each reviewable.
- **Open-source optionality**. `finance_lab` can plausibly go public once the claims are honest (see the [QCBM post]({% post_url 2026-2-15-QCBMSyntheticFinancialData %})'s "what I can say without overclaiming" section). The platform code can't. Keeping them together would have held the research hostage to the platform's IP posture.

## What didn't change

- The APIs each side exposes to the other. The carve-out produced a cleaner import boundary, not a new protocol.
- The data. Both sides still read from the same DuckDB store; `finance_lab` consumes, doesn't ingest.
- The overall architectural story. Nothing was rewritten; files moved.

## What did change

- Tests that depended on cross-repo fixtures had to be either inlined into each side or lifted into a small shared `sqetch-test-data` package. The latter is cleaner; the former is what actually happened on day one.
- CI per repo is faster individually. The combined pipeline used to be the bottleneck; two shorter pipelines are a better day-to-day experience.

## The wider point

Splitting a repo is a boring, unglamorous operation. It's easy to defer and easy to regret. In this case the split landed late enough that a few tests broke during the move; it should have happened a quarter earlier.

Private (`finance_lab`), part of the broader Anzaetek codebase.
