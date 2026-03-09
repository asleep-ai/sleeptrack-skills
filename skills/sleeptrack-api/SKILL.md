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

All successful responses follow this envelope structure:

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

**Request body** (all fields optional):

| Field | Type | Constraints |
|-------|------|-------------|
| metadata.birth_year | Integer | |
| metadata.birth_month | Integer | 1-12 |
| metadata.birth_day | Integer | 1-31 |
| metadata.gender | String | male, female, non_binary, other, prefer_not_to_say |
| metadata.height | Float | cm, 0-300 |
| metadata.weight | Float | kg, 0-1000 |

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
| last_session_info | Object with session_id, state, session_start_time, session_end_time |
| metadata | Object with demographic fields (birth_year, gender, height, weight, etc.) |

### PUT /ai/v1/users/{user_id} -- Update User

Updates user metadata. Send the same `metadata` object as in Create User.

```bash
curl -X PUT https://api.asleep.ai/ai/v1/users/USER_ID \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"metadata": {"height": 180.0, "weight": 75.0}}'
```

**Response (200 OK):** Returns the `user_id` in `result`.

### DELETE /ai/v1/users/{user_id} -- Delete User

Permanently removes a user and all associated sessions and data.

```bash
curl -X DELETE https://api.asleep.ai/ai/v1/users/USER_ID \
  -H "x-api-key: YOUR_API_KEY"
```

**Response (204 No Content):** Empty body on success.

## Session Data Endpoints

### GET /data/v3/sessions/{session_id} -- Get Session

Returns the full session object (sleep stages, snoring stages) and the stat object (all computed metrics) for a single session.

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
| stat | Object with all computed sleep metrics (latencies, durations, ratios, cycles) |

Sleep stages are recorded every 30 seconds: -1 (unknown), 0 (wake), 1 (light), 2 (deep), 3 (REM).

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
| limit | Integer (0-100) | 20 | Max records per request |

**Response (200 OK):** Contains `timezone` and `sleep_session_list` array. Each item includes `session_id`, `state`, `session_start_time`, `session_end_time`, `created_timezone`, `unexpected_end_time`, `last_received_seq_num`, and `time_in_bed`.

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
| period | Object with start_date, end_date, and days count |
| peculiarities | Array of conditions (e.g., NO_BREATHING_STABILITY) |
| average_stats | Object with averaged metrics: times, durations, ratios, event counts |
| slept_sessions | Array of session references where sleep was detected |
| never_slept_sessions | Array of session references where no sleep was detected |

See `references/response_schemas.md` for the full average_stats field listing.

## Webhook Events

Asleep sends HTTP POST requests to your configured callback URL when sleep data events occur. Configure your callback URL via the SDK `callbackUrl` parameter or through the Asleep Dashboard.

Webhook requests include `x-api-key` and `x-user-id` headers for authentication. Always use webhook version **V3** for new integrations.

### INFERENCE_COMPLETE

Fires every 5-40 minutes during active sleep tracking, providing incremental analysis data.

**Payload fields:** `event`, `version`, `user_id`, `session_id`, `seq_num`, `inference_seq_num`, `sleep_stages`, `snoring_stages`, `time_window` (start, end).

```bash
# Example: simulating a webhook POST to your endpoint
curl -X POST https://your-server.com/webhook \
  -H "x-api-key: YOUR_API_KEY" \
  -H "x-user-id: USER_ID" \
  -H "Content-Type: application/json" \
  -d '{"event":"INFERENCE_COMPLETE","version":"V3","session_id":"s123","sleep_stages":[1,1,2]}'
```

### SESSION_COMPLETE

Fires once when a session's analysis finishes. The payload mirrors the Get Session response structure, with the addition of top-level `event`, `version`, and `user_id` fields.

**Payload fields:** `event`, `version`, `user_id`, `session_id`, `session` (full object), `stat` (full object), `peculiarities`.

See `references/response_schemas.md` for complete webhook payload schemas with field tables and example JSON.

### Webhook Versions

| Version | Status | Notes |
|---------|--------|-------|
| V1 | Legacy | Original format |
| V2 | Legacy | Additional fields |
| V3 | Current | Recommended for all new integrations |

### Webhook Authentication

Asleep sends `x-api-key` and `x-user-id` as headers on every POST to your callback URL. Verify these headers match expected values before processing the payload.

### Webhook Configuration

Webhooks can be configured in two ways:

1. **SDK parameter:** Pass `callbackUrl` when initializing the Asleep SDK in your mobile app. The SDK will register the URL with the Asleep platform.
2. **Dashboard:** Set the webhook URL in the Asleep Dashboard under your application settings.

Your callback endpoint must be a publicly accessible HTTPS URL that responds with a 2xx status code within 30 seconds.

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
| User management | /ai/v1/ | Create, Get, Update, Delete user; Delete session |
| Session data (get) | /data/v3/ | Get single session |
| Session data (list) | /data/v1/ | List sessions |
| Statistics | /data/v1/ | Average stats |

**Version bump policy:** Adding new fields to a response does not trigger a version change. Version changes occur only when field names, types, or structural shapes change.

## Best Practices

**Cache completed sessions.** Once a session reaches COMPLETE state, its data is immutable. Store the response locally to avoid redundant API calls.

**Use exponential backoff on rate limits.** When you receive a 403 rate-limit response, wait with increasing delays (e.g., 1s, 2s, 4s) before retrying. Cap at a reasonable maximum (e.g., 60s).

**Ensure webhook idempotency.** Track processed session/event IDs to avoid duplicate processing if the same webhook fires more than once.

**Respond to webhooks quickly.** Return a 2xx status within a few seconds. Offload heavy processing to a background queue to avoid webhook timeouts.

**Store API keys securely.** Use environment variables or a secret manager. Never commit keys to version control.

## Links

- **Documentation:** https://docs-en.asleep.ai
- **Dashboard:** https://dashboard.asleep.ai
- **LLM-optimized reference:** https://docs-en.asleep.ai/llms.txt

## Reference Files

This skill includes detailed response schemas and field definitions:

- `references/response_schemas.md`: Complete field-by-field documentation for all response objects, webhook payloads, enumerations, and example JSON
