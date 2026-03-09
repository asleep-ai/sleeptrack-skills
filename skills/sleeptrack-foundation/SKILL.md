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

2. **Analysis**: The server AI processes uploaded audio in 5-minute batches,
   classifying each 30-second interval into a sleep stage (Wake, Light, Deep, or REM)
   and detecting snoring episodes.

3. **Delivery**: Completed analysis results are available via REST API polling
   or webhook push notification. Preliminary results can be accessed during
   an active session for real-time UI updates.

## Valid Session Requirements

For the AI to produce meaningful sleep analysis, a session must meet these minimums:

- At least 5 consecutive minutes of sound data uploaded.
- At least 70% overall data upload integrity (ratio of successfully uploaded
  chunks to expected chunks based on session duration).
- One active session per user at a time. Attempting to open a second session
  for the same user while one is OPEN will produce ERR_UPLOAD_FORBIDDEN.
- Microphone access must remain available throughout tracking. Loss of
  microphone access triggers ERR_MIC_PERMISSION.

## Session Lifecycle

The session lifecycle has two parallel state models: SDK states visible on the
client device, and API states tracked on the server.

### SDK States (client-side)

These states are reported by the iOS and Android SDKs:

| State | Meaning |
|---|---|
| IDLE | No tracking in progress |
| INITIALIZING | SDK preparing resources (allocating audio buffers, authenticating) |
| INITIALIZED | Ready to start tracking |
| TRACKING_STARTED | Actively recording and uploading audio |
| TRACKING_STOPPING | Finishing upload of remaining audio chunks |

The typical forward progression is IDLE -> INITIALIZING -> INITIALIZED ->
TRACKING_STARTED -> TRACKING_STOPPING -> IDLE.

### API States (server-side)

These states are returned when querying sessions through the REST API:

| State | Meaning |
|---|---|
| OPEN | Session created, server is accepting audio uploads |
| CLOSED | Session terminated by client, AI analysis in progress |
| COMPLETE | Analysis finished, all metrics and sleep stages available |

Transition: OPEN -> CLOSED -> COMPLETE. The CLOSED-to-COMPLETE transition
happens automatically once the AI finishes processing.

### Real-time Data Access

Preliminary sleep stage data becomes available after sequence 10 (approximately 5 minutes). Poll every 10 sequences thereafter for updates during an active session.

## Sleep Data Metrics

All duration metrics are returned in **seconds** by the API. Time-of-day fields use ISO 8601 timestamps.

### Sleep Index

`sleep_index` -- composite quality score ranging from 50 to 100. Derived from total sleep time, sleep efficiency, and awakening count. Higher is better.

### Duration Metrics (seconds)

| Metric | Definition |
|---|---|
| time_in_bed | Total tracking duration (session start to end) |
| time_in_sleep_period | Sleep onset to final awakening |
| time_in_sleep | Actual time spent asleep (excludes wake periods within sleep period) |
| sleep_latency | Time from session start to first sleep onset |
| wakeup_latency | Time from final awakening to session end |
| light_latency | Time from session start to first Light stage |
| deep_latency | Time from session start to first Deep stage |
| rem_latency | Time from session start to first REM stage |

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
| sleep_cycle_count | Total number of complete sleep cycles |
| sleep_cycle_time | Ordered list of individual cycle durations |

### Sleep Stages

Recorded every 30 seconds as integer values:

| Value | Stage |
|---|---|
| -1 | Unknown |
| 0 | Wake |
| 1 | Light |
| 2 | Deep |
| 3 | REM |

### Snoring Metrics

| Metric | Definition |
|---|---|
| snoring_stages | Binary timeline (0 = no snoring, 1 = snoring) recorded every 30 seconds |
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
| NO_BREATHING_STABILITY | Could not establish stable breathing baseline |

## Sleep Quality Benchmarks

### Sleep Efficiency

| Rating | Range |
|---|---|
| Good | > 85% |
| Fair | 75--85% |
| Poor | < 75% |

### Sleep Latency

| Rating | Range | Notes |
|---|---|---|
| Fast | < 10 min | May indicate sleep deprivation |
| Normal | 10--20 min | -- |
| Slow | > 20 min | May indicate insomnia |

### WASO

Lower values indicate fewer sleep disruptions. There is no universal threshold; track trends over time for a given user.

## Error Codes

### Critical Errors

These require stopping the tracking session.

| Code | Cause |
|---|---|
| ERR_MIC_PERMISSION | App lacks microphone access permission |
| ERR_AUDIO | Microphone unavailable or in use by another app |
| ERR_INVALID_URL | Malformed API endpoint URL in configuration |
| ERR_COMMON_EXPIRED | API rate limit exceeded or subscription plan expired |
| ERR_UPLOAD_FORBIDDEN | Multiple simultaneous tracking attempts with the same user ID |
| ERR_UPLOAD_NOT_FOUND | Attempting to upload to a non-existent or ended session |
| ERR_CLOSE_NOT_FOUND | Attempting to close a non-existent or already-ended session |

### Warning Errors

These are transient; tracking continues automatically.

| Code | Cause |
|---|---|
| ERR_AUDIO_SILENCED | Audio temporarily unavailable, SDK compensates |
| ERR_AUDIO_UNSILENCED | Audio restored after a silence period |
| ERR_UPLOAD_FAILED | Network issue during data upload, SDK retries automatically |

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
Breaking changes are introduced under a new version number; existing versions
remain supported during a deprecation window announced through the official
documentation and Dashboard notifications. Always pin your integration to a
specific version rather than following latest implicitly.

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
