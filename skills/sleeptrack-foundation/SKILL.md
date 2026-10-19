---
name: sleeptrack-foundation
description: Core domain knowledge about the Asleep sleep tracking platform -- data model, metrics definitions, session lifecycle, error codes, and API conventions. Reference this skill to understand Asleep concepts before working with platform-specific skills.
---

# Sleeptrack Foundation

## What is Asleep?

Asleep is an audio-based sleep tracking platform. It uses the device microphone to capture ambient sound, then applies AI analysis to determine sleep stages, snoring, and overall sleep quality. No wearable hardware is required.

## How It Works

Audio-based sleep analysis proceeds in three stages:

1. **Capture**: The mobile SDK records ambient audio through the device microphone
   and uploads chunks every 30 seconds to the Asleep server.

2. **Analysis**: The server AI processes uploaded audio in analysis increments,
   classifying each 30-second interval into a sleep stage (Wake, Light, Deep, or REM)
   and detecting snoring episodes.

3. **Delivery**: Completed analysis results are available via REST API polling
   or webhook push notification. Preliminary results can be accessed during
   an active session for real-time UI updates.

## Valid Session Requirements

For the AI to produce meaningful sleep analysis, a session must meet these minimums:

- Enough valid sound data for the selected analysis. The public session contract describes less than 5 minutes as `TOO_SHORT_FOR_ANALYSIS`; missing or defective data can produce `TOO_MANY_DEFECTS_IN_SLEEP_STAGES`. Do not assume one numerical integrity threshold for all models.
- One active session per user at a time. Attempting to create another session before the previous one is closed produces `ERR_CREATE_CONFLICT` (Android code 22409).
- Microphone access must remain available throughout tracking. Loss of
  microphone access triggers ERR_MIC_PERMISSION.

## Session Lifecycle

The session lifecycle has two parallel state models: SDK states visible on the
client device, and API states tracked on the server.

### SDK Tracking Status (client-side)

Android and iOS expose `getTrackingStatus()` / `TrackingStatus` with nullable `sessionId`. This is not a shared IDLE/INITIALIZING/TRACKING enum. Android documents the session ID as available from `TrackingListener.onCreate` until the session closes. Use each pinned SDK's listener/delegate callbacks for setup, recording, interruption, and completion; do not infer server analysis state from local tracking status.

### API States (server-side)

These states are returned when querying sessions through the REST API:

| State | Meaning |
|---|---|
| OPEN | Session created, server is accepting audio uploads |
| CLOSED | Session terminated by client, AI analysis in progress |
| COMPLETE | Analysis completed; report fields can still be null or unavailable |

Transition: OPEN -> CLOSED -> COMPLETE. The CLOSED-to-COMPLETE transition
happens automatically once the AI finishes processing.

### Measurement and analysis ranges

The actual measurement window is preserved by required `measurement_start_time` and nullable `measurement_end_time`. Session `start_time` and nullable `end_time` reflect the selected analysis range. Get Session v4 adds a nullable `analysis` object for epoch boundaries and durations; use the v4 contract for range-aware timelines. Reanalysis can change a completed report, so refetch after success and invalidate cached report data. Check the API skill for public request fields and guidance on bounds within the recorded measurement window.

### Real-time Data Access

Analysis is delivered in increments that depend on the integration/model. Do not assume that every SDK/model polls on the same sequence boundary. For webhook inference events use `inference_seq_num` for ordering and deduplication.

## Sleep Data Metrics

Duration metrics use **seconds**. Per-session timestamps use ISO 8601 with an offset; Average Stats clock fields use HH:MM:SS on the wire. Convert clock times at the presentation boundary. Preserve null for unavailable metrics.

### Sleep Index

`sleep_index` is a nullable beta integer score documented in the 50-100 range. Do not assume a universal scoring formula or a clinical interpretation from this schema.

### Duration Metrics (seconds)

| Metric | Definition |
|---|---|
| time_in_bed | Time from the start of sleep measurement to its end; for ranged sessions, see v4 `analysis` |
| time_in_sleep_period | Sleep onset to final awakening |
| time_in_sleep | Actual time spent asleep (excludes wake periods within sleep period) |
| sleep_latency | Time from analysis start to first sleep onset |
| wakeup_latency | Time from final awakening to analysis end |
| light_latency | Time from sleep onset to first Light stage |
| deep_latency | Time from sleep onset to first Deep stage |
| rem_latency | Time from sleep onset to first REM stage |

### Sleep Efficiency

`sleep_efficiency` = time_in_sleep / time_in_bed (decimal ratio, e.g. 0.83 = 83%)

### Stage Durations (seconds)

| Metric | Definition |
|---|---|
| time_in_wake | Total wake time within the sleep period |
| time_in_light | Total Light sleep time |
| time_in_deep | Total Deep sleep time |
| time_in_rem | Total REM sleep time |

### Stage Ratios

Proportions relative to `time_in_sleep_period`:

| Metric | Definition |
|---|---|
| wake_ratio | time_in_wake / time_in_sleep_period |
| sleep_ratio | time_in_sleep / time_in_sleep_period |
| light_ratio | time_in_light / time_in_sleep_period |
| deep_ratio | time_in_deep / time_in_sleep_period |
| rem_ratio | time_in_rem / time_in_sleep_period |

### Wake After Sleep Onset (WASO)

| Metric | Definition |
|---|---|
| waso_count | Number of awakenings after initial sleep onset |
| longest_waso | Duration of the longest single awakening (seconds) |

