# Complete ViewModel Implementation

This reference provides full implementation examples for iOS sleep tracking using Combine and SwiftUI.

## Full ViewModel with Combine

```swift
import Foundation
import Combine
import AsleepSDK

final class SleepTrackingViewModel: ObservableObject {
    // MARK: - SDK Components
    private(set) var trackingManager: Asleep.SleepTrackingManager?
    private(set) var reports: Asleep.Reports?

    // MARK: - Published State
    @Published var userId: String?
    @Published var sessionId: String?
    @Published var sequenceNumber: Int?
    @Published var error: String?
    @Published var isTracking = false
    @Published var currentReport: Asleep.Model.Report?
    @Published var reportList: [Asleep.Model.SleepSession]?
    @Published private(set) var config: Asleep.Config?

    // MARK: - Initialization
    func initAsleepConfig(
        apiKey: String,
        userId: String,
        baseUrl: URL? = nil,
        callbackUrl: URL? = nil
    ) {
        Asleep.initAsleepConfig(
            apiKey: apiKey,
            userId: userId.isEmpty ? nil : userId,
            baseUrl: baseUrl,
            callbackUrl: callbackUrl,
            delegate: self
        )

        // Optional: Enable debug logging
        Asleep.setDebugLoggerDelegate(self)
    }

    func initSleepTrackingManager() {
        guard let config else { return }
        trackingManager = Asleep.createSleepTrackingManager(
            config: config,
            delegate: self
        )
    }

    func initReports() {
        guard let config else { return }
        reports = Asleep.createReports(config: config)
    }

    // MARK: - Tracking Control
    func startTracking() {
        trackingManager?.startTracking()
    }

    func stopTracking() {
        trackingManager?.stopTracking()
        initReports()
    }
}

// MARK: - AsleepConfigDelegate
extension SleepTrackingViewModel: AsleepConfigDelegate {
    func userDidJoin(userId: String, config: Asleep.Config) {
        Task { @MainActor in
            self.config = config
            self.userId = userId
            initSleepTrackingManager()
        }
    }

    func didFailUserJoin(error: Asleep.AsleepError) {
        Task { @MainActor in
            self.error = "Failed to join: \(error.localizedDescription)"
        }
    }

    func userDidDelete(userId: String) {
        Task { @MainActor in
            self.userId = nil
            self.config = nil
        }
    }
}

// MARK: - AsleepSleepTrackingManagerDelegate
extension SleepTrackingViewModel: AsleepSleepTrackingManagerDelegate {
    func didCreate() {
        Task { @MainActor in
            self.isTracking = true
            self.error = nil
        }
    }

    func didUpload(sequence: Int) {
        Task { @MainActor in
            self.sequenceNumber = sequence
        }
    }

    func didClose(sessionId: String) {
        Task { @MainActor in
            self.isTracking = false
            self.sessionId = sessionId
        }
    }

    func didFail(error: Asleep.AsleepError) {
        switch error {
        case let .httpStatus(code, _, message) where code == 403 || code == 404:
            Task { @MainActor in
                self.isTracking = false
                self.error = "\(code): \(message ?? "Unknown error")"
            }
        default:
            Task { @MainActor in
                self.error = error.localizedDescription
            }
        }
    }

    func didInterrupt() {
        Task { @MainActor in
            self.error = "Tracking interrupted (e.g., phone call)"
        }
    }

    func didResume() {
        Task { @MainActor in
            self.error = nil
        }
    }

    func micPermissionWasDenied() {
        Task { @MainActor in
            self.isTracking = false
            self.error = "Microphone permission denied. Please enable in Settings."
        }
    }

    func analysing(session: Asleep.Model.Session) {
        // Optional: Handle real-time analysis data
        print("Real-time analysis:", session)
    }
}

// MARK: - AsleepDebugLoggerDelegate (Optional)
extension SleepTrackingViewModel: AsleepDebugLoggerDelegate {
    func didPrint(message: String) {
        print("[Asleep SDK]", message)
    }
}
```

## Complete SwiftUI View

