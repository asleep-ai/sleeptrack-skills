---
name: sleeptrack-api
description: Asleep REST API integration reference. Use when building server-side applications, API clients, webhook handlers, or any backend that interacts with the Asleep sleep tracking platform. Covers authentication, user management, session data retrieval, statistics, and webhook events.
---

# Asleep REST API Reference

## Overview

This skill is a language-agnostic REST API reference for the Asleep sleep tracking platform. It documents endpoints, request/response formats, webhook events, and integration best practices without prescribing a specific programming language or framework.

For detailed response field definitions and example payloads, see `references/response_schemas.md`.

## Authentication

All requests require an API key passed via the `x-api-key` header.

**Obtaining an API key:**
1. Sign up or log in at https://dashboard.asleep.ai
2. Navigate to the Settings tab
3. Generate an API key for your application

**Base URL:**
```
https://api.asleep.ai
```

```bash
curl -H "x-api-key: YOUR_API_KEY" https://api.asleep.ai/ai/v1/users/USER_ID
```

## Common Headers

| Header | Type | Required | Description |
|--------|------|----------|-------------|
| x-api-key | String | Yes | API authentication key |
| x-user-id | String | Conditional | Required for session operations |
| timezone | String | No | IANA timezone for response timestamps (default: UTC) |

## Response Format

Most successful JSON responses use this envelope. Do not require a particular `detail` string (`OK` and `success` both occur). DELETE success is HTTP 204 with no body; Close and Reanalyze return only `detail`:

```json
{
  "detail": "success",
  "result": { }
}
```

Error responses omit the `result` field:

```json
{
  "detail": "user does not exist"
}
```

## User Management Endpoints

### POST /ai/v1/users -- Create User

Creates a new user. The body is optional; include `metadata` to attach demographic information.

```bash
curl -X POST https://api.asleep.ai/ai/v1/users \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"metadata": {"birth_year": 1990, "gender": "male", "height": 175.5, "weight": 70.0}}'
```

**Request body** (all metadata fields optional and nullable):

| Field | Type | Constraints |
|-------|------|-------------|
| metadata.birth_year | Integer or null | 1900 through current year |
| metadata.birth_month | Integer or null | 1-12 |
| metadata.birth_day | Integer or null | 1-31; valid calendar date |
| metadata.gender | String or null | male, female, non_binary, other, prefer_not_to_say |
| metadata.height | Float or null | cm, 0-300; at most one decimal place |
| metadata.weight | Float or null | kg, 0-1000; at most one decimal place |

**Response (201 Created):**
Returns the generated `user_id` in `result`.

```json
{
  "detail": "success",
  "result": {
    "user_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

### GET /ai/v1/users/{user_id} -- Get User

Retrieves user profile, deletion status, last session info, and metadata.

```bash
curl https://api.asleep.ai/ai/v1/users/USER_ID \
  -H "x-api-key: YOUR_API_KEY"
```

**Response (200 OK):**

| Field | Description |
|-------|-------------|
| user_id | User identifier |
| to_be_deleted | Boolean indicating pending deletion |
| last_session_info | Object or null; fields: session_id, inference_type, service, start_time, nullable end_time, state, nullable complete_time |
| metadata | Object or null with demographic fields (birth_year, gender, height, weight, etc.) |

### PUT /ai/v1/users/{user_id} -- Update User

Updates user metadata. The `metadata` member is required and may be null; its fields are optional and nullable. Do not assume omitted fields are preserved without checking the current update contract.

```bash
curl -X PUT https://api.asleep.ai/ai/v1/users/USER_ID \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"metadata": {"height": 180.0, "weight": 75.0}}'
```

**Response (200 OK):** Returns `user_id` and `metadata` in `result`.

### DELETE /ai/v1/users/{user_id} -- Delete User

Permanently removes a user and all associated sessions and data.

```bash
curl -X DELETE https://api.asleep.ai/ai/v1/users/USER_ID \
  -H "x-api-key: YOUR_API_KEY"
```

**Response (204 No Content):** Empty body on success.

## Session Data Endpoints

### GET /data/v3/sessions/{session_id} -- Get Session (deprecated)

Returns session metadata and nullable analysis results. For new integrations use [Get Session v4](https://docs-en.asleep.ai/docs/data-api-get-session-v4.md), which adds a nullable `analysis` object with epoch-range information. The schema reference below describes v3 and the V3 webhook; do not infer that API and webhook version numbers are interchangeable.

With a selected analysis range, `start_time`/`end_time` describe that range; required `measurement_start_time` and nullable `measurement_end_time` retain the actual measurement window.

```bash
curl https://api.asleep.ai/data/v3/sessions/SESSION_ID \
  -H "x-api-key: YOUR_API_KEY" \
  -H "x-user-id: USER_ID" \
  -H "timezone: America/New_York"
