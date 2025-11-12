# Asleep REST API Reference

This reference provides comprehensive documentation for the Asleep REST API endpoints.

## Base URL

```
https://api.asleep.ai
```

## Authentication

All API requests require authentication via the `x-api-key` header:

```http
x-api-key: YOUR_API_KEY
```

Obtain your API key from the [Asleep Dashboard](https://dashboard.asleep.ai).

## Common Headers

| Header | Type | Required | Description |
|--------|------|----------|-------------|
| x-api-key | String | Yes | API authentication key |
| x-user-id | String | Conditional | Required for session operations |
| timezone | String | No | Response timezone (default: UTC) |

## Response Format

All responses follow this structure:

```json
{
  "detail": "message about the result",
  "result": { /* response data */ }
}
```

## Common Error Codes

| Status | Error | Description |
|--------|-------|-------------|
| 401 | Unauthorized | API Key missing or invalid |
| 403 | Plan expired | Subscription period ended |
| 403 | Rate limit exceeded | Request quota temporarily exceeded |
| 403 | Quota exceeded | Total usage limit surpassed |
| 404 | Not Found | Resource doesn't exist |

---

## User Management APIs

### [POST] Create User

Creates a new user for sleep tracking.

**Endpoint:**
```
POST https://api.asleep.ai/ai/v1/users
```

**Headers:**
```
x-api-key: YOUR_API_KEY
```

**Request Body (Optional):**
```json
{
  "metadata": {
    "birth_year": 1990,
    "birth_month": 5,
    "birth_day": 15,
    "gender": "male",
    "height": 175.5,
    "weight": 70.0
  }
}
```

**Metadata Fields:**
- `birth_year` (Integer): User's birth year
- `birth_month` (Integer): User's birth month (1-12)
- `birth_day` (Integer): User's birth day (1-31)
- `gender` (String): One of: `male`, `female`, `non_binary`, `other`, `prefer_not_to_say`
- `height` (Float): Height in cm (0-300)
- `weight` (Float): Weight in kg (0-1000)

**Response (201 Created):**
```json
{
  "detail": "success",
  "result": {
    "user_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**Example (curl):**
```bash
curl -X POST "https://api.asleep.ai/ai/v1/users" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "metadata": {
      "birth_year": 1990,
      "gender": "male",
      "height": 175.5,
      "weight": 70.0
    }
  }'
```

**Example (Python):**
```python
import requests

response = requests.post(
    "https://api.asleep.ai/ai/v1/users",
    headers={"x-api-key": "YOUR_API_KEY"},
    json={
        "metadata": {
            "birth_year": 1990,
            "gender": "male",
            "height": 175.5,
            "weight": 70.0
        }
    }
)
user_id = response.json()["result"]["user_id"]
```

---

### [GET] Get User

Retrieves user information and last session data.

**Endpoint:**
```
GET https://api.asleep.ai/ai/v1/users/{user_id}
```

**Headers:**
```
x-api-key: YOUR_API_KEY
```

**Path Parameters:**
- `user_id` (String): User identifier

**Response (200 OK):**
```json
{
  "detail": "success",
  "result": {
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "to_be_deleted": false,
    "last_session_info": {
      "session_id": "abc123",
      "state": "COMPLETE",
      "session_start_time": "2024-01-20T22:00:00+00:00",
      "session_end_time": "2024-01-21T06:30:00+00:00"
    },
    "metadata": {
      "birth_year": 1990,
      "birth_month": 5,
      "birth_day": 15,
      "gender": "male",
      "height": 175.5,
      "weight": 70.0
    }
  }
}
```

**Session States:**
- `OPEN`: Session in progress, audio uploads available
- `CLOSED`: Session terminated, analysis in progress
- `COMPLETE`: All analysis completed

**Error Response (404):**
```json
{
  "detail": "user does not exist"
}
```

**Example (curl):**
```bash
curl -X GET "https://api.asleep.ai/ai/v1/users/USER_ID" \
  -H "x-api-key: YOUR_API_KEY"
```

**Example (Python):**
```python
import requests

