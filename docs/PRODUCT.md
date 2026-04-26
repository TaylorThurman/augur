# Crypto Oracle — Product Overview

**Version:** v1
**Date:** 2026-04-25
**Status:** Draft

## Problem Statement & Vision

Crypto swing trading requires synthesizing a wide range of signals — macro conditions, on-chain flows, supply dynamics, institutional activity, and regulatory news — into a directional thesis. Doing this manually every day is time-consuming, inconsistent, and prone to recency bias. Traders either over-index on whatever headline they saw last or skip the analysis entirely and trade on gut feel.

Crypto Oracle is an autonomous research tool that performs this synthesis daily using structured data collection and LLM-powered analysis. It produces directional predictions with price ranges and confidence scores, then re-evaluates those predictions every day against new data. When the outlook changes, it generates a new prediction while preserving the original — building a growing track record that can be measured for accuracy over time.

The end state is a tool that produces swing trade signals with a measurable, provable directional hit rate. If the tool can consistently beat 60% directional accuracy over a meaningful sample, it becomes a genuine edge. If it can't, the prediction log will show that clearly rather than hiding behind selective memory.

## Users & Stakeholders

**Primary User:** Solo trader (Taylor). The only user for v1. Interacts with the system passively — receives Discord notifications, reviews predictions and accuracy metrics. Does not interact with a dashboard or UI in v1.

**Future Users:** None planned. This is a personal research tool. If the prediction engine proves accurate, it may eventually feed signals into an existing autonomous trading bot, but that integration is out of scope for now.

## Constraints

**C-001: Free Data Sources Only**
All market data, on-chain metrics, and news must come from free/public APIs. No paid subscriptions. This limits data granularity and rate limits but keeps operating costs at zero during the proving phase.

**C-002: Railway Deployment**
The system deploys on Railway's free tier ($5/month credit). This constrains compute to a single service with modest resource allocation. The system must be lightweight enough to run within these limits.

**C-003: Claude API Cost**
LLM analysis uses the Anthropic Claude API. Each daily evaluation requires at least one API call with potentially large context (price data, on-chain metrics, news summaries). Cost should be monitored but is accepted as the primary operating expense.

**C-004: Single Coin Scope**
v1 supports Bitcoin only. The architecture should not preclude multi-coin expansion, but no effort should be spent on abstraction or generalization until the single-coin pipeline is proven.

**C-005: No Frontend**
v1 has no dashboard or web UI. All output is delivered via Discord. A frontend may be added later but is explicitly out of scope.

## Quality Attributes

**Reliability:** The daily cron must run every day without manual intervention. If a data source is unavailable, the system should degrade gracefully — log the gap, use available data, and note reduced confidence rather than failing silently or skipping the run entirely.

**Data Integrity:** Prediction records are immutable once created. The original prediction is never modified or deleted when a new prediction supersedes it. The full history must be preserved for accuracy measurement.

**Latency:** Not critical. The daily evaluation can take minutes if needed. There is no real-time requirement.

**Observability:** Every run should produce a structured log entry. Failures, data gaps, and prediction changes should all be visible in Discord notifications so the user knows the system is healthy without checking logs.

## Success Criteria

**SC-001:** The system runs autonomously for 30+ consecutive days without manual intervention or failure.

**SC-002:** After 20+ predictions, directional hit rate can be calculated and is visible in Discord summaries.

**SC-003:** Directional hit rate exceeds 55% over a 50+ prediction sample. Below 55% indicates the tool is not meaningfully better than chance and the approach needs rethinking.

**SC-004:** Every prediction record includes: direction, price range, confidence score, reasoning summary, data sources used, and timestamp. No prediction is created without all fields populated.

**SC-005:** When the daily evaluation generates a new prediction, the original prediction is preserved with its final status (hit, miss, or superseded) and the relationship between old and new predictions is tracked.

## Assumptions & Dependencies

**A-001:** Free APIs (CoinGecko, Glassnode free tier, etc.) will remain available and provide sufficient data for meaningful analysis. If a key source goes down or restricts access, the system's prediction quality may degrade.

**A-002:** Claude API will remain available and capable of the synthesis task. Model changes or API outages directly impact the system.

**A-003:** Bitcoin will continue to be influenced by the five fundamental categories identified (macro, on-chain, supply events, institutional flows, regulatory/news). If market dynamics shift fundamentally, the analysis framework may need updating.

**A-004:** A 30-day prediction window is appropriate for swing trading. This assumption will be validated by the accuracy tracking — if predictions consistently become stale before 30 days, the window may need shortening.

**A-005:** Railway's free tier provides sufficient uptime and compute for a once-daily cron job with modest API calls and database writes.

## Changelog

### v1 — 2026-04-25
- Initial draft
