# Node.js Client Implementation Guide

This reference provides complete Node.js client implementations for the Asleep API, including webhook servers and production patterns.

## Complete Node.js API Client

```javascript
const axios = require('axios');

class AsleepClient {
  constructor(apiKey, baseURL = 'https://api.asleep.ai') {
    this.apiKey = apiKey;
    this.baseURL = baseURL;
    this.client = axios.create({
      baseURL: baseURL,
      headers: { 'x-api-key': apiKey }
    });
  }

  async _request(method, path, options = {}) {
    try {
      const response = await this.client.request({
        method,
        url: path,
        ...options
      });
      return response.data;
    } catch (error) {
      if (error.response) {
        const status = error.response.status;
        const detail = error.response.data?.detail || 'Unknown error';

        if (status === 401) {
          throw new Error('Invalid API key');
        } else if (status === 403) {
          throw new Error(`API access error: ${detail}`);
        } else if (status === 404) {
          throw new Error('Resource not found');
        }
      }
      throw error;
    }
  }

  // User management
  async createUser(metadata = null) {
    const data = metadata ? { metadata } : {};
    const result = await this._request('POST', '/ai/v1/users', { data });
    return result.result.user_id;
  }

  async getUser(userId) {
    const result = await this._request('GET', `/ai/v1/users/${userId}`);
    return result.result;
  }

  async deleteUser(userId) {
    await this._request('DELETE', `/ai/v1/users/${userId}`);
  }

  // Session management
  async getSession(sessionId, userId, timezone = 'UTC') {
    const result = await this._request('GET', `/data/v3/sessions/${sessionId}`, {
      headers: { 'x-user-id': userId, 'timezone': timezone }
    });
    return result.result;
  }

  async listSessions(userId, options = {}) {
    const { dateGte, dateLte, offset = 0, limit = 20, orderBy = 'DESC' } = options;
    const params = { offset, limit, order_by: orderBy };
    if (dateGte) params.date_gte = dateGte;
    if (dateLte) params.date_lte = dateLte;

    const result = await this._request('GET', '/data/v1/sessions', {
      headers: { 'x-user-id': userId },
      params
    });
    return result.result;
  }

  async deleteSession(sessionId, userId) {
    await this._request('DELETE', `/ai/v1/sessions/${sessionId}`, {
      headers: { 'x-user-id': userId }
    });
  }

  // Statistics
  async getAverageStats(userId, startDate, endDate, timezone = 'UTC') {
    const result = await this._request('GET', `/data/v1/users/${userId}/average-stats`, {
      headers: { 'timezone': timezone },
      params: { start_date: startDate, end_date: endDate }
    });
    return result.result;
  }
}

// Usage
const client = new AsleepClient(process.env.ASLEEP_API_KEY);
```

## Express Webhook Server

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

## Retry with Exponential Backoff

```javascript
async function retryWithExponentialBackoff(
  func,
  maxRetries = 3,
  baseDelay = 1000,
  maxDelay = 60000
) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await func();
    } catch (error) {
      if (error.response?.status === 403) {
        const detail = error.response.data?.detail || '';
        if (detail.toLowerCase().includes('rate limit')) {
          if (attempt < maxRetries - 1) {
            const delay = Math.min(baseDelay * Math.pow(2, attempt), maxDelay);
            console.log(`Rate limited, retrying in ${delay}ms...`);
            await new Promise(resolve => setTimeout(resolve, delay));
            continue;
          }
        }
      }
      throw error;
    }
  }
}

// Usage
const result = await retryWithExponentialBackoff(
  () => client.getSession('session123', 'user123')
);
```

## Basic Usage Examples

### Creating Users

```javascript
// Create user with metadata
const userId = await client.createUser({
  birth_year: 1990,
  gender: 'male',
  height: 175.5,
  weight: 70.0
});
console.log(`Created user: ${userId}`);

// Create user without metadata
const userId = await client.createUser();
```

### Getting Sessions

```javascript
// Get sessions for date range
const sessions = await client.listSessions('user123', {
  dateGte: '2024-01-01',
  dateLte: '2024-01-31',
  limit: 50,
  orderBy: 'DESC'
});

console.log(`Found ${sessions.sleep_session_list.length} sessions`);

sessions.sleep_session_list.forEach(session => {
  console.log(`Session ${session.session_id}: ${session.session_start_time}`);
  console.log(`  State: ${session.state}, Time in bed: ${session.time_in_bed}s`);
});
```

