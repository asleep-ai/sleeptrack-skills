# Testing Guide for Android Sleep Tracking

This guide provides comprehensive testing patterns for Android sleep tracking applications.

## Unit Testing

### ViewModel Testing Setup

```kotlin
@ExperimentalCoroutinesApi
class SleepTrackingViewModelTest {

    @get:Rule
    val instantExecutorRule = InstantTaskExecutorRule()

    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    private lateinit var viewModel: SleepTrackingViewModel
    private lateinit var mockContext: Application

    @Before
    fun setup() {
        mockContext = mock(Application::class.java)
        viewModel = SleepTrackingViewModel(mockContext)
    }

    @Test
    fun `initializeSDK should transition to INITIALIZED on success`() = runTest {
        // Given
        val userId = "test_user_123"

        // When
        viewModel.initializeSDK(userId)

        // Simulate success callback
        // (requires mocking Asleep SDK)

        // Then
        assertEquals(AsleepState.STATE_INITIALIZED, viewModel.trackingState.value)
        assertEquals(userId, viewModel.userId.value)
    }

    @Test
    fun `startTracking should fail if not initialized`() = runTest {
        // When
        viewModel.startTracking()

        // Then
        assertNotEquals(AsleepState.STATE_TRACKING_STARTED, viewModel.trackingState.value)
    }

    @Test
    fun `stopTracking should transition state correctly`() = runTest {
        // Given
        viewModel.initializeSDK("test_user")
        // Simulate initialization success
        viewModel.startTracking()
        // Simulate tracking started

        // When
        viewModel.stopTracking()

        // Then
        assertEquals(AsleepState.STATE_TRACKING_STOPPING, viewModel.trackingState.value)
    }

    @Test
    fun `error handling should classify warnings correctly`() = runTest {
        // Given
        val warningError = AsleepError(
            AsleepErrorCode.ERR_AUDIO_SILENCED,
            "Microphone temporarily unavailable"
        )

        // When
        val isWarning = isWarning(warningError.code)

        // Then
        assertTrue(isWarning)
    }
}
```

### MainDispatcherRule for Coroutines

```kotlin
@ExperimentalCoroutinesApi
class MainDispatcherRule(
    private val dispatcher: TestDispatcher = UnconfinedTestDispatcher()
) : TestWatcher() {

    override fun starting(description: Description) {
        Dispatchers.setMain(dispatcher)
    }

    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}
```

### Testing StateFlow

```kotlin
@Test
fun `trackingState should emit correct states during tracking lifecycle`() = runTest {
    // Given
    val states = mutableListOf<AsleepState>()
    val job = launch {
        viewModel.trackingState.collect { state ->
            states.add(state)
        }
    }

    // When
    viewModel.initializeSDK("test_user")
    advanceUntilIdle()

    // Then
    assertTrue(states.contains(AsleepState.STATE_INITIALIZING))
    assertTrue(states.contains(AsleepState.STATE_INITIALIZED))

    job.cancel()
}
```

## Integration Testing

### Activity Testing

```kotlin
@RunWith(AndroidJUnit4::class)
class SleepTrackingIntegrationTest {

    @get:Rule
    val activityRule = ActivityScenarioRule(MainActivity::class.java)

    @Test
    fun trackingFlow_complete() {
        // Check permissions granted
        onView(withId(R.id.btn_start_stop))
            .check(matches(isEnabled()))

        // Start tracking
        onView(withId(R.id.btn_start_stop))
            .perform(click())

        // Verify tracking started
        onView(withId(R.id.tracking_indicator))
            .check(matches(isDisplayed()))

        // Wait for sequences
        Thread.sleep(60000) // 1 minute

        // Stop tracking
        onView(withId(R.id.btn_start_stop))
            .perform(click())

        // Verify stopped
        onView(withId(R.id.tracking_indicator))
            .check(matches(not(isDisplayed())))
    }

    @Test
    fun errorDisplay_showsCorrectMessage() {
        // Simulate error state
        // Verify error message displayed
        onView(withId(R.id.error_text))
            .check(matches(isDisplayed()))
            .check(matches(withText(containsString("Microphone permission"))))
    }
}
```

### Fragment Testing

