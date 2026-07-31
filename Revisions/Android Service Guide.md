# Android Service Guide — Last-Minute Revision Cheat Sheet

## Overview: Service Types at a Glance

| Type | Notification | Survives Kill | Survives Reboot | Battery-Aware | Best For |
|---|---|---|---|---|---|
| **Background Service** | No | No | No | No | Short, immediate tasks (app in foreground) |
| **Foreground Service** | Required | Yes | No (needs BroadcastReceiver) | No | Long user-aware tasks |
| **WorkManager** | No | Yes | Yes | Yes (Doze-aware) | Deferrable, guaranteed tasks |

- **API 26+**: Background services killed shortly after app leaves foreground.
- **API 34+**: Foreground services must declare `foregroundServiceType` + matching permission.

---

## Service Lifecycle Callbacks

- **`onCreate()`** — First creation only.
- **`onStartCommand()`** — Called each time via `startService()`. Returns restart policy.
- **`onBind()`** / **`onUnbind()`** / **`onRebind()`** — Binding lifecycle.
- **`onDestroy()`** — Final cleanup.

### `onStartCommand()` Return Values

| Value | Behavior |
|---|---|
| `START_NOT_STICKY` | Don't recreate if killed. Use for restartable tasks. |
| `START_STICKY` | Recreate on kill, no intent redelivery. Use for long-running (e.g., music). |
| `START_REDELIVER_INTENT` | Recreate AND redeliver last intent. Use for must-complete tasks (e.g., upload). |

---

## Background Service

- Runs silently, no notification.
- **Killed aggressively on API 26+** when app is in background.
- Use only for **short, immediate tasks** while app is in foreground.
- **Modern alternative**: Kotlin Coroutines in ViewModel/lifecycle scope.

```kotlin
override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
    serviceScope.launch { doWork(); stopSelf() }
    return START_NOT_STICKY
}
```

**Manifest:** `<service android:name=".MyService" />`

---

## Foreground Service

- Must call `startForeground(id, notification)` — shows a **persistent notification**.
- Protected from OS kill; survives app backgrounding.
- Use for: audio playback, navigation, ongoing downloads, real-time tracking.

### Foreground Service Types (API 34+ — Mandatory)

| Type | Use Case | Required Permission |
|---|---|---|
| `camera` | Background camera | `FOREGROUND_SERVICE_CAMERA` |
| `connectedDevice` | Bluetooth/USB | `FOREGROUND_SERVICE_CONNECTED_DEVICE` |
| `dataSync` | Data transfer | `FOREGROUND_SERVICE_DATA_SYNC` |
| `health` | Fitness sensors | `FOREGROUND_SERVICE_HEALTH` |
| `location` | Real-time location | `FOREGROUND_SERVICE_LOCATION` |
| `mediaPlayback` | Audio/video | `FOREGROUND_SERVICE_MEDIA_PLAYBACK` |
| `mediaProjection` | Screen recording | `FOREGROUND_SERVICE_MEDIA_PROJECTION` |
| `microphone` | Audio recording | `FOREGROUND_SERVICE_MICROPHONE` |
| `phoneCall` | Ongoing call | `FOREGROUND_SERVICE_PHONE_CALL` |
| `remoteMessaging` | Messaging on another device | `FOREGROUND_SERVICE_REMOTE_MESSAGING` |
| `mediaProcessing` | Transcoding (API 35+, 6-hr limit) | `FOREGROUND_SERVICE_MEDIA_PROCESSING` |
| `shortService` | Short critical tasks (~3 min) | None |
| `specialUse` | No other type fits (Play Store justification needed) | `FOREGROUND_SERVICE_SPECIAL_USE` |
| `systemExempted` | Allowlisted cases (device admin, VPN, etc.) | `FOREGROUND_SERVICE_SYSTEM_EXEMPTED` |

- Multiple types combinable: `android:foregroundServiceType="location|microphone"`
- **Missing type on API 34+** → throws `MissingForegroundServiceTypeException`.

### API 35 Timeout Restrictions

| Type | Timeout | Action Required |
|---|---|---|
| `dataSync` | 6 hrs / 24-hr period | Call `stopSelf()` in `onTimeout(int, int)` (API 35+) |
| `mediaProcessing` | 6 hrs | Call `stopSelf()` in `onTimeout(int, int)` (API 35+) |
| `shortService` | ~3 min (non-resettable) | Call `stopSelf()` in `onTimeout()` (API 34+) |

```kotlin
override fun onTimeout(startId: Int, fgsType: Int) {
    stopSelf() // Must call within seconds or ANR is triggered
}
```

### Starting a Foreground Service

```kotlin
// From Activity/Context
ContextCompat.startForegroundService(this, Intent(this, MyForegroundService::class.java))
```

**Manifest (API 34+ example):**
```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

<service
    android:name=".MusicPlayerService"
    android:foregroundServiceType="mediaPlayback" />
```

---

## Bound Service

