# Accuracy Tracking

**Priority:** Must
**Status:** Draft
**Date:** 2026-04-25
**Related Features:** [features/prediction-storage.md](prediction-storage.md), [features/discord-notifications.md](discord-notifications.md)

## Overview

Accuracy tracking is the feedback loop that turns this from a prediction generator into a provably useful (or provably useless) research tool. It computes aggregate metrics across the prediction history and surfaces them in Discord summaries. Without this feature, the tool is just an opinion machine — with it, the tool has a track record.

Tracking is split into two layers. Layer 1 (directional hit rate) is the primary metric and the one that determines whether the tool is worth using for swing trading. Layer 2 (range accuracy) is a refinement that matters for position sizing and exit planning, but only becomes relevant after Layer 1 proves out.

## Requirements

**FR-001: Directional Hit Rate Calculation**
The system shall compute the percentage of scored predictions (expired or superseded) where the directional call was correct. Bullish is correct if price at scoring time is higher than price at prediction time. Bearish is correct if price is lower. Neutral is scored separately or excluded from directional accuracy.
Acceptance: Given N scored predictions, the system returns a hit rate as a percentage with the count of hits and misses. Neutral predictions are tracked but excluded from the directional hit rate denominator.
Priority: Must

**FR-002: Range Accuracy Calculation**
The system shall compute the percentage of scored predictions where the final price fell within the predicted range.
Acceptance: Given N scored predictions, the system returns range accuracy as a percentage.
Priority: Should

**FR-003: Confidence Calibration**
The system shall compute directional hit rate grouped by confidence score buckets (1–3, 4–6, 7–10). This reveals whether high-confidence predictions actually hit more often than low-confidence ones.
Acceptance: Given a sufficient sample (10+ predictions per bucket), the system shows hit rates per bucket. If high-confidence predictions don't outperform low-confidence ones, the confidence scoring needs recalibration.
Priority: Should

**FR-004: Rolling Accuracy**
The system shall compute accuracy metrics over rolling windows (last 10 predictions, last 20, all-time) to detect whether performance is improving, degrading, or stable.
Acceptance: Rolling metrics are computable and included in periodic Discord summaries.
Priority: Should

**FR-005: Minimum Sample Gate**
The system shall not present accuracy metrics as meaningful until at least 20 predictions have been scored. Below 20, the sample is too small for statistical significance and could be misleading.
Acceptance: Discord summaries include accuracy metrics only after 20+ scored predictions. Before that, they show the running count with a note that the sample is not yet significant.
Priority: Must

**FR-006: Weekly Accuracy Summary**
The system shall include a weekly accuracy summary in Discord notifications (e.g., every Sunday). This includes: total predictions scored, directional hit rate, range accuracy, and confidence calibration (if sample size is sufficient).
Acceptance: A weekly summary is sent to Discord. The summary is skipped if no new predictions were scored that week.
Priority: Must

## Scope

**In Scope:** Directional hit rate, range accuracy, confidence calibration, rolling windows, weekly summaries. All computed from stored prediction records.

**Out of Scope:** PnL simulation (computing hypothetical profit if trades were taken). Sharpe ratio or other portfolio metrics. Comparison against benchmark strategies (buy-and-hold, etc.). Statistical significance testing beyond the minimum sample gate.

## User Workflows

#### Weekly Accuracy Review
**Trigger:** Weekly cron (Sunday) or triggered as part of the daily loop on Sundays.
**Steps:**
1. Query all scored predictions.
2. Compute directional hit rate (all-time and rolling last 10/20).
3. Compute range accuracy.
4. Compute confidence calibration if sample is sufficient.
5. Format into a Discord summary.
6. Send to Discord.
**Expected Outcome:** User receives a clear, concise scorecard showing how the tool is performing.

## Open Questions

- Should superseded predictions count toward accuracy metrics? They didn't run their full 30-day window, so scoring them is arguably unfair. But excluding them means only predictions where the outlook never changed get scored, which biases toward stable periods.
- Is 20 predictions the right minimum sample gate, or should it be higher (e.g., 30)?

## Changelog

### v1 — 2026-04-25
- Initial draft