```kotlin
@RunWith(AndroidJUnit4::class)
class TrackingFragmentTest {

    @Test
    fun fragmentLaunch_displaysCorrectUI() {
        // Launch fragment in container
        launchFragmentInContainer<TrackingFragment>(
            themeResId = R.style.Theme_SleepTracking
        )

        // Verify initial UI state
        onView(withId(R.id.btnTrack))
            .check(matches(isDisplayed()))
            .check(matches(withText("Start Tracking")))
    }

    @Test
    fun clickStartButton_requestsPermissions() {
        launchFragmentInContainer<TrackingFragment>()

        // Click start button
        onView(withId(R.id.btnTrack))
            .perform(click())

        // Verify permission dialog or state change
        // This depends on permission state
    }
}
```

## Compose UI Testing

### Basic Compose Testing

```kotlin
@RunWith(AndroidJUnit4::class)
class SleepTrackingScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun initialState_showsStartButton() {
        // Given
        val viewModel = mockViewModel(AsleepState.STATE_INITIALIZED)

        // When
        composeTestRule.setContent {
            SleepTrackingScreen(viewModel = viewModel)
        }

        // Then
        composeTestRule.onNodeWithText("Start Sleep Tracking")
            .assertIsDisplayed()
    }

    @Test
    fun trackingActive_showsStopButton() {
        // Given
        val viewModel = mockViewModel(AsleepState.STATE_TRACKING_STARTED)

        // When
        composeTestRule.setContent {
            SleepTrackingScreen(viewModel = viewModel)
        }

        // Then
        composeTestRule.onNodeWithText("Stop Tracking")
            .assertIsDisplayed()
        composeTestRule.onNodeWithText("Tracking in progress")
            .assertIsDisplayed()
    }

    @Test
    fun errorState_displaysErrorMessage() {
        // Given
        val error = AsleepError(AsleepErrorCode.ERR_MIC_PERMISSION, "Permission denied")
        val viewModel = mockViewModel(AsleepState.STATE_ERROR(error))

        // When
        composeTestRule.setContent {
            SleepTrackingScreen(viewModel = viewModel)
        }

        // Then
        composeTestRule.onNodeWithText("Microphone permission is required")
            .assertIsDisplayed()
    }

    private fun mockViewModel(state: AsleepState): SleepTrackingViewModel {
        return mock(SleepTrackingViewModel::class.java).apply {
            whenever(trackingState).thenReturn(MutableStateFlow(state))
            whenever(sequence).thenReturn(MutableLiveData(0))
        }
    }
}
```

### Testing User Interactions

```kotlin
@Test
fun clickStartButton_startsTracking() {
    // Given
    val viewModel = SleepTrackingViewModel(mockContext)
    composeTestRule.setContent {
        SleepTrackingScreen(viewModel = viewModel)
    }

    // When
    composeTestRule.onNodeWithText("Start Sleep Tracking")
        .performClick()

    // Then
    verify(viewModel).startTracking()
}

@Test
fun clickStopButton_stopsTracking() {
    // Given
    val viewModel = mockViewModel(AsleepState.STATE_TRACKING_STARTED)
    composeTestRule.setContent {
        SleepTrackingScreen(viewModel = viewModel)
    }

    // When
    composeTestRule.onNodeWithText("Stop Tracking")
        .performClick()

    // Then
    verify(viewModel).stopTracking()
}
```

## Permission Testing

### Testing Permission Manager

```kotlin
@RunWith(AndroidJUnit4::class)
class PermissionManagerTest {

    private lateinit var activity: AppCompatActivity
    private lateinit var permissionManager: PermissionManager

    @Before
    fun setup() {
        val scenario = ActivityScenario.launch(TestActivity::class.java)
        scenario.onActivity { act ->
            activity = act
            permissionManager = PermissionManager(activity)
        }
    }

    @Test
    fun requestPermissions_checksAllPermissions() {
        // When
        permissionManager.requestPermissions()

        // Then
        // Verify permission checks called
        verify(permissionManager).isBatteryOptimizationIgnored()
        verify(permissionManager).hasMicrophonePermission()
    }

    @Test
    fun allPermissionsGranted_updatesLiveData() {
        // Given
        // Mock all permissions as granted

        // When
        permissionManager.checkAndRequestNext()

        // Then
        assertTrue(permissionManager.allPermissionsGranted.value == true)
    }
}
```

## Instrumentation Testing

### Testing Foreground Service

