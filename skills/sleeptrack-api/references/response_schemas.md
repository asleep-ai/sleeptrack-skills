# Asleep API Response Schemas

Detailed field definitions for all Asleep REST API response objects, webhook payloads, and enumerations.

## Get Session Response

**Endpoint:** `GET /data/v3/sessions/{session_id}`

### Top-Level Fields

| Field | Type | Description |
|-------|------|-------------|
| timezone | String | IANA timezone applied to timestamps in the response |
| peculiarities | Array[String] | Special conditions affecting this session (see Peculiarities) |
| missing_data_ratio | Float | Proportion of missing audio data (0.0-1.0) |
| session | Object | Session metadata and stage timelines |
| stat | Object | Computed sleep metrics (null if session is not COMPLETE) |

### Session Object

| Field | Type | Description |
|-------|------|-------------|
| id | String | Session identifier |
| state | String | Session state: OPEN, CLOSED, or COMPLETE |
| start_time | String (ISO 8601) | Session start timestamp |
| end_time | String (ISO 8601) | Session end timestamp |
| sleep_stages | Array[Integer] | Sleep stage value per 30-second epoch (-1, 0, 1, 2, 3) |
| snoring_stages | Array[Integer] | Snoring detection per 30-second epoch (0, 1) |

### Stat Object

**Time metrics (seconds unless noted):**

| Field | Type | Description |
|-------|------|-------------|
| sleep_time | String (HH:MM:SS) | Clock time when user fell asleep |
| wake_time | String (HH:MM:SS) | Clock time when user woke up |
| sleep_index | Float | Composite sleep quality score, range 50-100 |
| sleep_latency | Integer | Seconds from session start to sleep onset |
| wakeup_latency | Integer | Seconds from final wake to session end |
| time_in_bed | Integer | Total session duration in seconds |
| time_in_sleep_period | Integer | Seconds from sleep onset to final wake |
| time_in_sleep | Integer | Total seconds spent asleep |
| time_in_wake | Integer | Total seconds spent awake during the sleep period |
| time_in_light | Integer | Seconds in light sleep (stage 1) |
| time_in_deep | Integer | Seconds in deep sleep (stage 2) |
| time_in_rem | Integer | Seconds in REM sleep (stage 3) |
| light_latency | Integer | Seconds from sleep onset to first light sleep |
| deep_latency | Integer | Seconds from sleep onset to first deep sleep |
| rem_latency | Integer | Seconds from sleep onset to first REM sleep |

**Ratio metrics (0.0-1.0):**

| Field | Type | Description |
|-------|------|-------------|
| sleep_efficiency | Float | time_in_sleep / time_in_bed (decimal ratio, e.g. 0.83 = 83%) |
| sleep_ratio | Float | Proportion of sleep period spent asleep |
| wake_ratio | Float | Proportion of sleep period spent awake |
| light_ratio | Float | Proportion of sleep time in light sleep |
| deep_ratio | Float | Proportion of sleep time in deep sleep |
| rem_ratio | Float | Proportion of sleep time in REM sleep |

**Wake and cycle metrics:**

| Field | Type | Description |
|-------|------|-------------|
| waso_count | Integer | Number of wake-after-sleep-onset episodes |
| longest_waso | Integer | Duration of longest WASO episode in seconds |
| sleep_cycle_count | Integer | Number of complete NREM+REM sleep cycles |
| sleep_cycle | Array[Object] | Individual cycle details (see below) |

**Sleep cycle item:**

| Field | Type | Description |
|-------|------|-------------|
| order | Integer | Cycle number (1-based) |
| start_time | String (ISO 8601) | Cycle start timestamp |
| end_time | String (ISO 8601) | Cycle end timestamp |

**Snoring metrics:**

| Field | Type | Description |
|-------|------|-------------|
| snoring_count | Integer | Number of discrete snoring episodes |
| time_in_snoring | Integer | Total snoring duration in seconds |
| time_in_no_snoring | Integer | Total non-snoring duration in seconds |
| snoring_ratio | Float | Proportion of sleep period with snoring (0.0-1.0) |
| no_snoring_ratio | Float | Proportion of sleep period without snoring (0.0-1.0) |

### Example JSON

