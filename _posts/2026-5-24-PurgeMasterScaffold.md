---
layout: post
title: "Purge Master: agents for security — scanner orchestration, LLM triage, agent-assisted pen-testing (scaffold)"
---

Early flag: this is a scaffold post. The repo is at "initial commit — appsec CLI scaffold + fixtures + planning docs." What follows is the shape it's being built into, not a shipped tool. Purge Master is a **companion to [Seal DAO]({% post_url 2026-4-12-SealDAOArchitecture %})** — the security program for a post-quantum chain with a lot of Rust and a DEX needs its own toolchain.

## The shape

One Go binary, `appsec`, orchestrates the scanning layer end to end:

- **Scan** — SAST / DAST / SCA / secrets, each scanner in a pinned container.
- **Normalize** — every scanner emits SARIF; SARIF is the immutable contract.
- **Store** — findings land in a DuckDB store, queried and triaged in SQL.
- **Triage** — an LLM (local, via Ollama) estimates exploitability and drafts patch suggestions.
- **Pen-test** — agent-assisted: drive the dynamic side and the suppression-lab targets, not just static rules.

Claude Code skills (`/scan-repo`, `/scan-running-app`, `/triage`) are thin wrappers over the binary — the orchestration logic lives in Go, not in the prompt.

## One principle worth quoting

From `docs/ARCHITECTURE.md`:

> **No Python on the host.** Every Python scanner runs inside a container (podman/docker). The host sees one `appsec` Go binary. [...] a security program whose orchestrator is a `pip`-managed Python venv breaks every time a shared OS package updates `libssl`. The CLI you run daily must not itself be a source of entropy.

That's the whole design temperament: the thing you run every day is one static binary; everything noisy is sandboxed behind a SARIF boundary.

## Two halves of "agents for security"

The tagline isn't only about triage. There are two distinct agent roles:

1. **Triage agent** — reads SARIF findings, ranks exploitability, proposes fixes. LLM-assisted, human-in-the-loop. The honest framing is *assisted*, not autonomous: a 14B local model is not GPT-class, so retrieval and the SARIF contract do the heavy lifting and the model annotates.
2. **Pen-testing agent** — drives the dynamic / DAST side against running targets (the plan pairs Juice Shop / DVWA fixtures), probing rather than just pattern-matching.

## What's actually there today

Planning docs — `PLAN.md`, `ARCHITECTURE.md`, a self-`CRITIQUE.md`, `DEMO.md`, `MANUAL-TESTING.md`, `STATUS.md` — plus the CLI scaffold and fixtures. The scanner orchestration, the DuckDB triage pipeline, and the pen-testing agent are the build-out, not the current state. Coming soon, said plainly.