```kotlin
@RunWith(AndroidJUnit4::class)
class ForegroundServiceTest {

    @get:Rule
    val serviceRule = ServiceTestRule()

    @Test
    fun trackingService_startsForeground() {
        // Given
        val context = InstrumentationRegistry.getInstrumentation().targetContext

        // When
        val serviceIntent = Intent(context, SleepTrackingService::class.java)
        serviceRule.startService(serviceIntent)

        // Then
        // Verify service is in foreground
        assertTrue(isServiceForeground(context, SleepTrackingService::class.java))
    }

    private fun isServiceForeground(context: Context, serviceClass: Class<*>): Boolean {
        val manager = context.getSystemService(Context.ACTIVITY_SERVICE) as ActivityManager
        return manager.getRunningServices(Integer.MAX_VALUE)
            .any { it.service.className == serviceClass.name && it.foreground }
    }
}
```

## Mock SDK Testing

### Creating SDK Mocks

```kotlin
class MockAsleepSDK {
    companion object {
        fun mockInitSuccess(userId: String, listener: Asleep.AsleepConfigListener) {
            val config = mock(AsleepConfig::class.java)
            listener.onSuccess(userId, config)
        }

        fun mockInitFail(errorCode: Int, listener: Asleep.AsleepConfigListener) {
            listener.onFail(errorCode, "Test error")
        }

        fun mockTrackingStart(sessionId: String, listener: Asleep.AsleepTrackingListener) {
            listener.onStart(sessionId)
        }

        fun mockTrackingPerform(sequence: Int, listener: Asleep.AsleepTrackingListener) {
            listener.onPerform(sequence)
        }
    }
}
```

### Using Mocks in Tests

```kotlin
@Test
fun initializeSDK_successFlow() = runTest {
    // Given
    val userId = "test_user"

    // Mock Asleep SDK
    MockAsleepSDK.mockInitSuccess(userId, any())

    // When
    viewModel.initializeSDK(userId)
    advanceUntilIdle()

    // Then
    assertEquals(AsleepState.STATE_INITIALIZED, viewModel.trackingState.value)
}
```

## Test Dependencies

Add to your `build.gradle`:

```gradle
dependencies {
    // Unit testing
    testImplementation 'junit:junit:4.13.2'
    testImplementation 'org.mockito:mockito-core:5.3.1'
    testImplementation 'org.mockito.kotlin:mockito-kotlin:5.0.0'
    testImplementation 'androidx.arch.core:core-testing:2.2.0'
    testImplementation 'org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3'

    // Android instrumentation testing
    androidTestImplementation 'androidx.test.ext:junit:1.1.5'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.5.1'
    androidTestImplementation 'androidx.test:runner:1.5.2'
    androidTestImplementation 'androidx.test:rules:1.5.0'

    // Fragment testing
    debugImplementation 'androidx.fragment:fragment-testing:1.6.1'

    // Compose testing
    androidTestImplementation 'androidx.compose.ui:ui-test-junit4:1.5.4'
    debugImplementation 'androidx.compose.ui:ui-test-manifest:1.5.4'
}
```

## Testing Best Practices

1. **Unit Tests**: Test ViewModels, business logic, and state management
2. **Integration Tests**: Test UI interactions and component integration
3. **Use Test Rules**: Leverage JUnit rules for setup/teardown
4. **Mock External Dependencies**: Mock Asleep SDK calls for predictable tests
5. **Test Coroutines**: Use `runTest` and `TestDispatcher` for coroutine testing
6. **Test Permissions**: Verify permission flows but avoid actual permission dialogs
7. **Compose Tests**: Use semantic properties and avoid hardcoded strings
8. **CI/CD**: Run tests in continuous integration pipeline

## Debugging Tests

### Enable Debug Logging

```kotlin
@Before
fun setup() {
    // Enable debug logging for tests
    Log.setDebug(true)
}
```

### Capture Screenshots on Failure

```kotlin
@Rule
fun testRule = TestRule { base, description ->
    object : Statement() {
        override fun evaluate() {
            try {
                base.evaluate()
            } catch (t: Throwable) {
                // Capture screenshot
                Screenshot.capture()
                throw t
            }
        }
    }
}
```

## Test Coverage

Aim for:
- **Unit Tests**: 80%+ coverage of ViewModels and business logic
- **Integration Tests**: Key user flows and state transitions
- **UI Tests**: Critical user interactions

Use JaCoCo for coverage reporting:

```gradle
apply plugin: 'jacoco'

jacoco {
    toolVersion = "0.8.10"
}
```