### Getting Session Details

```javascript
const session = await client.getSession(
  'session123',
  'user123',
  'America/New_York'
);

console.log(`Sleep efficiency: ${session.stat.sleep_efficiency.toFixed(1)}%`);
console.log(`Total sleep time: ${session.stat.sleep_time}`);
console.log(`Sleep stages: ${session.session.sleep_stages}`);
console.log(`Sleep cycles: ${session.stat.sleep_cycle.length}`);
```

### Getting Statistics

```javascript
const stats = await client.getAverageStats(
  'user123',
  '2024-01-01',
  '2024-01-31',
  'UTC'
);

const avg = stats.average_stats;
console.log(`Average sleep time: ${avg.sleep_time}`);
console.log(`Average efficiency: ${avg.sleep_efficiency.toFixed(1)}%`);
console.log(`Average bedtime: ${avg.start_time}`);
console.log(`Average wake time: ${avg.end_time}`);
console.log(`Light sleep ratio: ${(avg.light_ratio * 100).toFixed(1)}%`);
console.log(`Deep sleep ratio: ${(avg.deep_ratio * 100).toFixed(1)}%`);
console.log(`REM sleep ratio: ${(avg.rem_ratio * 100).toFixed(1)}%`);
console.log(`Number of sessions: ${stats.slept_sessions.length}`);
```

## Asynchronous Webhook Processing

```javascript
const Queue = require('bull');

const webhookQueue = new Queue('asleep-webhooks', {
  redis: {
    host: 'localhost',
    port: 6379
  }
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
  } else if (event.event === 'INFERENCE_COMPLETE') {
    await handleInferenceComplete(event);
  }
});
```

## Idempotency Pattern

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

## Comprehensive Error Handling

```javascript
class AsleepAPIError extends Error {
  constructor(message, statusCode, detail) {
    super(message);
    this.name = 'AsleepAPIError';
    this.statusCode = statusCode;
    this.detail = detail;
  }
}

class RateLimitError extends AsleepAPIError {
  constructor(detail) {
    super('Rate limit exceeded', 403, detail);
    this.name = 'RateLimitError';
  }
}

class ResourceNotFoundError extends AsleepAPIError {
  constructor(detail) {
    super('Resource not found', 404, detail);
    this.name = 'ResourceNotFoundError';
  }
}

async function safeApiRequest(requestFunc) {
  try {
    return await requestFunc();
  } catch (error) {
    if (error.response) {
      const status = error.response.status;
      const detail = error.response.data?.detail || 'Unknown error';

      if (status === 401) {
        throw new AsleepAPIError('Authentication failed', 401, detail);
      } else if (status === 403) {
        if (detail.toLowerCase().includes('rate limit')) {
          throw new RateLimitError(detail);
        } else {
          throw new AsleepAPIError('Access forbidden', 403, detail);
        }
      } else if (status === 404) {
        throw new ResourceNotFoundError(detail);
      } else {
        throw new AsleepAPIError(`API error (${status})`, status, detail);
      }
    }
    throw error;
  }
}

// Usage
try {
  const user = await safeApiRequest(() => client.getUser('user123'));
} catch (error) {
  if (error instanceof ResourceNotFoundError) {
    console.log('User not found, creating new user...');
    const userId = await client.createUser();
  } else if (error instanceof RateLimitError) {
    console.log('Rate limited, try again later');
  } else if (error instanceof AsleepAPIError) {
    console.error(`API error: ${error.message}`);
  }
}
```

## Production Configuration

```javascript
// config.js
require('dotenv').config();

class Config {
  static get ASLEEP_API_KEY() {
    return process.env.ASLEEP_API_KEY;
  }

  static get ASLEEP_BASE_URL() {
    return process.env.ASLEEP_BASE_URL || 'https://api.asleep.ai';
  }

  static get DATABASE_URL() {
    return process.env.DATABASE_URL;
  }

  static get REDIS_URL() {
    return process.env.REDIS_URL;
  }

  static get WEBHOOK_SECRET() {
    return process.env.WEBHOOK_SECRET;
  }

  static get ENABLE_CACHING() {
    return process.env.ENABLE_CACHING !== 'false';
  }

  static validate() {
    if (!this.ASLEEP_API_KEY) {
      throw new Error('ASLEEP_API_KEY environment variable required');
    }
    if (!this.DATABASE_URL) {
      throw new Error('DATABASE_URL environment variable required');
    }
  }
}

module.exports = Config;
```
