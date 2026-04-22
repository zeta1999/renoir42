---
layout: post
title: "notbbg: a market-data terminal across CEX, DEX, macro, and news (updated)"
---

[this-is-not-bbg-public](https://github.com/zeta1999/this-is-not-bbg-public) is a market-data terminal that aggregates ~17 feeds (CEX, DEX, macro, on-chain, news) into a single Go server and three clients. The name is a joke.

## Clients

Three, not one:

- **TUI** — Bubble Tea terminal app with seven panels: OHLC charts, LOB, news, alerts, feed monitor, server log, agent.
- **Desktop** — Electron GUI for the same topics.
- **Phone** — React Native (Expo) app, marked experimental in the README. QR / manual pairing from TUI or desktop.

## Feeds

All adapters live under `server/internal/feeds/`:

- `ccxt/`: Binance, OKX, Bybit, Bitget, Coinbase, Kraken.
- `dex/`: Hyperliquid, Jupiter, Raydium, Uniswap, plus DeFi-protocol HTTP clients.
- `world/`: CoinGecko, Yahoo Finance, FRED, Fear/Greed index, mempool.space.
- `rss/`: 20+ news feeds through a single module.

Each adapter normalizes into topic-tagged bus messages (`ohlc.*.*`, `lob.*.*`, `trade.*.*`, `news`, `perp.*.*`, `indicator.*`).

## Bus and backpressure

Two things, in two different places:

- `server/internal/bus/bus.go` — topic pub/sub with per-topic ring buffers so a late-joining subscriber gets history. No credit scheme; overflow drops old entries in the ring.
- `server/cmd/notbbg-server/relay.go` — per-client relay with credit-based flow control for *bulk* topics (OHLC backfill). Realtime topics bypass credits and send as soon as the client is writable. Credits are refilled by client acks; the design is GenStage-shaped, not a straight port.

The draft I had before put the credit scheme in the bus. It's in the relay, and it only gates bulk data.

## PQC layer

`server/internal/transport/pqc.go` implements an ML-KEM-768 handshake (via Cloudflare's `circl/kem/mlkem/mlkem768`), HKDF into a ChaCha20-Poly1305 session key, running *on top of* TLS 1.3 — not as TLS's own key exchange. The collector push path uses this double layer so a classical TLS break wouldn't immediately expose the replayed stream. Forward-looking rather than strictly necessary for public market data; the same transport backs config + secrets sync.

Secrets encryption (`server/internal/config/encrypted.go`): Argon2id → SHA3-256 sequential stretch → XChaCha20-Poly1305. Same shape as [rust-secure-memory](https://github.com/zeta1999/rust-secure-memory-public).

## Datalake

`server/internal/datalake/writer.go` subscribes to configured topic globs and writes Hive-partitioned JSONL under `type/exchange/instrument/date/`. Daily or hourly rotation via config. One file handle per active partition, append-only.

## AI panel

The seventh TUI panel shells out to `claude -p <prompt>` (the Claude Code CLI binary on the machine) with a 60-second timeout — `transport/http.go:handleAgentExec`. It's a thin wrapper, not an embedded LLM.

Shelling out to a code-agent CLI is, in my experience, the part that actually earns its keep for data-science-style use against a live terminal: the agent can read the Hive-partitioned JSONL under the datalake path, write a quick Go or Python script, grep the bus log, or pull a specific topic window on demand — all without the terminal needing to ship a bespoke query language. The same mechanism works with Codex / any agent CLI that respects `-p` semantics; Claude Code is the default because that's what's on my `$PATH`.

## What I'd do next

- Normalize the feed-side backpressure too: right now a misbehaving CCXT connection can fan out to the bus faster than the datalake writer drains. A per-publisher token bucket or ring-rejection counter would at least surface that.
- Back the ring buffer with a tiered on-disk store for longer replay windows than memory allows.
- Put a public schema on the bus message payloads (the proto/ dir has some of this already) and use it as a pluggability point for external consumers instead of re-subscribing over HTTP.
- Windows client: I only run the TUI on Linux/macOS day-to-day.

Code @ [github.com/zeta1999/this-is-not-bbg-public](https://github.com/zeta1999/this-is-not-bbg-public).
