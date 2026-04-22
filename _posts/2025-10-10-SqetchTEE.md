---
layout: post
title: "sqetch-tee-fl — federated learning inside a TEE"
---

Short note on `sqetch-tee-fl` (private, Sep 2025). A prototype for federated learning under TEE (Trusted Execution Environment) constraints inside the Sqetch fintech platform.

## What the setup was

- Multiple participants, each holding a private slice of training data they can't share.
- A federated-learning coordinator that schedules training rounds.
- The aggregation step runs inside a TEE (Intel SGX / AMD SEV shape). Each participant encrypts their gradient update under a key attested to the TEE; the TEE decrypts, aggregates, returns the aggregated update.

## What the TEE bought

The narrow claim: the coordinator can't see individual gradients. The TEE enforces that aggregated updates are the only thing any non-participant party gets. This mattered for a specific compliance posture — some participants couldn't share individual-customer-derived gradients even in encrypted form with a non-TEE coordinator.

## What the TEE didn't buy

- Protection against a compromised participant. Classical federated-learning attacks (gradient-inversion, backdoor injection via poisoned updates) still apply. The TEE doesn't help.
- Protection against side-channel attacks on the TEE itself. Those aren't hypothetical — Foreshadow, LVI, and the SEV class of attacks all exist. You trust the TEE only to the degree you trust its microarchitectural state.
- Protection against a malicious coordinator who has the attestation key. The attestation flow has to actually run, not just be described in the spec.

## Where it got stuck

The deployment side, honestly. Getting a TEE-enabled host commissioned with the right microcode, the right attestation chain, and the right kernel support, outside of a single cloud vendor's specific managed offering, is still not routine in 2025. A prototype where each participant has to do that themselves before any training happens is a deployment problem, not a cryptography problem.

We reached "one-site prototype works end to end" and didn't reach "N-site deployment in the wild."

## What I'd keep

- The design of the attestation + key-wrapping + aggregation flow. That part is clean and worth documenting even if the deployment pieces aren't.
- The integration with the rest of the Sqetch platform — the FL coordinator talks to the same model-registry service as the non-FL training paths, which kept the story consistent.

Private repo, part of the broader Sqetch fintech platform.
