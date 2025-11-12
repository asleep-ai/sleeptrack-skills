# Complete ViewModel and Activity Implementation

This reference provides complete, production-ready implementations for Android MVVM architecture with the Asleep SDK.

## Complete ViewModel with State Management

```kotlin
@HiltViewModel
class SleepTrackingViewModel @Inject constructor(
    @ApplicationContext private val applicationContext: Application
) : ViewModel() {

    // State management
    private val _trackingState = MutableStateFlow<AsleepState>(AsleepState.STATE_IDLE)
    val trackingState: StateFlow<AsleepState> = _trackingState.asStateFlow()

    private val _userId = MutableLiveData<String?>()
    val userId: LiveData<String?> = _userId

    private val _sessionId = MutableStateFlow<String?>(null)
    val sessionId: StateFlow<String?> = _sessionId.asStateFlow()

    private val _sequence = MutableLiveData<Int>()
    val sequence: LiveData<Int> = _sequence

    private var asleepConfig: AsleepConfig? = null

    // Tracking listener
    private val trackingListener = object : Asleep.AsleepTrackingListener {
        override fun onStart(sessionId: String) {
            _sessionId.value = sessionId
            _trackingState.value = AsleepState.STATE_TRACKING_STARTED
        }

        override fun onPerform(sequence: Int) {
            _sequence.postValue(sequence)

            // Check real-time data every 10 sequences after sequence 10
            if (sequence > 10 && sequence % 10 == 0) {
                getCurrentSleepData()
            }
        }

        override fun onFinish(sessionId: String?) {
            _sessionId.value = sessionId
            _trackingState.value = AsleepState.STATE_IDLE
        }

        override fun onFail(errorCode: Int, detail: String) {
            handleError(AsleepError(errorCode, detail))
        }
    }

    fun initializeSDK(userId: String) {
        if (_trackingState.value != AsleepState.STATE_IDLE) return

        _trackingState.value = AsleepState.STATE_INITIALIZING

        Asleep.initAsleepConfig(
            context = applicationContext,
            apiKey = BuildConfig.ASLEEP_API_KEY,
            userId = userId,
            baseUrl = "https://api.asleep.ai",
            callbackUrl = null,
            service = applicationContext.getString(R.string.app_name),
            asleepConfigListener = object : Asleep.AsleepConfigListener {
                override fun onSuccess(userId: String?, config: AsleepConfig?) {
                    asleepConfig = config
                    _userId.value = userId
                    _trackingState.value = AsleepState.STATE_INITIALIZED
                }

                override fun onFail(errorCode: Int, detail: String) {
                    _trackingState.value = AsleepState.STATE_ERROR(
                        AsleepError(errorCode, detail)
                    )
                }
            }
        )
    }

    fun startTracking() {
        val config = asleepConfig ?: return
        if (_trackingState.value != AsleepState.STATE_INITIALIZED) return

        _trackingState.value = AsleepState.STATE_TRACKING_STARTING

        Asleep.beginSleepTracking(
            asleepConfig = config,
            asleepTrackingListener = trackingListener,
            notificationTitle = "Sleep Tracking",
            notificationText = "Recording your sleep",
            notificationIcon = R.drawable.ic_notification,
            notificationClass = MainActivity::class.java
        )
    }

    fun stopTracking() {
        if (Asleep.isSleepTrackingAlive(applicationContext)) {
            _trackingState.value = AsleepState.STATE_TRACKING_STOPPING
            Asleep.endSleepTracking()
        }
    }

    fun reconnectToTracking() {
        if (Asleep.isSleepTrackingAlive(applicationContext)) {
            Asleep.connectSleepTracking(trackingListener)
            _trackingState.value = AsleepState.STATE_TRACKING_STARTED
        }
    }

    private fun getCurrentSleepData() {
        Asleep.getCurrentSleepData(
            asleepSleepDataListener = object : Asleep.AsleepSleepDataListener {
                override fun onSleepDataReceived(session: Session) {
                    // Process real-time sleep data
                    val currentStage = session.sleepStages?.lastOrNull()
                    Log.d("Sleep", "Current stage: $currentStage")
                }

                override fun onFail(errorCode: Int, detail: String) {
                    Log.e("Sleep", "Failed to get data: $errorCode")
                }
            }
        )
    }

    private fun handleError(error: AsleepError) {
        if (isWarning(error.code)) {
            // Log warning but continue tracking
            Log.w("Sleep", "Warning: ${error.code} - ${error.message}")
        } else {
            // Critical error - stop tracking
            _trackingState.value = AsleepState.STATE_ERROR(error)
        }
    }
}
```

