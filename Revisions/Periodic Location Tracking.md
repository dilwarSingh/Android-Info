# Periodic Location Tracking — Last-Minute Revision

## Core Decision Rule

- **WorkManager** → best-effort, deferrable, timing-flexible (~30+ min)
- **Foreground Service (FGS)** → user-visible, ongoing, notification-required
- **Geofencing API** → zone enter/exit detection; always prefer over polling
- **Passive location** → opportunistic, minimal battery cost, no delivery guarantee
- **No silent, exact, indefinite 30-min background polling API exists on modern Android.**

---

## Use Case → Best Choice

| Scenario | Choice |
|---|---|
| Delivery / ride-hailing / field worker | **Foreground Service** |
| Fitness route recording | **Foreground Service** |
| Geofence enter/exit | **Geofencing API** |
| Check-in / attendance / audit log | **WorkManager** |
| Analytics / coarse snapshot | **WorkManager** or passive location |
| "Silent exact every 30 min forever" | Not achievable on Android |

---

## Option 1: WorkManager (Best-Effort Periodic)

**Guarantees:**
- Persistent scheduling; survives process death and reboot
- Integrates with Doze / App Standby / battery optimizations
- **Does not** survive user **force-stop** — resumes only after relaunch

**Does NOT guarantee:**
- Exact 30-min cadence — expect **30+ minutes** (Doze, App Standby bucket, OEM limits)
- Minimum repeat interval: **15 minutes**
- Guaranteed fresh location fix in background

**Best for:** check-in logs, telemetry, coarse snapshots, deferred uploads

```kotlin
val request = PeriodicWorkRequestBuilder<LocationWorker>(30, TimeUnit.MINUTES)
    .setConstraints(Constraints.Builder().setRequiresBatteryNotLow(true).build())
    .build()
WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "periodic_location_snapshot", ExistingPeriodicWorkPolicy.UPDATE, request)
```

**Inside the Worker:** use `getCurrentLocation(Priority.PRIORITY_BALANCED_POWER_ACCURACY, token)` — `lastLocation` may be null/stale. No-fix is a normal outcome; return `Result.success()` regardless.

**Permissions required:**
```xml
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
```

---

## Option 2: Foreground Service (User-Visible Tracking)

**Advantages over WorkManager:**
- Stronger system signal → more timely location delivery
- Supports continuous collection with `LocationCallback`
- "While-in-use" classification → `ACCESS_BACKGROUND_LOCATION` **not needed**

**Does NOT guarantee:**
- Immunity from Doze or OEM power management
- Automatic restart after force-stop (`START_STICKY` only helps after system-initiated kill)
- Exact 30-min wall-clock delivery — `intervalMillis` is a **request**, not a promise

```kotlin
val request = LocationRequest.Builder(Priority.PRIORITY_BALANCED_POWER_ACCURACY, 30 * 60 * 1000L)
    .setMinUpdateIntervalMillis(30 * 60 * 1000L)
    .setWaitForAccurateLocation(false)
    .build()
fusedClient.requestLocationUpdates(request, locationCallback, Looper.getMainLooper())
```

- Use `LocationRequest` — **not** `Handler.postDelayed()` loops.

**Permissions + manifest (API 34+):**
```xml
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
<service android:name=".LocationTrackingService"
    android:foregroundServiceType="location"
    android:exported="false" />
```
> Without `FOREGROUND_SERVICE_LOCATION`, calling `startForeground()` with the `location` service type throws a `SecurityException`.

---

## Battery, Reliability & Policy Comparison

| Factor | WorkManager | Foreground Service |
|---|---|---|
| Battery impact | Lower | Higher |
| Timing accuracy | Inexact / best-effort | More timely, still not exact |
| Reboot behavior | Auto-rescheduled | Not restored unless custom logic |
| Survives process death | Yes (except user force-stop) | Sometimes (`START_STICKY`); not after force-stop |
| Notification required | No | Yes (ongoing) |
| Doze impact | Significantly deferred | Better, not fully exempt |
| Play policy risk | Lower | Higher scrutiny |
| Silent background polling | Only if inexact OK | Poor policy fit |

---

## Exact Alarms — Avoid for This

**`AlarmManager.setExactAndAllowWhileIdle()`** is **not** the answer:
- Intended for **user-facing, time-critical events** — not silent polling
- Heavily restricted on modern Android
- Still bound by location permission, background execution, and FGS rules

