# Asleep Webhook Reference

This reference provides comprehensive documentation for implementing Asleep webhooks in backend applications.

## Overview

Asleep webhooks enable real-time notifications about sleep session events. The system sends HTTP POST requests to your configured callback URL when specific events occur.

## Webhook Configuration

Webhooks are configured by providing a callback URL during session operations (via SDK) or through the Asleep Dashboard.

**Callback URL Requirements:**
- Must be publicly accessible HTTPS endpoint
- Should respond with 2xx status code
- Should handle requests within 30 seconds

## Authentication

Webhook requests include authentication headers:

```http
x-api-key: YOUR_API_KEY
x-user-id: USER_ID
```

**Security Best Practices:**
- Verify the `x-api-key` matches your expected API key
- Validate the `x-user-id` belongs to your system
- Use HTTPS for your webhook endpoint
- Implement request signing if needed
- Log all webhook attempts for audit

## Supported Events

Asleep webhooks support two primary event types:

### 1. INFERENCE_COMPLETE

Triggered during sleep session analysis at regular intervals (every 5 or 40 minutes).

**Use Cases:**
- Real-time sleep stage monitoring
- Live dashboard updates
- Progressive data analysis
- User notifications during tracking

**Timing:**
- Fires every 5 minutes during active tracking
- May also fire at 40-minute intervals
- Multiple events per session

### 2. SESSION_COMPLETE

Triggered when complete sleep session analysis finishes.

**Use Cases:**
- Final report generation
- User notifications
- Data storage
- Statistics calculation
- Integration with other systems

**Timing:**
- Fires once per session
- Occurs after session end
- Contains complete analysis

## Webhook Payload Schemas

### INFERENCE_COMPLETE Payload

Provides incremental sleep analysis data.

**Structure:**
```json
{
  "event": "INFERENCE_COMPLETE",
  "version": "V3",
  "timestamp": "2024-01-21T06:15:00Z",
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "session_id": "session123",
  "seq_num": 60,
  "inference_seq_num": 12,
  "sleep_stages": [1, 1, 2, 2, 2],
  "breath_stages": [0, 0, 0, 0, 0],
  "snoring_stages": [0, 0, 1, 1, 0],
  "time_window": {
    "start": "2024-01-21T06:10:00Z",
    "end": "2024-01-21T06:15:00Z"
  }
}
```

**Field Descriptions:**

| Field | Type | Description |
|-------|------|-------------|
| event | String | Always "INFERENCE_COMPLETE" |
| version | String | API version (V1, V2, V3) |
| timestamp | String (ISO 8601) | Event generation time |
| user_id | String | User identifier |
| session_id | String | Session identifier |
| seq_num | Integer | Audio data upload sequence number |
| inference_seq_num | Integer | Analysis sequence (5-minute increments) |
| sleep_stages | Array[Integer] | Sleep stage values for time window |
| breath_stages | Array[Integer] | Breathing stability indicators |
| snoring_stages | Array[Integer] | Snoring detection values |
| time_window | Object | Time range for this analysis chunk |

**Sleep Stage Values:**
- `-1`: Unknown/No data
- `0`: Wake
- `1`: Light sleep
- `2`: Deep sleep
- `3`: REM sleep

**Snoring Stage Values:**
- `0`: No snoring
- `1`: Snoring detected

**Example Handler (Python):**
```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/webhook', methods=['POST'])
def handle_inference():
    # Verify authentication
    api_key = request.headers.get('x-api-key')
    user_id = request.headers.get('x-user-id')

    if api_key != EXPECTED_API_KEY:
        return jsonify({"error": "Unauthorized"}), 401

    # Parse payload
    data = request.json

    if data['event'] == 'INFERENCE_COMPLETE':
        session_id = data['session_id']
        sleep_stages = data['sleep_stages']

        # Process incremental data
        update_live_dashboard(session_id, sleep_stages)

        # Store for real-time analysis
        store_incremental_data(data)

    return jsonify({"status": "received"}), 200
```