## Complete Activity Implementation

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {

    private lateinit var binding: ActivityMainBinding
    private val viewModel: SleepTrackingViewModel by viewModels()
    private lateinit var permissionManager: PermissionManager

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        permissionManager = PermissionManager(this)

        setupUI()
        observeState()
        checkExistingSession()
    }

    private fun setupUI() {
        binding.btnStartStop.setOnClickListener {
            when (viewModel.trackingState.value) {
                AsleepState.STATE_INITIALIZED -> {
                    if (permissionManager.allPermissionsGranted.value == true) {
                        viewModel.startTracking()
                    } else {
                        permissionManager.requestPermissions()
                    }
                }
                AsleepState.STATE_TRACKING_STARTED -> {
                    viewModel.stopTracking()
                }
                else -> {
                    // Button disabled
                }
            }
        }
    }

    private fun observeState() {
        lifecycleScope.launch {
            viewModel.trackingState.collect { state ->
                updateUI(state)
            }
        }

        viewModel.sequence.observe(this) { sequence ->
            binding.tvSequence.text = "Sequence: $sequence"
        }
    }

    private fun updateUI(state: AsleepState) {
        when (state) {
            AsleepState.STATE_IDLE -> {
                binding.btnStartStop.isEnabled = false
                binding.btnStartStop.text = "Initializing..."
            }
            AsleepState.STATE_INITIALIZED -> {
                binding.btnStartStop.isEnabled = true
                binding.btnStartStop.text = "Start Tracking"
            }
            AsleepState.STATE_TRACKING_STARTED -> {
                binding.btnStartStop.isEnabled = true
                binding.btnStartStop.text = "Stop Tracking"
                binding.trackingIndicator.visibility = View.VISIBLE
            }
            AsleepState.STATE_TRACKING_STOPPING -> {
                binding.btnStartStop.isEnabled = false
                binding.btnStartStop.text = "Stopping..."
            }
            is AsleepState.STATE_ERROR -> {
                showErrorDialog(state.errorCode)
            }
            else -> {}
        }
    }

    private fun checkExistingSession() {
        if (Asleep.isSleepTrackingAlive(applicationContext)) {
            viewModel.reconnectToTracking()
        } else {
            viewModel.initializeSDK(getUserId())
        }
    }

    private fun getUserId(): String {
        // Load from SharedPreferences or generate
        val prefs = getSharedPreferences("asleep", MODE_PRIVATE)
        return prefs.getString("user_id", null) ?: run {
            val newId = UUID.randomUUID().toString()
            prefs.edit().putString("user_id", newId).apply()
            newId
        }
    }
}
```

## Complete PermissionManager

```kotlin
class PermissionManager(private val activity: AppCompatActivity) {
    private val context = activity.applicationContext

    // Permission launchers
    private val batteryOptimizationLauncher =
        activity.registerForActivityResult(
            ActivityResultContracts.StartActivityForResult()
        ) {
            checkAndRequestNext()
        }

    private val micPermissionLauncher =
        activity.registerForActivityResult(
            ActivityResultContracts.RequestPermission()
        ) { granted ->
            if (granted) checkAndRequestNext()
            else showPermissionDeniedDialog("Microphone")
        }

    private val notificationPermissionLauncher =
        activity.registerForActivityResult(
            ActivityResultContracts.RequestPermission()
        ) { granted ->
            if (granted) checkAndRequestNext()
            else showPermissionDeniedDialog("Notifications")
        }