```

**Response (200 OK):**

| Field | Description |
|-------|-------------|
| timezone | IANA timezone applied to response timestamps |
| peculiarities | Array of special conditions (e.g., NEVER_SLEPT, IN_PROGRESS) |
| missing_data_ratio | Proportion of missing audio data (0.0-1.0) |
| session | Object with id, state, start_time, end_time, sleep_stages, snoring_stages |
| stat | Nullable computed sleep metrics; inspect peculiarities and field nullability |

Sleep stage values for four-stage analysis are -1 (invalid audio or analysis error), 0 (wake), 1 (light), 2 (deep), 3 (REM). Two-stage analysis uses 0 (wake) and 1 (sleep). For v4 ranged sessions, use `analysis.epoch_durations`; do not assume every epoch is 30 seconds.

See `references/response_schemas.md` for the complete field-by-field breakdown of the session and stat objects.

### GET /data/v1/sessions -- List Sessions

Returns a paginated list of sessions for a user.

```bash
curl "https://api.asleep.ai/data/v1/sessions?date_gte=2024-01-01&date_lte=2024-01-31&order_by=DESC&limit=50" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "x-user-id: USER_ID"
```

**Query parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| date_gte | String (YYYY-MM-DD) | -- | Sessions on or after this date |
| date_lte | String (YYYY-MM-DD) | -- | Sessions on or before this date |
| order_by | ASC or DESC | DESC | Sort by start time |
| offset | Integer | 0 | Records to skip |
| limit | Integer (0-100) | 10 | Max records per request |

**Response (200 OK):** Contains `timezone` and `sleep_session_list`. Items include `session_id`, `state`, analysis-range `session_start_time`/nullable `session_end_time`, required `measurement_start_time`, nullable `measurement_end_time`, `created_timezone`, nullable `unexpected_end_time`, `last_received_seq_num`, and `time_in_bed`. See the schema reference for nullability.

### POST /ai/v1/sessions/{session_id}/close -- Close Session

Ends recording. Requires `x-api-key` and `x-user-id`. The public [Close Session contract](https://docs-en.asleep.ai/docs/close-session.md) documents optional JSON `unexpected` (`0` or `1`, default `0`) for abnormal termination. Let the SDK close sessions it owns. Success is HTTP 200 with `{"detail":"success"}`; invalid `unexpected` values return 400, an invalid `x-user-id` returns 401, an already closed session returns 403, and a missing session or user returns 404. This reference does not publish undocumented Close request fields.

### POST /data/v1/sessions/{session_id}/reanalyze -- Reanalyze Session

Re-runs analysis of a COMPLETE session. Requires `x-api-key` and `x-user-id`. Send a JSON object body (`{}` when no bounds are selected). Optional **flat** `start_time` and `end_time` bounds must be UTC and ordered. After success, refetch the report and invalidate cached values.

```bash
curl -X POST https://api.asleep.ai/data/v1/sessions/SESSION_ID/reanalyze \
  -H "x-api-key: YOUR_API_KEY" \
  -H "x-user-id: USER_ID" \
  -H "Content-Type: application/json" \
  -d '{"start_time":"2025-01-15T03:00:00+00:00","end_time":"2025-01-15T07:00:00+00:00"}'
```

Success is HTTP 200 with `{"detail":"reanalyze success"}` or `{"detail":"already analyzed"}`.

| Status | Publicly documented condition |
|--------|-------------------------------|
| 400 | Invalid session ID, non-UTC or unordered bounds, or range exceeding available data |
| 401 / 403 | Authentication failure / inaccessible customer |
| 404 | Session **or sleep analysis** not found |
| 409 | Reanalysis already in progress |
| 410 | Analysis result no longer exists |
| 422 | Session not COMPLETE |

Send bounds inside the recorded measurement window (`measurement_start_time`..`measurement_end_time`). Handle 400 for invalid, unordered, or out-of-range bounds, and do not rely on the server to adjust bounds. Do not infer model-specific minimum durations from this public reference.

### DELETE /ai/v1/sessions/{session_id} -- Delete Session

Permanently removes a session, its uploaded audio, and analysis data.

```bash
curl -X DELETE https://api.asleep.ai/ai/v1/sessions/SESSION_ID \
  -H "x-api-key: YOUR_API_KEY" \
  -H "x-user-id: USER_ID"
