# Data Collection Pipeline

**Priority:** Must
**Status:** Draft
**Date:** 2026-04-25
**Related Features:** [features/prediction-engine.md](prediction-engine.md), [features/daily-evaluation.md](daily-evaluation.md), [features/prediction-storage.md](prediction-storage.md)

## Overview

The data collection pipeline gathers the raw quantitative inputs the prediction engine needs to form a thesis. It runs as the first step of every daily evaluation, assembling a structured data snapshot that gets passed to the LLM for analysis.

For MVP, the pipeline integrates with **two external APIs only** — CoinGecko (price/history) and mempool.space (on-chain) — plus one local computation (halving cycle math). Macro indicators, institutional ETF flows, and news/regulatory context are explicitly **not** collected by this pipeline; they are gathered by the prediction engine itself via Claude's web search at synthesis time. This split keeps the pipeline narrow, deterministic, and cheap to operate, while letting the LLM handle the qualitative/unstructured categories where building a free-tier scraper would dominate the work.

The quality of predictions is bounded by the quality of data collection. If a category is missing or stale, the prediction engine should know about it and factor that into its confidence score.

## Requirements

**FR-001: Price Data Collection**
The system shall collect current Bitcoin price, 24h change, 7d change, 30d change, and recent price history (daily candles for at least 90 days) from CoinGecko's free public API.
Acceptance: Price data is returned with timestamps and stored in a structured format. Spot-check against CoinGecko's website shows data is accurate within 1%.
Priority: Must

**FR-003: On-Chain Flow Collection**
The system shall collect Bitcoin on-chain signals — including mempool/fee state and recent block-level activity that can be derived as a proxy for accumulation vs distribution — from mempool.space's free public API.
Acceptance: On-chain data includes at minimum a directional read on recent network activity over the trailing 7 days. If mempool.space is unavailable, this category is reported as a gap (see FR-007).
Priority: Must

**FR-004: Supply Event Context**
The system shall include current halving cycle positioning — days since last halving, estimated days to next halving, and a brief historical context note for where this point in the cycle has typically fallen in past cycles. This is a local computation, not an external API call.
Acceptance: Halving data is accurate (computed from known block heights and average block times). Historical cycle comparison is included.
Priority: Must

**FR-007: Data Gap Handling**
The system shall track which data categories were successfully collected and which failed or returned stale data. This metadata is passed to the prediction engine alongside the data itself.
Acceptance: If any data source fails, the pipeline still completes with available data and includes a structured "gaps" field listing what's missing and why. If price data (FR-001) specifically is unavailable, the run is aborted and a Discord alert is sent — price is the one non-negotiable input.
Priority: Must

**FR-009: Raw Snapshot Persistence**
The system shall persist the full assembled data snapshot for each daily run alongside the prediction it produced. The snapshot is stored as a JSONB column on the prediction record itself (not a separate table) so that the inputs and the resulting prediction travel together.
Acceptance: Every prediction record contains a `raw_snapshot` JSONB field holding the complete data passed to the LLM (price block, on-chain block, halving block, gaps report, and any other inputs). The snapshot is queryable and is preserved immutably for the life of the prediction record.
Priority: Must
Rationale: Enables debugging anomalous predictions, replaying historical days against revised prompts, and detecting silent data-source drift.

**FR-010: Stale Data Handling**
The system shall pass through stale data (older than the freshness threshold for its category) to the prediction engine with an explicit `stale: true` flag rather than treating it as a missing gap. The LLM is responsible for deciding how much weight to place on stale inputs.
Acceptance: When a source returns data older than its freshness threshold, the snapshot includes the data, the timestamp, and a `stale: true` marker. The category is not added to the gaps list.
Priority: Must

## Scope

**In Scope:**
- Bitcoin only.
- Three input categories handled by this pipeline: price (CoinGecko), on-chain (mempool.space), halving cycle (local computation).
- Daily collection frequency, triggered by the daily evaluation cron.
- Raw snapshot persistence on the prediction record.

**Out of Scope (delegated to the prediction engine via Claude web search at synthesis time):**
- Macro indicators (Fed policy sentiment, DXY, broad risk-on/risk-off conditions). Previously specified as FR-002; removed from this pipeline because there is no clean free API and the LLM can summarize current macro context directly.
- Institutional ETF flows (Bitcoin spot ETF net inflow/outflow). Previously specified as FR-005; removed because the de facto source (Farside Investors) has no API, and the LLM can pull current numbers via web search.
- News and regulatory headlines. Previously specified as FR-006; removed because Claude web search at prediction time is simpler than maintaining a news aggregator integration.

**Out of Scope (entirely):** Real-time or intraday data collection. Paid API integrations. Multi-coin data collection. Sentiment analysis from social media (Twitter/X, Reddit). Technical indicator computation (moving averages, RSI, etc.).

## User Workflows

#### Daily Data Collection
**Trigger:** Daily cron fires the evaluation loop.
**Steps:**
1. Pipeline calls CoinGecko for price + 90d daily candles.
2. Pipeline calls mempool.space for on-chain signals.
3. Pipeline computes halving cycle context locally.
4. Pipeline assembles a unified snapshot containing all three blocks plus a gaps report and stale-data flags.
5. Snapshot is passed to the prediction engine, which augments it with macro/ETF/news context via Claude web search.
6. After the prediction is generated, the snapshot is persisted as the `raw_snapshot` JSONB field on the new prediction record.
**Expected Outcome:** A complete (or partially complete with documented gaps) data snapshot ready for analysis, and a persisted record of exactly what the LLM saw on this day.
**What Could Go Wrong:** A source is down or has changed its API. The system logs the failure, marks the gap, and continues. If price data specifically is unavailable, the run is skipped entirely and a Discord alert is sent — price is the one non-negotiable input.

## Open Questions

_None. Previous open questions resolved in v1.1:_
- _Specific APIs:_ CoinGecko (price), mempool.space (on-chain). Macro/ETF/news delegated to LLM web search.
- _Snapshot storage:_ Yes — stored as a JSONB column on the prediction record (FR-009).
- _Macro free API:_ None used. Sourced via LLM web search at prediction time.

## Changelog

### v1.1 — 2026-04-25
- MVP simplification: pipeline scoped to two external APIs (CoinGecko, mempool.space) plus local halving math.
- Removed FR-002 (Macro), FR-005 (ETF), FR-006 (News) from this pipeline; documented in Out of Scope as delegated to the prediction engine via Claude web search.
- Removed FR-008 (Rate Limit Compliance) — over-specified for a daily run against two free sources; if reintroduced later it belongs in the architecture blueprint, not the product spec.
- Added FR-009 (Raw Snapshot Persistence) — full daily snapshot stored as a JSONB column on the prediction record.
- Added FR-010 (Stale Data Handling) — stale data passed through with a flag rather than treated as a gap.
- Resolved all prior open questions.

### v1 — 2026-04-25
- Initial draft.