**Example Handler (Node.js):**
```javascript
const express = require('express');
const app = express();

app.use(express.json());

app.post('/webhook', async (req, res) => {
  // Verify authentication
  const apiKey = req.headers['x-api-key'];
  const userId = req.headers['x-user-id'];

  if (apiKey !== process.env.ASLEEP_API_KEY) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  const { event, session_id, sleep_stages } = req.body;

  if (event === 'INFERENCE_COMPLETE') {
    // Update real-time dashboard
    await updateLiveDashboard(session_id, sleep_stages);

    // Store incremental data
    await storeIncrementalData(req.body);
  }

  res.status(200).json({ status: 'received' });
});
```

---

### SESSION_COMPLETE Payload

Provides comprehensive final sleep analysis.

**Structure:**
```json
{
  "event": "SESSION_COMPLETE",
  "version": "V3",
  "timestamp": "2024-01-21T06:30:00Z",
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "session_id": "session123",
  "session": {
    "id": "session123",
    "state": "COMPLETE",
    "start_time": "2024-01-20T22:00:00+00:00",
    "end_time": "2024-01-21T06:30:00+00:00",
    "timezone": "UTC",
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
  },
  "peculiarities": []
}
```

**Field Descriptions:**

| Field | Type | Description |
|-------|------|-------------|
| event | String | Always "SESSION_COMPLETE" |
| version | String | API version (V1, V2, V3) |
| timestamp | String (ISO 8601) | Event generation time |
| user_id | String | User identifier |
| session_id | String | Session identifier |
| session | Object | Complete session data |
| stat | Object | Comprehensive sleep statistics |
| peculiarities | Array[String] | Special session conditions |

**Session Object Fields:**
- `id`: Session identifier
- `state`: Always "COMPLETE" for this event
- `start_time`, `end_time`: Session timestamps (ISO 8601)
- `timezone`: Timezone of the session
- `sleep_stages`: Complete sleep stage timeline
- `snoring_stages`: Complete snoring timeline

**Stat Object Fields:**
- `sleep_time`: Total sleep duration (HH:MM:SS)
- `sleep_index`: Overall sleep quality score (0-100)
- `sleep_latency`: Time to fall asleep (seconds)
- `time_in_bed`: Total time in bed (seconds)
- `time_in_sleep`: Total actual sleep time (seconds)
- `time_in_light/deep/rem`: Stage durations (seconds)
- `sleep_efficiency`: Percentage of time spent sleeping
- `waso_count`: Wake after sleep onset episodes
- `longest_waso`: Longest wake episode (seconds)
- `sleep_cycle`: Array of sleep cycle objects

