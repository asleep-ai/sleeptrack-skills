# Asleep API Reference

## Official Documentation

- **Main Documentation**: https://docs-en.asleep.ai
- **LLM-Optimized Docs**: https://docs-en.asleep.ai/llms.txt
- **Dashboard**: https://dashboard.asleep.ai

## API Authentication

All Asleep API requests require authentication using an API key.

### Getting an API Key

1. Sign up for the Asleep Dashboard at https://dashboard.asleep.ai
2. Navigate to the API key generation section
3. Generate your API key for your application
4. Store the API key securely (never commit to version control)

### Authentication Method

Include the API key in request headers:
```
X-API-Key: your_api_key_here
```

## Core Concepts

### User Management

Each application user must be registered with Asleep before tracking sleep data.

- **User ID**: Unique identifier for each user (managed by your application)
- User data is associated with sessions and reports
- Users can have multiple sleep tracking sessions

### Sleep Sessions

A sleep session represents one tracking period from start to stop.

**Session States**:
- IDLE: Not tracking
- INITIALIZING: Preparing to track
- INITIALIZED: Ready to track
- TRACKING_STARTED: Active tracking in progress
- TRACKING_STOPPING: Ending tracking session

**Key Session Characteristics**:
- Minimum tracking time: 5 minutes for valid session
- Real-time data available after sequence 10, then every 10 sequences
- Each session generates a comprehensive sleep report

### Sleep Reports

Reports contain detailed analysis of a completed sleep session.

**Core Metrics**:
- Sleep stages: Wake, Light, Deep, REM
- Sleep efficiency: Percentage of time actually sleeping
- Sleep latency: Time to fall asleep
- Total sleep time
- Time in bed
- Wake after sleep onset (WASO)
- Sleep stage ratios and durations
- Snoring detection and analysis

### Statistics

Aggregated sleep metrics across multiple sessions.

**Available Statistics**:
- Average sleep duration
- Average sleep efficiency
- Sleep stage distribution
- Trends over time periods

## Error Codes

### Critical Errors (Stop Tracking)

- **ERR_MIC_PERMISSION**: Microphone access permission denied
- **ERR_AUDIO**: Microphone in use by another app or hardware issue
- **ERR_INVALID_URL**: Malformed API endpoint URL
- **ERR_COMMON_EXPIRED**: API rate limit exceeded or plan expired
- **ERR_UPLOAD_FORBIDDEN**: Multiple simultaneous tracking attempts with same user ID
- **ERR_UPLOAD_NOT_FOUND**: Session does not exist or already ended
- **ERR_CLOSE_NOT_FOUND**: Attempted to close non-existent session

### Warning Errors (Continue Tracking)

- **ERR_AUDIO_SILENCED**: Audio temporarily unavailable but tracking continues
- **ERR_AUDIO_UNSILENCED**: Audio restored after silence
- **ERR_UPLOAD_FAILED**: Network issue during upload, will retry

## REST API Endpoints

### User Management

**Create User**
```
POST /users
Body: { "user_id": "string" }
```

**Get User**
```
GET /users/{user_id}
```

**Update User**
```
PUT /users/{user_id}
Body: { /* user properties */ }
```

**Delete User**
```
DELETE /users/{user_id}
```

### Session Management

**Get Session**
```
GET /sessions/{session_id}
```

**List Sessions**
```
GET /sessions?user_id={user_id}&from={date}&to={date}
```

**Delete Session**
```
DELETE /sessions/{session_id}
```

### Statistics

**Get Average Stats**
```
GET /users/{user_id}/statistics/average?from={date}&to={date}
```

## Webhooks

Asleep supports webhooks for real-time event notifications.

**Supported Events**:
- Session started
- Session completed
- Report generated
- User created/updated/deleted

**Webhook Configuration**:
Configure webhook URLs in the Asleep Dashboard.

## Platform SDKs

### Android SDK

- **Language**: Kotlin
- **Minimum SDK**: Check latest documentation
- **Key Classes**: AsleepConfig, SleepTrackingManager
- **Architecture**: MVVM patterns, Hilt dependency injection
- **Permissions Required**:
  - RECORD_AUDIO (microphone)
  - POST_NOTIFICATIONS
  - REQUEST_IGNORE_BATTERY_OPTIMIZATIONS
  - FOREGROUND_SERVICE

### iOS SDK

- **Language**: Swift
- **Minimum iOS Version**: Check latest documentation
- **Key Classes**: AsleepConfig, SleepTrackingManager
- **Architecture**: Delegate patterns, Combine framework
- **Permissions Required**:
  - Microphone access (NSMicrophoneUsageDescription)
  - Notifications
  - Background modes

## Data Structures

### Session

Represents a sleep tracking session with metadata and tracking state information.

### Report

Comprehensive sleep analysis including:
- Sleep stages timeline
- Statistical metrics
- Snoring analysis
- Quality indicators

### Statistics

Aggregated metrics across multiple sessions for trend analysis.

## Best Practices

### API Key Security

- Never hardcode API keys in source code
- Use environment variables or secure storage
- Rotate keys periodically
- Monitor usage in Dashboard

### User ID Management

- Use consistent user IDs across sessions
- Consider user privacy in ID scheme
- Implement proper user consent flows

### Error Handling

- Distinguish between critical errors (stop tracking) and warnings (continue)
- Provide user-friendly error messages
- Implement retry logic for network failures
- Log errors for debugging

### Session Management

- Validate minimum tracking time (5 minutes)
- Handle app lifecycle properly (don't lose sessions)
- Implement reconnection logic for interrupted sessions
- Clean up resources when stopping tracking

### Performance

- Check real-time data appropriately (after sequence 10, every 10 sequences)
- Cache reports when appropriate
- Batch API requests when possible
- Monitor API rate limits

## Common Use Cases

### Health & Fitness Apps

Integrate sleep tracking alongside activity tracking for comprehensive health insights.

### Healthcare & Wellness

Clinical-grade sleep monitoring for patient care and wellness programs.

### Sleep Tech

Dedicated sleep improvement applications with detailed analysis and recommendations.

### Smart Home & IoT

Integrate sleep data with smart home automation for optimized sleep environment.

## Support & Resources

- **Documentation**: https://docs-en.asleep.ai
- **Dashboard**: https://dashboard.asleep.ai
- **Email**: Contact through dashboard for technical support