- Components bind via `bindService()` and interact through an **IBinder** interface.
- **Lives only while at least one client is bound**; destroyed automatically on last unbind.
- Use for: two-way communication, IPC (AIDL), component-scoped service lifetime.

```kotlin
inner class MyBinder : Binder() {
    fun getService(): MyService = this@MyService
}
override fun onBind(intent: Intent?): IBinder = binder
```

**Binding from Activity:**
```kotlin
bindService(intent, connection, Context.BIND_AUTO_CREATE)   // onStart()
unbindService(connection)                                    // onStop()
```

### Started + Bound (Hybrid)

- `startService()` keeps alive even when unbound.
- `bindService()` allows interaction.
- Destroyed only when **both stopped AND unbound**.

---

## WorkManager

**Recommended default** for all deferrable background work.

- Guaranteed execution even after app kill or device reboot.
- Supports **constraints** (network, charging, battery), **chaining**, **retry**, and **Doze-aware** scheduling.
- Internally uses `JobScheduler` (API 23+) or a built-in scheduler + `AlarmManager` on older APIs.

### Key Worker Result Values

| Value | Meaning |
|---|---|
| `Result.success()` | Work completed. |
| `Result.failure()` | Work failed, no retry. |
| `Result.retry()` | Retry using backoff policy. |

```kotlin
class MyWorker(ctx: Context, params: WorkerParameters) : CoroutineWorker(ctx, params) {
    override suspend fun doWork(): Result {
        return try { doTask(); Result.success() }
        catch (e: Exception) { if (runAttemptCount < 3) Result.retry() else Result.failure() }
    }
}
```

**Enqueue a periodic request with constraints:**
```kotlin
val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.UNMETERED)
    .setRequiresCharging(true)
    .build()

WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "unique_work_name",
    ExistingPeriodicWorkPolicy.KEEP,
    PeriodicWorkRequestBuilder<MyWorker>(1, TimeUnit.HOURS)
        .setConstraints(constraints)
        .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 15, TimeUnit.MINUTES)
        .build()
)
```

### Work Chaining

```kotlin
// Sequential: compress → upload → notify
WorkManager.getInstance(context)
    .beginWith(compressWork)
    .then(uploadWork)
    .then(notifyWork)
    .enqueue()

// Parallel then sequential: (c1 || c2) → upload → notify
WorkManager.getInstance(context)
    .beginWith(listOf(compress1, compress2))
    .then(uploadWork).then(notifyWork).enqueue()
```

### Expedited Work (API 31+)

- For **urgent, short tasks** that must start immediately but don't need a persistent notification.
- Falls back to a foreground service internally on older APIs.

```kotlin
val work = OneTimeWorkRequestBuilder<UrgentWorker>()
    .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
    .build()
```

### User-Initiated Data Transfer (API 34+)

- For **long user-triggered transfers** (e.g., large file download). Distinct from Expedited Work.
- **No Jetpack library, including WorkManager, supports UIDT jobs** — must use `JobScheduler` directly via `JobInfo.Builder(...).setUserInitiated(true)`.
- Requires `android.permission.RUN_USER_INITIATED_JOBS` permission.
- The `JobService` must call `setNotification()` to display a notification so the user is aware.
- Must be scheduled while the app is **in the foreground**.

```kotlin
val jobInfo = JobInfo.Builder(JOB_ID, ComponentName(context, MyTransferJobService::class.java))
    .setUserInitiated(true)
    .setRequiredNetworkType(JobInfo.NETWORK_TYPE_ANY)
    .setEstimatedNetworkBytes(500 * 1024 * 1024L, 0)
    .build()
jobScheduler.schedule(jobInfo)
```