```swift
import SwiftUI

struct SleepTrackingView: View {
    @StateObject private var viewModel = SleepTrackingViewModel()
    @AppStorage("sampleapp+apikey") private var apiKey = ""
    @AppStorage("sampleapp+userid") private var userId = ""

    @State private var startTime: Date?
    @State private var showingReport = false

    var body: some View {
        VStack(spacing: 20) {
            // Configuration Section
            VStack(alignment: .leading, spacing: 8) {
                Text("Configuration")
                    .font(.headline)

                TextField("API Key", text: $apiKey)
                    .textFieldStyle(.roundedBorder)
                    .disabled(viewModel.isTracking)

                TextField("User ID", text: $userId)
                    .textFieldStyle(.roundedBorder)
                    .disabled(viewModel.isTracking)
            }
            .padding()

            // Status Section
            VStack(alignment: .leading, spacing: 8) {
                Text("Status")
                    .font(.headline)

                if let error = viewModel.error {
                    Text("Error: \(error)")
                        .foregroundColor(.red)
                        .font(.caption)
                }

                if viewModel.isTracking {
                    HStack {
                        ProgressView()
                        Text("Tracking...")
                    }

                    if let sequence = viewModel.sequenceNumber {
                        Text("Sequence: \(sequence)")
                            .font(.caption)
                    }
                }

                if let sessionId = viewModel.sessionId {
                    Text("Session ID: \(sessionId)")
                        .font(.caption)
                        .lineLimit(1)
                }
            }
            .padding()

            // Tracking Control
            Button(action: {
                if viewModel.isTracking {
                    stopTracking()
                } else {
                    startTracking()
                }
            }) {
                Text(viewModel.isTracking ? "Stop Tracking" : "Start Tracking")
                    .frame(maxWidth: .infinity)
                    .padding()
                    .background(viewModel.isTracking ? Color.red : Color.blue)
                    .foregroundColor(.white)
                    .cornerRadius(10)
            }
            .disabled(apiKey.isEmpty || userId.isEmpty)
            .padding(.horizontal)

            // Report Access
            if viewModel.sessionId != nil {
                Button("View Report") {
                    fetchReport()
                }
                .padding()
            }

            Spacer()
        }
        .padding()
        .sheet(isPresented: $showingReport) {
            ReportView(report: viewModel.currentReport)
        }
    }

    private func startTracking() {
        viewModel.sessionId = nil
        viewModel.sequenceNumber = nil

        if viewModel.config == nil {
            viewModel.initAsleepConfig(
                apiKey: apiKey,
                userId: userId
            )
        } else {
            viewModel.startTracking()
        }

        startTime = Date()
    }

    private func stopTracking() {
        viewModel.stopTracking()
    }

    private func fetchReport() {
        guard let sessionId = viewModel.sessionId else { return }

        Task {
            do {
                let report = try await viewModel.reports?.report(sessionId: sessionId)
                await MainActor.run {
                    viewModel.currentReport = report
                    showingReport = true
                }
            } catch {
                await MainActor.run {
                    viewModel.error = "Failed to fetch report: \(error.localizedDescription)"
                }
            }
        }
    }
}
```

## Complete Report View

```swift
import SwiftUI
import AsleepSDK

struct ReportView: View {
    @Environment(\.dismiss) private var dismiss
    let report: Asleep.Model.Report?

    var body: some View {
        NavigationView {
            ScrollView {
                if let report = report {
                    VStack(alignment: .leading, spacing: 16) {
                        // Session Information
                        Section("Session Information") {
                            InfoRow(label: "Session ID", value: report.session.id)
                            InfoRow(label: "Start Time", value: report.session.startTime.formatted())
                            if let endTime = report.session.endTime {
                                InfoRow(label: "End Time", value: endTime.formatted())
                            }
                            InfoRow(label: "State", value: report.session.state.rawValue)
                        }

                        Divider()

                        // Sleep Statistics
                        if let stat = report.stat {
                            Section("Sleep Statistics") {
                                StatRow(label: "Sleep Efficiency", value: stat.sleepEfficiency, unit: "%")
                                StatRow(label: "Sleep Latency", value: stat.sleepLatency, unit: "min")
                                StatRow(label: "Total Sleep Time", value: stat.sleepTime, unit: "min")
                                StatRow(label: "Time in Bed", value: stat.timeInBed, unit: "min")
                            }

                            Divider()

                            Section("Sleep Stages") {
                                StatRow(label: "Deep Sleep", value: stat.timeInDeep, unit: "min")
                                StatRow(label: "Light Sleep", value: stat.timeInLight, unit: "min")
                                StatRow(label: "REM Sleep", value: stat.timeInRem, unit: "min")
                                StatRow(label: "Wake Time", value: stat.timeInWake, unit: "min")
                            }

                            Divider()

                            Section("Snoring Analysis") {
                                StatRow(label: "Time Snoring", value: stat.timeInSnoring, unit: "min")
                                StatRow(label: "Snoring Count", value: stat.snoringCount, unit: "times")
                            }
                        }
                    }
                    .padding()
                } else {
                    Text("No report available")
                        .foregroundColor(.secondary)
                }
            }
            .navigationTitle("Sleep Report")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    Button("Done") {
                        dismiss()
                    }
                }
            }
        }
    }
}

struct InfoRow: View {
    let label: String
    let value: String

    var body: some View {
        HStack {
            Text(label)
                .foregroundColor(.secondary)
            Spacer()
            Text(value)
        }
    }
}

struct StatRow: View {
    let label: String
    let value: Int?
    let unit: String

    var body: some View {
        HStack {
            Text(label)
                .foregroundColor(.secondary)
            Spacer()
            if let value = value {
                Text("\(value) \(unit)")
            } else {
                Text("N/A")
                    .foregroundColor(.secondary)
            }
        }
    }
}

struct Section<Content: View>: View {
    let title: String
    let content: Content

    init(_ title: String, @ViewBuilder content: () -> Content) {
        self.title = title
        self.content = content()
    }

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            Text(title)
                .font(.headline)
            content
        }
    }
}
```
