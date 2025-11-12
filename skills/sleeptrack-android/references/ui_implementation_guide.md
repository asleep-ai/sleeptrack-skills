# UI Implementation Guide

This guide provides complete UI implementations for Android sleep tracking with both ViewBinding and Jetpack Compose.

## ViewBinding Implementation

### Complete Fragment with ViewBinding

```kotlin
class TrackingFragment : Fragment() {
    private var _binding: FragmentTrackingBinding? = null
    private val binding get() = _binding!!
    private val viewModel: SleepTrackingViewModel by viewModels()

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = FragmentTrackingBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        binding.btnTrack.setOnClickListener {
            viewModel.startTracking()
        }

        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.trackingState.collect { state ->
                when (state) {
                    AsleepState.STATE_TRACKING_STARTED -> {
                        binding.progressIndicator.visibility = View.VISIBLE
                        binding.btnTrack.text = "Stop Tracking"
                    }
                    AsleepState.STATE_INITIALIZED -> {
                        binding.progressIndicator.visibility = View.GONE
                        binding.btnTrack.text = "Start Tracking"
                    }
                    else -> {}
                }
            }
        }
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

### Layout XML (fragment_tracking.xml)

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:padding="16dp">

    <ProgressBar
        android:id="@+id/progressIndicator"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:visibility="gone"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintBottom_toTopOf="@id/btnTrack" />

    <TextView
        android:id="@+id/tvSequence"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Sequence: 0"
        android:textSize="18sp"
        app:layout_constraintTop_toBottomOf="@id/progressIndicator"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintBottom_toTopOf="@id/btnTrack" />

    <Button
        android:id="@+id/btnTrack"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:text="Start Tracking"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

## Jetpack Compose Implementation

### Complete Tracking Screen

```kotlin
@Composable
fun SleepTrackingScreen(
    viewModel: SleepTrackingViewModel = hiltViewModel()
) {
    val trackingState by viewModel.trackingState.collectAsState()
    val sequence by viewModel.sequence.observeAsState(0)

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        when (trackingState) {
            AsleepState.STATE_TRACKING_STARTED -> {
                CircularProgressIndicator()
                Spacer(modifier = Modifier.height(16.dp))
                Text("Tracking in progress")
                Text("Sequence: $sequence")
                Spacer(modifier = Modifier.height(32.dp))
                Button(onClick = { viewModel.stopTracking() }) {
                    Text("Stop Tracking")
                }
            }
            AsleepState.STATE_INITIALIZED -> {
                Button(onClick = { viewModel.startTracking() }) {
                    Text("Start Sleep Tracking")
                }
            }
            is AsleepState.STATE_ERROR -> {
                val error = (trackingState as AsleepState.STATE_ERROR).errorCode
                ErrorDisplay(error = error)
            }
            else -> {
                CircularProgressIndicator()
                Text("Initializing...")
            }
        }
    }
}

@Composable
fun ErrorDisplay(error: AsleepError) {
    Column(
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Icon(
            imageVector = Icons.Default.Warning,
            contentDescription = "Error",
            tint = MaterialTheme.colorScheme.error
        )
        Spacer(modifier = Modifier.height(8.dp))
        Text(
            text = getUserFriendlyMessage(error),
            style = MaterialTheme.typography.bodyLarge,
            textAlign = TextAlign.Center
        )
    }
}
```

### Advanced Compose UI with Sleep Stages

```kotlin
@Composable
fun DetailedTrackingScreen(
    viewModel: SleepTrackingViewModel = hiltViewModel()
) {
    val trackingState by viewModel.trackingState.collectAsState()
    val sequence by viewModel.sequence.observeAsState(0)
    val sessionId by viewModel.sessionId.collectAsState()

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Sleep Tracking") }
            )
        }
    ) { padding ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
                .padding(16.dp)
        ) {
            when (trackingState) {
                AsleepState.STATE_TRACKING_STARTED -> {
                    TrackingActiveContent(
                        sequence = sequence,
                        sessionId = sessionId,
                        onStop = { viewModel.stopTracking() }
                    )
                }
                AsleepState.STATE_INITIALIZED -> {
                    TrackingIdleContent(
                        onStart = { viewModel.startTracking() }
                    )
                }
                else -> {
                    LoadingContent()
                }
            }
        }
    }
}

@Composable
fun TrackingActiveContent(
    sequence: Int,
    sessionId: String?,
    onStop: () -> Unit
) {
    Column(
        modifier = Modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.SpaceBetween
    ) {
        // Status card
        Card(
            modifier = Modifier.fillMaxWidth(),
            colors = CardDefaults.cardColors(
                containerColor = MaterialTheme.colorScheme.primaryContainer
            )
        ) {
            Column(
                modifier = Modifier.padding(16.dp),
                horizontalAlignment = Alignment.CenterHorizontally
            ) {
                Text(
                    text = "Tracking Active",
                    style = MaterialTheme.typography.headlineSmall
                )
                Spacer(modifier = Modifier.height(8.dp))
                Text(
                    text = "Session: ${sessionId?.take(8)}...",
                    style = MaterialTheme.typography.bodyMedium
                )
            }
        }

        Spacer(modifier = Modifier.height(24.dp))

        // Progress indicator
        Box(
            contentAlignment = Alignment.Center
        ) {
            CircularProgressIndicator(
                modifier = Modifier.size(120.dp),
                strokeWidth = 8.dp
            )
            Column(
                horizontalAlignment = Alignment.CenterHorizontally
            ) {
                Text(
                    text = "$sequence",
                    style = MaterialTheme.typography.headlineLarge
                )
                Text(
                    text = "sequences",
                    style = MaterialTheme.typography.bodySmall
                )
            }
        }

        Spacer(modifier = Modifier.height(24.dp))

        // Duration estimate
        val minutes = sequence / 2
        Text(
            text = "Approximately $minutes minutes",
            style = MaterialTheme.typography.bodyLarge
        )

        Spacer(modifier = Modifier.weight(1f))

        // Stop button
        Button(
            onClick = onStop,
            modifier = Modifier.fillMaxWidth(),
            colors = ButtonDefaults.buttonColors(
                containerColor = MaterialTheme.colorScheme.error
            )
        ) {
            Text("Stop Tracking")
        }
    }
}

