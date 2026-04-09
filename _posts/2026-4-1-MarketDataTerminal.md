---
layout: post
title: "A Market Data Terminal From Scratch"
---

[notbbg](https://github.com/zeta1999/this-is-not-bbg-public) is a real-time market data terminal that aggregates feeds from 17+ sources into a single interface. TUI for the terminal, React Native for mobile. The name is a joke, obviously.

## Why Build One

Bloomberg terminals cost $24K/year. For crypto and cross-asset monitoring — where you need CEX feeds, DEX feeds, FX, commodities, news, and on-chain data in one place — there's no single product that does it well. You end up with 15 browser tabs and a Discord window. I wanted one screen.

## Architecture

The core is a feed aggregation layer with 17 adapters:

**Exchanges**: Binance, OKX, Bybit, Bitget, Hyperliquid (WebSocket streams for orderbook and trades)
**Aggregators**: CoinGecko, Yahoo Finance (REST polling for broader market data)
**News**: 20+ RSS feeds from crypto and financial news sources
**On-chain**: Block explorers and DeFi protocol APIs

Each adapter runs independently and pushes normalized messages into a central event bus. The bus implements credit-based backpressure — inspired by Erlang's GenStage — so a slow consumer (say, the chart renderer) doesn't cause the feed handlers to buffer unboundedly or drop data.

## The Seven Panels

The TUI interface has seven panels that you can navigate between:

1. **OHLC charts** — multi-timeframe candlesticks with basic indicators
2. **LOB** — live order book depth with bid/ask visualization
3. **News feed** — aggregated and deduplicated across RSS sources
4. **Alerts** — configurable price/volume/spread alerts
5. **Feed monitor** — health status and latency for each adapter
6. **Server log** — raw event stream for debugging
7. **AI agent** — embedded Claude terminal for ad-hoc queries against the data

## Post-Quantum Encrypted Backup

The remote collector uses TLS 1.3 with ML-KEM-768 for key exchange. Market data streams to a remote server over this channel, written to a Hive-partitioned JSONL datalake. The encryption is overkill for market data, but the same infrastructure backs up configuration, API keys, and alert rules — where PQ encryption is forward-looking insurance.

Secrets (API keys, auth tokens) are stored locally encrypted with Argon2id key derivation.

## What's Next

The terminal feeds directly into [gpu-backtest](https://github.com/zeta1999/gpu-backtest-public), a GPU-accelerated backtesting engine I'm building in parallel. Recent work there includes a Monte Carlo scenario engine for robust meta-parameter optimization, synthetic data generation for reproducible testing, Metal/OpenCL backend parity with CUDA, and a growing library of classical HFT strategies written in a custom DSL. The idea is that notbbg collects and normalizes the data, and gpu-backtest consumes it for strategy development.

The repo is at [github.com/zeta1999/this-is-not-bbg-public](https://github.com/zeta1999/this-is-not-bbg-public).
