# Battery & Power Optimization

Concise reference for Android's power-management system: Doze, App Standby, App Standby Buckets, resource limits, exemptions, and network/radio efficiency.

> For choosing the right background-execution API (WorkManager vs Foreground Service vs background service), see `Android Service Guide.md`. This guide covers the **power-management restrictions** those APIs operate under.

---

## Table of Contents

1. [Doze Mode](#1-doze-mode)
2. [App Standby](#2-app-standby)
3. [App Standby Buckets](#3-app-standby-buckets)
4. [Resource Limits by Device/App State](#4-resource-limits-by-deviceapp-state)
5. [Battery Optimization Exemptions](#5-battery-optimization-exemptions)
6. [Radio & Network Efficiency](#6-radio--network-efficiency)
7. [Testing Doze & App Standby](#7-testing-doze--app-standby)
8. [Best Practices Checklist](#8-best-practices-checklist)
9. [Further Reading](#9-further-reading)

---

## 1. Doze Mode

Triggered when the device is **unplugged, stationary, and screen-off** for a while. Applies to all apps on Android 6.0+ regardless of target SDK.

| Restriction | Detail |
|---|---|
| Network access | Suspended |
| Wake locks | Ignored |
| `AlarmManager` alarms | Deferred to next maintenance window (`setExact`/`setWindow`) |
| Wi-Fi scans | Not performed |
| Sync adapters | Don't run |
| `JobScheduler` / `WorkManager` | Don't run (WorkManager uses JobScheduler internally) |

The system periodically opens a **maintenance window** to flush pending jobs/syncs/alarms and allow network access, then re-enters Doze; windows get **less frequent** the longer the device stays idle.

**Alarms that still work in Doze:**
| API | Behavior |
|---|---|
| `setAndAllowWhileIdle()` / `setExactAndAllowWhileIdle()` | Fire in Doze, but **max once per 9 minutes per app** |
| `setAlarmClock()` | Fires normally; system briefly exits Doze just before |

**Doze checklist:**
- Prefer **FCM** for downstream messaging over a custom persistent connection.
- Use **FCM high-priority** messages only when they result in a user-visible notification.
- Pack enough info into the initial FCM payload to avoid a follow-up network round-trip.
- Use `setAndAllowWhileIdle()`/`setExactAndAllowWhileIdle()` only for truly critical alarms.

---

## 2. App Standby

An app is marked **idle** when the user hasn't touched it in a while AND none of these hold: user explicitly launched it, it has a foreground activity/foreground service, or it's showing a lock-screen/tray notification.

- Idle apps: background network access deferred; roughly **once-a-day** network access window while idle.
- Plugging in power **releases** all apps from standby.
- **FCM** is Doze/Standby-aware: high-priority messages grant temporary network + partial wake lock access even while idle.

> Don't start a foreground service *purely* to dodge idle detection — reserve foreground services for work the user actually expects to see running (per Play policy and this repo's `Android Service Guide.md`).

---

## 3. App Standby Buckets

Since Android 9 (API 28), every app is placed in one of five (+1) priority buckets based on usage recency/frequency — **don't try to manipulate which bucket you're in**; design for graceful behavior in any bucket.

| Bucket | Trigger | Restriction level |
|---|---|---|
| **Active** | Currently used / very recently used / long-running FGS / tapped from notification | Minimal — generous job quota (Android 16+) |
| **Working set** | Used often (e.g. daily) | Mild |
| **Frequent** | Used regularly, not daily | Stronger |
| **Rare** | Rarely used | Strict — network access **disabled** outside windows |
| **Restricted** (Android 12+) | 8+ days (Android 13+; 45 on 12/12L) without interaction, or excessive broadcasts/bindings in 24h | Most severe — applies **even while charging** |
| **Never** | Installed but never run | Severe restrictions |

**Restricted bucket specifics:** jobs run once/day in a 10-min batched session (must share the slot with another pending job), fewer expedited jobs allowed, one alarm/day.

**Exempt from Restricted bucket:** companion-device apps, device/profile owner apps, persistent apps, VPN apps, dialer-role apps, apps with active widgets, apps the user manually marked "unrestricted," and apps holding `USE_EXACT_ALARM` or `ACCESS_BACKGROUND_LOCATION`.

**Check your bucket:**
```kotlin
val bucket = usageStatsManager.appStandbyBucket // compare against UsageStatsManager.STANDBY_BUCKET_*
```
```bash
adb shell am get-standby-bucket <package>
```

**Design implications:**
- No launcher activity → your app may never reach the Active bucket. Give it one.
- Non-interactive notifications → users can't promote you to Active by tapping. Design interactive notifications where sensible.
- Misusing high-priority FCM (not resulting in a notification) can get future messages silently downgraded (pre-API 32).
- Multi-package apps can land in **different buckets per package** — test each independently.

---

## 4. Resource Limits by Device/App State

Device/app state can **override** bucket-based limits:

| Device state | Jobs | Alarms | Network | FCM |
|---|---|---|---|---|
| Charging | No limits (except Restricted bucket) | No limits | No restrictions | No restrictions |
| Screen on | Bucket-based | Bucket + process based | Bucket/process based | No restrictions |
| Screen off + Doze active | Bucket-based, deferred to maintenance window | Regular: deferred; while-idle: max 7/hour | Restricted | High-priority: none; normal: deferred |

| App process state | Jobs | Alarms | Network |
|---|---|---|---|
| Visible / foreground | No limits | No limits | No restrictions |
| Running a foreground service | Bucket-based (changed in Android 16) | Bucket-based frequency | No restrictions |
| User manually restricted battery | Restricted | Restricted | Bucket-dependent |
| User manually unrestricted | Generous | No limits | Unrestricted (unless Data Saver) |

**Approximate bucket-based job/alarm/network quotas** (guideline only — subject to change):

| Bucket | Regular jobs | Expedited jobs | Alarms | Network |
|---|---|---|---|---|
| Active | ~20 min / rolling 60 min | ~30 min / rolling 24h | No limits | None |
| Working set | ~10 min / rolling 4h | ~15 min / rolling 24h | 10/hour | None |
| Frequent | ~10 min / rolling 12h | ~10 min / rolling 24h | 2/hour | None |
| Rare | ~10 min / rolling 24h | ~10 min / rolling 24h | 1/hour | **Disabled** |
| Restricted | Once/day, ~10 min (batched w/ other jobs) | ~5 min / rolling 24h | 1/day | **Disabled** |

**Android 16 (API 36) change:** job execution quota is now also adjusted by whether the job runs while the app is in a **top (foreground)** state or running a **foreground service**, in addition to standby bucket.

---

## 5. Battery Optimization Exemptions

An app on the exemption list can use network + partial wake locks during Doze/Standby (other restrictions still apply — e.g. jobs/alarms are still deferred on older APIs).

```kotlin
val isExempt = powerManager.isIgnoringBatteryOptimizations(packageName)
```

| Intent | Effect |
|---|---|
| `ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS` | Sends user to system settings list |
| `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` | Direct opt-in dialog — **only for apps with an acceptable use case** |

**Google Play policy:** exemption requests are prohibited unless Doze/Standby genuinely breaks your core function and FCM high-priority messaging isn't viable.

| Use case | Exemption acceptable? |
|---|---|
| Chat/VOIP app that *can* use FCM high-priority | ❌ No — use FCM instead |
| Chat/VOIP app with a **technical** dependency blocking FCM | ✅ Yes |
| Safety app | ✅ Yes (if applicable) |
| Task automation app | ✅ Yes (if applicable) |
| Peripheral companion app needing persistent connection for the peripheral's own internet access | ✅ Yes |
| Peripheral companion app only syncing periodically / standard BT profile devices | ❌ No |

---

## 6. Radio & Network Efficiency

The wireless radio has 3 power states: **full power** → **low power** (~50% less, ~1.5s to return to full) → **standby** (minimal, ~2s+ to return to full). Every new connection triggers full power **plus tail time** (~5s) **plus low-power dwell** (~12s) — roughly **18 seconds of drain per transfer session**, even for a 1-second payload.

| Technique | Why |
|---|---|
| **Bundle transfers** | 3× one-second transfers/minute keeps the radio perpetually active; one 3-second transfer/minute lets it sleep ~40s/minute |
| **Prefetch** | Front-load likely-needed data in one burst instead of many small requests; ~1-2 MB / 6 seconds is a reasonable threshold for ~50%-likely-to-be-used data |
| **Check connectivity first** | Searching for signal is one of the most expensive radio operations — check `ConnectivityManager` before a user-initiated request when offline is plausible |
| **Pool connections** | Reuse HTTP connections (OkHttp/`HttpURLConnection` do this by default) instead of opening new ones per request |
| **Defer non-urgent transfers** | Use WorkManager with `NetworkType.UNMETERED` + `setRequiresCharging(true)` for large/non-urgent downloads |

Rule of thumb: aim to need a new network request only **every 2–5 minutes**, transferring **1–5 MB** at a time; chunk large downloads (e.g. video) accordingly rather than streaming continuously.

Full HTTP client setup (timeouts, caching, interceptors) → `Android Networking Guide.md`.

---

## 7. Testing Doze & App Standby

```bash
# --- Doze ---
adb shell dumpsys deviceidle force-idle   # force Doze
adb shell dumpsys deviceidle unforce      # exit Doze
adb shell dumpsys battery reset           # reset battery/charging simulation

# --- App Standby ---
adb shell dumpsys battery unplug
adb shell am set-inactive <package> true   # force idle
adb shell am set-inactive <package> false  # wake
adb shell am get-inactive <package>        # check idle state

# --- Standby bucket ---
adb shell am get-standby-bucket <package>
```

Always verify your app **recovers gracefully** after exiting Doze/Standby — pending syncs resume, UI reflects fresh state, no crashes from deferred callbacks firing late.

---

## 8. Best Practices Checklist

- [ ] Use FCM (high-priority only for notification-producing messages) instead of a custom persistent connection
- [ ] Use `setAndAllowWhileIdle()`/`setExactAndAllowWhileIdle()` only for genuinely critical alarms
- [ ] Never start a foreground service solely to avoid idle/standby detection
- [ ] Give the app a launcher activity and interactive notifications so it can reach the Active bucket
- [ ] Don't request battery-optimization exemption unless your use case matches an accepted category
- [ ] Bundle/batch network calls; prefetch likely-needed data instead of many small requests
- [ ] Check `ConnectivityManager` before user-initiated requests to avoid needless radio wake-ups
- [ ] Use WorkManager constraints (`UNMETERED`, `setRequiresCharging`) for large/deferrable transfers
- [ ] Test explicitly with `adb shell dumpsys deviceidle` / `am set-inactive` before shipping background-heavy features
- [ ] Re-test each package separately in multi-package apps — buckets are assigned per package

---

## 9. Further Reading

| Resource | Link |
|---|---|
| Optimize for Doze and App Standby | https://developer.android.com/training/monitoring-device-state/doze-standby |
| App Standby Buckets | https://developer.android.com/topic/performance/appstandby |
| Power management resource limits | https://developer.android.com/topic/performance/power/power-details |
| Optimizing network access for battery life | https://developer.android.com/develop/connectivity/network-ops/network-access-optimization |

---

*Last Updated: July 2026 · Covers Android 6.0 (API 23) through Android 16 (API 36) power management behavior.*
