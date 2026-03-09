---
name: sleeptrack-android
description: Asleep Android SDK integration reference. Use when building Android apps with sleep tracking -- covers SDK setup, API surface, permissions, foreground service, error codes, and lifecycle constraints. Self-contained; does not require sleeptrack-foundation.
---

# Asleep Android SDK Reference

## Overview

The Asleep Android SDK wraps audio capture, upload, and session management into a single library. It records ambient sound via the device microphone, streams audio to Asleep servers for analysis, and returns sleep stage classifications in real time and as a final report. The SDK manages its own foreground service to keep tracking alive in the background.

## Requirements

| Requirement | Value |
|---|---|
| minSdk | 24 (Android 7.0) |
| targetSdk | 34 |
| Java | 17 |
| Kotlin | 1.9.24+ |

## Dependencies

```gradle
dependencies {
    implementation 'com.squareup.okhttp3:okhttp:4.11.0'
    implementation 'com.google.code.gson:gson:2.10'
    implementation 'ai.asleep:asleepsdk:3.1.4'
}
```

## Manifest Permissions

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MICROPHONE" />
<uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS" />
<!-- Android 13+ -->
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
```

## SDK API Surface

### Initialization

```
Asleep.initAsleepConfig(
    context: Context,
    apiKey: String,
    userId: String?,
    baseUrl: String?,
    callbackUrl: String?,
    service: String?,
    asleepConfigListener: AsleepConfigListener
)
```

Initializes the SDK configuration. Pass `null` for `userId` to have the server generate one (returned in `onSuccess`). The `callbackUrl` is an optional webhook endpoint. The `service` string identifies your app in analytics.

**AsleepConfigListener**

- `onSuccess(userId: String?, asleepConfig: AsleepConfig?)` -- SDK ready. Store the `asleepConfig` for tracking calls and persist `userId` for future sessions.
- `onFail(errorCode: Int, detail: String)` -- Initialization failed. Check API key validity and network connectivity.

### Start Tracking

```
Asleep.beginSleepTracking(
    asleepConfig: AsleepConfig,
    asleepTrackingListener: AsleepTrackingListener,
    notificationTitle: String,
    notificationText: String,
    notificationIcon: Int,
    notificationClass: Class<*>
)
```

Starts audio capture and opens a sleep session. Automatically creates a foreground service with the provided notification parameters. `notificationClass` is the Activity opened when the user taps the notification.

**AsleepTrackingListener**

- `onStart(sessionId: String)` -- Tracking session opened. Store `sessionId` if needed.
- `onPerform(sequence: Int)` -- Called approximately every 30 seconds. Each call represents one sequence (one audio chunk uploaded).
- `onFinish(sessionId: String?)` -- Tracking stopped and session closed. Final report available via API.
- `onFail(errorCode: Int, detail: String)` -- Error during tracking. See Error Codes below.

### Stop Tracking

```
Asleep.endSleepTracking()
```

Stops audio capture, closes the session, and tears down the foreground service.

### Check Tracking Status

```
Asleep.isSleepTrackingAlive(context: Context): Boolean
```

Returns `true` if the tracking foreground service is currently running. Use this on app resume to reconnect UI to an active session.

### Get Real-Time Data

```
Asleep.getCurrentSleepData(
    asleepSleepDataListener: AsleepSleepDataListener
)
```

Fetches preliminary sleep stage data for the current session. Only available after sequence 10; call every 10 sequences for updated results.

**AsleepSleepDataListener**

- `onSleepDataReceived(session: Session)` -- Contains `sleepStages` and `snoringStages` arrays with classifications up to the current point.
- `onFail(errorCode: Int, detail: String)` -- Data fetch failed.

## Tracking States

The SDK progresses through these states during its lifecycle:

| State | Description |
|---|---|
| `STATE_IDLE` | No configuration loaded |
| `STATE_INITIALIZING` | `initAsleepConfig` called, waiting for response |
| `STATE_INITIALIZED` | Config ready, can start tracking |
| `STATE_TRACKING_STARTING` | `beginSleepTracking` called, service launching |
| `STATE_TRACKING_STARTED` | Audio capture active, sequences incrementing |
| `STATE_TRACKING_STOPPING` | `endSleepTracking` called, finalizing session |
| `STATE_ERROR` | Error occurred, check error code |

## Foreground Service

The SDK automatically manages a foreground service during tracking. Key behaviors:

- The service starts when `beginSleepTracking` is called and stops when `endSleepTracking` completes.
- The notification parameters (`notificationTitle`, `notificationText`, `notificationIcon`, `notificationClass`) control what the user sees in the notification shade.
- The service survives the user swiping the app away from recents.
- Do NOT call `endSleepTracking()` in `onDestroy()`. The service is designed to outlive the Activity.

## Real-Time Data

Preliminary sleep data becomes available after sequence 10. Best practice is to check every 10 sequences (i.e., when `sequence > 10 && sequence % 10 == 0` in the `onPerform` callback). Data includes sleep stage and snoring classifications up to the current point. This data is preliminary and may differ from the final processed report.

## Error Codes

### Critical Errors (stop tracking)

| Code | Description |
|---|---|
| `ERR_MIC_PERMISSION` | Microphone permission revoked or unavailable |
| `ERR_AUDIO` | Audio capture failed (hardware or OS conflict) |
| `ERR_INVALID_URL` | Base URL or callback URL malformed |
| `ERR_COMMON_EXPIRED` | API key or session expired |
| `ERR_UPLOAD_FORBIDDEN` | Upload rejected (e.g., same user_id active on another device) |
| `ERR_UPLOAD_NOT_FOUND` | Session not found on server |
| `ERR_CLOSE_NOT_FOUND` | Attempted to close a non-existent session |

### Warning Errors (continue tracking)

| Code | Description |
|---|---|
| `ERR_AUDIO_SILENCED` | Microphone receiving silence (blocked or muted) |
| `ERR_AUDIO_UNSILENCED` | Microphone resumed receiving audio after silence |
| `ERR_UPLOAD_FAILED` | Temporary upload failure (will retry automatically) |

## Lifecycle Constraints

- **onStart**: If `isSleepTrackingAlive(context)` returns true, reconnect your UI to the active tracking session (re-register listeners).
- **onStop / onDestroy**: Do not stop tracking. The foreground service continues independently.
- **onResume**: Re-check runtime permissions. If a required permission was revoked while tracking was active, handle gracefully.
- **Battery optimization**: Must be disabled for the app to ensure reliable background tracking. Without this, the OS may kill the foreground service during extended sleep sessions.

## Permission Request Order

Request permissions sequentially, not in parallel:

1. **Microphone** (RECORD_AUDIO) -- required before tracking can start
2. **Notification** (POST_NOTIFICATIONS) -- Android 13+ only, needed for foreground service visibility
3. **Battery optimization** (REQUEST_IGNORE_BATTERY_OPTIMIZATIONS) -- ensures background reliability

Show rationale before each request. Handle denial by guiding the user to app settings.

## Common Issues

- **Tracking stops unexpectedly**: Check battery optimization is disabled, notification permission is granted, microphone permission is still held, and no other app has claimed exclusive mic access.
- **No real-time data**: Verify sequence count exceeds 10 before calling `getCurrentSleepData`.
- **ERR_UPLOAD_FORBIDDEN**: The same `user_id` is actively tracking on another device. Use unique user IDs per device or check for active sessions before starting.
- **Service killed on some OEMs**: Some manufacturers (Xiaomi, Huawei, Samsung) aggressively kill background services. Guide users to vendor-specific battery settings (see https://dontkillmyapp.com).

## Authentication

- Obtain an API key from the Asleep Dashboard under the Settings tab.
- Pass the key as the `apiKey` parameter to `initAsleepConfig`.
- Base URL: `https://api.asleep.ai`

## Links

- Android Getting Started: https://docs-en.asleep.ai/docs/android-get-started.md
- AsleepConfig Reference: https://docs-en.asleep.ai/docs/android-asleep-config.md
- SleepTrackingManager: https://docs-en.asleep.ai/docs/android-sleep-tracking-manager.md
- Error Codes: https://docs-en.asleep.ai/docs/android-error-codes.md
- Android Permissions Guide: https://developer.android.com/guide/topics/permissions/overview
- Foreground Services: https://developer.android.com/develop/background-work/services/foreground-services
