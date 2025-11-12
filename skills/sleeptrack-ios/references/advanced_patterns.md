# Advanced Integration Patterns

Comprehensive patterns for different app architectures and advanced error handling strategies.

## Pattern 1: Simple Single-View App

Best for: Basic sleep tracking app with minimal features.

```swift
@main
struct SimpleSleepApp: App {
    var body: some Scene {
        WindowGroup {
            SleepTrackingView()
        }
    }
}
```

### Advantages

- Simple architecture
- Easy to understand and maintain
- Minimal code overhead
- Fast development

### Use Cases

- MVP or prototype apps
- Single-purpose sleep trackers
- Learning/demo applications

## Pattern 2: Multi-View App with Navigation

Best for: Apps with reports, settings, and history.

```swift
struct ContentView: View {
    var body: some View {
        TabView {
            SleepTrackingView()
                .tabItem {
                    Label("Track", systemImage: "moon.zzz")
                }

            ReportHistoryView()
                .tabItem {
                    Label("History", systemImage: "chart.bar")
                }

            SettingsView()
                .tabItem {
                    Label("Settings", systemImage: "gear")
                }
        }
    }
}
```

### Supporting Views

```swift
struct ReportHistoryView: View {
    @StateObject private var viewModel = ReportHistoryViewModel()

    var body: some View {
        NavigationView {
            List(viewModel.sessions) { session in
                NavigationLink {
                    ReportDetailView(sessionId: session.id)
                } label: {
                    SessionRow(session: session)
                }
            }
            .navigationTitle("Sleep History")
            .onAppear {
                viewModel.loadSessions()
            }
        }
    }
}

struct SettingsView: View {
    @AppStorage("sleepapp+apikey") private var apiKey = ""
    @AppStorage("sleepapp+userid") private var userId = ""
    @AppStorage("sleepapp+notifications") private var notificationsEnabled = false

    var body: some View {
        NavigationView {
            Form {
                Section("Account") {
                    TextField("User ID", text: $userId)
                    SecureField("API Key", text: $apiKey)
                }

                Section("Preferences") {
                    Toggle("Enable Notifications", isOn: $notificationsEnabled)
                }
            }
            .navigationTitle("Settings")
        }
    }
}
```

### Advantages

- Clear separation of concerns
- Scalable for additional features
- Familiar tab-based navigation
- Easy to add new sections

## Pattern 3: Centralized SDK Manager

Best for: Complex apps sharing SDK instance across views.

```swift
final class AsleepSDKManager: ObservableObject {
    static let shared = AsleepSDKManager()

    @Published var config: Asleep.Config?
    @Published var isInitialized = false
    @Published var error: String?

    private var trackingManager: Asleep.SleepTrackingManager?
    private var reports: Asleep.Reports?

    private init() {}

    func initialize(apiKey: String, userId: String) {
        Asleep.initAsleepConfig(
            apiKey: apiKey,
            userId: userId,
            delegate: self
        )
    }

    func getTrackingManager() -> Asleep.SleepTrackingManager? {
        guard let config else { return nil }
        if trackingManager == nil {
            trackingManager = Asleep.createSleepTrackingManager(
                config: config,
                delegate: self
            )
        }
        return trackingManager
    }

    func getReports() -> Asleep.Reports? {
        guard let config else { return nil }
        if reports == nil {
            reports = Asleep.createReports(config: config)
        }
        return reports
    }

    func reset() {
        trackingManager = nil
        reports = nil
        config = nil
        isInitialized = false
    }
}

// MARK: - Delegates
extension AsleepSDKManager: AsleepConfigDelegate {
    func userDidJoin(userId: String, config: Asleep.Config) {
        Task { @MainActor in
            self.config = config
            self.isInitialized = true
        }
    }

    func didFailUserJoin(error: Asleep.AsleepError) {
        Task { @MainActor in
            self.error = "Failed to join: \(error.localizedDescription)"
        }
    }

    func userDidDelete(userId: String) {
        Task { @MainActor in
            reset()
        }
    }
}

// Usage in views
struct SleepTrackingView: View {
    @ObservedObject var sdkManager = AsleepSDKManager.shared
    @StateObject private var trackingState = TrackingStateViewModel()

    var body: some View {
        VStack {
            if sdkManager.isInitialized {
                TrackingControls(manager: sdkManager.getTrackingManager())
            } else {
                ConfigurationView()
            }
        }
    }
}

struct ReportHistoryView: View {
    @ObservedObject var sdkManager = AsleepSDKManager.shared

    var body: some View {
        NavigationView {
            if sdkManager.isInitialized {
                ReportList(reports: sdkManager.getReports())
            } else {
                Text("Please configure the app first")
            }
        }
    }
}
```

### Advantages

- Single source of truth for SDK state
- Prevents duplicate SDK instances
- Centralized error handling
- Easy to manage lifecycle across app

### Testing Strategy