response = requests.get(
    f"https://api.asleep.ai/ai/v1/users/{user_id}",
    headers={"x-api-key": "YOUR_API_KEY"}
)
user_data = response.json()["result"]
```

---

### [DELETE] Delete User

Permanently removes a user and all associated data.

**Endpoint:**
```
DELETE https://api.asleep.ai/ai/v1/users/{user_id}
```

**Headers:**
```
x-api-key: YOUR_API_KEY
```

**Path Parameters:**
- `user_id` (String): User identifier

**Response (204 No Content):**
User information successfully deleted.

**Error Responses:**

401 Unauthorized:
```json
{
  "detail": "user_id is invalid"
}
```

404 Not Found:
```json
{
  "detail": "user does not exist"
}
```

**Example (curl):**
```bash
curl -X DELETE "https://api.asleep.ai/ai/v1/users/USER_ID" \
  -H "x-api-key: YOUR_API_KEY"
```

**Example (Python):**
```python
import requests

response = requests.delete(
    f"https://api.asleep.ai/ai/v1/users/{user_id}",
    headers={"x-api-key": "YOUR_API_KEY"}
)
# 204 No Content on success
```

---

## Session Management APIs

### [GET] Get Session

Retrieves comprehensive sleep analysis data for a specific session.

**Endpoint:**
```
GET https://api.asleep.ai/data/v3/sessions/{session_id}
```

**Headers:**
```
x-api-key: YOUR_API_KEY
x-user-id: USER_ID
timezone: Asia/Seoul  # Optional, defaults to UTC
```

**Path Parameters:**
- `session_id` (String): Session identifier

**Query Parameters:**
None

**Response (200 OK):**
```json
{
  "detail": "success",
  "result": {
    "timezone": "UTC",
    "peculiarities": [],
    "missing_data_ratio": 0.0,
    "session": {
      "id": "session123",
      "state": "COMPLETE",
      "start_time": "2024-01-20T22:00:00+00:00",
      "end_time": "2024-01-21T06:30:00+00:00",
      "sleep_stages": [0, 0, 1, 1, 2, 3, 2, 1, 0],
      "snoring_stages": [0, 0, 0, 1, 1, 0, 0, 0, 0]
    },
    "stat": {
      "sleep_time": "06:30:00",
      "sleep_index": 85.5,
      "sleep_latency": 900,
      "time_in_bed": 30600,
      "time_in_sleep": 27000,
      "time_in_light": 13500,
      "time_in_deep": 6750,
      "time_in_rem": 6750,
      "sleep_efficiency": 88.24,
      "waso_count": 2,
      "longest_waso": 300,
      "sleep_cycle": [
        {
          "order": 1,
          "start_time": "2024-01-20T22:15:00+00:00",
          "end_time": "2024-01-21T01:30:00+00:00"
        }
      ]
    }
  }
}
```

**Sleep Stages Values:**
- `-1`: Unknown/No data
- `0`: Wake
- `1`: Light sleep
- `2`: Deep sleep
- `3`: REM sleep

**Snoring Stages Values:**
- `0`: No snoring
- `1`: Snoring detected

**Peculiarities:**
- `IN_PROGRESS`: Session still being analyzed
- `NEVER_SLEPT`: No sleep detected in session
- `TOO_SHORT_FOR_ANALYSIS`: Session duration < 5 minutes

**Error Responses:**

400 Bad Request:
```json
{
  "detail": "Invalid timezone format"
}
```

404 Not Found:
```json
{
  "detail": "Session not found"
}
```

**Example (curl):**
```bash
curl -X GET "https://api.asleep.ai/data/v3/sessions/SESSION_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "x-user-id: USER_ID" \
  -H "timezone: UTC"
```

**Example (Python):**
```python
import requests

