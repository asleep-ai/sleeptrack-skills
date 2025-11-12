# Production Patterns and Best Practices

This reference provides comprehensive production-ready patterns for deploying and maintaining Asleep API integrations in production environments.

## Caching Strategies

### Session Caching

Sessions are immutable once complete, making them ideal for caching:

```python
from functools import lru_cache
from datetime import datetime, timedelta
import json
import redis

redis_client = redis.Redis(host='localhost', port=6379, db=0)

class CachedAsleepClient(AsleepClient):
    """Client with response caching"""

    @lru_cache(maxsize=128)
    def get_session_cached(self, session_id: str, user_id: str) -> Dict:
        """Get session with caching (sessions are immutable once complete)"""
        return self.get_session(session_id, user_id)

    def get_recent_sessions(self, user_id: str, days: int = 7) -> List[Dict]:
        """Get recent sessions with Redis caching"""
        cache_key = f"sessions:{user_id}:{days}"
        cached = redis_client.get(cache_key)

        if cached:
            return json.loads(cached)

        end_date = datetime.now()
        start_date = end_date - timedelta(days=days)

        result = self.list_sessions(
            user_id=user_id,
            date_gte=start_date.strftime("%Y-%m-%d"),
            date_lte=end_date.strftime("%Y-%m-%d")
        )

        # Cache for 5 minutes
        redis_client.setex(cache_key, 300, json.dumps(result))

        return result

    def invalidate_user_cache(self, user_id: str):
        """Invalidate all caches for a user"""
        pattern = f"sessions:{user_id}:*"
        for key in redis_client.scan_iter(match=pattern):
            redis_client.delete(key)
```

## Rate Limiting

### Application-Level Rate Limiting

Protect your backend from being overwhelmed:

```python
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(
    app,
    key_func=get_remote_address,
    default_limits=["100 per hour"]
)

@app.route('/api/sessions/<session_id>')
@limiter.limit("10 per minute")
def get_session(session_id):
    """Rate-limited session endpoint"""
    # Implementation
    pass

@app.route('/api/statistics')
@limiter.limit("5 per minute")
def get_statistics():
    """Statistics endpoint with stricter rate limiting"""
    # Implementation
    pass
```

### API Request Rate Limiting

Respect Asleep API rate limits with request throttling:

```python
import time
from collections import deque
from threading import Lock

class RateLimitedClient(AsleepClient):
    """Client with built-in rate limiting"""

    def __init__(self, api_key: str, requests_per_second: int = 10):
        super().__init__(api_key)
        self.requests_per_second = requests_per_second
        self.request_times = deque()
        self.lock = Lock()

    def _wait_for_rate_limit(self):
        """Wait if necessary to stay within rate limits"""
        with self.lock:
            now = time.time()

            # Remove requests older than 1 second
            while self.request_times and self.request_times[0] < now - 1:
                self.request_times.popleft()

            # If at limit, wait
            if len(self.request_times) >= self.requests_per_second:
                sleep_time = 1 - (now - self.request_times[0])
                if sleep_time > 0:
                    time.sleep(sleep_time)
                    self.request_times.popleft()

            self.request_times.append(time.time())

    def _request(self, method: str, path: str, **kwargs):
        """Rate-limited request"""
        self._wait_for_rate_limit()
        return super()._request(method, path, **kwargs)
```

## Connection Pooling

### HTTP Session with Connection Pool

Reuse connections for better performance:

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

def create_session_with_retries():
    """Create session with connection pooling and retries"""
    session = requests.Session()

    retry_strategy = Retry(
        total=3,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504],
        method_whitelist=["HEAD", "GET", "OPTIONS", "POST", "PUT", "DELETE"]
    )

    adapter = HTTPAdapter(
        max_retries=retry_strategy,
        pool_connections=10,
        pool_maxsize=20
    )

    session.mount("https://", adapter)
    session.mount("http://", adapter)

    return session

class PooledAsleepClient(AsleepClient):
    """Client with connection pooling"""

    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.asleep.ai"
        self.session = create_session_with_retries()
        self.session.headers.update({"x-api-key": api_key})
```

## Monitoring and Logging

### Structured Logging

```python
import logging
import json
from datetime import datetime

