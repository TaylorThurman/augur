# Daily Evaluation Loop

**Priority:** Must
**Status:** Draft
**Date:** 2026-04-25
**Related Features:** [features/data-collection.md](data-collection.md), [features/prediction-engine.md](prediction-engine.md), [features/prediction-storage.md](prediction-storage.md), [features/discord-notifications.md](discord-notifications.md)

## Overview

The daily evaluation loop is the orchestrator. It is the single entry point for all system activity — a cron job that fires once per day, determines what needs to happen, and executes the appropriate flow. There is no other trigger in the system. No manual endpoints, no event-driven hooks. The cron is the heartbeat.

The loop follows a simple decision tree: check if an active prediction exists. If not, create one. If yes, evaluate it against fresh data. If the evaluation says the outlook has changed, create a new prediction and supersede the old one. Along the way, check if any predictions have reached their 30-day expiration and score them.

## Requirements

**FR-001: Daily Cron Execution**
The system shall execute the evaluation loop once per day at a configured time.
Acceptance: The loop runs every day within a 5-minute window of the configured time. Missed runs are detected and logged.
Priority: Must

**FR-002: No-Prediction Path**
When no active prediction exists in the database, the system shall trigger the data collection pipeline followed by the prediction engine in initial prediction mode.
Acceptance: A new active prediction is created and stored. Discord notification is sent.
Priority: Must

**FR-003: Active-Prediction Path**
When an active prediction exists, the system shall trigger the data collection pipeline followed by the prediction engine in re-evaluation mode.
Acceptance: The active prediction is either confirmed ("on track") or superseded by a new prediction. Discord notification is sent reflecting the outcome.
Priority: Must

**FR-004: Prediction Expiration Check**
Before running the evaluation, the system shall check if any active predictions have reached their 30-day expiration. Expired predictions are scored (hit or miss based on current price vs. predicted direction and range) and marked as "expired."
Acceptance: An expired prediction is never re-evaluated. Its final status reflects whether the directional call and price range were accurate at expiration.
Priority: Must

**FR-005: Run Logging**
Every execution of the loop shall produce a structured log entry including: timestamp, path taken (no-prediction / evaluation / new-prediction / error), data sources successfully queried, data gaps, prediction outcome, and any errors encountered.
Acceptance: Logs are stored in the database. Any run that produced an error is distinguishable from a successful run.
Priority: Must

**FR-006: Error Recovery**
If the loop fails partway through (data collection error, API timeout, malformed LLM response), it shall not leave the system in an inconsistent state. No partial predictions are stored. The failure is logged and a Discord alert is sent. The system retries on the next daily run.
Acceptance: After any failure, the database state is identical to what it was before the run started. The next day's run picks up cleanly.
Priority: Must

## Scope

**In Scope:** Single daily cron. Decision tree logic. Expiration scoring. Error handling and logging.

**Out of Scope:** Manual trigger endpoints. Real-time or event-driven evaluation. Multiple runs per day. Retry logic within the same day (if it fails, it waits for tomorrow).

## User Workflows

#### Typical Day — Prediction On Track
**Trigger:** Cron fires at configured time.
**Steps:**
1. Check for expired predictions — none found.
2. Check for active prediction — found.
3. Run data collection pipeline.
4. Run prediction engine in re-evaluation mode.
5. Result: "on track."
6. Log the run.
7. Send Discord summary: "BTC prediction still on track. Current price $X, predicted range $Y–$Z. Confidence unchanged at N/10."
**Expected Outcome:** User receives a brief daily confirmation. No database changes beyond the run log.

#### Outlook Changed — New Prediction
**Trigger:** Cron fires at configured time.
**Steps:**
1. Check for expired predictions — none found.
2. Check for active prediction — found.
3. Run data collection pipeline.
4. Run prediction engine in re-evaluation mode.
5. Result: outlook changed, new prediction generated.
6. Old prediction marked "superseded." New prediction stored as "active."
7. Log the run.
8. Send Discord notification with both old prediction summary and new prediction details.
**Expected Outcome:** User receives a detailed notification showing what changed and why.

#### Prediction Expires
**Trigger:** Cron fires at configured time.
**Steps:**
1. Check for expired predictions — found one that hit its 30-day mark.
2. Score the expired prediction: compare current price to predicted direction and range.
3. Mark prediction as "expired" with final score (direction hit/miss, range hit/miss).
4. No active prediction now exists.
5. Run data collection pipeline.
6. Run prediction engine in initial prediction mode.
7. Store new active prediction.
8. Log the run.
9. Send Discord notification with expiration scorecard and new prediction.
**Expected Outcome:** User sees how the last prediction performed and gets a fresh one.

## Open Questions

- What time of day should the cron run? After US market close (4 PM ET / 3 PM CT) might be ideal since macro signals are freshest then, but Bitcoin trades 24/7 so the choice is somewhat arbitrary.
- Should the system detect missed runs (e.g., Railway had an outage) and compensate, or just accept the gap and run normally the next day?

## Changelog

### v1 — 2026-04-25
- Initial draft
