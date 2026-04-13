---
layout: post
title: "Building a Multi-Exchange Feed Handler: 17 Exchanges, One Interface"
---

In a [previous post]({% post_url 2026-4-1-MarketDataTerminal %}) I introduced [notbbg](https://github.com/zeta1999/this-is-not-bbg-public). This post goes deeper into the feed handler architecture — how you normalize data from 17 different exchanges with different APIs, protocols, and quirks into a single coherent interface.

## The Diversity Problem

No two exchanges speak the same language. Some give you WebSocket streams of JSON. Some use FIX protocol. Some have REST-only APIs with rate limits. Some provide multicast UDP feeds for co-located clients. Some send full order book snapshots every second; others send incremental deltas and expect you to maintain state.

The data formats are equally inconsistent. Timestamps might be Unix milliseconds, microseconds, ISO 8601, or exchange-local time. Prices might be floats, strings, or integers with an implicit decimal point. Instrument identifiers follow no standard — the same asset is "BTCUSDT" on one exchange and "BTC-USDT" on another.

You need an adapter per exchange, and you need every adapter to produce the same normalized output.

## The Adapter Pattern

Each exchange gets its own adapter module that handles:

1. **Connection management** — WebSocket reconnection with exponential backoff, session keepalive, authentication token refresh
2. **Protocol handling** — parsing the exchange's specific message format (JSON, FIX, binary)
3. **Normalization** — converting to a common internal format: `(timestamp_ns, exchange, instrument, event_type, price, quantity, side)`
4. **Rate limiting** — respecting per-exchange API limits, queueing requests when throttled
5. **Clock synchronization** — converting exchange timestamps to a common nanosecond epoch, accounting for clock drift

The adapter interface is a trait (in Go terms, an interface) that every exchange module implements. The rest of the system — LOB construction, OHLC aggregation, alerting, storage — only sees normalized events.

## The Hard Parts

**Reconnection under load.** WebSocket connections drop. The exchange might restart, your network might blip, or the exchange might disconnect idle clients. The reconnection logic needs to:
- Detect the disconnect quickly (heartbeat timeout)
- Reconnect without losing data (request a snapshot after reconnection)
- Handle the gap between disconnect and reconnection (mark the book state as stale)
- Not thundering-herd all 17 adapters when your network comes back

**Sequence gaps.** Some exchanges provide sequence numbers on their delta feeds. If you see sequence 1001 after 999, you missed 1000. You need to detect this, request a fresh snapshot, and rebuild the book. If the exchange doesn't provide sequence numbers, you're relying on snapshot frequency to catch drift.

**Rate limits and backpressure.** REST-based exchanges impose rate limits (e.g., 1200 requests/minute). If you're polling 50 instruments at 1-second intervals, you're already at 3000 requests/minute. The adapter needs to batch requests, prioritize active instruments, and degrade gracefully when throttled.

## The Data Lake

Normalized events flow into a Hive-partitioned datalake — partitioned by exchange, instrument, and date. This serves two purposes:

1. **Historical replay** — feed the same data into gpu-backtest for backtesting
2. **Post-quantum encrypted backup** — the datalake is encrypted at rest with ML-KEM-768 derived keys, so historical market data is protected against future quantum attacks

The backup encryption might seem paranoid for market data, but the same infrastructure handles order flow and position data — and those are worth protecting.

## Terminal and Mobile

The TUI (terminal) interface renders live order books, OHLC charts, and aggregated views across exchanges. React Native provides a mobile interface for monitoring on the go. Both consume the same normalized event stream.

The repo is at [github.com/zeta1999/this-is-not-bbg-public](https://github.com/zeta1999/this-is-not-bbg-public).
