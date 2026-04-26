# Discord Notifications

**Priority:** Must
**Status:** Draft
**Date:** 2026-04-25
**Related Features:** [features/daily-evaluation.md](daily-evaluation.md), [features/accuracy-tracking.md](accuracy-tracking.md)

## Overview

Discord is the sole user interface for v1. Every meaningful event in the system — new predictions, daily evaluations, prediction changes, expiration scorecards, accuracy summaries, and errors — is communicated through Discord webhook messages. The notifications must be concise, scannable, and informative enough that the user never needs to check logs or a database to understand what the system is doing.

## Requirements

**FR-001: New Prediction Notification**
When a new prediction is created (initial or after superseding an old one), the system shall send a Discord message containing: direction, price range, confidence score, current price, reasoning summary (abbreviated), and expiration date.
Acceptance: The message renders cleanly in Discord with embedded formatting. All fields are present and human-readable.
Priority: Must

**FR-002: On-Track Notification**
When the daily evaluation confirms the active prediction is still on track, the system shall send a brief Discord message containing: current price, predicted range, how far into the 30-day window the prediction is, and a one-line rationale for why it's still on track.
Acceptance: The message is short — no more than a few lines. It confirms the system ran without requiring the user to parse a wall of text.
Priority: Must

**FR-003: Outlook Changed Notification**
When the daily evaluation generates a new prediction, the system shall send a Discord message containing: the old prediction summary, what changed, and the full new prediction details.
Acceptance: The message clearly separates old vs. new and highlights the reason for the change.
Priority: Must

**FR-004: Expiration Scorecard**
When a prediction expires and is scored, the system shall send a Discord message containing: the original prediction details, the actual price at expiration, direction result (hit/miss), range result (hit/miss), and running accuracy stats if sample size is sufficient.
Acceptance: The scorecard is clear and definitive — no ambiguity about whether the prediction was right or wrong.
Priority: Must

**FR-005: Error Alert**
When the daily loop fails for any reason, the system shall send a Discord message containing: what failed, at which step, and the error message. The alert should be visually distinct from normal notifications (e.g., red embed color).
Acceptance: Errors are immediately visible and distinguishable from normal messages. The user knows the system had a problem without checking logs.
Priority: Must

**FR-006: Weekly Accuracy Summary**
The system shall send a weekly Discord message (Sundays) with aggregate accuracy metrics as defined in the accuracy tracking feature.
Acceptance: Weekly summary is sent even if no new predictions were scored (showing current stats with a note that nothing changed). Skipped only if zero predictions have been scored total.
Priority: Must

**FR-007: Webhook Configuration**
The Discord webhook URL shall be configurable via environment variable, not hardcoded.
Acceptance: Changing the webhook URL requires only updating the environment variable and restarting the service.
Priority: Must

## Scope

**In Scope:** Discord webhook integration. Six notification types (new prediction, on-track, outlook changed, expiration scorecard, error, weekly summary). Embedded message formatting.

**Out of Scope:** Discord bot (slash commands, interactive buttons). Two-way interaction (receiving commands from Discord). Multiple channel support. Notification preferences or filtering. Mobile push notifications.

## User Workflows

#### Daily Check-In
**Trigger:** User glances at their Discord channel.
**Steps:**
1. User sees today's notification — either "on track" (brief) or "outlook changed" (detailed).
2. User reads the key metrics: current price, predicted range, confidence.
3. User decides whether to act on the information.
**Expected Outcome:** User gets a complete picture of the tool's current thesis in under 30 seconds of reading.

## Open Questions

- Should notifications include a chart or visual (e.g., a price chart image showing the predicted range)? This would require generating images server-side, which adds complexity but significantly improves scannability.
- Should different notification types go to different Discord channels, or all to one?

## Changelog

### v1 — 2026-04-25
- Initial draft