class StructuredLogger:
    """Structured logging for API requests"""

    def __init__(self, name: str):
        self.logger = logging.getLogger(name)

    def log_request(self, method: str, path: str, user_id: str = None):
        """Log API request"""
        self.logger.info(json.dumps({
            'event': 'api_request',
            'timestamp': datetime.now().isoformat(),
            'method': method,
            'path': path,
            'user_id': user_id
        }))

    def log_response(self, method: str, path: str, status_code: int, duration: float):
        """Log API response"""
        self.logger.info(json.dumps({
            'event': 'api_response',
            'timestamp': datetime.now().isoformat(),
            'method': method,
            'path': path,
            'status_code': status_code,
            'duration_ms': duration * 1000
        }))

    def log_error(self, method: str, path: str, error: Exception, duration: float):
        """Log API error"""
        self.logger.error(json.dumps({
            'event': 'api_error',
            'timestamp': datetime.now().isoformat(),
            'method': method,
            'path': path,
            'error_type': type(error).__name__,
            'error_message': str(error),
            'duration_ms': duration * 1000
        }))

class MonitoredAsleepClient(AsleepClient):
    """Client with comprehensive logging"""

    def __init__(self, api_key: str):
        super().__init__(api_key)
        self.logger = StructuredLogger(__name__)

    def _request(self, method: str, path: str, **kwargs):
        """Monitored API request"""
        start_time = datetime.now()
        user_id = kwargs.get('headers', {}).get('x-user-id')

        self.logger.log_request(method, path, user_id)

        try:
            result = super()._request(method, path, **kwargs)
            duration = (datetime.now() - start_time).total_seconds()
            self.logger.log_response(method, path, 200, duration)
            return result

        except Exception as e:
            duration = (datetime.now() - start_time).total_seconds()
            self.logger.log_error(method, path, e, duration)
            raise
```

### Metrics Collection

```python
from datadog import statsd

class MetricsClient(AsleepClient):
    """Client with metrics collection"""

    def _request(self, method: str, path: str, **kwargs):
        """Request with metrics"""
        start_time = datetime.now()

        try:
            result = super()._request(method, path, **kwargs)

            duration = (datetime.now() - start_time).total_seconds()

            # Record success metrics
            statsd.increment('asleep_api.request.success')
            statsd.timing('asleep_api.request.duration', duration)
            statsd.histogram('asleep_api.response_time', duration)

            return result

        except Exception as e:
            duration = (datetime.now() - start_time).total_seconds()

            # Record error metrics
            statsd.increment('asleep_api.request.error')
            statsd.increment(f'asleep_api.error.{type(e).__name__}')
            statsd.timing('asleep_api.request.duration', duration)

            raise
```

## Security Best Practices

### API Key Management

```python
import os
from dotenv import load_dotenv

class SecureConfig:
    """Secure configuration management"""

    def __init__(self):
        load_dotenv()
        self._validate_config()

    def _validate_config(self):
        """Validate required environment variables"""
        required = ['ASLEEP_API_KEY', 'DATABASE_URL']
        missing = [var for var in required if not os.getenv(var)]

        if missing:
            raise ValueError(f"Missing required environment variables: {', '.join(missing)}")

    @property
    def asleep_api_key(self) -> str:
        """Get API key from environment"""
        return os.getenv('ASLEEP_API_KEY')

    @property
    def database_url(self) -> str:
        """Get database URL from environment"""
        return os.getenv('DATABASE_URL')

    @property
    def redis_url(self) -> str:
        """Get Redis URL from environment"""
        return os.getenv('REDIS_URL', 'redis://localhost:6379')
```

### Webhook Security

```python
import hmac
import hashlib

def verify_webhook_signature(payload: bytes, signature: str, secret: str) -> bool:
    """Verify webhook payload signature"""
    expected_signature = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(signature, expected_signature)

@app.route('/asleep-webhook', methods=['POST'])
def secure_webhook():
    """Webhook endpoint with signature verification"""
    # Verify API key
    api_key = request.headers.get('x-api-key')
    if api_key != EXPECTED_API_KEY:
        return jsonify({"error": "Unauthorized"}), 401

    # Verify signature (if implemented)
    signature = request.headers.get('x-signature')
    if signature:
        if not verify_webhook_signature(request.data, signature, WEBHOOK_SECRET):
            return jsonify({"error": "Invalid signature"}), 401

    # Process webhook
    event = request.json
    process_webhook(event)

    return jsonify({"status": "success"}), 200
```

## Deployment Configuration

### Environment-Based Configuration

```python
import os
from enum import Enum