    private val _allPermissionsGranted = MutableLiveData(false)
    val allPermissionsGranted: LiveData<Boolean> = _allPermissionsGranted

    fun requestPermissions() {
        checkAndRequestNext()
    }

    private fun checkAndRequestNext() {
        when {
            !isBatteryOptimizationIgnored() -> requestBatteryOptimization()
            !hasMicrophonePermission() -> requestMicrophone()
            !hasNotificationPermission() -> requestNotification()
            else -> {
                _allPermissionsGranted.value = true
            }
        }
    }

    private fun requestBatteryOptimization() {
        val intent = Intent().apply {
            action = Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS
            data = Uri.parse("package:${context.packageName}")
        }
        batteryOptimizationLauncher.launch(intent)
    }

    private fun requestMicrophone() {
        if (ActivityCompat.shouldShowRequestPermissionRationale(
                activity,
                Manifest.permission.RECORD_AUDIO
            )) {
            showPermissionRationale(
                "Microphone access is required to record sleep sounds"
            )
        } else {
            micPermissionLauncher.launch(Manifest.permission.RECORD_AUDIO)
        }
    }

    private fun requestNotification() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            notificationPermissionLauncher.launch(
                Manifest.permission.POST_NOTIFICATIONS
            )
        } else {
            checkAndRequestNext()
        }
    }

    private fun isBatteryOptimizationIgnored(): Boolean {
        val pm = context.getSystemService(Context.POWER_SERVICE) as PowerManager
        return pm.isIgnoringBatteryOptimizations(context.packageName)
    }

    private fun hasMicrophonePermission(): Boolean {
        return ContextCompat.checkSelfPermission(
            context,
            Manifest.permission.RECORD_AUDIO
        ) == PackageManager.PERMISSION_GRANTED
    }

    private fun hasNotificationPermission(): Boolean {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            ContextCompat.checkSelfPermission(
                context,
                Manifest.permission.POST_NOTIFICATIONS
            ) == PackageManager.PERMISSION_GRANTED
        } else {
            NotificationManagerCompat.from(context).areNotificationsEnabled()
        }
    }

    private fun showPermissionRationale(message: String) {
        AlertDialog.Builder(activity)
            .setTitle("Permission Required")
            .setMessage(message)
            .setPositiveButton("OK") { _, _ ->
                checkAndRequestNext()
            }
            .show()
    }

    private fun showPermissionDeniedDialog(permissionName: String) {
        AlertDialog.Builder(activity)
            .setTitle("Permission Denied")
            .setMessage("$permissionName permission is required for sleep tracking. Please grant it in app settings.")
            .setPositiveButton("Settings") { _, _ ->
                val intent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
                    data = Uri.parse("package:${context.packageName}")
                }
                activity.startActivity(intent)
            }
            .setNegativeButton("Cancel", null)
            .show()
    }
}
```

## Minimal Production Templates

### Minimal ViewModel

```kotlin
@HiltViewModel
class SleepTrackingViewModel @Inject constructor(
    @ApplicationContext private val app: Application
) : ViewModel() {

    private val _trackingState = MutableStateFlow<AsleepState>(AsleepState.STATE_IDLE)
    val trackingState: StateFlow<AsleepState> = _trackingState

    private var config: AsleepConfig? = null

    private val listener = object : Asleep.AsleepTrackingListener {
        override fun onStart(sessionId: String) {
            _trackingState.value = AsleepState.STATE_TRACKING_STARTED
        }
        override fun onPerform(sequence: Int) {}
        override fun onFinish(sessionId: String?) {
            _trackingState.value = AsleepState.STATE_IDLE
        }
        override fun onFail(errorCode: Int, detail: String) {
            _trackingState.value = AsleepState.STATE_ERROR(AsleepError(errorCode, detail))
        }
    }

    fun initializeSDK(userId: String) {
        Asleep.initAsleepConfig(
            context = app,
            apiKey = BuildConfig.ASLEEP_API_KEY,
            userId = userId,
            baseUrl = "https://api.asleep.ai",
            callbackUrl = null,
            service = "MyApp",
            asleepConfigListener = object : Asleep.AsleepConfigListener {
                override fun onSuccess(userId: String?, asleepConfig: AsleepConfig?) {
                    config = asleepConfig
                    _trackingState.value = AsleepState.STATE_INITIALIZED
                }
                override fun onFail(errorCode: Int, detail: String) {
                    _trackingState.value = AsleepState.STATE_ERROR(AsleepError(errorCode, detail))
                }
            }
        )
    }

    fun startTracking() {
        config?.let {
            Asleep.beginSleepTracking(
                asleepConfig = it,
                asleepTrackingListener = listener,
                notificationTitle = "Sleep Tracking",
                notificationText = "Active",
                notificationIcon = R.drawable.ic_notification,
                notificationClass = SleepActivity::class.java
            )
        }
    }

    fun stopTracking() {
        Asleep.endSleepTracking()
    }
}
```

### Minimal Activity

```kotlin
@AndroidEntryPoint
class SleepActivity : AppCompatActivity() {

