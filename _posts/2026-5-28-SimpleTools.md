---
layout: post
title: "simple tools: libraries for local agent development (simple-ui, simple-network, limited-shell)"
---

"simple tools" is an umbrella for three Rust libraries I keep reaching for when building local agents and developer tools: a TUI/widget layer, a networking layer, and a capability-scoped execution layer. Different maturities — worth being precise about which is which.

## simple-ui — TUI widgets + markdown engine

The most mature of the three. A widget library plus a markdown engine that renders interactive documents. The widgets — `DataTable` (scroll, selection, sort), `DepthLadder`, `TimeAndSales`, `Chart` — were upstreamed from an internal trading terminal into a reusable crate. The markdown engine has a unix-domain-socket IPC path (so a process can push AST updates into a live view) and a formula evaluator with WASM bindings for user-defined functions.

Decision on record: **TUI-only for v1**. The graphical / React target is parked rather than half-shipped. For agent work this is the rendering layer — readable terminal UIs an agent can drive and update in real time.

## simple-network — Erlang patterns + distributed algorithms

A Rust networking library organized around Erlang-style process patterns — `gen_server` (RPC), `gen_statem` (stateful handlers), `net_kernel` (clustering), pub/sub — over a pluggable transport (TCP/UDP/TLS, mTLS pairing). Underneath sit the distributed-systems building blocks: Raft, SWIM, 2PC, Kademlia DHT, Gossip, vector clocks, OR-Set CRDTs. Each algorithm ships with TLA+ and Lean4 proof skeletons alongside the implementation, plus `cxx` (C++) and `cgo` (Go) FFI bridges.

Status, stated plainly: transitioning from skeleton to implementation. `gen_statem` and pub/sub are filled in; several algorithm modules are logic + proofs with the loops not yet fleshed out. Good for prototyping and learning, not production. For agents this is the substrate for distributing work across machines.

## limited-shell — a capability-scoped DSL

A DSL (Rust target) for multi-machine, capabilities-based workflows. You declare roles with parent/child hierarchies and `can`/`cannot` rules over resources; typed devices (CPU, GPU, MACGPU, Disk, Network, QRNG) carry extents and cost rules; a scheduler places task graphs across heterogeneous machines under those constraints. Policy is deny-first, most-specific-match-wins.

It's the furthest along on language plumbing: a hand-written recursive-descent parser and an execution engine — expression evaluator, statement executor, for/while loops, `try`/`catch`/`finally`, builtins, div-by-zero handling. The point for agent work: run an agent's actions inside an explicit capability and cost envelope instead of an open shell.

## Where they sit

| Library | Role for agent dev | Maturity |
|---|---|---|
| simple-ui | terminal UI / live views | v1, TUI-only |
| simple-network | distributed substrate | skeleton → impl |
| limited-shell | capability-scoped execution | parser + exec engine landed |

All early, none overclaimed. simple-ui is usable today; the other two are honest works-in-progress with the foundations (proofs, parser, execution engine) in place.
