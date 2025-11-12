# iOS-Specific Features

Detailed implementation guides for iOS platform features including Siri Shortcuts, background modes, permissions, lifecycle management, and persistent storage.

## Siri Shortcuts Integration

Enable voice-activated tracking with App Intents (iOS 16+):

```swift
// StartSleepIntent.swift
import AppIntents

@available(iOS 16, *)
struct StartSleepIntent: AppIntent {
    static var title: LocalizedStringResource = "Start Sleep"
    static var description = IntentDescription("Start Sleep Tracking")

    func perform() async throws -> some IntentResult {
        NotificationCenter.default.post(name: .startSleep, object: nil)
        return .result()
    }
}

// StopSleepIntent.swift
@available(iOS 16, *)
struct StopSleepIntent: AppIntent {
    static var title: LocalizedStringResource = "Stop Sleep"
    static var description = IntentDescription("Stop Sleep Tracking")

    func perform() async throws -> some IntentResult {
        NotificationCenter.default.post(name: .stopSleep, object: nil)
        return .result()
    }
}

// Notification extensions
extension Notification.Name {
    static let startSleep = Notification.Name("startSleep")
    static let stopSleep = Notification.Name("stopSleep")
}
```

### Handling Shortcuts in Views

```swift
struct SleepTrackingView: View {
    @StateObject private var viewModel = SleepTrackingViewModel()

    var body: some View {
        // ... view content ...
        .onReceive(NotificationCenter.default.publisher(for: .startSleep)) { _ in
            if !viewModel.isTracking {
                startTracking()
            }
        }
        .onReceive(NotificationCenter.default.publisher(for: .stopSleep)) { _ in
            if viewModel.isTracking {
                stopTracking()
            }
        }
    }
}
```

Users can then say: "Hey Siri, start sleep" or "Hey Siri, stop sleep"

## Background Audio Mode

Configure background audio to maintain tracking during sleep.

### Info.plist Configuration

```xml
<key>UIBackgroundModes</key>
<array>
    <string>audio</string>
</array>
```

### Implementation Notes

- iOS automatically maintains background audio session during tracking
- App remains active in background while microphone is in use
- User sees audio indicator (red bar/pill) showing active recording
- No additional code needed beyond Info.plist configuration
- System handles audio session management automatically

### Best Practices

1. Inform users why the app needs background audio mode
2. Display clear status indicators when tracking is active
3. Handle audio interruptions gracefully (phone calls, other apps)
4. Test background behavior thoroughly on physical devices

## Microphone Permission Handling

Request and handle microphone permission properly:

```swift
import AVFoundation

func requestMicrophonePermission() async -> Bool {
    switch AVAudioSession.sharedInstance().recordPermission {
    case .granted:
        return true
    case .denied:
        return false
    case .undetermined:
        return await AVAudioSession.sharedInstance().requestRecordPermission()
    @unknown default:
        return false
    }
}

// Usage in SwiftUI
Button("Start Tracking") {
    Task {
        let hasPermission = await requestMicrophonePermission()
        if hasPermission {
            startTracking()
        } else {
            showPermissionAlert = true
        }
    }
}
```

### Permission Alert Handling

```swift
struct PermissionAlert: ViewModifier {
    @Binding var showAlert: Bool

    func body(content: Content) -> some View {
        content
            .alert("Microphone Access Required", isPresented: $showAlert) {
                Button("Open Settings") {
                    if let url = URL(string: UIApplication.openSettingsURLString) {
                        UIApplication.shared.open(url)
                    }
                }
                Button("Cancel", role: .cancel) {}
            } message: {
                Text("Please enable microphone access in Settings to track sleep.")
            }
    }
}
```

### Checking Permission Status

```swift
func checkMicrophonePermission() -> Bool {
    let status = AVAudioSession.sharedInstance().recordPermission
    return status == .granted
}
```

## App Lifecycle Management

Handle app state transitions gracefully:

```swift
struct SleepTrackingView: View {
    @Environment(\.scenePhase) private var scenePhase

    var body: some View {
        // ... view content ...
        .onChange(of: scenePhase) { newPhase in
            switch newPhase {
            case .active:
                print("App is active")
                // Refresh UI state if needed
            case .inactive:
                print("App is inactive")
                // Prepare for potential backgrounding
            case .background:
                // Tracking continues in background with audio mode
                print("App is in background")
                // Minimal operations only
            @unknown default:
                break
            }
        }
    }
}
```

### Advanced Lifecycle Handling

```swift
class AppLifecycleObserver: ObservableObject {
    @Published var isActive = true
    private var cancellables = Set<AnyCancellable>()

    init() {
        NotificationCenter.default.publisher(for: UIApplication.didEnterBackgroundNotification)
            .sink { [weak self] _ in
                self?.handleEnterBackground()
            }
            .store(in: &cancellables)

        NotificationCenter.default.publisher(for: UIApplication.willEnterForegroundNotification)
            .sink { [weak self] _ in
                self?.handleEnterForeground()
            }
            .store(in: &cancellables)
    }

    private func handleEnterBackground() {
        isActive = false
        // Save state, reduce operations
    }

    private func handleEnterForeground() {
        isActive = true
        // Refresh state, resume operations
    }
}
```

## Persistent Storage with AppStorage

Store configuration persistently across app launches:

```swift
struct SleepTrackingView: View {
    @AppStorage("sleepapp+apikey") private var apiKey = ""
    @AppStorage("sleepapp+userid") private var userId = ""
    @AppStorage("sleepapp+baseurl") private var baseUrl = ""

    // Values automatically persist across app launches
    // Uses UserDefaults under the hood
}
```

### Custom Storage Keys

```swift
extension String {
    static let apiKeyStorage = "sleepapp+apikey"
    static let userIdStorage = "sleepapp+userid"
    static let baseUrlStorage = "sleepapp+baseurl"
}

struct SleepTrackingView: View {
    @AppStorage(.apiKeyStorage) private var apiKey = ""
    @AppStorage(.userIdStorage) private var userId = ""
    @AppStorage(.baseUrlStorage) private var baseUrl = ""
}
```

### Advanced Persistent Storage

```swift
class PersistentSettings: ObservableObject {
    @AppStorage("sleepapp+apikey") var apiKey = ""
    @AppStorage("sleepapp+userid") var userId = ""
    @AppStorage("sleepapp+baseurl") var baseUrl = ""
    @AppStorage("sleepapp+notifications") var notificationsEnabled = false
    @AppStorage("sleepapp+lastSessionId") var lastSessionId = ""

    func clearAll() {
        apiKey = ""
        userId = ""
        baseUrl = ""
        notificationsEnabled = false
        lastSessionId = ""
    }
}

// Usage
struct SettingsView: View {
    @StateObject private var settings = PersistentSettings()

    var body: some View {
        Form {
            TextField("API Key", text: $settings.apiKey)
            TextField("User ID", text: $settings.userId)
            Toggle("Notifications", isOn: $settings.notificationsEnabled)
        }
    }
}
```
