# Prediction Engine

**Priority:** Must
**Status:** Draft
**Date:** 2026-04-25
**Related Features:** [features/data-collection.md](data-collection.md), [features/prediction-storage.md](prediction-storage.md)

## Overview

The prediction engine is the analytical core of the system. It receives a structured data snapshot from the data collection pipeline and synthesizes it into a directional prediction with a price range, confidence score, and reasoning. It uses the Claude API to perform the synthesis — the LLM acts as the analyst, weighing signals across all five fundamental categories and producing a structured output.

The engine runs in two modes: initial prediction (when no active prediction exists) and re-evaluation (when an active prediction exists and needs to be assessed against new data). Both modes produce the same structured output format, but re-evaluation also includes an assessment of whether the current prediction still holds.

## Requirements

**FR-001: Initial Prediction Generation**
The system shall generate a prediction when no active prediction exists. The prediction includes: direction (bullish/bearish/neutral), price target range (low–high), 30-day time horizon from prediction date, confidence score (1–10), and a reasoning summary explaining which signals drove the call.
Acceptance: A prediction record with all required fields is produced and stored. The reasoning summary references specific data points from the collection snapshot.
Priority: Must

**FR-002: Structured LLM Prompt**
The system shall pass the data snapshot to Claude via a structured prompt that includes: all collected data organized by category, any data gaps flagged by the collection pipeline, the current Bitcoin price, and instructions to produce output in a defined JSON schema.
Acceptance: The prompt reliably produces valid JSON output matching the prediction schema. The prompt includes explicit instructions for the confidence score scale and what each level means.
Priority: Must

**FR-003: Re-evaluation Mode**
The system shall pass the current active prediction alongside the new data snapshot to Claude and ask for an assessment: is the prediction still on track, or has the outlook changed? If the outlook has changed, a new prediction is generated.
Acceptance: The re-evaluation produces one of two outcomes — "on track" (with a brief rationale) or "new prediction" (with a full prediction record). The decision is based on two triggers: price has broken outside the predicted range, or a major catalyst has materially changed the outlook.
Priority: Must

**FR-004: Confidence Score Calibration**
The system shall use a defined confidence scale where the score reflects how many signals align and how strong they are. A 7+ means most signals agree and data quality is high. A 4–6 means mixed signals or notable data gaps. A 1–3 means the call is speculative with significant uncertainty.
Acceptance: The confidence score definition is embedded in the LLM prompt so the model applies it consistently across predictions.
Priority: Must

**FR-005: Data Gap Awareness**
The system shall include data gap information in the LLM prompt so the model can factor missing data into its confidence score and reasoning. A prediction made with missing on-chain data, for example, should reflect that uncertainty.
Acceptance: When data gaps exist, the reasoning summary explicitly mentions them and the confidence score is adjusted downward.
Priority: Must

**FR-006: Prediction Schema Enforcement**
The system shall validate LLM output against the expected JSON schema before storing it. If the output is malformed, the system retries once. If the retry also fails, the run is logged as a failure and a Discord alert is sent.
Acceptance: Malformed LLM output never results in a corrupt or partial prediction record in the database. Failures are logged and alerted.
Priority: Must

## Scope

**In Scope:** Bitcoin predictions only. Claude API as the sole LLM provider. Two modes (initial and re-evaluation). Structured JSON output.

**Out of Scope:** Technical analysis or chart pattern recognition by the LLM. Multi-model ensemble (using multiple LLMs and averaging). Fine-tuning or custom model training. Backtesting predictions against historical data.

## User Workflows

#### First Run (No Existing Prediction)
**Trigger:** Daily cron fires, system detects no active prediction in the database.
**Steps:**
1. Data collection pipeline assembles snapshot.
2. Prediction engine constructs initial prediction prompt with snapshot data.
3. Claude API call returns structured prediction.
4. Output is validated against schema.
5. Prediction is stored with status "active."
6. Discord notification sent with the new prediction.
**Expected Outcome:** A new active prediction exists in the database and the user receives a Discord message with the full prediction details.
**What Could Go Wrong:** Claude API is down or returns malformed output. System retries once, then logs failure and sends Discord alert. No prediction is created — the system will try again on the next daily run.

#### Re-evaluation (Active Prediction Exists)
**Trigger:** Daily cron fires, system detects an active prediction.
**Steps:**
1. Data collection pipeline assembles snapshot.
2. Prediction engine constructs re-evaluation prompt with snapshot data AND the current active prediction.
3. Claude API call returns either "on track" or a new prediction.
4. If "on track": log the evaluation, send a brief Discord summary.
5. If "new prediction": store the new prediction as active, update the old prediction's status to "superseded," send Discord notification with both the old and new predictions.
**Expected Outcome:** The active prediction is either confirmed or replaced, and the user is notified.
**What Could Go Wrong:** The LLM may be overly sensitive or insensitive to changes — either generating new predictions too frequently (noise) or never changing (stubbornness). This will be tuned through prompt engineering based on observed behavior.

## Open Questions

- What Claude model should be used? Sonnet for cost efficiency, or Opus for higher quality analysis? The tradeoff is cost vs. prediction quality.
- Should the raw LLM response be stored alongside the parsed prediction for debugging and prompt tuning?
- How should the prompt handle conflicting signals (e.g., bullish on-chain flows but bearish macro)? Should it be instructed to default to neutral, or to weigh categories differently?

## Changelog

### v1 — 2026-04-25
- Initial draft