```swift
protocol AsleepSDKManagerProtocol {
    var config: Asleep.Config? { get }
    var isInitialized: Bool { get }
    func initialize(apiKey: String, userId: String)
    func getTrackingManager() -> Asleep.SleepTrackingManager?
}

// For testing
class MockSDKManager: AsleepSDKManagerProtocol, ObservableObject {
    @Published var config: Asleep.Config?
    @Published var isInitialized = false

    func initialize(apiKey: String, userId: String) {
        isInitialized = true
    }

    func getTrackingManager() -> Asleep.SleepTrackingManager? {
        nil
    }
}
```

## Error Recovery Patterns

### Automatic Retry with Exponential Backoff

```swift
final class SleepTrackingViewModel: ObservableObject {
    private var retryCount = 0
    private let maxRetries = 3

    func startTrackingWithRetry() {
        trackingManager?.startTracking()
    }

    func didFail(error: Asleep.AsleepError) {
        if isTransientError(error) && retryCount < maxRetries {
            retryCount += 1
            let delay = pow(2.0, Double(retryCount))

            DispatchQueue.main.asyncAfter(deadline: .now() + delay) { [weak self] in
                self?.startTrackingWithRetry()
            }
        } else {
            retryCount = 0
            handleError(error)
        }
    }

    private func isTransientError(_ error: Asleep.AsleepError) -> Bool {
        switch error {
        case .networkError, .uploadFailed:
            return true
        default:
            return false
        }
    }
}
```

### Error Categorization and Handling

```swift
extension SleepTrackingViewModel {
    func handleError(_ error: Asleep.AsleepError) {
        switch error {
        case .micPermission:
            showAlert(
                title: "Microphone Access Required",
                message: "Please enable microphone access in Settings to track sleep.",
                action: openSettings
            )

        case .audioSessionError:
            showAlert(
                title: "Audio Unavailable",
                message: "Another app is using the microphone. Please close it and try again."
            )

        case let .httpStatus(code, _, message):
            switch code {
            case 403:
                showAlert(
                    title: "Session Already Active",
                    message: "Another device is tracking with this user ID."
                )
            case 404:
                showAlert(
                    title: "Session Not Found",
                    message: "The tracking session could not be found."
                )
            default:
                showAlert(
                    title: "Error \(code)",
                    message: message ?? "An unknown error occurred"
                )
            }

        default:
            showAlert(
                title: "Error",
                message: error.localizedDescription
            )
        }
    }

    private func openSettings() {
        if let url = URL(string: UIApplication.openSettingsURLString) {
            UIApplication.shared.open(url)
        }
    }
}
```

### Report Fetching with Retry

```swift
func fetchReportWithRetry(sessionId: String, maxAttempts: Int = 5) async {
    for attempt in 1...maxAttempts {
        do {
            let report = try await reports?.report(sessionId: sessionId)
            await MainActor.run {
                self.currentReport = report
            }
            return
        } catch {
            if attempt < maxAttempts {
                // Wait before retrying (exponential backoff)
                let delay = UInt64(pow(2.0, Double(attempt))) * 1_000_000_000
                try? await Task.sleep(nanoseconds: delay)
            } else {
                await MainActor.run {
                    self.error = "Report not ready. Please try again later."
                }
            }
        }
    }
}
```

### Graceful Degradation

```swift
struct SleepTrackingView: View {
    @StateObject private var viewModel = SleepTrackingViewModel()
    @State private var offlineMode = false

    var body: some View {
        VStack {
            if offlineMode {
                OfflineModeView()
            } else {
                OnlineModeView(viewModel: viewModel)
            }
        }
        .onReceive(viewModel.$error) { error in
            if let error = error, isNetworkError(error) {
                offlineMode = true
            }
        }
    }

    private func isNetworkError(_ error: String) -> Bool {
        error.contains("network") || error.contains("connection")
    }
}

struct OfflineModeView: View {
    var body: some View {
        VStack(spacing: 20) {
            Image(systemName: "wifi.slash")
                .font(.system(size: 60))
                .foregroundColor(.secondary)

            Text("Offline Mode")
                .font(.headline)

            Text("Sleep tracking data will sync when connection is restored")
                .font(.caption)
                .foregroundColor(.secondary)
                .multilineTextAlignment(.center)
                .padding(.horizontal)
        }
    }
}
```

## Testing Patterns

### Dependency Injection for Testing

```swift
protocol SleepTrackingManagerProtocol {
    func startTracking()
    func stopTracking()
}

final class SleepTrackingViewModel: ObservableObject {
    private let trackingManager: SleepTrackingManagerProtocol

    init(trackingManager: SleepTrackingManagerProtocol) {
        self.trackingManager = trackingManager
    }

    func startTracking() {
        trackingManager.startTracking()
    }
}

// For testing
class MockTrackingManager: SleepTrackingManagerProtocol {
    var startTrackingCalled = false
    var stopTrackingCalled = false

    func startTracking() {
        startTrackingCalled = true
    }

    func stopTracking() {
        stopTrackingCalled = true
    }
}

// Usage in tests
func testStartTracking() {
    let mockManager = MockTrackingManager()
    let viewModel = SleepTrackingViewModel(trackingManager: mockManager)

    viewModel.startTracking()

    XCTAssertTrue(mockManager.startTrackingCalled)
}
```