### Sleep Cycles

| Metric | Definition |
|---|---|
| sleep_cycle | Average duration of one sleep cycle (seconds) |
| sleep_cycle_count | Number of sleep cycles |
| sleep_cycle_time | Ordered ISO 8601 cycle-boundary timestamps, beginning with the first cycle onset |

### Sleep Stages

Sleep stages use these integer values for four-stage analysis:

| Value | Stage |
|---|---|
| -1 | Error (invalid audio or analysis error) |
| 0 | Wake |
| 1 | Light |
| 2 | Deep |
| 3 | REM |

Two-stage analysis uses 0 (wake) and 1 (sleep). For v4 ranged sessions, use `analysis.epoch_durations`; do not assume every epoch is 30 seconds.

### Snoring Metrics

| Metric | Definition |
|---|---|
| snoring_stages | Nullable timeline (-1 = error, 0 = no snoring, 1 = snoring), one value per 30 seconds |
| time_in_snoring | Total snoring duration (seconds) |
| snoring_ratio | Proportion of sleep period spent snoring |
| snoring_count | Number of distinct snoring episodes |

### Peculiarities

When a session cannot produce standard results, one of these flags is set:

| Peculiarity | Meaning |
|---|---|
| IN_PROGRESS | Analysis is still running |
| NEVER_SLEPT | No sleep detected during the session |
| TOO_SHORT_FOR_ANALYSIS | Insufficient data for meaningful analysis |
| FAILED | Sleep analysis failed |
| TOO_LONG_FOR_ANALYSIS | Analysis completed but exceeds the documented 24-hour reliability boundary |
| TOO_MANY_DEFECTS_IN_SLEEP_STAGES | Missing/defective audio leaves insufficient analysis results |
| NO_BREATHING_STABILITY | Customer contract does not support breathing-stability analysis |

## Interpretation limits

The API schema is not a clinical diagnosis or a universal quality-rating policy. Do not invent sleep-efficiency or latency cutoffs as platform requirements. Use an explicitly approved product/scientific definition for labels, and keep raw ratios distinct from percentages used for display.

## SDK Error Codes

Error handling depends on platform and SDK version. Consult the pinned SDK's official error table instead of classifying every error as fatal or transient across platforms.

| Android code | Meaning / version boundary |
|---|---|
| ERR_CREATE_CONFLICT (22409) | Previous session is not closed |
| ERR_UPLOAD_FORBIDDEN (23403, legacy) | Upload to an already closed session |
| ERR_UPLOAD_NOT_FOUND (23404, legacy) | Upload session does not exist |
| ERR_UPLOAD_TRACKING_TERMINATED (23499) | From 3.2.0, unifies/deprecates individual upload 4xx codes; SDK handling |
| ERR_CLOSE_FORBIDDEN (24403) | Session already closed |
| ERR_CLOSE_NOT_FOUND (24404) | Session does not exist |
| ERR_MIC_PERMISSION (11002) | Missing microphone permission; client handling |
| ERR_AUDIO (11003) | Android audio error; SDK stops |
| ERR_AUDIO_SILENCED / ERR_AUDIO_UNSILENCED | Audio silenced/restored; SDK handling, not proof of usable audio |
| ERR_UPLOAD_FAILED (23000) | Upload/network failure; SDK handling |
| ERR_COMMON_EXPIRED (11403) | Plan expired, rate limit, or quota exceeded; SDK stops |

These numeric codes are Android-specific. See [Android errors](https://docs-en.asleep.ai/docs/android-error-codes.md) and [iOS errors](https://docs-en.asleep.ai/docs/ios-error-codes.md) for platform-specific behavior and newly added errors.

## API Authentication

All API requests require the `x-api-key` header with a valid API key.

To obtain an API key:

1. Sign up or log in at https://dashboard.asleep.ai
2. Navigate to the Settings tab.
3. Create an API key for your application.

Store API keys securely. Never commit them to version control. Use separate
keys for development and production environments.

## API Base URL

    https://api.asleep.ai

## API Versioning

Asleep versions its API via the URL path (e.g., `/data/v1/`, `/data/v3/`).
Get Session v3 is deprecated in favor of v4. Pin endpoint and SDK versions, tolerate additive fields, and consult official migration notices; this reference makes no fixed support-window guarantee.

## Official Documentation

**Primary resources**:

- Main docs: https://docs-en.asleep.ai
- LLM-optimized reference: https://docs-en.asleep.ai/llms.txt
- Dashboard: https://dashboard.asleep.ai

**Key pages**:

- QuickStart: https://docs-en.asleep.ai/docs/quickstart.md
- Sleep data overview: https://docs-en.asleep.ai/docs/sleep-data.md
- System overview: https://docs-en.asleep.ai/docs/system-overview.md
- API basics: https://docs-en.asleep.ai/docs/api-basics.md
- Webhook guide: https://docs-en.asleep.ai/docs/webhook.md
- Sleep environment guidelines: https://docs-en.asleep.ai/docs/sleep-environment-guideline.md

Contract verified 2026-10-19 against [Get Session](https://docs-en.asleep.ai/docs/data-api-get-session.md), [Average Stats](https://docs-en.asleep.ai/docs/get-users-average-stats.md), and [Android TrackingStatus](https://docs-en.asleep.ai/docs/android-sleep-tracking-manager.md). For exact wire types, nullability, webhook envelopes, and source-version caveats, use the sleeptrack-api schema reference.
