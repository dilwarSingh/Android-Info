# Android Permissions Guide

Concise reference for Android's permission model: types, declaring, requesting at runtime, and the version-by-version changes that affect what you must ask for and when.

---

## Table of Contents

1. [Permission Types](#1-permission-types)
2. [Declaring Permissions](#2-declaring-permissions)
3. [Requesting Runtime Permissions](#3-requesting-runtime-permissions)
4. [Rationale & Denial Handling](#4-rationale--denial-handling)
5. [Location Permissions](#5-location-permissions)
6. [Storage & Media Permissions](#6-storage--media-permissions)
7. [Notifications Permission (API 33+)](#7-notifications-permission-api-33)
8. [Bluetooth Permissions (API 31+)](#8-bluetooth-permissions-api-31)
9. [Special Permissions](#9-special-permissions)
10. [Permission Changes by Android Version](#10-permission-changes-by-android-version)
11. [Testing & Debugging](#11-testing--debugging)
12. [Best Practices Checklist](#12-best-practices-checklist)
13. [Further Reading](#13-further-reading)

---

## 1. Permission Types

| Type | Protection Level | Granted | Examples |
|---|---|---|---|
| **Normal** | `normal` | Automatically at install | `INTERNET`, `ACCESS_NETWORK_STATE`, `VIBRATE` |
| **Signature** | `signature` | Only if app is signed with the same cert as the permission's definer | Privileged system/OEM permissions |
| **Runtime (dangerous)** | `dangerous` | User prompted at runtime (API 23+) | `CAMERA`, `ACCESS_FINE_LOCATION`, `RECORD_AUDIO`, `READ_CONTACTS` |
| **Special** | `appop` | User grants via **Settings → Apps → Special app access** | `SYSTEM_ALERT_WINDOW`, `SCHEDULE_EXACT_ALARM`, `MANAGE_EXTERNAL_STORAGE` |

- **Install-time** = Normal + Signature. Devices on API ≤22 auto-grant *all* declared permissions (runtime prompts didn't exist yet).
- **Permission groups** (e.g. "Location") only affect how many dialogs the system batches together — don't assume grouping semantics or that a grouping is stable across OS versions.
- Full list: [`Manifest.permission` reference](https://developer.android.com/reference/android/Manifest.permission).

---

## 2. Declaring Permissions

```xml
<manifest>
    <uses-permission android:name="android.permission.CAMERA" />

    <!-- Only needed up to API 28 -->
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
        android:maxSdkVersion="28" />

    <!-- Optional hardware: don't block installs on devices without it -->
    <uses-feature android:name="android.hardware.camera" android:required="false" />
</manifest>
```

```kotlin
// Check optional hardware at runtime
if (packageManager.hasSystemFeature(PackageManager.FEATURE_CAMERA_FRONT)) { /* ... */ }
```

- `android:maxSdkVersion` — stop requesting a permission once a newer API makes it unnecessary.
- `<uses-permission-sdk-23>` — declare a permission only on devices that support runtime requests.

---

## 3. Requesting Runtime Permissions

**Flow:** check → (maybe) show rationale → request → handle result. Always ask **in context** (when the feature is used), never at app launch.

```kotlin
val requestPermissionLauncher = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { granted -> if (granted) startFeature() else showDeniedUi() }

when {
    ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA) ==
        PackageManager.PERMISSION_GRANTED -> startFeature()

    shouldShowRequestPermissionRationale(Manifest.permission.CAMERA) ->
        showRationaleThenLaunch { requestPermissionLauncher.launch(Manifest.permission.CAMERA) }

    else -> requestPermissionLauncher.launch(Manifest.permission.CAMERA)
}
```

**Compose:**
```kotlin
val launcher = rememberLauncherForActivityResult(ActivityResultContracts.RequestPermission()) { granted -> /* ... */ }
Button(onClick = { launcher.launch(Manifest.permission.CAMERA) }) { Text("Enable camera") }
```

Requesting several at once → `ActivityResultContracts.RequestMultiplePermissions()`.

---

## 4. Rationale & Denial Handling

| State | `checkSelfPermission()` | `shouldShowRequestPermissionRationale()` | Action |
|---|---|---|---|
| Never asked | `DENIED` | `false` | Request directly |
| Denied once | `DENIED` | `true` | Show rationale UI, then request |
| Denied twice ("don't ask again") — API 30+ | `DENIED` | `false` | No dialog shown; send to app Settings only if essential |
| Granted | `GRANTED` | — | Use the API |

**Do:** degrade gracefully (keep the rest of the app usable) · be specific about which feature is unavailable and why · re-check permission state on every use (it can change via Settings, auto-reset, or hibernation).

**Don't:** nag repeatedly after a firm denial · block the whole UI with a full-screen permission wall · cache granted/denied state in `SharedPreferences`/`DataStore` — always query fresh.

---

## 5. Location Permissions

| Permission | API | Grants |
|---|---|---|
| `ACCESS_COARSE_LOCATION` | 1+ | Approximate (~1–3 km) |
| `ACCESS_FINE_LOCATION` | 1+ | Precise (GPS), implies coarse |
| `ACCESS_BACKGROUND_LOCATION` | 29+ | Access with no visible UI / no location-type foreground service |

- API 31+: user can choose **Precise** or **Approximate** even if fine is requested — handle both outcomes.
- A foreground service with `foregroundServiceType="location"` counts as "while-in-use"; a background-started FGS still needs `ACCESS_BACKGROUND_LOCATION`.
- Full strategy (WorkManager vs. Foreground Service vs. Geofencing, permission combinations) → see `Periodic Location Tracking.md`.

---

## 6. Storage & Media Permissions

| Target/Device API | What you need |
|---|---|
| ≤ 28 | `READ_EXTERNAL_STORAGE` / `WRITE_EXTERNAL_STORAGE` |
| 29+ (Scoped Storage) | No permission needed to **add** your own files to shared storage |
| 33+ | `READ_MEDIA_IMAGES`, `READ_MEDIA_VIDEO`, `READ_MEDIA_AUDIO` (granular, replaces `READ_EXTERNAL_STORAGE`) |
| 34+ (Selected Photos Access) | `READ_MEDIA_VISUAL_USER_SELECTED` — user can grant only some photos/videos |
| 36+ | Photo picker pre-selects previously-granted app-owned photos |

**Recommended:** use the [Photo Picker](https://developer.android.com/training/data-storage/shared/photopicker) — needs **no storage permission at all**. Only request media permissions if you maintain a custom gallery picker.

```kotlin
if (Build.VERSION.SDK_INT >= 34) requestPermissions.launch(arrayOf(READ_MEDIA_IMAGES, READ_MEDIA_VIDEO, READ_MEDIA_VISUAL_USER_SELECTED))
else if (Build.VERSION.SDK_INT >= 33) requestPermissions.launch(arrayOf(READ_MEDIA_IMAGES, READ_MEDIA_VIDEO))
else requestPermissions.launch(arrayOf(READ_EXTERNAL_STORAGE))
```

**Pitfalls:** don't cache access level; re-query on `onResume()` (user can change partial selection without closing your app); treat granted `Uri`s as temporary once partial access is revoked.

---

## 7. Notifications Permission (API 33+)

`POST_NOTIFICATIONS` — runtime permission gating **all** notifications (including foreground-service notifications) on API 33+.

| Scenario | Behavior |
|---|---|
| Fresh install, API 33+ | Notifications **off by default** until granted |
| App update, notifications already enabled pre-13 | Auto pre-granted (no prompt) if eligible |
| App update, notifications disabled pre-13 | Denial persists |
| Media session / self-managed call notifications | Exempt — no permission needed |

```kotlin
val launcher = rememberLauncherForActivityResult(ActivityResultContracts.RequestPermission()) { }
Button(onClick = {
    if (Build.VERSION.SDK_INT >= 33) launcher.launch(Manifest.permission.POST_NOTIFICATIONS)
}) { Text("Enable notifications") }
```

Check before sending: `NotificationManagerCompat.areNotificationsEnabled()`. Ask **in context** (e.g. after the user opts into an alert), not on first launch.

---

## 8. Bluetooth Permissions (API 31+)

| Target API | Permissions |
|---|---|
| ≤ 30 | `BLUETOOTH`, `BLUETOOTH_ADMIN`, `ACCESS_FINE_LOCATION` (scans could infer location) |
| 31+ | `BLUETOOTH_SCAN`, `BLUETOOTH_ADVERTISE`, `BLUETOOTH_CONNECT` (each runtime, granular) |

```xml
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<!-- Skip ACCESS_FINE_LOCATION if you never derive location from scans: -->
<uses-permission android:name="android.permission.BLUETOOTH_SCAN"
    android:usesPermissionFlags="neverForLocation" />
```

`neverForLocation` avoids the location permission entirely but filters some BLE beacons out of scan results.

---

## 9. Special Permissions

Guard sensitive system-level actions. No dialog — user grants via a **Settings** page you deep-link to; check on `onResume()`.

| Permission | Purpose | Check | Settings Intent |
|---|---|---|---|
| `SCHEDULE_EXACT_ALARMS` | Exact alarms | `alarmManager.canScheduleExactAlarms()` | `ACTION_REQUEST_SCHEDULE_EXACT_ALARM` |
| `SYSTEM_ALERT_WINDOW` | Draw over other apps | `Settings.canDrawOverlays(context)` | `ACTION_MANAGE_OVERLAY_PERMISSION` |
| `MANAGE_EXTERNAL_STORAGE` | All-files access | `Environment.isExternalStorageManager()` | `ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION` |

```kotlin
if (!alarmManager.canScheduleExactAlarms()) {
    startActivity(Intent(ACTION_REQUEST_SCHEDULE_EXACT_ALARM))
}
// Check the result in onResume()
```

Use sparingly — Play policy scrutinizes special permissions; show a clear in-app rationale before redirecting to Settings.

---

## 10. Permission Changes by Android Version

| Version (API) | Key Permission Changes |
|---|---|
| 6.0 (23) | Runtime permission model introduced |
| 10 (29) | Scoped Storage; `ACCESS_BACKGROUND_LOCATION` split out; no permission needed to add own files to shared storage |
| 11 (30) | One-time "Only this time" grant; auto-reset of unused apps' permissions; deny-twice = permanent (no more dialog); `SYSTEM_ALERT_WINDOW` flow changes; `READ_PHONE_NUMBERS` split from `READ_PHONE_STATE` |
| 12 (31) | Granular Bluetooth permissions (`SCAN`/`ADVERTISE`/`CONNECT`); precise-vs-approximate location choice |
| 13 (33) | `POST_NOTIFICATIONS`; granular media permissions (`READ_MEDIA_IMAGES`/`VIDEO`/`AUDIO`) |
| 14 (34) | Selected Photos Access (`READ_MEDIA_VISUAL_USER_SELECTED`); apps can self-revoke unused permissions (`revokeSelfPermissionOnKill`) |
| 15 (35) | Permission checks on content URIs; query most-recent photo selection (`QUERY_ARG_LATEST_SELECTION_ONLY`) |
| 16 (36) | Health Connect permissions move to granular `android.permissions.health.*`; new **Local Network Permission** required to access LAN; photo picker pre-selects previously-granted app-owned photos |
| 17 (Beta) | Privacy-focused Contact Picker |

---

## 11. Testing & Debugging

```bash
# Grant every declared runtime permission on install
adb shell install -g PATH_TO_APK

# Inspect grant/denial flags (USER_SET = denied once, USER_FIXED = denied permanently)
adb shell dumpsys package PACKAGE_NAME

# Reset flags so the dialog reappears
adb shell pm clear-permission-flags PACKAGE_NAME PERMISSION_NAME user-set user-fixed

# Simulate a fresh-install POST_NOTIFICATIONS state
adb shell pm revoke PACKAGE_NAME android.permission.POST_NOTIFICATIONS
```

---

## 12. Best Practices Checklist

- [ ] Request only permissions the feature truly needs (check if a [permission-free API](https://developer.android.com/training/permissions/evaluating) exists first)
- [ ] Request in context, tied to a specific user action — never on app launch
- [ ] Show rationale only when `shouldShowRequestPermissionRationale()` returns `true`
- [ ] Handle denial gracefully; never block the entire UI
- [ ] Re-check permission state on every use — don't cache it
- [ ] Use Photo Picker instead of storage permissions where possible
- [ ] Prefer `ACCESS_COARSE_LOCATION` unless precision is essential
- [ ] Declare special permissions only when justified; explain before redirecting to Settings
- [ ] Set `android:maxSdkVersion` on legacy permissions no longer needed for current `targetSdk`
- [ ] Test denial, "don't ask again," and one-time-permission flows via `adb`

---

## 13. Further Reading

| Resource | Link |
|---|---|
| Permissions overview | https://developer.android.com/guide/topics/permissions/overview |
| Requesting runtime permissions | https://developer.android.com/training/permissions/requesting |
| Requesting special permissions | https://developer.android.com/training/permissions/requesting-special |
| Declaring permissions | https://developer.android.com/training/permissions/declaring |
| Photo picker | https://developer.android.com/training/data-storage/shared/photopicker |
| Bluetooth permissions | https://developer.android.com/develop/connectivity/bluetooth/bt-permissions |
| Notification runtime permission | https://developer.android.com/develop/ui/compose/notifications/notification-permission |
| Permission best practices | https://developer.android.com/training/permissions/usage-notes |

---

*Last Updated: July 2026 · Covers Android 6.0 (API 23) through Android 16 (API 36), with Android 17 Beta notes.*