---

## Geofencing API (Prefer Over Polling)

Use when the real requirement is: *"Did user enter/exit a zone?"*
- **Event-driven**, not timer-driven → less battery
- Subject to platform heuristics; not millisecond-precise
- Use WorkManager only as a **fallback/cleanup layer** around geofence workflow

---

## Passive Location

- Receives updates when **other apps or system** already requested them
- Minimal active battery cost; **no delivery guarantee**
- Good for: lightweight analytics, opportunistic journaling, context awareness

---

## Location Permissions — Full Breakdown

| Permission | API Level | Controls |
|---|---|---|
| `ACCESS_COARSE_LOCATION` | 1+ | Approximate location (~1–3 km, Wi-Fi/cell) |
| `ACCESS_FINE_LOCATION` | 1+ | Precise location (~1–50 m, GPS/GNSS); implicitly includes COARSE |
| `ACCESS_BACKGROUND_LOCATION` | 29+ | **When** app can access location (not accuracy); needed for WorkManager, geofencing, PendingIntent |
| `FOREGROUND_SERVICE` | 28+ | Required for any `startForeground()` call |
| `FOREGROUND_SERVICE_LOCATION` | 34+ | Required to start FGS with `foregroundServiceType="location"`; install-time, not runtime |

**`ACCESS_BACKGROUND_LOCATION` versioning:**
- **API 28 and below:** no separate background permission — foreground permission is enough
- **API 29:** introduced, can be requested alongside foreground perm
- **API 30+:** must be requested **separately**; user directed to Settings
- **API 33+:** further UI friction discouraging background grants
- **Play policy:** requires data safety declaration + justification

**Android 12+ (API 31+) — Approximate vs Precise choice at runtime:**
```kotlin
val hasFine = ContextCompat.checkSelfPermission(context, Manifest.permission.ACCESS_FINE_LOCATION) == PERMISSION_GRANTED
val hasCoarse = ContextCompat.checkSelfPermission(context, Manifest.permission.ACCESS_COARSE_LOCATION) == PERMISSION_GRANTED
// hasCoarse=true, hasFine=false → user chose approximate
```
- App must handle both outcomes. Request `FINE` only when the use case demands it.

**Permission combos by scenario:**

| Use Case | Permissions |
|---|---|
| Map while app open | `ACCESS_FINE_LOCATION` |
| FGS tracking (API 34+) | `FINE` + `FOREGROUND_SERVICE` + `FOREGROUND_SERVICE_LOCATION` |
| FGS tracking (API 28–33) | `FINE` + `FOREGROUND_SERVICE` |
| WorkManager background snapshot | `FINE` (or `COARSE`) + `ACCESS_BACKGROUND_LOCATION` |
| Geofence while app closed | `FINE` + `ACCESS_BACKGROUND_LOCATION` |
| Passive while app closed | `FINE` + `ACCESS_BACKGROUND_LOCATION` (coarse-only works with the fused `PRIORITY_PASSIVE` path; `LocationManager.PASSIVE_PROVIDER` requires FINE) |

---

## Platform Policy Rules (Do Not Skip)

- **Background FGS starts** are increasingly restricted — start from a **user action**.
- **BOOT_COMPLETED** + auto-starting an FGS is allowed but subject to background-start restrictions.
- **Google Play** requires justification for `ACCESS_BACKGROUND_LOCATION`: why it's core, why less invasive alternatives fail, how user is informed.
- Production apps must provide: disclosure screen, stop-tracking control, retention/privacy info, and active-tracking notification.

---

## Hybrid Strategy

```text
User actively tracking       →  Foreground Service
Low-priority / no session    →  WorkManager
Zone detection               →  Geofencing API
Opportunistic low-power      →  Passive location
```

---

## Final Decision Tree

```
Exact wall-clock execution required?
├── YES → No policy-friendly Android solution. Re-evaluate.
└── NO
    ├── User actively aware of tracking?
    │   ├── YES → Foreground Service
    │   └── NO
    │       ├── Zone entry/exit is the real requirement?
    │       │   ├── YES → Geofencing API
    │       │   └── NO  → WorkManager or passive location
    └── Power more important than timing?
        ├── YES → WorkManager / passive
        └── NO  → Foreground Service (if user-visible + policy-appropriate)
```
