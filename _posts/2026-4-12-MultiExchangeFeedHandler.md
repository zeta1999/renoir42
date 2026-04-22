---
layout: post
title: "notbbg feed adapters: what's actually in the ~17 of them (updated)"
---

Follow-on to the [notbbg terminal post]({% post_url 2026-4-1-MarketDataTerminal %}). This one is the feed-adapter layer, described honestly against the code rather than as a sales page.

The earlier draft of this post made several features sound more sophisticated than they are — exponential backoff, sequence-gap resync, clock-drift handling, queued prioritised rate limiting, encrypted-at-rest datalake. None of those are in the repo. This is what's actually there.

## The ~17 "feeds" are mostly not exchanges

`server/internal/feeds/` is organised by source *kind*, not by "exchange":

- `ccxt/` — 6 centralised venues over WebSocket: Binance, OKX, Bybit, Bitget, Coinbase, Kraken.
- `dex/` — Hyperliquid (perps), plus Jupiter, Raydium, Uniswap HTTP clients, plus `defi_protocols.go` / `more_protocols.go`.
- `world/` — CoinGecko, Yahoo Finance, FRED, Fear/Greed index, mempool.space. These are macro / aggregator, not exchanges.
- `rss/` — a single module sweeping 20+ news feeds.

So "17 exchanges" was wrong. It's ~17 adapters across CEX, DEX, macro, and news.

## What each CCXT adapter actually does

Looking at `binance.go`, `bybit.go`, `okx.go`, `bitget.go`, `coinbase.go`, `kraken.go`, they're the same shape:

1. Open a WebSocket to the venue, subscribe to the requested topics (orderbook / kline / trades / tickers).
2. Parse incoming JSON into Go structs.
3. Convert into `feeds.LOBSnapshot` / `feeds.Trade` / etc., push onto the bus with a topic like `lob.binance.BTCUSDT`.
4. If the connection drops, `time.Sleep(5 * time.Second)` and reconnect. That is the entire backoff policy — a fixed five seconds, not exponential, not jittered.
5. Rate limiting is `time.Sleep(time.Second / max(rateLimit, 1))` between REST calls. No priority queue, no per-instrument weighting, no graceful degradation when throttled — just a fixed-interval throttle.

Things you might expect and that are **not** there:

- No FIX protocol support. All CCXT adapters are WebSocket + JSON.
- No multicast UDP adapter. Everything is TCP/WebSocket.
- No sequence-gap detection. Grep for `sequence`, `lastUpdateId`, `update_id`, `seqnum` returns nothing in the adapters. If a delta is missed between snapshots, the downstream book is wrong until the next snapshot overwrites it. This is the single biggest gap if you care about book correctness.
- No clock-drift handling. No NTP integration, no per-adapter offset estimation.
- No auth-token refresh flows (the adapters only subscribe to public channels).

So the CCXT layer is a straightforward WebSocket-subscribe-and-normalize, not a hardened feed handler.

## DEX / HTTP adapters

`dex/` and `world/` are HTTP-polling shaped: a tick interval, a `GET`, parse, push to bus. Same idea, different transport. Jupiter and Raydium pull DEX trades; Uniswap is on-chain via its subgraph-shaped API. `world/mempool.go` polls mempool.space for L1 fee data; `world/fred.go` pulls macro series; `world/feargreed.go` is a daily sentiment pull.

The common thread: polling loops with a ticker, no backpressure from the bus back to the poller. If the bus is slow, the ring buffer in `bus/bus.go` drops oldest entries — the poller doesn't slow down.

## Normalised schema

Topics land on the bus in the shape `{type}.{venue}.{instrument}` — e.g. `lob.binance.BTCUSDT`, `trade.bybit.ETHUSDT`, `ohlc.yahoo.AAPL`. Payloads are Go structs under `server/internal/feeds/` with `price` / `qty` / `timestamp` as normalised fields. That normalisation *is* the meaningful work the adapters do: timestamps into nanoseconds, strings-of-floats into `float64`, instrument names into a lowercase canonical form.

## Clients

Three, not two: TUI (Bubble Tea), Desktop (Electron), Phone (React Native, marked experimental). The last post already corrected this — repeating it here because the earlier draft of this post repeated the same miss.

## Datalake is not encrypted at rest

Correcting the previous draft directly: `server/internal/datalake/writer.go` writes plain Hive-partitioned JSONL (or CSV) under `type/exchange/instrument/date/`. No encryption applied at the file level. The ML-KEM-768 layer in `server/internal/transport/pqc.go` is for the **transport** between the main server and an optional remote collector — it protects the stream in flight, not the files on disk.

If you want encryption at rest, it's currently your filesystem's job (e.g. LUKS / FileVault), not notbbg's.

## What I'd do next

- Sequence-gap detection on venues that expose `u`/`U`/`lastUpdateId` (Binance, Bybit). Detect the gap, hit the REST snapshot endpoint, rebuild the side, resume applying deltas. This is the single biggest correctness improvement.
- Exponential backoff with jitter on reconnect. Five-second fixed sleep thunders on outages.
- Bus-side credit flow control back to adapters so a slow datalake writer doesn't invisibly drop old ring-buffer entries.
- Actual rate-limit budgets modelled per endpoint (weighted on Binance, segmented on Coinbase), not a global sleep divider.
- If the datalake ever holds position/order data rather than just public market data, an at-rest encryption layer (XChaCha20-Poly1305 with a session key, same shape as `rust-secure-memory`) rather than outsourcing to the filesystem.

Code @ [github.com/zeta1999/this-is-not-bbg-public](https://github.com/zeta1999/this-is-not-bbg-public).