**Peculiarities:**
- `IN_PROGRESS`: Analysis still ongoing (shouldn't occur for COMPLETE)
- `NEVER_SLEPT`: No sleep detected
- `TOO_SHORT_FOR_ANALYSIS`: Session < 5 minutes
- `NO_BREATHING_STABILITY`: Inconsistent breathing data

**Example Handler (Python):**
```python
from flask import Flask, request, jsonify
import logging

app = Flask(__name__)
logger = logging.getLogger(__name__)

@app.route('/webhook', methods=['POST'])
def handle_session_complete():
    # Verify authentication
    api_key = request.headers.get('x-api-key')
    user_id = request.headers.get('x-user-id')

    if api_key != EXPECTED_API_KEY:
        logger.warning(f"Unauthorized webhook attempt from {request.remote_addr}")
        return jsonify({"error": "Unauthorized"}), 401

    # Parse payload
    data = request.json

    if data['event'] == 'SESSION_COMPLETE':
        session_id = data['session_id']
        stat = data['stat']

        # Store complete report
        save_sleep_report(user_id, session_id, data)

        # Send user notification
        notify_user(user_id, {
            'session_id': session_id,
            'sleep_time': stat['sleep_time'],
            'sleep_efficiency': stat['sleep_efficiency'],
            'sleep_index': stat['sleep_index']
        })

        # Update user statistics
        update_user_statistics(user_id)

        # Trigger integrations
        sync_to_health_platform(user_id, data)

        logger.info(f"Processed SESSION_COMPLETE for {session_id}")

    return jsonify({"status": "processed"}), 200
```

**Example Handler (Node.js):**
```javascript
const express = require('express');
const app = express();

app.use(express.json());

app.post('/webhook', async (req, res) => {
  // Verify authentication
  const apiKey = req.headers['x-api-key'];
  const userId = req.headers['x-user-id'];

  if (apiKey !== process.env.ASLEEP_API_KEY) {
    console.warn('Unauthorized webhook attempt');
    return res.status(401).json({ error: 'Unauthorized' });
  }

  const { event, session_id, stat } = req.body;

  if (event === 'SESSION_COMPLETE') {
    try {
      // Store complete report
      await saveSleepReport(userId, session_id, req.body);

      // Send user notification
      await notifyUser(userId, {
        sessionId: session_id,
        sleepTime: stat.sleep_time,
        sleepEfficiency: stat.sleep_efficiency,
        sleepIndex: stat.sleep_index
      });

      // Update statistics
      await updateUserStatistics(userId);

      // Sync to integrations
      await syncToHealthPlatform(userId, req.body);

      console.log(`Processed SESSION_COMPLETE for ${session_id}`);

      res.status(200).json({ status: 'processed' });
    } catch (error) {
      console.error('Webhook processing error:', error);
      res.status(500).json({ error: 'Processing failed' });
    }
  } else {
    res.status(200).json({ status: 'received' });
  }
});
```

---

## Webhook Versioning

Webhooks support three format versions for backward compatibility:

### V1 (Legacy)
Original webhook format. Use V3 for new implementations.

### V2 (Legacy)
Updated format with additional fields. Use V3 for new implementations.

### V3 (Current)
Latest format with comprehensive data structures. Recommended for all new integrations.

**Version Selection:**
Configure webhook version through SDK initialization or Dashboard settings.

---

## Implementation Guide

### 1. Set Up Webhook Endpoint

Create a public HTTPS endpoint to receive webhook events:

**Python (Flask):**
```python
from flask import Flask, request, jsonify
import hmac
import hashlib

app = Flask(__name__)

@app.route('/asleep-webhook', methods=['POST'])
def asleep_webhook():
    # Verify authentication
    if not verify_webhook(request):
        return jsonify({"error": "Unauthorized"}), 401

    # Parse event
    event = request.json
    event_type = event.get('event')

    # Route to appropriate handler
    if event_type == 'INFERENCE_COMPLETE':
        handle_inference_complete(event)
    elif event_type == 'SESSION_COMPLETE':
        handle_session_complete(event)

    return jsonify({"status": "success"}), 200

def verify_webhook(request):
    api_key = request.headers.get('x-api-key')
    return api_key == EXPECTED_API_KEY

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=443, ssl_context='adhoc')
```

**Node.js (Express):**
```javascript
const express = require('express');
const https = require('https');
const fs = require('fs');

const app = express();
app.use(express.json());

app.post('/asleep-webhook', async (req, res) => {
  // Verify authentication
  if (!verifyWebhook(req)) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  const { event } = req.body;

  try {
    switch (event) {
      case 'INFERENCE_COMPLETE':
        await handleInferenceComplete(req.body);
        break;
      case 'SESSION_COMPLETE':
        await handleSessionComplete(req.body);
        break;
      default:
        console.warn(`Unknown event type: ${event}`);
    }

    res.status(200).json({ status: 'success' });
  } catch (error) {
    console.error('Webhook error:', error);
    res.status(500).json({ error: 'Processing failed' });
  }
});

function verifyWebhook(req) {
  const apiKey = req.headers['x-api-key'];
  return apiKey === process.env.ASLEEP_API_KEY;
}

// HTTPS server
const options = {
  key: fs.readFileSync('private-key.pem'),
  cert: fs.readFileSync('certificate.pem')
};

https.createServer(options, app).listen(443);
```

### 2. Configure Webhook URL

Configure your webhook URL through:
- SDK initialization (for mobile apps)
- Asleep Dashboard (for backend integrations)

**SDK Example (Android):**
```kotlin
AsleepConfig.init(
    apiKey = "YOUR_API_KEY",
    userId = "user123",
    callbackUrl = "https://your-domain.com/asleep-webhook"
)
```

### 3. Handle Webhook Events

Implement handlers for each event type:

**Python Example:**
```python
def handle_inference_complete(event):
    """Process incremental sleep data"""
    session_id = event['session_id']
    sleep_stages = event['sleep_stages']

    # Update real-time dashboard
    redis_client.set(f"session:{session_id}:latest", json.dumps(sleep_stages))

    # Notify connected clients via WebSocket
    websocket_broadcast(session_id, sleep_stages)

    # Store for analysis
    db.incremental_data.insert_one(event)

def handle_session_complete(event):
    """Process complete sleep report"""
    user_id = event['user_id']
    session_id = event['session_id']
    stat = event['stat']

    # Store complete report
    db.sleep_reports.insert_one({
        'user_id': user_id,
        'session_id': session_id,
        'date': event['session']['start_time'],
        'statistics': stat,
        'created_at': datetime.now()
    })

    # Update user's latest statistics
    update_user_stats(user_id)

    # Send push notification
    send_notification(user_id, {
        'title': 'Sleep Report Ready',
        'body': f"Sleep time: {stat['sleep_time']}, Efficiency: {stat['sleep_efficiency']:.1f}%"
    })

    # Trigger downstream processes
    calculate_weekly_trends(user_id)
    check_sleep_goals(user_id, stat)
```

### 4. Error Handling

Implement robust error handling:

**Retry Logic:**
```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=2, max=10))
def process_webhook(event):
    """Process webhook with automatic retry"""
    # Your processing logic here
    pass

@app.route('/webhook', methods=['POST'])
def webhook_endpoint():
    try:
        event = request.json
        process_webhook(event)
        return jsonify({"status": "success"}), 200
    except Exception as e:
        logger.error(f"Webhook processing failed: {e}")
        return jsonify({"status": "error", "message": str(e)}), 500
```

**Idempotency:**
```python
def handle_session_complete(event):
    session_id = event['session_id']

    # Check if already processed
    if db.processed_webhooks.find_one({'session_id': session_id}):
        logger.info(f"Session {session_id} already processed")
        return

    # Process event
    save_sleep_report(event)

    # Mark as processed
    db.processed_webhooks.insert_one({
        'session_id': session_id,
        'processed_at': datetime.now()
    })
```

### 5. Testing

Test webhook handling locally:

**ngrok for Local Testing:**
```bash
# Start your local server
python app.py

# In another terminal, expose with ngrok
ngrok http 5000

# Use the ngrok URL as your webhook URL
# Example: https://abc123.ngrok.io/webhook
```

**Mock Webhook Requests:**
```bash
# Test INFERENCE_COMPLETE
curl -X POST http://localhost:5000/webhook \
  -H "Content-Type: application/json" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "x-user-id: test_user" \
  -d '{
    "event": "INFERENCE_COMPLETE",
    "version": "V3",
    "session_id": "test123",
    "sleep_stages": [1, 1, 2]
  }'

# Test SESSION_COMPLETE
curl -X POST http://localhost:5000/webhook \
  -H "Content-Type: application/json" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "x-user-id: test_user" \
  -d '{
    "event": "SESSION_COMPLETE",
    "version": "V3",
    "session_id": "test123",
    "stat": {
      "sleep_time": "07:30:00",
      "sleep_efficiency": 88.5
    }
  }'
```

---

## Best Practices

### Security
- Always verify `x-api-key` header
- Use HTTPS for webhook endpoints
- Implement request signing if handling sensitive data
- Rate limit webhook endpoint
- Log all webhook attempts

### Reliability
- Respond quickly (< 5 seconds ideal)
- Process asynchronously if needed
- Implement idempotency checks
- Handle duplicate events gracefully
- Return 2xx status even if processing fails (retry logic)

### Performance
- Use message queues for heavy processing
- Implement caching where appropriate
- Batch database operations
- Monitor webhook response times
- Scale horizontally if needed

### Monitoring
- Log all webhook events
- Track processing success/failure rates
- Monitor response times
- Set up alerts for failures
- Dashboard for webhook metrics

### Error Handling
- Catch and log all exceptions
- Return appropriate HTTP status codes
- Implement exponential backoff
- Dead letter queue for failed events
- Manual review process for failures

---

## Common Use Cases

### Real-Time Dashboard Updates

```python
@app.route('/webhook', methods=['POST'])
def webhook():
    event = request.json

    if event['event'] == 'INFERENCE_COMPLETE':
        # Broadcast to connected WebSocket clients
        socketio.emit('sleep_update', {
            'session_id': event['session_id'],
            'sleep_stages': event['sleep_stages'],
            'timestamp': event['timestamp']
        }, room=event['user_id'])

    return jsonify({"status": "success"}), 200
```

### User Notifications

```python
def handle_session_complete(event):
    user_id = event['user_id']
    stat = event['stat']

    # Generate insights
    insights = generate_sleep_insights(stat)

    # Send push notification
    send_push_notification(user_id, {
        'title': 'Your Sleep Report is Ready!',
        'body': f"You slept for {stat['sleep_time']} with {stat['sleep_efficiency']:.0f}% efficiency",
        'data': {
            'session_id': event['session_id'],
            'insights': insights
        }
    })
```

### Data Analytics Pipeline

```python
def handle_session_complete(event):
    # Store in data warehouse
    bigquery_client.insert_rows_json('sleep_data.sessions', [{
        'user_id': event['user_id'],
        'session_id': event['session_id'],
        'date': event['session']['start_time'],
        'statistics': json.dumps(event['stat']),
        'ingested_at': datetime.now().isoformat()
    }])

    # Trigger analytics jobs
    trigger_weekly_report_job(event['user_id'])
    update_cohort_analysis()
```

### Integration with Other Systems

```python
def handle_session_complete(event):
    user_id = event['user_id']
    stat = event['stat']

    # Sync to Apple Health
    sync_to_apple_health(user_id, {
        'sleep_analysis': stat,
        'date': event['session']['start_time']
    })

    # Update CRM
    update_crm_profile(user_id, {
        'last_sleep_date': event['session']['start_time'],
        'avg_sleep_efficiency': calculate_avg_efficiency(user_id)
    })
```

---

## Troubleshooting

### Webhook Not Received

**Check:**
- Endpoint is publicly accessible
- HTTPS is properly configured
- Firewall allows incoming requests
- Webhook URL is correctly configured
- Server is running and healthy

### Authentication Failures

**Check:**
- `x-api-key` validation logic
- API key matches dashboard
- Headers are correctly parsed
- Case sensitivity of header names

### Duplicate Events

**Solution:**
```python
def handle_webhook(event):
    event_id = f"{event['session_id']}:{event['event']}:{event['timestamp']}"

    # Check if already processed
    if redis_client.exists(f"processed:{event_id}"):
        return

    # Process event
    process_event(event)

    # Mark as processed (expire after 24 hours)
    redis_client.setex(f"processed:{event_id}", 86400, "1")
```

### Processing Delays

**Solution:**
```python
from celery import Celery

celery = Celery('tasks', broker='redis://localhost:6379')

@app.route('/webhook', methods=['POST'])
def webhook():
    event = request.json

    # Queue for async processing
    process_webhook_async.delay(event)

    # Respond immediately
    return jsonify({"status": "queued"}), 200

@celery.task
def process_webhook_async(event):
    # Heavy processing here
    pass
```

---

## Resources

- **Official Documentation**: https://docs-en.asleep.ai/docs/webhook.md
- **API Basics**: https://docs-en.asleep.ai/docs/api-basics.md
- **Dashboard**: https://dashboard.asleep.ai
