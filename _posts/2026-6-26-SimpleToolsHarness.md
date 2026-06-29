---
layout: post
title: "simple tools, continued: remoting, sandboxes, and the harness they serve"
---

A while back I wrote up [simple tools](https://renoir42.com/simple-tools.html) as an umbrella for a few Rust libraries I kept reaching for — a TUI layer, a networking layer, a capability-scoped shell. It's worth saying out loud what the umbrella is actually *for*, because that's what decides where it goes next.

## The point is the harness

The brief for every one of these libraries is the same: keep it small enough that a **harness** — a coding agent that may be partly or entirely a local model, see [Le Harnais](https://github.com/zeta1999) — can wire it into a working tool in one sitting. No framework to learn, no plugin system, a surface you can read end-to-end and audit. That constraint is the design, not a nicety.

It also explains the recent direction of the libraries that already exist:

- **simple-network** grew a post-quantum secure channel (hybrid ML-KEM + XChaCha20 + ML-DSA) and an optional Tor onion carrier — both behind feature flags so the base stays lean.
- **simple-ui** gained an `AgentLog` widget: a rendering target for streaming agent activity, because if a model is driving the tool you want to *see* what it's doing.
- **simple-secrets** got a written threat model and fresh-secret TOTP minting.

## Two new members, by the same logic

If the harness is going to run tools, two gaps follow directly.

**simple-remote** — remoting for the capability-scoped shell. Run a `limited-shell` session on another host over `simple-network`, under the same explicit resource and policy limits. No ambient SSH trust; just the scope you grant.

**simple-sandbox** — a boundary around the blast radius. When the harness lets a model run tools, you want side effects contained, filesystem and network access mediated, and an audit trail of what actually happened. Paired with `limited-shell`, *what can run* and *what it can reach* are both scoped.

Both are coming soon. The umbrella now has [its own page](https://renoir42.com/simple-tools.html) laying out the libraries and the design rules — small over general, capabilities over trust, and public mirrors that carry just the code that compiles.