class Environment(Enum):
    DEVELOPMENT = "development"
    STAGING = "staging"
    PRODUCTION = "production"

class Config:
    """Environment-based configuration"""

    def __init__(self):
        self.env = Environment(os.getenv('ENVIRONMENT', 'development'))
        self.asleep_api_key = os.getenv('ASLEEP_API_KEY')
        self.asleep_base_url = os.getenv('ASLEEP_BASE_URL', 'https://api.asleep.ai')
        self.database_url = os.getenv('DATABASE_URL')
        self.redis_url = os.getenv('REDIS_URL')

        # Feature flags
        self.enable_caching = self._parse_bool('ENABLE_CACHING', True)
        self.enable_webhooks = self._parse_bool('ENABLE_WEBHOOKS', True)
        self.enable_metrics = self._parse_bool('ENABLE_METRICS', True)

        # Performance settings
        self.max_connections = int(os.getenv('MAX_CONNECTIONS', '100'))
        self.request_timeout = int(os.getenv('REQUEST_TIMEOUT', '30'))

        self._validate()

    def _parse_bool(self, key: str, default: bool) -> bool:
        """Parse boolean environment variable"""
        value = os.getenv(key, str(default)).lower()
        return value in ('true', '1', 'yes')

    def _validate(self):
        """Validate configuration"""
        if not self.asleep_api_key:
            raise ValueError("ASLEEP_API_KEY is required")

        if self.env == Environment.PRODUCTION:
            if not self.database_url:
                raise ValueError("DATABASE_URL is required in production")

    @property
    def is_production(self) -> bool:
        return self.env == Environment.PRODUCTION

    @property
    def is_development(self) -> bool:
        return self.env == Environment.DEVELOPMENT
```

### Health Check Endpoint

```python
from flask import Flask, jsonify
import requests

@app.route('/health')
def health_check():
    """Health check endpoint for load balancers"""
    checks = {
        'status': 'healthy',
        'timestamp': datetime.now().isoformat(),
        'environment': config.env.value,
        'checks': {}
    }

    # Check database connection
    try:
        db.command('ping')
        checks['checks']['database'] = 'ok'
    except Exception as e:
        checks['status'] = 'unhealthy'
        checks['checks']['database'] = f'error: {str(e)}'

    # Check Redis connection
    try:
        redis_client.ping()
        checks['checks']['redis'] = 'ok'
    except Exception as e:
        checks['status'] = 'unhealthy'
        checks['checks']['redis'] = f'error: {str(e)}'

    # Check Asleep API connectivity
    try:
        response = requests.get(
            f"{config.asleep_base_url}/health",
            headers={"x-api-key": config.asleep_api_key},
            timeout=5
        )
        if response.status_code == 200:
            checks['checks']['asleep_api'] = 'ok'
        else:
            checks['checks']['asleep_api'] = f'status: {response.status_code}'
    except Exception as e:
        checks['status'] = 'unhealthy'
        checks['checks']['asleep_api'] = f'error: {str(e)}'

    status_code = 200 if checks['status'] == 'healthy' else 503
    return jsonify(checks), status_code

@app.route('/ready')
def readiness_check():
    """Readiness check for Kubernetes"""
    # Check if app is ready to serve traffic
    if not app.initialized:
        return jsonify({'status': 'not ready'}), 503

    return jsonify({'status': 'ready'}), 200

@app.route('/live')
def liveness_check():
    """Liveness check for Kubernetes"""
    # Simple check that app is running
    return jsonify({'status': 'alive'}), 200
```

## Error Recovery

### Circuit Breaker Pattern

```python
from datetime import datetime, timedelta

