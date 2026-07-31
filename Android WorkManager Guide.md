# Android WorkManager Guide

Deep-dive reference for WorkManager internals: worker types & threading, unique work, observing/cancelling work, testing, debugging, and migrating from legacy schedulers.

> For the "which background API should I use" decision (WorkManager vs Foreground Service vs plain background service), constraints, chaining, and expedited/UIDT work, see `Android Service Guide.md`. This guide goes deeper into WorkManager itself.

---

## Table of Contents

1. [Worker Types & Threading](#1-worker-types--threading)
2. [Unique Work & Conflict Resolution](#2-unique-work--conflict-resolution)
3. [Observing & Querying Work](#3-observing--querying-work)
4. [Cancelling & Stopping Work](#4-cancelling--stopping-work)
5. [Testing Workers](#5-testing-workers)
6. [Debugging WorkManager](#6-debugging-workmanager)
7. [Multi-Process Workers](#7-multi-process-workers)
8. [Migrating from Legacy Schedulers](#8-migrating-from-legacy-schedulers)
9. [Hilt Integration](#9-hilt-integration)
10. [Best Practices Checklist](#10-best-practices-checklist)
11. [Further Reading](#11-further-reading)

---

## 1. Worker Types & Threading

| Type | For | Threading |
|---|---|---|
| **`Worker`** | Simple, synchronous work | WorkManager runs `doWork()` on a background thread automatically |
| **`CoroutineWorker`** | Kotlin (recommended) | `doWork()` is `suspend`; runs on a default `Dispatcher` (customizable) |
| **`RxWorker`** | Existing RxJava codebases | `createWork()` called on main thread; returned `Single` is **subscribed** on a background thread by default (override `getBackgroundScheduler()` to change) |
| **`ListenableWorker`** | Callback-based async APIs (base class of all above) | `startWork()` called on **main thread** — you own all threading |

```kotlin
// CoroutineWorker — recommended for Kotlin
class SyncWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result = try {
        repository.sync(); Result.success()
    } catch (e: Exception) { Result.retry() }
}
```

```kotlin
// RxWorker
class RxDownloadWorker(context: Context, params: WorkerParameters) : RxWorker(context, params) {
    override fun createWork(): Single<Result> =
        Observable.range(0, 100).flatMap { download(it) }.toList().map { Result.success() }
}
```

```kotlin
// ListenableWorker — full manual control via CallbackToFutureAdapter
class CallbackWorker(context: Context, params: WorkerParameters) : ListenableWorker(context, params) {
    override fun startWork(): ListenableFuture<Result> = CallbackToFutureAdapter.getFuture { completer ->
        val callback = object : Callback {
            override fun onFailure(call: Call, e: IOException) = completer.setException(e)
            override fun onResponse(call: Call, response: Response) = completer.set(Result.success())
        }
        completer.addCancellationListener(cancelDownloadRunnable, executor) // cleanup on stop
        downloadAsynchronously(url, callback)
        callback // the "tag" object for future.cancel() bookkeeping
    }
}
```

`ListenableFuture` requires either Guava's `ListeningExecutorService` or the lightweight `androidx.concurrent:concurrent-futures` (`CallbackToFutureAdapter`) — the future is **automatically cancelled** when WorkManager decides the work should stop.

---

## 2. Unique Work & Conflict Resolution

**Unique work** guarantees only one instance of work with a given human-readable name exists at a time — critical to avoid duplicate scheduling (e.g. calling "schedule daily sync" on every app launch).

```kotlin
WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "sendLogs", ExistingPeriodicWorkPolicy.KEEP,
    PeriodicWorkRequestBuilder<SendLogsWorker>(24, TimeUnit.HOURS).build(),
)
```

| Policy (`ExistingWorkPolicy`, one-time) | Behavior |
|---|---|
| `REPLACE` | Cancels existing work, replaces with new |
| `KEEP` | Ignores the new request if unfinished work exists |
| `APPEND` | Chains new work after existing; if existing work is `CANCELLED`/`FAILED`, the new work is too |
| `APPEND_OR_REPLACE` | Like `APPEND`, but new work still runs even if the prerequisite failed/was cancelled |

`ExistingPeriodicWorkPolicy` (periodic work) supports only `REPLACE`/`KEEP` (also `UPDATE` in newer versions, to change constraints without losing the schedule position).

---

## 3. Observing & Querying Work

```kotlin
// By id / unique name / tag
workManager.getWorkInfoById(id)                    // ListenableFuture<WorkInfo>
workManager.getWorkInfosForUniqueWork("sync")       // ListenableFuture<List<WorkInfo>>
workManager.getWorkInfosByTag("syncTag")            // ListenableFuture<List<WorkInfo>>

// Reactive (Flow) variant
workManager.getWorkInfoByIdFlow(id).collect { info ->
    if (info?.state == WorkInfo.State.SUCCEEDED) showSnackbar()
}
```

**Complex queries (2.4.0+)** — combine tag(s), state(s), and unique work name(s), AND-ed across categories, OR-ed within a category:
```kotlin
val query = WorkQuery.Builder
    .fromTags(listOf("syncTag"))
    .addStates(listOf(WorkInfo.State.FAILED, WorkInfo.State.CANCELLED))
    .addUniqueWorkNames(listOf("preProcess", "sync"))
    .build()
val infos: ListenableFuture<List<WorkInfo>> = workManager.getWorkInfos(query)
```

---

## 4. Cancelling & Stopping Work

```kotlin
workManager.cancelWorkById(id)
workManager.cancelUniqueWork("sync")
workManager.cancelAllWorkByTag("syncTag") // cancels ALL work carrying this tag
```

**Reasons a running Worker gets stopped:**
- Explicit cancellation call.
- Unique work re-enqueued with `REPLACE`.
- Constraints no longer met (e.g. network dropped).
- System-imposed **10-minute execution deadline** exceeded — work is rescheduled for retry.

**Cooperative shutdown — implement both:**
```kotlin
override fun onStopped() {
    // Called as soon as the Worker is stopped — close DB/file handles, cancel network calls
}
// In a long-running doWork() loop:
while (!isStopped) { processNextChunk() }
```
WorkManager **ignores** any `Result` returned after the stop signal — the work is already considered stopped.

---

## 5. Testing Workers

```kotlin
// build.gradle.kts
testImplementation("androidx.work:work-testing:2.11.2")
```

**Plain `Worker`** — `TestWorkerBuilder` lets you specify the executor:
```kotlin
@Test
fun testSleepWorker() {
    val worker = TestWorkerBuilder<SleepWorker>(
        context = context, executor = Executors.newSingleThreadExecutor(),
        inputData = workDataOf("SLEEP_DURATION" to 1000L),
    ).build()
    assertThat(worker.doWork(), `is`(Result.success()))
}
```

**`CoroutineWorker` / `RxWorker` / any `ListenableWorker`** — `TestListenableWorkerBuilder` defers to the worker's own threading:
```kotlin
@Test
fun testSleepWorker() = runBlocking {
    val worker = TestListenableWorkerBuilder<SleepCoroutineWorker>(context).build()
    assertThat(worker.doWork(), `is`(Result.success()))
}
```
For `RxWorker`, call `.createWork().subscribe { result -> assertThat(result, is(Result.success())) }` instead.

---

## 6. Debugging WorkManager

**Enable verbose logs** (look for the `WM-` tag prefix in Logcat) via custom on-demand initialization:
```kotlin
class MyApplication : Application(), Configuration.Provider {
    override val workManagerConfiguration
        get() = Configuration.Builder().setMinimumLoggingLevel(android.util.Log.DEBUG).build()
}
```
```xml
<!-- Disable the default auto-initializer first -->
<provider android:name="androidx.work.impl.WorkManagerInitializer"
    android:authorities="${applicationId}.workmanager-init" tools:node="remove" />
```

**Inspect scheduled jobs (API 23+):**
```bash
adb shell dumpsys jobscheduler   # shows required/satisfied/unsatisfied constraints per job, plus recent job history
```

**Request a diagnostics dump (debug builds, WorkManager 2.4.0+):**
```bash
adb shell am broadcast -a "androidx.work.diagnostics.REQUEST_DIAGNOSTICS" -p "<your.package.name>"
adb logcat   # look for WM-DiagnosticsWrkr — lists recently completed, running, and scheduled work
```

---

## 7. Multi-Process Workers

Bind a Worker to a specific process with `RemoteListenableWorker`:

```kotlin
val data = Data.Builder()
    .putString(ARGUMENT_PACKAGE_NAME, componentName.packageName)
    .putString(ARGUMENT_CLASS_NAME, componentName.className)
    .build()

OneTimeWorkRequest.Builder(ExampleRemoteListenableWorker::class.java).setInputData(data).build()
```
```xml
<service android:name="androidx.work.multiprocess.RemoteWorkerService"
    android:exported="false" android:process=":worker1" />
```

---

## 8. Migrating from Legacy Schedulers

| Legacy (Firebase JobDispatcher / GCM) | WorkManager |
|---|---|
| `JobService` (manual threading, `onStartJob` on main thread) | `ListenableWorker` |
| `SimpleJobService` (background thread handled for you) | `Worker` |
| `Job.Builder` | `OneTimeWorkRequest.Builder` / `PeriodicWorkRequest.Builder` |
| `Bundle` input extras | `Data` / `workDataOf(...)` |
| `Job.Builder.setConstraints(...)` | `Constraints.Builder()` |
| Tag + `setReplaceCurrent()` | `enqueueUniqueWork()` / `enqueueUniquePeriodicWork()` + `ExistingWorkPolicy` |
| `dispatcher.cancel(tag)` | `cancelUniqueWork(name)` |

```kotlin
// Firebase JobDispatcher-era Job → WorkManager
val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.UNMETERED)
    .setRequiresCharging(true)
    .build()

val request = OneTimeWorkRequestBuilder<MyWorker>()
    .setInputData(workDataOf("some_key" to "some_value"))
    .setInitialDelay(60, TimeUnit.SECONDS)
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30_000, TimeUnit.MILLISECONDS)
    .setConstraints(constraints)
    .build()

WorkManager.getInstance(context).enqueueUniqueWork("my-unique-name", ExistingWorkPolicy.KEEP, request)
```

Key behavioral difference: **WorkManager persists across device reboot automatically** — legacy `Lifetime.UNTIL_NEXT_BOOT`-style settings have no equivalent because persistence is the default.

---

## 9. Hilt Integration

```kotlin
@HiltWorker
class SyncWorker @AssistedInject constructor(
    @Assisted appContext: Context,
    @Assisted params: WorkerParameters,
    private val repository: SyncRepository, // regular Hilt-injected dependency
) : CoroutineWorker(appContext, params)

@HiltAndroidApp
class MyApplication : Application(), Configuration.Provider {
    @Inject lateinit var workerFactory: HiltWorkerFactory
    override val workManagerConfiguration
        get() = Configuration.Builder().setWorkerFactory(workerFactory).build()
}
```
Full DI mechanics (scopes, assisted injection details) → `Android Dependency Injection Guide.md` §9.

---

## 10. Best Practices Checklist

- [ ] Use `CoroutineWorker` for new Kotlin code; reserve `ListenableWorker` for callback-based APIs you can't wrap in coroutines
- [ ] Always use **unique work** (`enqueueUniqueWork`/`enqueueUniquePeriodicWork`) for anything scheduled repeatedly (app launch, periodic sync)
- [ ] Pick the `ExistingWorkPolicy` deliberately — `KEEP` for idempotent periodic jobs, `REPLACE` when new input should supersede old
- [ ] Implement `onStopped()` to release resources; poll `isStopped` in long-running loops
- [ ] Test `Worker`s with `TestWorkerBuilder`, and `CoroutineWorker`/`RxWorker`/`ListenableWorker` with `TestListenableWorkerBuilder`
- [ ] Enable `WM-` debug logging + `adb shell dumpsys jobscheduler` when work isn't firing as expected
- [ ] Inject dependencies into Workers via `@HiltWorker` + `HiltWorkerFactory`, not manual construction
- [ ] Remember the **10-minute execution limit** per work run — chain/split long operations instead of fighting it

---

## 11. Further Reading

| Resource | Link |
|---|---|
| Threading in WorkManager | https://developer.android.com/develop/background-work/background-tasks/persistent/threading |
| Manage work (unique work, cancellation) | https://developer.android.com/develop/background-work/background-tasks/persistent/how-to/manage-work |
| Testing WorkManager Workers | https://developer.android.com/develop/background-work/background-tasks/testing/persistent/worker-impl |
| Debugging WorkManager | https://developer.android.com/develop/background-work/background-tasks/testing/persistent/debug |
| Migrating from Firebase JobDispatcher | https://developer.android.com/develop/background-work/background-tasks/persistent/migrate-from-legacy/firebase |

---

*Last Updated: July 2026 · WorkManager 2.11.2.*