response = requests.get(
    f"https://api.asleep.ai/data/v3/sessions/{session_id}",
    headers={
        "x-api-key": "YOUR_API_KEY",
        "x-user-id": user_id,
        "timezone": "UTC"
    }
)
session_data = response.json()["result"]
```

---

### [GET] List Sessions

Retrieves multiple sessions with filtering and pagination.

**Endpoint:**
```
GET https://api.asleep.ai/data/v1/sessions
```

**Headers:**
```
x-api-key: YOUR_API_KEY
x-user-id: USER_ID
timezone: UTC  # Optional
```

**Query Parameters:**
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| date_gte | String (YYYY-MM-DD) | No | - | Sessions on or after this date |
| date_lte | String (YYYY-MM-DD) | No | - | Sessions on or before this date |
| order_by | String (ASC/DESC) | No | DESC | Sort direction by start time |
| offset | Integer | No | 0 | Number of records to skip |
| limit | Integer (0-100) | No | 20 | Maximum records per request |

**Response (200 OK):**
```json
{
  "detail": "success",
  "result": {
    "timezone": "UTC",
    "sleep_session_list": [
      {
        "session_id": "session123",
        "state": "COMPLETE",
        "session_start_time": "2024-01-20T22:00:00+00:00",
        "session_end_time": "2024-01-21T06:30:00+00:00",
        "created_timezone": "UTC",
        "unexpected_end_time": null,
        "last_received_seq_num": 156,
        "time_in_bed": 30600
      }
    ]
  }
}
```

**Error Response (400):**
```json
{
  "detail": "Invalid timezone"
}
```

**Example (curl):**
```bash
curl -X GET "https://api.asleep.ai/data/v1/sessions?date_gte=2024-01-01&limit=10" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "x-user-id: USER_ID"
```

**Example (Python):**
```python
import requests

response = requests.get(
    "https://api.asleep.ai/data/v1/sessions",
    headers={
        "x-api-key": "YOUR_API_KEY",
        "x-user-id": user_id
    },
    params={
        "date_gte": "2024-01-01",
        "date_lte": "2024-01-31",
        "limit": 50,
        "order_by": "DESC"
    }
)
sessions = response.json()["result"]["sleep_session_list"]
```

---

### [DELETE] Delete Session

Permanently removes a session and all associated data.

**Endpoint:**
```
DELETE https://api.asleep.ai/ai/v1/sessions/{session_id}
```

**Headers:**
```
x-api-key: YOUR_API_KEY
x-user-id: USER_ID
```

**Path Parameters:**
- `session_id` (String): Session identifier

**Response (204 No Content):**
Session, uploaded audio, and analysis data successfully deleted.

**Error Responses:**

401 Unauthorized:
```json
{
  "detail": "x-user-id is invalid"
}
```

404 Not Found (User):
```json
{
  "detail": "user does not exist"
}
```

404 Not Found (Session):
```json
{
  "detail": "session does not exist"
}
```

**Example (curl):**
```bash
curl -X DELETE "https://api.asleep.ai/ai/v1/sessions/SESSION_ID" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "x-user-id: USER_ID"
```

**Example (Python):**
```python
import requests

