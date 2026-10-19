# Asleep API Response Schemas

Response fields for the endpoints covered by this skill. Use the linked official contract for other endpoints and versions.

## Get Session Response

**Endpoint:** `GET /data/v3/sessions/{session_id}`

### Top-Level Fields

| Field | Type | Description |
|-------|------|-------------|
| timezone | String | IANA timezone applied to timestamps in the response |
| peculiarities | Array[String] | Special conditions affecting this session (see Peculiarities) |
| missing_data_ratio | Float | Proportion of missing audio data (0.0-1.0) |
| session | Object | Session metadata and stage timelines |
| stat | Object or null | Computed metrics or null; see nullability rules below |

### Session Object

| Field | Type | Description |
|-------|------|-------------|
| id | String | Session identifier |
| state | String | Session state: OPEN, CLOSED, or COMPLETE |
| start_time | String (ISO 8601) | Analysis range start timestamp |
| end_time | String (ISO 8601) or null | Analysis end timestamp; null while OPEN |
| measurement_start_time | String (ISO 8601) | Actual measurement start; required |
| measurement_end_time | String (ISO 8601) or null | Actual measurement end |
| unexpected_end_time | String (ISO 8601) or null | Time an abnormal termination was reported |
| created_timezone | String | Timezone at session creation |
| sleep_stages | Array[Integer] or null | Sleep stage value per 30-second epoch (-1, 0, 1, 2, 3) |
| snoring_stages | Array[Integer] or null | Snoring per 30-second epoch (-1 error, 0 no snoring, 1 snoring) |
| breath_stages | Array[Integer] or null | Breathing stability per epoch: 0 stable; 1 or 2 unstable |

For two-stage analysis, sleep stage values are 0 (wake) and 1 (sleep).

### Stat Object

**Time metrics (seconds unless noted):**