    private lateinit var binding: ActivitySleepBinding
    private val viewModel: SleepTrackingViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivitySleepBinding.inflate(layoutInflater)
        setContentView(binding.root)

        // Initialize SDK
        viewModel.initializeSDK(loadUserId())

        // Setup button
        binding.btnTrack.setOnClickListener {
            when (viewModel.trackingState.value) {
                AsleepState.STATE_INITIALIZED -> viewModel.startTracking()
                AsleepState.STATE_TRACKING_STARTED -> viewModel.stopTracking()
                else -> {}
            }
        }

        // Observe state
        lifecycleScope.launch {
            viewModel.trackingState.collect { state ->
                binding.btnTrack.text = when (state) {
                    AsleepState.STATE_INITIALIZED -> "Start"
                    AsleepState.STATE_TRACKING_STARTED -> "Stop"
                    else -> "Loading..."
                }
            }
        }
    }

    private fun loadUserId(): String {
        val prefs = getSharedPreferences("app", MODE_PRIVATE)
        return prefs.getString("user_id", null) ?: UUID.randomUUID().toString().also {
            prefs.edit().putString("user_id", it).apply()
        }
    }
}
```

## State Management Pattern

Define states using sealed classes:

```kotlin
sealed class AsleepState {
    data object STATE_IDLE: AsleepState()
    data object STATE_INITIALIZING : AsleepState()
    data object STATE_INITIALIZED : AsleepState()
    data object STATE_TRACKING_STARTING : AsleepState()
    data object STATE_TRACKING_STARTED : AsleepState()
    data object STATE_TRACKING_STOPPING : AsleepState()
    data class STATE_ERROR(val errorCode: AsleepError) : AsleepState()
}

data class AsleepError(val code: Int, val message: String)
```

## Real-Time Data Access

```kotlin
// In ViewModel
private var lastCheckedSequence = 0

private val trackingListener = object : Asleep.AsleepTrackingListener {
    override fun onPerform(sequence: Int) {
        _sequence.postValue(sequence)

        // Check after sequence 10, then every 10 sequences
        if (sequence > 10 && sequence - lastCheckedSequence >= 10) {
            getCurrentSleepData(sequence)
        }
    }
    // ... other callbacks
}

private fun getCurrentSleepData(sequence: Int) {
    Asleep.getCurrentSleepData(
        asleepSleepDataListener = object : Asleep.AsleepSleepDataListener {
            override fun onSleepDataReceived(session: Session) {
                lastCheckedSequence = sequence

                // Extract current data
                val currentSleepStage = session.sleepStages?.lastOrNull()
                val currentSnoringStage = session.snoringStages?.lastOrNull()

                _currentSleepStage.postValue(currentSleepStage)

                Log.d("Sleep", "Current stage: $currentSleepStage")
                Log.d("Sleep", "Snoring: $currentSnoringStage")
            }

            override fun onFail(errorCode: Int, detail: String) {
                Log.e("Sleep", "Failed to get current data: $errorCode")
            }
        }
    )
}
```