@Composable
fun TrackingIdleContent(onStart: () -> Unit) {
    Column(
        modifier = Modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(
            imageVector = Icons.Default.NightsStay,
            contentDescription = "Sleep",
            modifier = Modifier.size(120.dp),
            tint = MaterialTheme.colorScheme.primary
        )
        Spacer(modifier = Modifier.height(24.dp))
        Text(
            text = "Ready to Track Sleep",
            style = MaterialTheme.typography.headlineMedium
        )
        Spacer(modifier = Modifier.height(8.dp))
        Text(
            text = "Make sure you're in a quiet environment",
            style = MaterialTheme.typography.bodyMedium,
            textAlign = TextAlign.Center
        )
        Spacer(modifier = Modifier.height(32.dp))
        Button(
            onClick = onStart,
            modifier = Modifier.fillMaxWidth(0.8f)
        ) {
            Text("Start Tracking")
        }
    }
}

@Composable
fun LoadingContent() {
    Box(
        modifier = Modifier.fillMaxSize(),
        contentAlignment = Alignment.Center
    ) {
        Column(
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            CircularProgressIndicator()
            Spacer(modifier = Modifier.height(16.dp))
            Text("Initializing SDK...")
        }
    }
}
```

## Material Design 3 Theming

```kotlin
// Theme setup for Compose
@Composable
fun SleepTrackingTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colorScheme = if (darkTheme) {
        darkColorScheme(
            primary = Color(0xFF6200EE),
            secondary = Color(0xFF03DAC6),
            error = Color(0xFFCF6679)
        )
    } else {
        lightColorScheme(
            primary = Color(0xFF6200EE),
            secondary = Color(0xFF03DAC6),
            error = Color(0xFFB00020)
        )
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,
        content = content
    )
}
```

## Custom Views (XML)

### CircularProgressView

```kotlin
class CircularProgressView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {

    private var progress = 0
    private val paint = Paint().apply {
        style = Paint.Style.STROKE
        strokeWidth = 20f
        color = Color.BLUE
        isAntiAlias = true
    }

    fun setProgress(value: Int) {
        progress = value
        invalidate()
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        val centerX = width / 2f
        val centerY = height / 2f
        val radius = (min(width, height) / 2f) - 20f

        // Draw circle
        canvas.drawCircle(centerX, centerY, radius, paint)

        // Draw progress arc
        val rectF = RectF(
            centerX - radius,
            centerY - radius,
            centerX + radius,
            centerY + radius
        )
        paint.style = Paint.Style.FILL
        canvas.drawArc(rectF, -90f, (progress * 360f / 100f), true, paint)
    }
}
```

## Permission UI Patterns

### Permission Request Dialog (Compose)

```kotlin
@Composable
fun PermissionRequestDialog(
    permissionName: String,
    rationale: String,
    onConfirm: () -> Unit,
    onDismiss: () -> Unit
) {
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("Permission Required") },
        text = { Text(rationale) },
        confirmButton = {
            TextButton(onClick = onConfirm) {
                Text("Grant Permission")
            }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) {
                Text("Cancel")
            }
        }
    )
}
```

### Using with Accompanist Permissions

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun PermissionScreen(
    onPermissionsGranted: () -> Unit
) {
    val micPermissionState = rememberPermissionState(
        Manifest.permission.RECORD_AUDIO
    )

    when {
        micPermissionState.status.isGranted -> {
            onPermissionsGranted()
        }
        micPermissionState.status.shouldShowRationale -> {
            PermissionRequestDialog(
                permissionName = "Microphone",
                rationale = "Microphone access is required to record sleep sounds",
                onConfirm = { micPermissionState.launchPermissionRequest() },
                onDismiss = { /* Handle dismissal */ }
            )
        }
        else -> {
            Button(onClick = { micPermissionState.launchPermissionRequest() }) {
                Text("Grant Microphone Permission")
            }
        }
    }
}
```

## Navigation Integration

### Jetpack Navigation with Compose

```kotlin
@Composable
fun SleepTrackingNavHost(
    navController: NavHostController = rememberNavController()
) {
    NavHost(
        navController = navController,
        startDestination = "tracking"
    ) {
        composable("tracking") {
            SleepTrackingScreen(
                onSessionComplete = { sessionId ->
                    navController.navigate("results/$sessionId")
                }
            )
        }
        composable(
            "results/{sessionId}",
            arguments = listOf(navArgument("sessionId") { type = NavType.StringType })
        ) { backStackEntry ->
            ResultsScreen(
                sessionId = backStackEntry.arguments?.getString("sessionId")
            )
        }
    }
}
```

## Best Practices

1. **ViewBinding**: Always nullify binding in `onDestroyView()` for fragments
2. **Compose**: Use `collectAsState()` for StateFlow and `observeAsState()` for LiveData
3. **State Management**: Handle all possible states in UI, including loading and error states
4. **Accessibility**: Add content descriptions for all interactive elements
5. **Dark Mode**: Support both light and dark themes
6. **Orientation**: Handle configuration changes with ViewModels
7. **Touch Targets**: Ensure buttons are at least 48dp in size