Types below follow the public v3 tables. Preserve unavailable values as null rather than coercing them. The [public versioning policy](https://docs-en.asleep.ai/docs/api-basics.md) permits additive response fields; tolerate unknown keys.

| Field | Type | Description |
|-------|------|-------------|
| sleep_time | String (ISO 8601) or null | Timestamp when user fell asleep |
| wake_time | String (ISO 8601) or null | Timestamp when user woke up |
| sleep_index | Integer or null | Beta score, range 50-100 |
| sleep_latency | Integer or null | Seconds from analysis start to sleep onset |
| wakeup_latency | Integer or null | Seconds from final wake to analysis end |
| time_in_bed | Integer | Seconds from the start of sleep measurement to its end; for ranged sessions, see v4 `analysis` |
| time_in_sleep_period | Integer | Seconds from sleep onset to final wake |
| time_in_sleep | Integer | Total seconds spent asleep |
| time_in_wake | Integer | Total seconds spent awake during the sleep period |
| time_in_light | Integer or null | Seconds in light sleep (stage 1) |
| time_in_deep | Integer or null | Seconds in deep sleep (stage 2) |
| time_in_rem | Integer or null | Seconds in REM sleep (stage 3) |
| light_latency | Integer or null | Seconds from sleep onset to first light sleep |
| deep_latency | Integer or null | Seconds from sleep onset to first deep sleep |
| rem_latency | Integer or null | Seconds from sleep onset to first REM sleep |

**Ratio metrics (0.0-1.0):**

| Field | Type | Description |
|-------|------|-------------|
| sleep_efficiency | Float | time_in_sleep / time_in_bed (decimal ratio, e.g. 0.83 = 83%) |
| sleep_ratio | Float or null | Proportion of sleep period spent asleep |
| wake_ratio | Float or null | Proportion of sleep period spent awake |
| light_ratio | Float or null | time_in_light / time_in_sleep_period |
| deep_ratio | Float or null | time_in_deep / time_in_sleep_period |
| rem_ratio | Float or null | time_in_rem / time_in_sleep_period |

**Wake and cycle metrics:**

| Field | Type | Description |
|-------|------|-------------|
| waso_count | Integer or null | Number of wake-after-sleep-onset episodes |
| longest_waso | Integer or null | Duration of longest WASO episode in seconds |
| sleep_cycle_count | Integer or null | Number of sleep cycles |
| sleep_cycle | Integer or null | Average sleep-cycle duration in seconds |
| sleep_cycle_time | Array[String (ISO 8601)] | Ordered cycle boundaries: first onset, first end, second end, and so on |

**Snoring metrics:**

| Field | Type | Description |
|-------|------|-------------|
| snoring_count | Integer or null | Number of discrete snoring episodes |
| time_in_snoring | Integer or null | Total snoring duration in seconds |
| time_in_no_snoring | Integer or null | Total non-snoring duration in seconds |
| snoring_ratio | Float or null | Proportion of sleep period with snoring (0.0-1.0) |
| no_snoring_ratio | Float or null | Proportion of sleep period without snoring (0.0-1.0) |

**Breathing metrics (nullable when unsupported):**

| Field | Type | Description |
|-------|------|-------------|
| time_in_stable_breath, time_in_unstable_breath | Integer or null | Durations in seconds |
| stable_breath_ratio, unstable_breath_ratio | Float or null | Ratios (0-1) during the sleep period |
| breathing_index | Float or null | Breathing instability index |
| breathing_pattern | String or null | STABLE_BREATH, MILDLY_UNSTABLE_BREATH, MODERATELY_UNSTABLE_BREATH, SEVERELY_UNSTABLE_BREATH |
| unstable_breath_count | Integer or null | Count of unstable breathing episodes |

### Nullability and version boundaries

`stat` is null for `IN_PROGRESS`, `TOO_SHORT_FOR_ANALYSIS`, or `TOO_MANY_DEFECTS_IN_SLEEP_STAGES`. Treat failure results as unavailable rather than assuming COMPLETE implies a usable object. Public documentation lists `FAILED` but its nullability bullet does not enumerate that flag; do not turn that omission into a non-null guarantee.

Two-stage analysis uses 0 (wake) and 1 (sleep). With two-stage analysis, light/deep/REM durations, ratios, and latencies are null. With `NEVER_SLEPT`, sleep/wake timestamps, sleep/wakeup latency, stage ratios/latencies, sleep_cycle, longest_waso, and sleep_index are null. New metrics may be unavailable for older sessions. Preserve null instead of coercing it to zero.

Get Session v3 is deprecated. [v4](https://docs-en.asleep.ai/docs/data-api-get-session-v4.md) adds nullable `analysis` containing `start_epoch_index`, `end_epoch_index`, `epoch_durations`, and `total_epoch_count`. For selected ranges, stage arrays cover that range; use `epoch_durations` and do not assume every epoch is 30 seconds in a ranged session. Use the v4 contract when implementing range-aware timelines.

### Example JSON

A full 300-second session with ten 30-second epochs; the 240-second sleep period contains only light sleep. This synthetic sample illustrates the wire format, not clinical validity. It is illustrative only; real sessions must exceed the selected model's minimum duration.

```json
{
  "detail": "OK",
  "result": {
    "timezone": "UTC",
    "peculiarities": [
      "NO_BREATHING_STABILITY"
    ],
    "missing_data_ratio": 0.0,
    "session": {
      "id": "20250115030000_abcde",
      "state": "COMPLETE",
      "start_time": "2025-01-15T03:00:00+00:00",
      "end_time": "2025-01-15T03:05:00+00:00",
      "measurement_start_time": "2025-01-15T03:00:00+00:00",
      "measurement_end_time": "2025-01-15T03:05:00+00:00",
      "unexpected_end_time": null,
      "created_timezone": "UTC",
      "sleep_stages": [
        0,
        1,
        1,
        1,
        1,
        1,
        1,
        1,
        1,
        0
      ],
      "breath_stages": null,
      "snoring_stages": [
        0,
        0,
        0,
        0,
        0,
        0,
        0,
        0,
        0,
        0
      ]
    },
    "stat": {
      "sleep_time": "2025-01-15T03:00:30+00:00",
      "wake_time": "2025-01-15T03:04:30+00:00",
      "sleep_index": 50,
      "sleep_latency": 30,
      "wakeup_latency": 30,
      "light_latency": 0,
      "deep_latency": null,
      "rem_latency": null,
      "time_in_bed": 300,
      "time_in_sleep_period": 240,
      "time_in_sleep": 240,
      "time_in_wake": 0,
      "time_in_light": 240,
      "time_in_deep": 0,
      "time_in_rem": 0,
      "time_in_stable_breath": null,
      "time_in_unstable_breath": null,
      "time_in_snoring": 0,
      "time_in_no_snoring": 240,
      "sleep_efficiency": 0.8,
      "sleep_ratio": 1.0,
      "wake_ratio": 0.0,
      "light_ratio": 1.0,
      "deep_ratio": 0.0,
      "rem_ratio": 0.0,
      "stable_breath_ratio": null,
      "unstable_breath_ratio": null,
      "snoring_ratio": 0.0,
      "no_snoring_ratio": 1.0,
      "breathing_index": null,
      "breathing_pattern": null,
      "waso_count": 0,
      "longest_waso": 0,
      "sleep_cycle_count": 0,
      "sleep_cycle": null,
      "sleep_cycle_time": [],
      "unstable_breath_count": null,
      "snoring_count": 0
    }
  }
}
```

## List Sessions Response

**Endpoint:** `GET /data/v1/sessions`

### Top-Level Fields

| Field | Type | Description |
|-------|------|-------------|
| timezone | String | IANA timezone applied to timestamps |
| sleep_session_list | Array[Object] | List of session summary items |

### Session List Item

| Field | Type | Description |
|-------|------|-------------|
| session_id | String | Session identifier |
| state | String | OPEN, CLOSED, or COMPLETE |
| session_start_time | String (ISO 8601) | Analysis start timestamp |
| session_end_time | String (ISO 8601) or null | Analysis end; null while OPEN |
| measurement_start_time | String (ISO 8601) | Actual measurement start; required |
| measurement_end_time | String (ISO 8601) or null | Actual measurement end |
| created_timezone | String | Timezone when session was created |
| unexpected_end_time | String (ISO 8601) or null | Timestamp if session ended unexpectedly |
| last_received_seq_num | Integer or null | Last audio upload sequence number received |
| time_in_bed | Integer or null | Seconds from the start of sleep measurement to its end; for ranged sessions, see v4 `analysis` |

## Get Average Stats Response

**Endpoint:** `GET /data/v1/users/{user_id}/average-stats`

### Top-Level Fields

| Field | Type | Description |
|-------|------|-------------|
| period | Object | Queried date range |
| peculiarities | Array[String] | Conditions across the period (e.g., NO_BREATHING_STABILITY) |
| average_stats | Object or null | Averages across slept_sessions; null when no eligible sleep sessions |
| slept_sessions | Array[Object] | Sessions where sleep was detected |
| never_slept_sessions | Array[Object] | Sessions where no sleep was detected |

### Period Object

| Field | Type | Description |
|-------|------|-------------|
| start_date | String (YYYY-MM-DD) | Period start |
| end_date | String (YYYY-MM-DD) | Period end |
| timezone | String | Requested IANA timezone; no days field |

### Average Stats Object

Averages are calculated over `slept_sessions`, not the no-sleep array. Duration/count outputs are integers; ratio outputs are floats. Time-of-day fields use HH:MM:SS on the wire. Do not assume every Get Session metric is included (no aggregate sleep_index, stage latencies, breathing_pattern, sleep_cycle, or sleep_cycle_time).

| Field | Type | Description |
|-------|------|-------------|
| start_time | String (HH:MM:SS) | Average bedtime (session start) |
| end_time | String (HH:MM:SS) | Average session end time |
| sleep_time | String (HH:MM:SS) | Average time of falling asleep |
| wake_time | String (HH:MM:SS) | Average time of waking up |
| sleep_latency | Integer | Average seconds to fall asleep |
| wakeup_latency | Integer | Average seconds from final wake to session end |
| time_in_bed | Integer | Average total time in bed (seconds) |
| time_in_sleep_period | Integer | Average sleep-onset-to-wake duration (seconds) |
| time_in_sleep | Integer | Average actual sleep time (seconds) |
| time_in_wake | Integer | Average wake time during sleep period (seconds) |
| time_in_light | Integer or null | Average light sleep duration (seconds) |
| time_in_deep | Integer or null | Average deep sleep duration (seconds) |
| time_in_rem | Integer or null | Average REM sleep duration (seconds) |
| time_in_snoring | Integer or null | Average snoring duration (seconds) |
| time_in_no_snoring | Integer or null | Average non-snoring duration (seconds) |
| sleep_efficiency | Float | Average sleep efficiency (ratio, 0-1) |
| wake_ratio | Float | Average wake proportion (0.0-1.0) |
| sleep_ratio | Float | Average sleep proportion (0.0-1.0) |
| light_ratio | Float or null | Average light sleep proportion (0.0-1.0) |
| deep_ratio | Float or null | Average deep sleep proportion (0.0-1.0) |
| rem_ratio | Float or null | Average REM sleep proportion (0.0-1.0) |
| snoring_ratio | Float or null | Average snoring proportion (0.0-1.0) |
| no_snoring_ratio | Float or null | Average non-snoring proportion (0.0-1.0) |
| waso_count | Integer or null | Average WASO episodes per session |
| longest_waso | Integer or null | Average longest WASO duration (seconds) |
| sleep_cycle_count | Integer or null | Average sleep cycles per session |
| snoring_count | Integer or null | Average snoring episodes per session |

Additional nullable average fields: `time_in_stable_breath`, `time_in_unstable_breath`, `unstable_breath_count` (integers), and `stable_breath_ratio`, `unstable_breath_ratio`, `breathing_index` (floats). These appear in the public response example; the prose table omits the legacy breathing fields.

### Never Slept Session Object

`id`, `start_time`, `end_time`, and `completed_time` (ISO 8601 timestamps except the string ID). These are objects, not `{session_id, session_start_time}` references.

### Slept Session Object

Each object contains `id`, `created_timezone`, `start_time`, `end_time`, and `completed_time`, plus flattened per-session metrics:

- `sleep_time`, `wake_time`: nullable ISO 8601 timestamps.
- `sleep_latency`, `wakeup_latency`, `light_latency`, `deep_latency`, `rem_latency`: seconds, nullable where unavailable.
- `time_in_bed`, `time_in_sleep_period`, `time_in_sleep`, `time_in_wake`, `time_in_light`, `time_in_deep`, `time_in_rem`: integer seconds (stage-specific values may be null).
- `time_in_stable_breath`, `time_in_unstable_breath`, `time_in_snoring`, `time_in_no_snoring`: nullable integer seconds.
- `sleep_efficiency`, `sleep_ratio`, `wake_ratio`, `light_ratio`, `deep_ratio`, `rem_ratio`, `stable_breath_ratio`, `unstable_breath_ratio`, `snoring_ratio`, `no_snoring_ratio`: decimal ratios, preserving null when unavailable.
- `breathing_index`, `breathing_pattern`: nullable breathing metrics.
- `waso_count`, `longest_waso`, `sleep_cycle_count`, `sleep_cycle`, `unstable_breath_count`, `snoring_count`: nullable integer metrics; `longest_waso` and `sleep_cycle` use seconds.

This is the public Average Stats schema, not an exact alias of Get Session's stat object. The [public versioning policy](https://docs-en.asleep.ai/docs/api-basics.md) permits additive response fields; tolerate additional keys and consult the pinned contract before requiring them.

## Webhook Payloads (V3)

Top-level keys are `event`, `version`, and `data`. No `time_window` is documented.

### INFERENCE_COMPLETE

`data` contains `user_id`, `session_id`, integer `seq_num` and `inference_seq_num`, and stage arrays. Public documentation's field table says `stage_list`, while its V3 example says `sleep_stages`; this sample follows the example. Confirm this naming discrepancy for the pinned integration. `breath_stages` and `snoring_stages` can be null. Sequence numbers identify successive inference results, not a timestamp window.

```json
{
  "event": "INFERENCE_COMPLETE",
  "version": "V3",
  "data": {
    "user_id": "USER_ID",
    "session_id": "20250115030000_abcde",
    "seq_num": 9,
    "inference_seq_num": 0,
    "sleep_stages": [
      0,
      1,
      1,
      1,
      1,
      1,
      1,
      1,
      1,
      0
    ],
    "breath_stages": null,
    "snoring_stages": [
      0,
      0,
      0,
      0,
      0,
      0,
      0,
      0,
      0,
      0
    ]
  }
}
```

### SESSION_COMPLETE

`data` follows the sleep-result shape and adds `user_id`. Identify the session using `data.session.id`, not an invented `data.session_id`. The public V3 example omits the REST measurement-range fields; do not require them in webhook payloads. The stat field types/nullability follow Get Session.

```json
{
  "event": "SESSION_COMPLETE",
  "version": "V3",
  "data": {
    "timezone": "UTC",
    "peculiarities": [
      "NO_BREATHING_STABILITY"
    ],
    "missing_data_ratio": 0.0,
    "session": {
      "id": "20250115030000_abcde",
      "state": "COMPLETE",
      "start_time": "2025-01-15T03:00:00+00:00",
      "end_time": "2025-01-15T03:05:00+00:00",
      "unexpected_end_time": null,
      "created_timezone": "UTC",
      "sleep_stages": [
        0,
        1,
        1,
        1,
        1,
        1,
        1,
        1,
        1,
        0
      ],
      "breath_stages": null,
      "snoring_stages": [
        0,
        0,
        0,
        0,
        0,
        0,
        0,
        0,
        0,
        0
      ]
    },
    "stat": {
      "sleep_time": "2025-01-15T03:00:30+00:00",
      "wake_time": "2025-01-15T03:04:30+00:00",
      "sleep_index": 50,
      "sleep_latency": 30,
      "wakeup_latency": 30,
      "light_latency": 0,
      "deep_latency": null,
      "rem_latency": null,
      "time_in_bed": 300,
      "time_in_sleep_period": 240,
      "time_in_sleep": 240,
      "time_in_wake": 0,
      "time_in_light": 240,
      "time_in_deep": 0,
      "time_in_rem": 0,
      "time_in_stable_breath": null,
      "time_in_unstable_breath": null,
      "time_in_snoring": 0,
      "time_in_no_snoring": 240,
      "sleep_efficiency": 0.8,
      "sleep_ratio": 1.0,
      "wake_ratio": 0.0,
      "light_ratio": 1.0,
      "deep_ratio": 0.0,
      "rem_ratio": 0.0,
      "stable_breath_ratio": null,
      "unstable_breath_ratio": null,
      "snoring_ratio": 0.0,
      "no_snoring_ratio": 1.0,
      "breathing_index": null,
      "breathing_pattern": null,
      "waso_count": 0,
      "longest_waso": 0,
      "sleep_cycle_count": 0,
      "sleep_cycle": null,
      "sleep_cycle_time": [],
      "unstable_breath_count": null,
      "snoring_count": 0
    },
    "user_id": "USER_ID"
  }
}
```

Handle delivery idempotently: completion uses session ID plus event; inference additionally uses inference_seq_num. Authentication, HMAC, ordering, and retry rules are in the API skill.

## Enumerations

### Sleep Stage Values

| Value | Meaning |
|-------|---------|
| -1 | Error (invalid audio or analysis error) |
| 0 | Wake |
| 1 | Light sleep |
| 2 | Deep sleep |
| 3 | REM sleep |

For two-stage analysis, 0 means wake and 1 means sleep.

For v4 ranged sessions, use `analysis.epoch_durations`; do not assume every epoch is 30 seconds.

### Snoring Stage Values

| Value | Meaning |
|-------|---------|
| -1 | Error (invalid audio or analysis error) |
| 0 | No snoring |
| 1 | Snoring detected |

For v4 ranged sessions, use `analysis.epoch_durations`; do not assume every epoch is 30 seconds.

### Session States

| State | Description |
|-------|-------------|
| OPEN | Session created, audio data uploads in progress |
| CLOSED | Session terminated, AI analysis in progress |
| COMPLETE | Analysis completed; inspect peculiarities and nullable metrics before use |

### Peculiarities

| Value | Description |
|-------|-------------|
| IN_PROGRESS | Session analysis is still running |
| NEVER_SLEPT | No sleep was detected during the session |
| TOO_SHORT_FOR_ANALYSIS | Session duration was less than 5 minutes of audio data |
| FAILED | Sleep analysis failed |
| TOO_LONG_FOR_ANALYSIS | Analysis completed but the range exceeds 24 hours and is considered unreliable |
| TOO_MANY_DEFECTS_IN_SLEEP_STAGES | Missing/defective audio leaves insufficient analysis results |
| NO_BREATHING_STABILITY | Customer contract does not support breathing-stability analysis |

## Metadata Fields

User metadata fields accepted by Create User and Update User endpoints.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| birth_year | Integer or null | 1900 through current year | User's birth year |
| birth_month | Integer or null | 1-12 | User's birth month |
| birth_day | Integer or null | 1-31 | User's birth day |
| gender | String or null | male, female, non_binary, other, prefer_not_to_say | Gender identity |
| height | Float or null | 0-300, one decimal place | Height in centimeters |
| weight | Float or null | 0-1000, one decimal place | Weight in kilograms |

Metadata fields are optional and nullable. Create User may omit metadata; Update User requires the metadata member (which may be null). Do not infer patch/merge semantics for omitted values.

## Public sources

Verified 2026-10-19: [Get Session v3](https://docs-en.asleep.ai/docs/data-api-get-session.md), [v4](https://docs-en.asleep.ai/docs/data-api-get-session-v4.md), [List Sessions](https://docs-en.asleep.ai/docs/data-api-list-sessions.md), [Average Stats](https://docs-en.asleep.ai/docs/get-users-average-stats.md), and [Webhook V3](https://docs-en.asleep.ai/docs/webhook.md). Public documentation governs required customer fields; released server additions are not automatically mandatory public fields.
