# Android Architecture Patterns for Asleep SDK

This document contains key architecture patterns from the Asleep Android sample app.

## 1. Application Setup with Hilt

```kotlin
@HiltAndroidApp
class SampleApplication : Application() {
    companion object {
        private lateinit var instance: SampleApplication
        val ACTION_AUTO_TRACKING: String by lazy {
            instance.packageName + ".ACTION_AUTO_TRACKING"
        }
    }

    override fun onCreate() {
        super.onCreate()
        instance = this
    }
}
```

## 2. State Management with Sealed Classes

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
```

## 3. ViewModel with Asleep Integration

```kotlin
@HiltViewModel
class AsleepViewModel @Inject constructor(
    @ApplicationContext private val applicationContext: Application
) : ViewModel() {

    // State management
    private var _asleepState = MutableStateFlow<AsleepState>(AsleepState.STATE_IDLE)
    val asleepState: StateFlow<AsleepState> get() = _asleepState

    // User and session data
    private var _asleepUserId = MutableLiveData<String?>(null)
    val asleepUserId: LiveData<String?> get() = _asleepUserId

    private var _sessionId = MutableStateFlow<String?>(null)
    val sessionId: StateFlow<String?> get() = _sessionId

    // Initialize SDK
    fun initAsleepConfig() {
        if (_asleepState.value != AsleepState.STATE_IDLE) return

        _asleepState.value = AsleepState.STATE_INITIALIZING
        val storedUserId = PreferenceHelper.getAsleepUserId(applicationContext)

        Asleep.initAsleepConfig(
            context = applicationContext,
            apiKey = Constants.ASLEEP_API_KEY,
            userId = storedUserId,
            baseUrl = Constants.BASE_URL,
            callbackUrl = Constants.CALLBACK_URL,
            service = Constants.SERVICE_NAME,
            asleepConfigListener = object : Asleep.AsleepConfigListener {
                override fun onFail(errorCode: Int, detail: String) {
                    _asleepErrorCode.value = AsleepError(errorCode, detail)
                    _asleepState.value = AsleepState.STATE_ERROR(AsleepError(errorCode, detail))
                }

                override fun onSuccess(userId: String?, asleepConfig: AsleepConfig?) {
                    _asleepConfig.value = asleepConfig
                    _asleepUserId.value = userId
                    userId?.let { PreferenceHelper.putAsleepUserId(applicationContext, it) }
                    _asleepState.value = AsleepState.STATE_INITIALIZED
                }
            }
        )
    }

    // Tracking lifecycle
    private val asleepTrackingListener = object: Asleep.AsleepTrackingListener {
        override fun onStart(sessionId: String) {
            _asleepState.value = AsleepState.STATE_TRACKING_STARTED
        }

        override fun onPerform(sequence: Int) {
            _sequence.postValue(sequence)
            if (sequence > 10 && (sequence % 10 == 1 || sequence - (_analyzedSeq ?: 0) > 10)) {
                getCurrentSleepData(sequence)
            }
        }

        override fun onFinish(sessionId: String?) {
            _sessionId.value = sessionId
            if (asleepState.value is AsleepState.STATE_ERROR) {
                // Exit due to Error
            } else {
                // Successful Finish
                _asleepState.value = AsleepState.STATE_IDLE
            }
        }

        override fun onFail(errorCode: Int, detail: String) {
            handleErrorOrWarning(AsleepError(errorCode, detail))
        }
    }

    fun beginSleepTracking() {
        if (_asleepState.value == AsleepState.STATE_INITIALIZED) {
            _asleepState.value = AsleepState.STATE_TRACKING_STARTING
            _asleepConfig.value?.let {
                Asleep.beginSleepTracking(
                    asleepConfig = it,
                    asleepTrackingListener = asleepTrackingListener,
                    notificationTitle = applicationContext.getString(R.string.app_name),
                    notificationText = "",
                    notificationIcon = R.mipmap.ic_app,
                    notificationClass = MainActivity::class.java
                )
            }
            PreferenceHelper.saveStartTrackingTime(applicationContext, System.currentTimeMillis())
        }
    }

    fun endSleepTracking() {
        if (Asleep.isSleepTrackingAlive(applicationContext)) {
            _asleepState.value = AsleepState.STATE_TRACKING_STOPPING
            Asleep.endSleepTracking()
        }
    }

    fun connectSleepTracking() {
        Asleep.connectSleepTracking(asleepTrackingListener)
        _asleepUserId.value = PreferenceHelper.getAsleepUserId(applicationContext)
        _asleepState.value = AsleepState.STATE_TRACKING_STARTED
    }

    // Error handling
    fun handleErrorOrWarning(asleepError: AsleepError) {
        val code = asleepError.code
        val message = asleepError.message
        if (isWarning(code)) {
            // Log warning, continue tracking
            _warningMessage.postValue("$existingMessage\n${getCurrentTime()} $code - $message")
        } else {
            // Critical error, stop tracking
            _asleepErrorCode.postValue(asleepError)
            _asleepState.value = AsleepState.STATE_ERROR(asleepError)
        }
    }
}
```

## 4. Activity State Handling

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding
    private lateinit var permissionManager: PermissionManager
    private val asleepViewModel: AsleepViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        permissionManager = PermissionManager(this)
        setPermissionObserver()
        permissionManager.checkAllPermissions()

        // State management with coroutines
        lifecycleScope.launch {
            asleepViewModel.asleepState.collect { state ->
                when (state) {
                    AsleepState.STATE_IDLE -> {
                        checkRunningService()
                    }
                    AsleepState.STATE_INITIALIZING -> {
                        binding.btnControlTracking.text = "No user id"
                        binding.btnControlTracking.isEnabled = false
                    }
                    AsleepState.STATE_INITIALIZED -> {
                        binding.btnControlTracking.apply {
                            isEnabled = true
                            text = getString(R.string.button_text_start_tracking)
                            setOnClickListener {
                                if (permissionManager.allPermissionsGranted.value == true) {
                                    asleepViewModel.beginSleepTracking()
                                } else {
                                    permissionManager.checkAndRequestPermissions()
                                }
                            }
                        }
                    }
                    AsleepState.STATE_TRACKING_STARTED -> {
                        binding.btnControlTracking.apply {
                            isEnabled = true
                            text = getString(R.string.button_text_stop_tracking)
                            setOnClickListener {
                                if (asleepViewModel.isEnoughTrackingTime()) {
                                    asleepViewModel.endSleepTracking()
                                } else {
                                    showInsufficientTimeDialog()
                                }
                            }
                        }
                    }
                    is AsleepState.STATE_ERROR -> {
                        binding.btnControlTracking.isEnabled = false
                        showErrorDialog(supportFragmentManager)
                    }
                }
            }
        }
    }

    private fun checkRunningService() {
        val isRunningService = Asleep.isSleepTrackingAlive(applicationContext)
        if (isRunningService) {
            asleepViewModel.connectSleepTracking()
        } else {
            asleepViewModel.initAsleepConfig()
        }
    }
}
```