class CircuitBreaker:
    """Circuit breaker for API calls"""

    def __init__(self, failure_threshold: int = 5, timeout: int = 60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = 'closed'  # closed, open, half_open

    def call(self, func, *args, **kwargs):
        """Execute function with circuit breaker"""
        if self.state == 'open':
            if datetime.now() - self.last_failure_time > timedelta(seconds=self.timeout):
                self.state = 'half_open'
            else:
                raise Exception("Circuit breaker is open")

        try:
            result = func(*args, **kwargs)

            if self.state == 'half_open':
                self.state = 'closed'
                self.failure_count = 0

            return result

        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = datetime.now()

            if self.failure_count >= self.failure_threshold:
                self.state = 'open'

            raise

class ResilientAsleepClient(AsleepClient):
    """Client with circuit breaker"""

    def __init__(self, api_key: str):
        super().__init__(api_key)
        self.circuit_breaker = CircuitBreaker()

    def _request(self, method: str, path: str, **kwargs):
        """Request with circuit breaker"""
        return self.circuit_breaker.call(
            super()._request,
            method,
            path,
            **kwargs
        )
```

## Database Patterns

### Session Storage

```python
from pymongo import MongoClient
from datetime import datetime

class SessionStore:
    """Store and retrieve sleep sessions"""

    def __init__(self, db):
        self.collection = db.sleep_sessions
        self._create_indexes()

    def _create_indexes(self):
        """Create database indexes for performance"""
        self.collection.create_index([('user_id', 1), ('session_start_time', -1)])
        self.collection.create_index([('session_id', 1)], unique=True)
        self.collection.create_index([('created_at', 1)])

    def store_session(self, session_data: Dict):
        """Store session in database"""
        doc = {
            'session_id': session_data['session']['id'],
            'user_id': session_data['user_id'],
            'session_start_time': session_data['session']['start_time'],
            'session_end_time': session_data['session']['end_time'],
            'statistics': session_data['stat'],
            'sleep_stages': session_data['session']['sleep_stages'],
            'created_at': datetime.now(),
            'updated_at': datetime.now()
        }

        self.collection.update_one(
            {'session_id': doc['session_id']},
            {'$set': doc},
            upsert=True
        )

    def get_user_sessions(self, user_id: str, limit: int = 10) -> List[Dict]:
        """Get recent sessions for user"""
        return list(
            self.collection
            .find({'user_id': user_id})
            .sort('session_start_time', -1)
            .limit(limit)
        )

    def get_sessions_by_date_range(
        self,
        user_id: str,
        start_date: str,
        end_date: str
    ) -> List[Dict]:
        """Get sessions within date range"""
        return list(
            self.collection.find({
                'user_id': user_id,
                'session_start_time': {
                    '$gte': start_date,
                    '$lte': end_date
                }
            })
            .sort('session_start_time', -1)
        )
```

## Background Job Processing

### Celery Task Queue

```python
from celery import Celery

celery = Celery('tasks', broker='redis://localhost:6379')

@celery.task(bind=True, max_retries=3)
def process_webhook_task(self, webhook_data: Dict):
    """Process webhook asynchronously"""
    try:
        if webhook_data['event'] == 'SESSION_COMPLETE':
            # Store in database
            store_session(webhook_data)

            # Send notification
            send_notification(webhook_data['user_id'], webhook_data)

            # Update analytics
            update_user_stats(webhook_data['user_id'])

    except Exception as e:
        # Retry with exponential backoff
        raise self.retry(exc=e, countdown=2 ** self.request.retries)

@app.route('/webhook', methods=['POST'])
def webhook():
    """Webhook endpoint with async processing"""
    event = request.json

    # Queue for background processing
    process_webhook_task.delay(event)

    # Respond immediately
    return jsonify({"status": "queued"}), 200
```

## Performance Optimization

### Batch Processing

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def fetch_sessions_batch(client: AsleepClient, user_ids: List[str]) -> Dict[str, List]:
    """Fetch sessions for multiple users in parallel"""
    results = {}

    with ThreadPoolExecutor(max_workers=10) as executor:
        future_to_user = {
            executor.submit(client.list_sessions, user_id): user_id
            for user_id in user_ids
        }

        for future in as_completed(future_to_user):
            user_id = future_to_user[future]
            try:
                results[user_id] = future.result()
            except Exception as e:
                print(f"Error fetching sessions for {user_id}: {e}")
                results[user_id] = []

    return results
```

### Query Optimization

```python
def get_user_summary_optimized(client: AsleepClient, user_id: str) -> Dict:
    """Get user summary with optimized queries"""
    # Fetch only what's needed
    user_data = client.get_user(user_id)

    # Use average stats instead of fetching all sessions
    stats = client.get_average_stats(
        user_id=user_id,
        start_date=(datetime.now() - timedelta(days=30)).strftime("%Y-%m-%d"),
        end_date=datetime.now().strftime("%Y-%m-%d")
    )

    return {
        'user_id': user_id,
        'last_session': user_data.get('last_session_info'),
        'monthly_average': stats['average_stats'],
        'session_count': len(stats['slept_sessions'])
    }
```
