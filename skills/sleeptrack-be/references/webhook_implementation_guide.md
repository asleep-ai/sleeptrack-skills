# Webhook Implementation Guide

This reference provides complete webhook handler implementations for both Python and Node.js, including best practices and payload examples.

## Flask Webhook Handler (Python)

```python
from flask import Flask, request, jsonify
import os
import logging

app = Flask(__name__)
logger = logging.getLogger(__name__)

EXPECTED_API_KEY = os.getenv("ASLEEP_API_KEY")

@app.route('/asleep-webhook', methods=['POST'])
def asleep_webhook():
    """Handle Asleep webhook events"""

    # Verify authentication
    api_key = request.headers.get('x-api-key')
    user_id = request.headers.get('x-user-id')

    if api_key != EXPECTED_API_KEY:
        logger.warning(f"Unauthorized webhook attempt from {request.remote_addr}")
        return jsonify({"error": "Unauthorized"}), 401

    # Parse event
    event = request.json
    event_type = event.get('event')

    logger.info(f"Received {event_type} event for user {user_id}")

    try:
        if event_type == 'INFERENCE_COMPLETE':
            handle_inference_complete(event)
        elif event_type == 'SESSION_COMPLETE':
            handle_session_complete(event)
        else:
            logger.warning(f"Unknown event type: {event_type}")

        return jsonify({"status": "success"}), 200

    except Exception as e:
        logger.error(f"Webhook processing error: {e}", exc_info=True)
        return jsonify({"status": "error", "message": str(e)}), 500

def handle_inference_complete(event):
    """Process incremental sleep data"""
    session_id = event['session_id']
    user_id = event['user_id']
    sleep_stages = event['sleep_stages']

    # Update real-time dashboard
    update_live_dashboard(session_id, sleep_stages)

    # Store incremental data
    db.incremental_data.insert_one(event)

    logger.info(f"Processed INFERENCE_COMPLETE for session {session_id}")

def handle_session_complete(event):
    """Process complete sleep report"""
    session_id = event['session_id']
    user_id = event['user_id']
    stat = event['stat']

    # Store complete report
    db.sleep_reports.insert_one({
        'user_id': user_id,
        'session_id': session_id,
        'date': event['session']['start_time'],
        'statistics': stat,
        'session_data': event['session'],
        'created_at': datetime.now()
    })

    # Send user notification
    send_push_notification(user_id, {
        'title': 'Sleep Report Ready',
        'body': f"Sleep time: {stat['sleep_time']}, Efficiency: {stat['sleep_efficiency']:.1f}%"
    })

    # Update user statistics
    update_user_aggregated_stats(user_id)

    logger.info(f"Processed SESSION_COMPLETE for session {session_id}")

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

## Express Webhook Handler (Node.js)

```javascript
const express = require('express');
const app = express();

app.use(express.json());

const EXPECTED_API_KEY = process.env.ASLEEP_API_KEY;

app.post('/asleep-webhook', async (req, res) => {
  // Verify authentication
  const apiKey = req.headers['x-api-key'];
  const userId = req.headers['x-user-id'];

  if (apiKey !== EXPECTED_API_KEY) {
    console.warn('Unauthorized webhook attempt');
    return res.status(401).json({ error: 'Unauthorized' });
  }

  const { event, session_id, stat } = req.body;
  console.log(`Received ${event} event for user ${userId}`);

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
    console.error('Webhook processing error:', error);
    res.status(500).json({ error: 'Processing failed' });
  }
});

async function handleInferenceComplete(event) {
  const { session_id, user_id, sleep_stages } = event;

  // Update real-time dashboard
  await updateLiveDashboard(session_id, sleep_stages);

  // Store incremental data
  await db.collection('incremental_data').insertOne(event);

  console.log(`Processed INFERENCE_COMPLETE for session ${session_id}`);
}