response = requests.delete(
    f"https://api.asleep.ai/ai/v1/sessions/{session_id}",
    headers={
        "x-api-key": "YOUR_API_KEY",
        "x-user-id": user_id
    }
)
# 204 No Content on success
```

---

## Statistics APIs

### [GET] Get Average Stats

Retrieves average sleep metrics over a specified time period (up to 100 days).

**Endpoint:**
```
GET https://api.asleep.ai/data/v1/users/{user_id}/average-stats
```

**Headers:**
```
x-api-key: YOUR_API_KEY
timezone: UTC  # Optional
```

**Path Parameters:**
- `user_id` (String): User identifier

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| start_date | String (YYYY-MM-DD) | Yes | Period start date |
| end_date | String (YYYY-MM-DD) | Yes | Period end date (max 100 days from start) |

**Response (200 OK):**
```json
{
  "detail": "success",
  "result": {
    "period": {
      "start_date": "2024-01-01",
      "end_date": "2024-01-31",
      "days": 31
    },
    "peculiarities": [],
    "average_stats": {
      "start_time": "22:30:00",
      "end_time": "06:45:00",
      "sleep_time": "07:15:00",
      "wake_time": "06:45:00",
      "sleep_latency": 900,
      "wakeup_latency": 300,
      "time_in_bed": 30600,
      "time_in_sleep_period": 29700,
      "time_in_sleep": 26100,
      "time_in_wake": 3600,
      "time_in_light": 13050,
      "time_in_deep": 6525,
      "time_in_rem": 6525,
      "time_in_snoring": 1800,
      "time_in_no_snoring": 24300,
      "sleep_efficiency": 85.29,
      "wake_ratio": 0.12,
      "sleep_ratio": 0.88,
      "light_ratio": 0.50,
      "deep_ratio": 0.25,
      "rem_ratio": 0.25,
      "snoring_ratio": 0.07,
      "no_snoring_ratio": 0.93,
      "waso_count": 2.5,
      "longest_waso": 420,
      "sleep_cycle_count": 4.2,
      "snoring_count": 15.3
    },
    "never_slept_sessions": [],
    "slept_sessions": [
      {
        "session_id": "session123",
        "session_start_time": "2024-01-20T22:00:00+00:00"
      }
    ]
  }
}
```

**Metrics Explanation:**

**Time Metrics** (HH:MM:SS format or seconds):
- `start_time`: Average bedtime
- `end_time`: Average wake time
- `sleep_time`: Average time of falling asleep
- `wake_time`: Average time of waking up
- `sleep_latency`: Average time to fall asleep (seconds)
- `wakeup_latency`: Average time from wake to getting up (seconds)
- `time_in_bed`: Average total time in bed (seconds)
- `time_in_sleep_period`: Average time from sleep onset to wake (seconds)
- `time_in_sleep`: Average actual sleep time (seconds)
- `time_in_wake`: Average wake time during sleep period (seconds)

**Sleep Stage Durations** (seconds):
- `time_in_light`: Average light sleep duration
- `time_in_deep`: Average deep sleep duration
- `time_in_rem`: Average REM sleep duration

**Snoring Metrics** (seconds):
- `time_in_snoring`: Average snoring duration
- `time_in_no_snoring`: Average non-snoring duration

**Ratio Metrics** (0-1 decimal):
- `sleep_efficiency`: Sleep time / Time in bed
- `wake_ratio`, `sleep_ratio`: Wake/sleep proportions
- `light_ratio`, `deep_ratio`, `rem_ratio`: Sleep stage proportions
- `snoring_ratio`, `no_snoring_ratio`: Snoring proportions

**Event Counts**:
- `waso_count`: Average wake after sleep onset episodes
- `longest_waso`: Average longest wake episode (seconds)
- `sleep_cycle_count`: Average number of sleep cycles
- `snoring_count`: Average snoring episodes

**Peculiarities:**
- `NO_BREATHING_STABILITY`: Inconsistent breathing data

**Error Responses:**

400 Bad Request:
```json
{
  "detail": "The period should be less than or equal to 100 days"
}
```

404 Not Found:
```json
{
  "detail": "Unable to find the user of id {user_id}"
}
```

**Example (curl):**
```bash
curl -X GET "https://api.asleep.ai/data/v1/users/USER_ID/average-stats?start_date=2024-01-01&end_date=2024-01-31" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "timezone: UTC"
```

**Example (Python):**
```python
import requests

response = requests.get(
    f"https://api.asleep.ai/data/v1/users/{user_id}/average-stats",
    headers={
        "x-api-key": "YOUR_API_KEY",
        "timezone": "UTC"
    },
    params={
        "start_date": "2024-01-01",
        "end_date": "2024-01-31"
    }
)
stats = response.json()["result"]
```

---

## Rate Limiting

The Asleep API implements rate limiting to ensure fair usage:

- **Rate Limit Exceeded (403)**: Temporary quota exceeded
- **Quota Exceeded (403)**: Total usage limit reached
- **Plan Expired (403)**: Subscription period ended

Monitor your usage in the [Asleep Dashboard](https://dashboard.asleep.ai).

**Best Practices:**
- Implement exponential backoff for retries
- Cache responses when appropriate
- Batch requests when possible
- Monitor usage proactively

---

## API Versioning

The Asleep API uses versioned endpoints (e.g., `/v1/`, `/v3/`). Version upgrades occur when:

- Renaming response object fields
- Modifying data types or enum values
- Restructuring response objects
- Introducing breaking changes

Non-breaking changes (like adding new fields) don't trigger version upgrades.

**Current Versions:**
- User Management: `/ai/v1/`
- Session Data: `/data/v3/` (Get Session), `/data/v1/` (List Sessions)
- Statistics: `/data/v1/`