## 5. Permission Management

```kotlin
class PermissionManager(private val activity: AppCompatActivity) {
    private val context = activity.applicationContext

    private val batteryOptimizationLauncher =
        activity.registerForActivityResult(ActivityResultContracts.StartActivityForResult()) {
            checkAndRequestPermissions()
        }

    private val micPermissionLauncher =
        activity.registerForActivityResult(ActivityResultContracts.RequestPermission()) {
            checkAndRequestPermissions()
        }

    private val notificationPermissionLauncher =
        activity.registerForActivityResult(ActivityResultContracts.RequestPermission()) {
            checkAndRequestPermissions()
        }

    private val _allPermissionsGranted = MutableLiveData(false)
    val allPermissionsGranted: LiveData<Boolean> = _allPermissionsGranted

    fun checkAndRequestPermissions() {
        when {
            !isBatteryOptimizationIgnored() -> {
                val intent = Intent().apply {
                    action = Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS
                    data = Uri.parse("package:${context.packageName}")
                }
                batteryOptimizationLauncher.launch(intent)
            }

            context.checkSelfPermission(android.Manifest.permission.RECORD_AUDIO)
                != android.content.pm.PackageManager.PERMISSION_GRANTED -> {
                if (shouldShowRequestPermissionRationale(activity,
                    android.Manifest.permission.RECORD_AUDIO)) {
                    showPermissionDialog()
                } else {
                    micPermissionLauncher.launch(android.Manifest.permission.RECORD_AUDIO)
                }
            }

            hasNotificationPermission().not() -> {
                if (Build.VERSION.SDK_INT < Build.VERSION_CODES.TIRAMISU ||
                    shouldShowRequestPermissionRationale(activity,
                        android.Manifest.permission.POST_NOTIFICATIONS)) {
                    showPermissionDialog()
                } else {
                    notificationPermissionLauncher.launch(
                        android.Manifest.permission.POST_NOTIFICATIONS)
                }
            }

            else -> {
                checkAllPermissions()
            }
        }
    }

    fun checkAllPermissions() {
        _batteryOptimized.value = isBatteryOptimizationIgnored()
        _micPermission.value = context.checkSelfPermission(
            android.Manifest.permission.RECORD_AUDIO
        ) == android.content.pm.PackageManager.PERMISSION_GRANTED
        _notificationPermission.value = hasNotificationPermission()

        _allPermissionsGranted.value =
            (_batteryOptimized.value == true) &&
            (_micPermission.value == true) &&
            (_notificationPermission.value == true)
    }

    private fun isBatteryOptimizationIgnored(): Boolean {
        val powerManager = context.getSystemService(Context.POWER_SERVICE) as PowerManager
        val packageName = context.packageName
        return powerManager.isIgnoringBatteryOptimizations(packageName)
    }

    private fun hasNotificationPermission(): Boolean {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            context.checkSelfPermission(android.Manifest.permission.POST_NOTIFICATIONS)
                == android.content.pm.PackageManager.PERMISSION_GRANTED
        } else {
            NotificationManagerCompat.from(context).areNotificationsEnabled()
        }
    }
}
```