**Gradle (WorkManager, for the rest of this guide's APIs):**
```kotlin
implementation("androidx.work:work-runtime-ktx:2.11.2")
```

---

## IntentService (Deprecated — API 30+)

- Was a single-threaded, auto-stopping `Service` subclass.
- Subject to API 26+ background execution limits — unreliable.

| Old | Modern Replacement |
|---|---|
| `IntentService` for short tasks | Kotlin Coroutines in ViewModel |
| `IntentService` for queued work | `WorkManager` + `OneTimeWorkRequest` |
| `IntentService` for immediate background | `JobIntentService` (also deprecated) → use `WorkManager` |

---

## JobScheduler (System API — API 21+)

- Low-level scheduler; WorkManager wraps it internally.
- Use directly only for **system apps**, **libraries**, or when WorkManager lacks a needed feature.
- **For app developers: prefer WorkManager.**

```kotlin
val jobInfo = JobInfo.Builder(JOB_ID, ComponentName(this, MyJobService::class.java))
    .setRequiredNetworkType(JobInfo.NETWORK_TYPE_UNMETERED)
    .setRequiresCharging(true)
    .setPeriodic(15 * 60 * 1000L)   // 15-min minimum
    .setPersisted(true)             // Requires RECEIVE_BOOT_COMPLETED
    .build()
jobScheduler.schedule(jobInfo)
```

---

## Decision Flow

```
Task immediate AND short-lived (< ~10 sec)?
├── YES → Background Service (or better: Coroutine/Thread in ViewModel)
└── NO
    └── User needs to see it happening (audio, GPS, download in progress)?
        ├── YES → Foreground Service (with correct type + notification)
        └── NO
            └── Two-way communication with a component needed?
                ├── YES → Bound Service
                └── NO
                    └── Can it be deferred / needs guaranteed completion?
                        ├── YES → WorkManager ✅ (default choice)
                        └── NO → Foreground Service (with proper type)
```

---

## Location Tracking: WorkManager vs Foreground Service

| Scenario | Best Choice |
|---|---|
| Delivery driver, live GPS | Foreground Service |
| Fitness workout route | Foreground Service |
| Passive check-in every 30 min | WorkManager |
| Geo-fencing trigger | WorkManager (or Geofencing API) |
| Analytics / approximate location | WorkManager |

### WorkManager Caveats for Location

- **Minimum interval: 15 minutes** for `PeriodicWorkRequest`.
- Actual execution **not exact** — delayed by Doze, battery optimization, system batching.
- Use `fusedClient.lastLocation` (may be null) or `getCurrentLocation()` (available since play-services-location 17.0.0).
- Requires `ACCESS_BACKGROUND_LOCATION` on API 29+.

### Foreground Service Caveats for Location

- Persistent notification always visible — user can kill it.
- Higher battery drain — service stays alive.
- `LocationRequest` interval is a **hint**, not a hard guarantee.
- Does **not survive reboot** — must use `BroadcastReceiver` for `BOOT_COMPLETED`.

**Restart on reboot:**
```kotlin
class BootReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == Intent.ACTION_BOOT_COMPLETED)
            ContextCompat.startForegroundService(context, Intent(context, LocationTrackingService::class.java))
    }
}
```

**Location Foreground Service manifest (API 34+):**
```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
<service android:name=".LocationTrackingService"
    android:foregroundServiceType="location"
    android:exported="false" />
```

### Production Hybrid Pattern

```
App in foreground / user active  →  Foreground Service (precise)
App closed / idle                →  WorkManager (battery-friendly, survives reboot)
```

Use `ProcessLifecycleOwner` or `ActivityLifecycleCallbacks` to switch between them.

---

## Battery & Reliability Comparison (Location)

| Factor | WorkManager | Foreground Service |
|---|---|---|
| Battery Impact | Low (Doze-aware) | High (always running) |
| Timing Accuracy | Approximate (±15 min) | Near-exact (hint-based) |
| Survives Reboot | Yes (auto-reschedules) | No (needs `BootReceiver`) |
| User Notification | Not required | Required (persistent) |
| Doze Behavior | May be delayed | Exempt |
| Play Store Policy Friendly | Yes | Needs justification |
| Background Location Permission | Required (API 29+) | Not required for foreground starts (while-in-use); a background/boot start of a `location` FGS still needs `ACCESS_BACKGROUND_LOCATION` |
| API 34+ Requirement | No extra changes | `foregroundServiceType` + permission |

---

## Quick Scenario Reference

| Scenario | Use |
|---|---|
| Music playback (screen off) | Foreground Service (`mediaPlayback`) |
| Upload files on Wi-Fi overnight | WorkManager |
| Quick DB update on app launch | Coroutine in ViewModel |
| Real-time GPS run tracking | Foreground Service (`location`) |
| Daily background news sync | WorkManager |
| Compress + upload videos | WorkManager (chained) |
| Bluetooth device interaction | Foreground Service (`connectedDevice`) |
| Screen recording | Foreground Service (`mediaProjection`) |
| Urgent one-time sync | WorkManager + `setExpedited()` |
| Timer reporting back to UI | Bound Service + Foreground Service |

---

## Google's Recommendations (2025–2026+)

- **WorkManager is the default** for almost all background tasks.
- Use **Foreground Service** only when user initiates and is aware of the operation.
- Avoid plain **Background Services** on API 26+ — OS will kill them.
- **API 34+**: Always declare `foregroundServiceType` + corresponding permission.
- **API 35+**: Handle `onTimeout()` for `shortService` (API 34+) and `onTimeout(int, int)` for `dataSync`, `mediaProcessing`.
- **`health` FGS type on API 36+ (Android 16)**: `BODY_SENSORS` only satisfies the runtime prerequisite on API 35 and lower; on API 36+ use granular Health Connect permissions (`READ_HEART_RATE`, `READ_SKIN_TEMPERATURE`, `READ_OXYGEN_SATURATION`) or declare `HIGH_SAMPLING_RATE_SENSORS`.
- Always request `ACCESS_BACKGROUND_LOCATION` explicitly (separate dialog on API 30+).
- Consider the **Geofencing API** over periodic location polling where appropriate.
