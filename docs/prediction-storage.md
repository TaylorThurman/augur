# Prediction Storage

**Priority:** Must
**Status:** Draft
**Date:** 2026-04-25
**Related Features:** [features/prediction-engine.md](prediction-engine.md), [features/daily-evaluation.md](daily-evaluation.md), [features/accuracy-tracking.md](accuracy-tracking.md)

## Overview

Prediction storage is the system of record for every prediction the tool has ever made. It uses Postgres on Railway and enforces one core invariant: predictions are immutable once created. When the outlook changes, a new prediction is created — the old one is never edited, only its status field is updated to reflect that it was superseded, expired, or scored.

This immutability is what makes accuracy tracking meaningful. If predictions could be retroactively edited, the track record would be meaningless. The storage layer enforces the integrity that the entire tool's credibility rests on.

## Requirements

**FR-001: Prediction Record Schema**
The system shall store each prediction with the following fields: unique ID, creation timestamp, direction (bullish/bearish/neutral), price at prediction time, price range low, price range high, time horizon (expiration date = creation + 30 days), confidence score (1–10), reasoning summary, data sources used, data gaps noted, status (active/superseded/expired), and final score fields (direction hit, range hit) populated at expiration.
Acceptance: All fields are present in the database schema. No prediction can be stored with null values in required fields.
Priority: Must

**FR-002: Prediction Immutability**
Once a prediction record is created, only the following fields may be updated: status (from active to superseded or expired) and final score fields (populated when the prediction expires or is superseded). All other fields are immutable.
Acceptance: The application layer enforces immutability. There is no "update prediction" function that modifies direction, price range, confidence, or reasoning.
Priority: Must

**FR-003: Prediction Lineage**
When a new prediction supersedes an old one, the new prediction shall reference the old prediction's ID. This creates a chain showing how the outlook evolved over time.
Acceptance: Given any prediction, the system can retrieve the full chain of predictions that preceded it.
Priority: Must

**FR-004: Daily Evaluation Log Storage**
The system shall store a log entry for each daily evaluation run, including: timestamp, data snapshot summary, evaluation result (on track / new prediction / error), and reference to the active prediction at the time of the run.
Acceptance: Evaluation logs are queryable by date range. Each log entry links to the prediction it evaluated.
Priority: Must

**FR-005: Active Prediction Query**
The system shall support an efficient query for the current active prediction. At most one prediction should have status "active" at any time.
Acceptance: Querying the active prediction returns exactly zero or one result. The system enforces that creating a new active prediction first supersedes the existing one.
Priority: Must

**FR-006: Historical Query**
The system shall support querying all predictions within a date range, optionally filtered by status (active, superseded, expired) and final score (hit, miss).
Acceptance: Historical queries return complete prediction records. This supports the accuracy tracking feature.
Priority: Should

## Scope

**In Scope:** Postgres schema design. Prediction CRUD (create, read, status update only). Evaluation log storage. Lineage tracking.

**Out of Scope:** Raw data snapshot archival (open question — see data-collection.md). Dashboard query optimization. Data export functionality. Backup and disaster recovery (relies on Railway's built-in Postgres backups for v1).

## User Workflows

#### Store New Prediction
**Trigger:** Prediction engine produces a new prediction (initial or after re-evaluation).
**Steps:**
1. If an active prediction exists, update its status to "superseded" and populate its final score based on current price at time of supersession.
2. Insert the new prediction with status "active."
3. If the new prediction supersedes an old one, set the predecessor_id field.
**Expected Outcome:** Database has exactly one active prediction. The old prediction is preserved with its final status.

#### Score Expired Prediction
**Trigger:** Daily evaluation loop detects an active prediction past its 30-day expiration.
**Steps:**
1. Fetch current Bitcoin price.
2. Compare to prediction: was direction correct? Was final price inside the predicted range?
3. Update prediction status to "expired" and populate final score fields.
**Expected Outcome:** Expired prediction has a definitive hit/miss score that feeds into accuracy tracking.

## Open Questions

- Should the raw data snapshot from each collection run be stored in the database? Pro: enables retroactive analysis of what data the model had when it made each prediction. Con: increases storage significantly over time.
- When a prediction is superseded (not expired naturally), should it be scored against the price at the moment of supersession, or left unscored since it didn't run its full 30 days?

## Changelog

### v1 — 2026-04-25
- Initial draft