async function handleSessionComplete(event) {
  const { session_id, user_id, stat, session } = event;

  // Store complete report
  await db.collection('sleep_reports').insertOne({
    user_id,
    session_id,
    date: session.start_time,
    statistics: stat,
    session_data: session,
    created_at: new Date()
  });

  // Send user notification
  await sendPushNotification(user_id, {
    title: 'Sleep Report Ready',
    body: `Sleep time: ${stat.sleep_time}, Efficiency: ${stat.sleep_efficiency.toFixed(1)}%`
  });

  // Update user statistics
  await updateUserAggregatedStats(user_id);

  console.log(`Processed SESSION_COMPLETE for session ${session_id}`);
}

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => {
  console.log(`Webhook server listening on port ${PORT}`);
});
```

## Webhook Event Payloads

### INFERENCE_COMPLETE Event

Sent every 5-40 minutes during sleep tracking with incremental data.

```json
{
  "event": "INFERENCE_COMPLETE",
  "version": "V3",
  "timestamp": "2024-01-21T06:15:00Z",
  "user_id": "user123",
  "session_id": "session123",
  "seq_num": 60,
  "inference_seq_num": 12,
  "sleep_stages": [1, 1, 2, 2, 2],
  "snoring_stages": [0, 0, 1, 1, 0]
}
```

**Fields:**
- `event`: Event type identifier
- `version`: API version (V3)
- `timestamp`: Event timestamp in ISO 8601 format
- `user_id`: User identifier
- `session_id`: Sleep session identifier
- `seq_num`: Sequence number for raw data
- `inference_seq_num`: Sequence number for inference results
- `sleep_stages`: Array of sleep stage values (see sleep stages reference)
- `snoring_stages`: Array of snoring detection values (0 = no snoring, 1 = snoring)

### SESSION_COMPLETE Event

Sent when sleep session ends with complete analysis.

```json
{
  "event": "SESSION_COMPLETE",
  "version": "V3",
  "timestamp": "2024-01-21T06:30:00Z",
  "user_id": "user123",
  "session_id": "session123",
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
    "sleep_efficiency": 88.24,
    "time_in_bed": 30600,
    "time_in_sleep": 27000,
    "time_in_wake": 3600,
    "time_in_light": 14400,
    "time_in_deep": 7200,
    "time_in_rem": 5400,
    "waso_count": 2,
    "sleep_latency": 900,
    "sleep_cycle": [
      {
        "index": 0,
        "start_time": "2024-01-20T22:15:00+00:00",
        "end_time": "2024-01-21T01:45:00+00:00"
      },
      {
        "index": 1,
        "start_time": "2024-01-21T01:45:00+00:00",
        "end_time": "2024-01-21T05:15:00+00:00"
      }
    ]
  }
}
```

**Sleep Stage Values:**
- `-1`: Unknown/No data
- `0`: Wake
- `1`: Light sleep
- `2`: Deep sleep
- `3`: REM sleep

## Webhook Best Practices

### 1. Idempotency

Handle duplicate webhook deliveries gracefully:

**Python:**
```python
def handle_session_complete(event):
    session_id = event['session_id']

    # Check if already processed
    if db.processed_webhooks.find_one({'session_id': session_id, 'event': 'SESSION_COMPLETE'}):
        logger.info(f"Session {session_id} already processed, skipping")
        return

    # Process event
    save_sleep_report(event)

    # Mark as processed
    db.processed_webhooks.insert_one({
        'session_id': session_id,
        'event': 'SESSION_COMPLETE',
        'processed_at': datetime.now()
    })
```

**Node.js:**
```javascript
async function handleSessionComplete(event) {
  const sessionId = event.session_id;

  // Check if already processed
  const existing = await db.collection('processed_webhooks').findOne({
    session_id: sessionId,
    event: 'SESSION_COMPLETE'
  });

  if (existing) {
    console.log(`Session ${sessionId} already processed, skipping`);
    return;
  }

  // Process event
  await saveSleepReport(event);

  // Mark as processed
  await db.collection('processed_webhooks').insertOne({
    session_id: sessionId,
    event: 'SESSION_COMPLETE',
    processed_at: new Date()
  });
}
```

### 2. Asynchronous Processing

Process webhooks asynchronously to respond quickly:

**Python (Celery):**
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
    """Process webhook asynchronously"""
    if event['event'] == 'SESSION_COMPLETE':
        handle_session_complete(event)
```

**Node.js (Bull):**
```javascript
const Queue = require('bull');

const webhookQueue = new Queue('asleep-webhooks', {
  redis: { host: 'localhost', port: 6379 }
});

app.post('/webhook', async (req, res) => {
  const event = req.body;

  // Queue for async processing
  await webhookQueue.add(event);

  // Respond immediately
  res.status(200).json({ status: 'queued' });
});

// Process queued webhooks
webhookQueue.process(async (job) => {
  const event = job.data;

  if (event.event === 'SESSION_COMPLETE') {
    await handleSessionComplete(event);
  }
});
```

### 3. Security

Always verify webhook authenticity:

```python
@app.route('/webhook', methods=['POST'])
def webhook():
    # Verify API key
    api_key = request.headers.get('x-api-key')
    if api_key != EXPECTED_API_KEY:
        logger.warning(f"Unauthorized webhook from {request.remote_addr}")
        return jsonify({"error": "Unauthorized"}), 401

    # Verify user ID presence
    user_id = request.headers.get('x-user-id')
    if not user_id:
        logger.warning("Missing x-user-id header")
        return jsonify({"error": "Missing user ID"}), 400

    # Process webhook
    # ...
```

### 4. Error Handling

Implement robust error handling:

```python
@app.route('/webhook', methods=['POST'])
def webhook():
    try:
        event = request.json
        event_type = event.get('event')

        if event_type == 'SESSION_COMPLETE':
            handle_session_complete(event)
        elif event_type == 'INFERENCE_COMPLETE':
            handle_inference_complete(event)
        else:
            logger.warning(f"Unknown event type: {event_type}")
            return jsonify({"error": "Unknown event type"}), 400

        return jsonify({"status": "success"}), 200

    except ValueError as e:
        logger.error(f"Validation error: {e}")
        return jsonify({"error": str(e)}), 400
    except Exception as e:
        logger.error(f"Processing error: {e}", exc_info=True)
        return jsonify({"error": "Internal server error"}), 500
```

### 5. Logging

Log all webhook events for debugging:

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('webhooks.log'),
        logging.StreamHandler()
    ]
)

logger = logging.getLogger(__name__)

@app.route('/webhook', methods=['POST'])
def webhook():
    user_id = request.headers.get('x-user-id')
    event = request.json
    event_type = event.get('event')

    logger.info(f"Webhook received - Type: {event_type}, User: {user_id}, Session: {event.get('session_id')}")

    try:
        # Process webhook
        # ...
        logger.info(f"Webhook processed successfully - Session: {event.get('session_id')}")
    except Exception as e:
        logger.error(f"Webhook processing failed - Session: {event.get('session_id')}, Error: {e}", exc_info=True)
```

## Testing Webhooks Locally

### Using ngrok

```bash
# Start your local server
python app.py  # or npm start

# In another terminal, expose with ngrok
ngrok http 5000

# Use the ngrok URL as webhook URL in Asleep Dashboard
# Example: https://abc123.ngrok.io/asleep-webhook
```

### Mock Webhook for Testing

**Python:**
```python
import requests
import json

def send_test_webhook(url, event_type='SESSION_COMPLETE'):
    """Send test webhook to local server"""

    if event_type == 'SESSION_COMPLETE':
        payload = {
            "event": "SESSION_COMPLETE",
            "version": "V3",
            "timestamp": "2024-01-21T06:30:00Z",
            "user_id": "test_user",
            "session_id": "test_session",
            "session": {
                "id": "test_session",
                "state": "COMPLETE",
                "start_time": "2024-01-20T22:00:00+00:00",
                "end_time": "2024-01-21T06:30:00+00:00",
                "sleep_stages": [0, 1, 2, 3, 2, 1, 0]
            },
            "stat": {
                "sleep_time": "06:30:00",
                "sleep_efficiency": 88.24,
                "time_in_bed": 30600,
                "time_in_sleep": 27000
            }
        }
    else:
        payload = {
            "event": "INFERENCE_COMPLETE",
            "version": "V3",
            "timestamp": "2024-01-21T06:15:00Z",
            "user_id": "test_user",
            "session_id": "test_session",
            "seq_num": 60,
            "inference_seq_num": 12,
            "sleep_stages": [1, 1, 2, 2, 2]
        }

    headers = {
        'x-api-key': 'your_api_key',
        'x-user-id': 'test_user',
        'Content-Type': 'application/json'
    }

    response = requests.post(url, headers=headers, json=payload)
    print(f"Status: {response.status_code}")
    print(f"Response: {response.json()}")

# Test locally
send_test_webhook('http://localhost:5000/asleep-webhook')
```

## Common Webhook Patterns

### Real-time Dashboard Updates

```python
def handle_inference_complete(event):
    """Update real-time dashboard with incremental data"""
    session_id = event['session_id']
    user_id = event['user_id']
    sleep_stages = event['sleep_stages']

    # Broadcast to connected clients via WebSocket
    socketio.emit('sleep_update', {
        'session_id': session_id,
        'sleep_stages': sleep_stages,
        'timestamp': event['timestamp']
    }, room=user_id)
```

### Sleep Report Notifications

```python
def handle_session_complete(event):
    """Send notification when sleep report is ready"""
    user_id = event['user_id']
    stat = event['stat']

    # Send push notification
    send_push_notification(user_id, {
        'title': 'Your Sleep Report is Ready',
        'body': f"You slept for {stat['sleep_time']} with {stat['sleep_efficiency']:.1f}% efficiency",
        'data': {
            'session_id': event['session_id'],
            'action': 'view_report'
        }
    })
```

### Data Aggregation

```python
def handle_session_complete(event):
    """Update aggregated user statistics"""
    user_id = event['user_id']
    stat = event['stat']

    # Update rolling averages
    db.user_stats.update_one(
        {'user_id': user_id},
        {
            '$inc': {
                'total_sessions': 1,
                'total_sleep_time': stat['time_in_sleep']
            },
            '$push': {
                'recent_efficiency': {
                    '$each': [stat['sleep_efficiency']],
                    '$slice': -30  # Keep last 30 sessions
                }
            }
        },
        upsert=True
    )
```
