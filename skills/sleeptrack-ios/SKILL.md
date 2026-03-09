---
name: sleeptrack-ios
description: Asleep iOS SDK integration reference. Use when building iOS apps with sleep tracking -- covers SDK setup, API surface, permissions, background audio, delegate protocols, error handling, and Siri Shortcuts. Self-contained; does not require sleeptrack-foundation.
---

# Asleep iOS SDK Reference

## Overview

The Asleep iOS SDK wraps microphone-based audio capture, upload, and session management behind a delegate pattern. The SDK records ambient audio during sleep, uploads 30-second chunks to Asleep servers for analysis, and returns sleep stage classifications. All tracking state is communicated through delegate callbacks.

## Authentication

- **API key**: Passed as `x-api-key` header. Obtain from the Asleep Dashboard Settings tab.
- **Base URL**: `https://api.asleep.ai`

## Installation

Swift Package Manager. In Xcode: File > Add Packages, enter the URL. Or add to `Package.swift`:

```swift
dependencies: [
    .package(url: "https://github.com/asleep-ai/asleep-sdk-ios", from: "2.0.0")
]
```

Import: `import AsleepSDK`

## Info.plist Requirements

| Key | Value | Required |
|-----|-------|----------|
| `NSMicrophoneUsageDescription` | Usage string explaining audio-based sleep tracking | Yes |
| `UIBackgroundModes` | `audio` | Yes (for background tracking) |
| `NSUserNotificationsUsageDescription` | Usage string for tracking reminders | No |

## SDK API Surface

### AsleepConfig -- Configuration and User Management

Initialize the SDK and manage user lifecycle.

```
Asleep.initAsleepConfig(apiKey: String, userId: String, delegate: AsleepConfigDelegate)
```

**AsleepConfigDelegate** protocol:

| Method | Called when |
|--------|------------|
| `userDidJoin(userId: String, config: Asleep.Config)` | SDK initialized successfully |
| `didFailUserJoin(error: Asleep.AsleepError)` | Initialization failed |
| `userDidDelete(userId: String)` | User account deleted |

The `Asleep.Config` object returned by `userDidJoin` is required to create the tracking manager and reports instance.

### SleepTrackingManager -- Tracking Lifecycle

Create and control a tracking session.

```
Asleep.createSleepTrackingManager(config: Asleep.Config, delegate: AsleepSleepTrackingManagerDelegate) -> Asleep.SleepTrackingManager
```

**Instance methods**: `startTracking()`, `stopTracking()`

**AsleepSleepTrackingManagerDelegate** protocol:

| Method | Called when |
|--------|------------|
| `didCreate()` | Session created, tracking has started |
| `didUpload(sequence: Int)` | Audio chunk uploaded (~every 30 seconds) |
| `didClose(sessionId: String)` | Tracking stopped, session finalized |
| `didFail(error: Asleep.AsleepError)` | Error during tracking |
| `didInterrupt()` | Tracking interrupted (phone call, other audio) |
| `didResume()` | Tracking resumed after interruption |
| `micPermissionWasDenied()` | Microphone permission not granted |
| `analysing(session: Asleep.Model.Session)` | Preliminary real-time sleep data available |

### Reports -- Retrieving Sleep Data

Async/await API (not delegate-driven).

```
Asleep.createReports(config: Asleep.Config) -> Asleep.Reports
```

| Method | Returns |
|--------|---------|
| `report(sessionId: String) async throws` | Single session report |
| `reports(fromDate: String, toDate: String) async throws` | List of sessions in date range |

Date format: `"YYYY-MM-DD"`.

## Microphone Permission

Check `AVAudioSession.sharedInstance().recordPermission` before tracking.

| State | Action |
|-------|--------|
| `.granted` | Proceed with tracking |
| `.denied` | Direct user to Settings to enable |
| `.undetermined` | Call `requestRecordPermission()` to prompt |

## Background Audio

Adding `audio` to `UIBackgroundModes` in Info.plist is sufficient. The SDK handles background audio session configuration. Tracking continues automatically when the app moves to the background.

Do not stop tracking when the app backgrounds -- this is the normal sleep tracking flow.

## Real-time Data

Preliminary sleep stage data becomes available after sequence 10 and updates every 10 sequences. The SDK calls the `analysing(session:)` delegate automatically when new data is ready. The `Asleep.Model.Session` object contains `sleepStages` with preliminary classifications.

This data is preliminary and may differ from the final report.

## Error Handling

**Asleep.AsleepError** cases:

| Case | Description | Severity |
|------|-------------|----------|
| `micPermission` | Microphone access denied | Critical -- cannot track |
| `audioSessionError` | Audio session unavailable (another app using mic) | Critical -- cannot track |
| `httpStatus(code: Int, url: String, message: String)` | Server error | Transient -- retry |

Notable HTTP status codes:
- **403**: Session already active on another device
- **404**: Session not found

**Critical vs transient**: Microphone and audio session errors require user intervention (stop tracking, show guidance). Network/HTTP errors are transient and can be retried.

## Lifecycle Constraints

- Tracking continues in background via background audio mode. Do not stop tracking on app backgrounding.
- Handle `scenePhase` changes (active / inactive / background) for UI updates, but do not tie tracking stop/start to them.
- Phone call and other audio interruptions are handled by `didInterrupt()` and `didResume()` delegates. The SDK manages the audio session automatically.
- Memory pressure from iOS can terminate background audio. Ensure `UIBackgroundModes` is configured and test on real devices.

## Siri Shortcuts

Supported via App Intents framework (iOS 16+). Define `StartSleepIntent` and `StopSleepIntent` conforming to the `AppIntent` protocol. This allows users to say "Hey Siri, start sleep tracking" or "Hey Siri, stop sleep tracking".

Each intent needs a `title`, `description`, and `perform() async throws -> some IntentResult` method that calls the appropriate SDK method.

## Persistent Storage

Use `@AppStorage` to persist API key and user ID across app launches:

```
@AppStorage("sleepapp+apikey") var apiKey: String
@AppStorage("sleepapp+userid") var userId: String
```

This avoids re-authentication on every launch. The userId from `userDidJoin` should be saved for subsequent sessions.

## Common Issues

| Problem | Check |
|---------|-------|
| Tracking does not start | Microphone permission granted? API key non-empty? User ID valid? Another app using mic? |
| Background tracking stops | `UIBackgroundModes` includes `audio`? Device under memory pressure? App was force-closed by user? |
| Reports not available | Session still processing (allow time). Session duration at least 5 minutes? Network connectivity? |

## Links

- **SDK repository**: https://github.com/asleep-ai/asleep-sdk-ios
- **Sample app**: https://github.com/asleep-ai/asleep-sdk-ios-sampleapp-public
- **iOS Get Started**: https://docs-en.asleep.ai/docs/ios-get-started.md
- **iOS Error Codes**: https://docs-en.asleep.ai/docs/ios-error-codes.md
- **AsleepConfig Reference**: https://docs-en.asleep.ai/docs/ios-asleep-config.md
- **SleepTrackingManager Reference**: https://docs-en.asleep.ai/docs/ios-sleep-tracking-manager.md
- **AVAudioSession (Apple)**: https://developer.apple.com/documentation/avfaudio/avaudiosession
- **App Intents (Apple)**: https://developer.apple.com/documentation/appintents
- **Background Modes (Apple)**: https://developer.apple.com/documentation/xcode/configuring-background-execution-modes