```

**Response (204 No Content):** Empty body on success.

## Statistics Endpoint

### GET /data/v1/users/{user_id}/average-stats -- Get Average Stats

Returns aggregated sleep metrics over a date range.

```bash
curl "https://api.asleep.ai/data/v1/users/USER_ID/average-stats?start_date=2024-01-01&end_date=2024-01-31" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "timezone: UTC"
```

**Query parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| start_date | String (YYYY-MM-DD) | Yes | Period start |
| end_date | String (YYYY-MM-DD) | Yes | Period end (max 100 days from start) |

**Response (200 OK):**

| Field | Description |
|-------|-------------|
| period | Object with timezone, start_date, and end_date (no days field) |
| peculiarities | Array of conditions (e.g., NO_BREATHING_STABILITY) |
| average_stats | Object with averaged metrics: times, durations, ratios, event counts |
| slept_sessions | Array of flattened per-session metrics where sleep was detected (see schema) |
| never_slept_sessions | Array of no-sleep session objects (id, start_time, end_time, completed_time) |

See `references/response_schemas.md` for the full average_stats field listing.

## Webhook Events (V3)

Register `callback_url` when creating a session, or use the platform-specific SDK callback URL option. V3 payloads use top-level `event`, `version`, and **`data`**. See the schema reference for event-specific fields. Do not flatten `data` or invent `time_window`.

- `INFERENCE_COMPLETE`: repeated incremental results; deduplicate by `data.session_id + event + data.inference_seq_num`, and reject stale sequence updates.
- `SESSION_COMPLETE`: final session result under `data`, with `user_id` added. Its session identifier is `data.session.id`; deduplicate by that ID plus event.

The public webhook field table calls the inference stage field `stage_list`, while its V3 example uses `sleep_stages`. Confirm the actual payload for the pinned integration before hard-coding a deserializer; the reference example follows the V3 example. Do not silently conflate these names.

### Authentication and delivery

Requests include `x-api-key` and `x-user-id`. If a webhook secret has been issued, require `X-Asleep-Timestamp` and `X-Asleep-Signature`: verify HMAC-SHA256 over `timestamp + "." + raw_body`, using the exact raw request bytes, a timing-safe comparison, and any supported `v1=` signature in the comma-separated header (rotation may supply multiple). Reject malformed headers and timestamps; the official guide recommends a five-minute freshness tolerance. Parse JSON after verification. When no secret has been issued these signature headers are absent. See [HMAC verification](https://docs-en.asleep.ai/docs/webhook-hmac-signature-verification.md).

Return a 2xx response after durable acceptance and process asynchronously. The [delivery policy](https://docs-en.asleep.ai/docs/webhook.md) retries network/timeouts and HTTP 408, 429, or 5xx; duplicate and out-of-order delivery are possible. Do not assume exactly-once delivery or an undocumented 30-second response contract.

## Error Codes

| Status | Condition | Description |
|--------|-----------|-------------|
| 401 | Unauthorized | API key missing or invalid |
| 403 | Plan expired | Subscription period ended |
| 403 | Rate limit exceeded | Request quota temporarily exceeded |
| 403 | Quota exceeded | Total usage limit surpassed |
| 404 | Not Found | Requested resource does not exist |

## API Versioning

Endpoints use explicit version prefixes:

| Domain | Version | Endpoints |
|--------|---------|-----------|
| User management | /ai/v1/ | Create, Get, Update, Delete user; Close/Delete session |
| Session data (get) | /data/v4/ (recommended), /data/v3/ (deprecated) | Get single session |
| Session data (list/reanalyze) | /data/v1/ | List or reanalyze sessions |
| Statistics | /data/v1/ | Average stats |

Allow additive response fields and pin endpoint and SDK versions. Check official version notices before migration.

## Best Practices

**Invalidate reports after reanalysis.** COMPLETE is not an immutability guarantee. Refetch after a successful reanalysis.

**Use exponential backoff on rate limits.** When you receive a 403 rate-limit response, wait with increasing delays (e.g., 1s, 2s, 4s) before retrying. Cap at a reasonable maximum (e.g., 60s).

**Ensure webhook idempotency.** Use the event-specific keys above; session/event alone would discard later inference updates.

**Respond to webhooks quickly.** Return a 2xx status within a few seconds. Offload heavy processing to a background queue to avoid webhook timeouts.

**Store API keys securely.** Use environment variables or a secret manager. Never commit keys to version control.

## Links

- **Documentation:** https://docs-en.asleep.ai
- **Dashboard:** https://dashboard.asleep.ai
- **LLM-optimized reference:** https://docs-en.asleep.ai/llms.txt

## Reference Files

This skill includes detailed response schemas and field definitions:

- `references/response_schemas.md`: Complete field-by-field documentation for all response objects, webhook payloads, enumerations, and example JSON

## Contract sources

Verified against public documentation on 2026-10-19: [Get Session v3](https://docs-en.asleep.ai/docs/data-api-get-session.md), [v4](https://docs-en.asleep.ai/docs/data-api-get-session-v4.md), [List Sessions](https://docs-en.asleep.ai/docs/data-api-list-sessions.md), [Average Stats](https://docs-en.asleep.ai/docs/get-users-average-stats.md), [Create User](https://docs-en.asleep.ai/docs/session-api-create-user.md), [Update User](https://docs-en.asleep.ai/docs/put-update-user.md), and [Get User](https://docs-en.asleep.ai/docs/session-api-get-user.md). Public documentation defines the customer-facing contract; tolerate additive fields.