## 6. Error Handling Utilities

```kotlin
data class AsleepError(val code: Int, val message: String)

internal fun isWarning(errorCode: Int): Boolean {
    return errorCode in setOf(
        AsleepErrorCode.ERR_AUDIO_SILENCED,
        AsleepErrorCode.ERR_AUDIO_UNSILENCED,
        AsleepErrorCode.ERR_UPLOAD_FAILED,
    )
}

internal fun getDebugMessage(errorCode: AsleepError): String {
    if (isNetworkError(errorCode.message)) {
        return "Please check your network connection."
    }
    if (isMethodFormatInvalid(errorCode.message)) {
        return "Please check the method format, including argument values and types."
    }

    return when (errorCode.code) {
        AsleepErrorCode.ERR_MIC_PERMISSION ->
            "The app does not have microphone access permission."
        AsleepErrorCode.ERR_AUDIO ->
            "Another app is using the microphone, or there is an issue with the microphone settings."
        AsleepErrorCode.ERR_INVALID_URL ->
            "Please check the URL format."
        AsleepErrorCode.ERR_COMMON_EXPIRED ->
            "The API rate limit has been exceeded, or the plan has expired."
        AsleepErrorCode.ERR_UPLOAD_FORBIDDEN ->
            "initAsleepConfig() was performed elsewhere with the same ID during the tracking."
        AsleepErrorCode.ERR_UPLOAD_NOT_FOUND, AsleepErrorCode.ERR_CLOSE_NOT_FOUND ->
            "The session has already ended."
        else -> ""
    }
}
```