```json
{
  "detail": "success",
  "result": {
    "timezone": "UTC",
    "peculiarities": [],
    "missing_data_ratio": 0.0,
    "session": {
      "id": "abc-123",
      "state": "COMPLETE",
      "start_time": "2024-01-20T22:00:00+00:00",
      "end_time": "2024-01-21T06:30:00+00:00",
      "sleep_stages": [0, 0, 1, 1, 2, 2, 3, 3, 1, 1, 2, 2, 3, 1, 0],
      "snoring_stages": [0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 1, 0, 0, 0, 0]
    },
    "stat": {
      "sleep_time": "22:15:00", "wake_time": "06:25:00",
      "sleep_index": 85.5, "sleep_latency": 900, "wakeup_latency": 300,
      "time_in_bed": 30600, "time_in_sleep_period": 29400,
      "time_in_sleep": 27000, "time_in_wake": 2400,
      "time_in_light": 13500, "time_in_deep": 6750, "time_in_rem": 6750,
      "sleep_efficiency": 88.24, "sleep_ratio": 0.92, "wake_ratio": 0.08,
      "light_ratio": 0.50, "deep_ratio": 0.25, "rem_ratio": 0.25,
      "waso_count": 2, "longest_waso": 300,
      "sleep_cycle_count": 4,
      "sleep_cycle": [
        {"order": 1, "start_time": "2024-01-20T22:15:00+00:00", "end_time": "2024-01-21T00:00:00+00:00"}
      ],
      "snoring_count": 15, "time_in_snoring": 1800, "time_in_no_snoring": 25200,
      "snoring_ratio": 0.07, "no_snoring_ratio": 0.93
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
| session_start_time | String (ISO 8601) | Session start timestamp |
| session_end_time | String (ISO 8601) | Session end timestamp |
| created_timezone | String | Timezone when session was created |
| unexpected_end_time | String (ISO 8601) or null | Timestamp if session ended unexpectedly |
| last_received_seq_num | Integer | Last audio upload sequence number received |
| time_in_bed | Integer | Total session duration in seconds |

## Get Average Stats Response

**Endpoint:** `GET /data/v1/users/{user_id}/average-stats`

### Top-Level Fields

| Field | Type | Description |
|-------|------|-------------|
| period | Object | Queried date range |
| peculiarities | Array[String] | Conditions across the period (e.g., NO_BREATHING_STABILITY) |
| average_stats | Object | Averaged metrics across sessions |
| slept_sessions | Array[Object] | Sessions where sleep was detected |
| never_slept_sessions | Array[Object] | Sessions where no sleep was detected |

### Period Object

| Field | Type | Description |
|-------|------|-------------|
| start_date | String (YYYY-MM-DD) | Period start |
| end_date | String (YYYY-MM-DD) | Period end |
| days | Integer | Number of days in the period |

### Average Stats Object

All values are averages across the sessions in the period. Field names match the Stat Object from Get Session. Key differences: numeric values are Float (averages) rather than Integer, and time-of-day fields use HH:MM:SS format.

| Field | Type | Description |
|-------|------|-------------|
| start_time | String (HH:MM:SS) | Average bedtime (session start) |
| end_time | String (HH:MM:SS) | Average session end time |
| sleep_time | String (HH:MM:SS) | Average time of falling asleep |
| wake_time | String (HH:MM:SS) | Average time of waking up |
| sleep_latency | Float | Average seconds to fall asleep |
| wakeup_latency | Float | Average seconds from final wake to session end |
| time_in_bed | Float | Average total time in bed (seconds) |
| time_in_sleep_period | Float | Average sleep-onset-to-wake duration (seconds) |
| time_in_sleep | Float | Average actual sleep time (seconds) |
| time_in_wake | Float | Average wake time during sleep period (seconds) |
| time_in_light | Float | Average light sleep duration (seconds) |
| time_in_deep | Float | Average deep sleep duration (seconds) |
| time_in_rem | Float | Average REM sleep duration (seconds) |
| time_in_snoring | Float | Average snoring duration (seconds) |
| time_in_no_snoring | Float | Average non-snoring duration (seconds) |
| sleep_efficiency | Float | Average sleep efficiency (percentage, 0-100) |
| wake_ratio | Float | Average wake proportion (0.0-1.0) |
| sleep_ratio | Float | Average sleep proportion (0.0-1.0) |
| light_ratio | Float | Average light sleep proportion (0.0-1.0) |
| deep_ratio | Float | Average deep sleep proportion (0.0-1.0) |
| rem_ratio | Float | Average REM sleep proportion (0.0-1.0) |
| snoring_ratio | Float | Average snoring proportion (0.0-1.0) |
| no_snoring_ratio | Float | Average non-snoring proportion (0.0-1.0) |
| waso_count | Float | Average WASO episodes per session |
| longest_waso | Float | Average longest WASO duration (seconds) |
| sleep_cycle_count | Float | Average sleep cycles per session |
| snoring_count | Float | Average snoring episodes per session |

### Session Reference Item

Used in both `slept_sessions` and `never_slept_sessions` arrays.

| Field | Type | Description |
|-------|------|-------------|
| session_id | String | Session identifier |
| session_start_time | String (ISO 8601) | Session start timestamp |

## Webhook Payloads

### INFERENCE_COMPLETE

Sent every 5-40 minutes during active tracking.

| Field | Type | Description |
|-------|------|-------------|
| event | String | Always "INFERENCE_COMPLETE" |
| version | String | Webhook version (V1, V2, V3) |
| user_id | String | User identifier |
| session_id | String | Session identifier |
| seq_num | Integer | Audio upload sequence number |
| inference_seq_num | Integer | Analysis sequence number (increments per 5-min batch) |
| sleep_stages | Array[Integer] | Sleep stage values for this time window |
| snoring_stages | Array[Integer] | Snoring detection values for this time window |
| time_window | Object | Time range covered by this inference |
| time_window.start | String (ISO 8601) | Window start timestamp |
| time_window.end | String (ISO 8601) | Window end timestamp |

**Example JSON:**

```json
{
  "event": "INFERENCE_COMPLETE",
  "version": "V3",
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "session_id": "abc-123",
  "seq_num": 60,
  "inference_seq_num": 12,
  "sleep_stages": [1, 1, 2, 2, 2, 3, 3, 1, 1, 2],
  "snoring_stages": [0, 0, 1, 1, 0, 0, 0, 0, 0, 1],
  "time_window": {
    "start": "2024-01-21T01:00:00+00:00",
    "end": "2024-01-21T01:05:00+00:00"
  }
}
```

### SESSION_COMPLETE

Sent once when session analysis finishes. The payload includes the full session and stat objects, matching the structure returned by the Get Session endpoint.

| Field | Type | Description |
|-------|------|-------------|
| event | String | Always "SESSION_COMPLETE" |
| version | String | Webhook version (V1, V2, V3) |
| user_id | String | User identifier |
| session_id | String | Session identifier |
| session | Object | Full session object (same schema as Get Session) |
| stat | Object | Full stat object (same schema as Get Session) |
| peculiarities | Array[String] | Special session conditions |

**Example JSON:**

```json
{
  "event": "SESSION_COMPLETE",
  "version": "V3",
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "session_id": "abc-123",
  "session": {
    "id": "abc-123",
    "state": "COMPLETE",
    "start_time": "2024-01-20T22:00:00+00:00",
    "end_time": "2024-01-21T06:30:00+00:00",
    "timezone": "UTC",
    "sleep_stages": [0, 0, 1, 1, 2, 2, 3, 3, 1, 1, 2, 3, 1, 0, 0],
    "snoring_stages": [0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 1, 0, 0, 0, 0]
  },
  "stat": {
    "sleep_time": "22:15:00",
    "wake_time": "06:25:00",
    "sleep_index": 85.5,
    "sleep_latency": 900,
    "time_in_bed": 30600,
    "time_in_sleep": 27000,
    "sleep_efficiency": 88.24,
    "waso_count": 2,
    "sleep_cycle_count": 4,
    "snoring_count": 15
  },
  "peculiarities": []
}
```

The `session` and `stat` objects use the same schema as the Get Session response. The example above is abbreviated; the full stat object includes all fields documented in the Stat Object section.

## Enumerations

### Sleep Stage Values

| Value | Meaning |
|-------|---------|
| -1 | Unknown / no data |
| 0 | Wake |
| 1 | Light sleep |
| 2 | Deep sleep |
| 3 | REM sleep |

Recorded every 30 seconds throughout the session.

### Snoring Stage Values

| Value | Meaning |
|-------|---------|
| 0 | No snoring |
| 1 | Snoring detected |

Recorded every 30 seconds throughout the session.

### Session States

| State | Description |
|-------|-------------|
| OPEN | Session created, audio data uploads in progress |
| CLOSED | Session terminated, AI analysis in progress |
| COMPLETE | All analysis finished, metrics fully available |

### Peculiarities

| Value | Description |
|-------|-------------|
| IN_PROGRESS | Session analysis is still running |
| NEVER_SLEPT | No sleep was detected during the session |
| TOO_SHORT_FOR_ANALYSIS | Session duration was less than 5 minutes of audio data |
| NO_BREATHING_STABILITY | Breathing data was too inconsistent for reliable analysis |

## Metadata Fields

User metadata fields accepted by Create User and Update User endpoints.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| birth_year | Integer | -- | User's birth year |
| birth_month | Integer | 1-12 | User's birth month |
| birth_day | Integer | 1-31 | User's birth day |
| gender | String | male, female, non_binary, other, prefer_not_to_say | Gender identity |
| height | Float | 0-300 | Height in centimeters |
| weight | Float | 0-1000 | Weight in kilograms |

All metadata fields are optional. Partial updates are supported -- only the fields included in the request body will be modified.
