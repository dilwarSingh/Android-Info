# UI Performance & Memory Leaks — Cheat Sheet

## What is UI Jank?

- **Jank** = missed frame rendering deadlines → visible stutter/dropped frames
- **60 FPS** target → each frame must render in **≤ 16.6ms**
- **90/120Hz** displays → **≤ 11.1ms / 8.3ms** per frame
- Blocked or overloaded main thread → GPU waits → frame dropped → **JANK**

---

## Root Causes of Jank

| Cause | Detail |
|---|---|
| **Heavy work on main thread** | Network, DB, file I/O blocking UI thread |
| **Complex layouts** | Deep hierarchies, redundant measure/layout passes |
| **Overdraw** | Pixels drawn multiple times per frame — wasted GPU cycles |
| **GC pressure** | Frequent garbage collection pauses freeze UI |
| **Bitmap misuse** | Large bitmaps decoded on main thread, no caching |
| **Recomposition storms** | Unstable lambdas/objects → unnecessary Compose recompositions |
| **Object allocation in `onDraw()`** | New objects each frame → GC triggered every frame |
| **Animation on main thread** | Heavy animations not offloaded to RenderThread |
| **Slow RecyclerView** | Heavy `onBindViewHolder`, no `DiffUtil`, no view recycling |
| **`SharedPreferences.commit()`** | Synchronous disk write on main thread — use `apply()` |
| **Too many ContentProviders** | Each adds ~2ms to cold start — use App Startup library |

---

## Frame Pipeline

```
App Code → Main Thread → RenderThread → GPU → Display
               ↑
          If THIS is slow → JANK
```

- `Choreographer` fires VSYNC → app has **16ms** to: handle input → logic → layout → draw
- Exceeding 16ms → frame **skipped** → user sees stutter

---

## Profiling Tools

### Android Studio Profiler
- **CPU Profiler → System Trace** — best for jank; shows frame timings, Choreographer, RenderThread, main thread work
- **Memory Profiler** — detects leaks, GC pauses, heap dumps for retained objects
- **Layout Inspector** — live view hierarchy, detects overdraw and deep nesting

### Perfetto (replaces Systrace)
```bash
adb shell perfetto -o /data/misc/perfetto-traces/trace.pb \
  -t 10s sched freq idle am wm gfx view binder_driver
adb pull /data/misc/perfetto-traces/trace.pb ~/trace.pb
```
Open at **https://ui.perfetto.dev** — look for: `Choreographer#doFrame` duration, `View#draw/measure/layout`, Binder calls on main thread.

### GPU Overdraw Visualization
`Developer Options → Debug GPU Overdraw → Show overdraw areas`

| Color | Meaning |
|---|---|
| No color | No overdraw (drawn 1x — perfect) |
| Blue | Overdrawn 1 time (drawn 2x total — acceptable) |
| Green | Overdrawn 2 times (drawn 3x total — warning) |
| Pink | Overdrawn 3 times (drawn 4x total — fix this) |
| Red | Overdrawn 4+ times (drawn 5x+ total — critical!) |

### Profile GPU Rendering
`Developer Options → Profile GPU Rendering → On screen as bars`
- Each bar = one frame; **green line = 16ms threshold**; bars above = **JANK**

### StrictMode
```kotlin
if (BuildConfig.DEBUG) {
    StrictMode.setThreadPolicy(
        StrictMode.ThreadPolicy.Builder()
            .detectDiskReads().detectDiskWrites().detectNetwork()
            .detectCustomSlowCalls()
            .penaltyLog().penaltyFlashScreen().build()
    )
    StrictMode.setVmPolicy(
        StrictMode.VmPolicy.Builder()
            .detectLeakedSqlLiteObjects().detectLeakedClosableObjects()
            .detectActivityLeaks().penaltyLog().build()
    )
}
```

### Compose Recomposition Tracker
- `Android Studio → Layout Inspector → Enable "Highlight recompositions"`
- Compiler metrics (Kotlin 2.0+):
```kotlin
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_metrics")
    metricsDestination = layout.buildDirectory.dir("compose_metrics")
}
```
Check `/build/compose_metrics/` — `*-composables.txt` (skippable?), `*-classes.txt` (stable?).

### ADB gfxinfo
```bash
adb shell dumpsys gfxinfo com.your.package framestats
```
- **Janky frames should be < 1%**; 95th percentile > 16ms = jank

---

## Memory Leaks

### What Is a Memory Leak?
- Object **no longer needed** but still referenced → GC cannot collect it
- Heap grows over time → GC runs more often → UI pauses → **OOM crash**
- Silent and gradual — often only noticed after extended use

### Common Leak Causes

| Cause | Detail |
|---|---|
| **Static reference to Activity/Context** | Activity held forever in static/singleton field |
| **Non-static inner/anonymous classes** | Implicitly hold reference to outer `Activity` |
| **Unregistered listeners/callbacks** | Sensor, GPS, BroadcastReceiver never unregistered |
| **`GlobalScope` coroutines** | Outlive Activity lifecycle |
| **Singleton holding Context** | Singleton initialized with `Activity` context instead of `Application` |
| **Bitmap not recycled** | Large `Bitmap` held after no longer displayed |
| **Handler.postDelayed with `this`** | Runnable captures Activity — keeps it alive |
| **ViewModel holding View refs** | ViewModel survives rotation; View does not |
| **ViewBinding not nulled in Fragment** | Binding retains entire View tree after `onDestroyView` |
| **Unclosed Cursor/InputStream** | Leaks memory and file descriptors |
| **Compose: State captured in lambdas** | Holding `State<T>` wrapper instead of `.value` in a long-lived lambda — **a stale-read/correctness pitfall, not a leak** |

### The Leak Chain
```
Activity (destroyed on rotation)
    ↑ referenced by
MyListener (registered, never unregistered)
    ↑ referenced by
SomeManager.instance (singleton — lives forever)

Result: Activity never GC'd → LEAK
```

---

## Memory Leak Detection

### LeakCanary
```kotlin
debugImplementation("com.squareup.leakcanary:leakcanary-android:2.14")
```
Zero config — auto-hooks lifecycle. Provides readable reference chain:
```
GC Root: static field MyManager.instance → MyManager.listener → MyActivity (mDestroyed=true)
```

### Android Studio Memory Profiler
- `View → Tool Windows → Profiler → Memory`
- Rotate screen → **Dump Java Heap** → filter by package → look for multiple Activity instances
- **Signs of leak:** heap grows without dropping after GC; sawtooth/shark-fin GC pattern

### ADB
```bash
adb shell dumpsys meminfo com.your.package
adb shell am dumpheap com.your.package /data/local/tmp/heap.hprof
adb pull /data/local/tmp/heap.hprof ~/heap.hprof
```

---

## Memory Leak Fixes

### Fix 1: No Static Activity/View References
```kotlin
// Use applicationContext in singletons — safe, lives as long as process
object MyManager {
    lateinit var appContext: Context
    fun init(context: Context) { appContext = context.applicationContext }
}
```

### Fix 2: Handler — Extract Runnable, Always Remove
```kotlin
private val handler = Handler(Looper.getMainLooper())
private val updateRunnable = Runnable { updateUI() }

override fun onStart() { handler.postDelayed(updateRunnable, 5000) }
override fun onStop() { handler.removeCallbacks(updateRunnable) }
```

### Fix 3: Always Unregister Listeners
```kotlin
override fun onDestroyView() {
    super.onDestroyView()
    sensorManager.unregisterListener(this) // symmetric with onViewCreated
}
```

### Fix 4: Scope Coroutines to Lifecycle
- `GlobalScope` → **never use in Activity/Fragment**
- Use `lifecycleScope.launch { }` (auto-cancelled on destroy)
- Use `viewModelScope.launch { }` (auto-cancelled on ViewModel clear)

### Fix 5: ViewModel Must Never Hold Views
- ViewModel exposes `StateFlow`; UI observes and updates itself
- Always collect with `repeatOnLifecycle(Lifecycle.State.STARTED)` — stops collection when backgrounded

### Fix 6: Null ViewBinding in Fragments
```kotlin
private var _binding: MyFragmentBinding? = null
private val binding get() = _binding!!

override fun onDestroyView() {
    super.onDestroyView()
    _binding = null // allows View tree to be GC'd
}
```

### Fix 7: Close Resources with `.use {}`
```kotlin
db.rawQuery("SELECT * FROM users WHERE id=?", arrayOf("$id")).use { cursor ->
    if (cursor.moveToFirst()) cursor.toUser() else null
}
```

### Fix 8: Compose — Collect State Safely
```kotlin
val state by viewModel.uiState.collectAsStateWithLifecycle() // captures value, not State wrapper
```

### Memory Leak Quick Reference

| Scenario | Fix |
|---|---|
| Activity leaked on rotation | `WeakReference` or `applicationContext` |
| Multiple Activity instances in heap | Extract Runnable / top-level class |
| Fragment view not released | `_binding = null` in `onDestroyView` |
| ViewModel crash after rotation | Expose `StateFlow`, never store Views |
| Coroutine running after destroy | `lifecycleScope` / `viewModelScope` |
| OOM after heavy list scrolling | Coil/Glide with lifecycle awareness |
| Sensor keeps running in background | Unregister in `onDestroyView` / `onStop` |
| DB/file descriptor exhaustion | `.use {}` or try-finally |

---

## ANR (Application Not Responding)

### Timeouts
- **Input event**: not handled within **5 seconds**
- **BroadcastReceiver.onReceive()**: **10s** (foreground) / **60s** (background) on Android 13 and lower — Android 14+ dynamically scales to **10–20s** (foreground) / **60–120s** (background) depending on CPU starvation
- **Service.onCreate/onStartCommand()**: **20s** (foreground) / **200s** (background, API 26+)

### Common Causes
- Network/DB/disk I/O on main thread
- `SharedPreferences.commit()` instead of `apply()`
- Deadlock between threads
- Heavy work in `BroadcastReceiver.onReceive()`

### Diagnosing ANRs
```bash
adb pull /data/anr/traces.txt ~/anr_traces.txt   # ANR trace files
adb bugreport ~/bugreport.zip                     # Android 11+
```
- **Google Play Console → Android Vitals → ANR rate** — target **< 0.47%**

### Fix: Async Patterns
```kotlin
// SharedPreferences — always apply(), never commit()
prefs.edit().putString(key, value).apply()

// BroadcastReceiver — use goAsync() to extend deadline
val pendingResult = goAsync()
CoroutineScope(Dispatchers.IO).launch {
    try { /* heavy work */ } finally { pendingResult.finish() }
}
```

---

## Common Performance Fixes

### Move Work Off Main Thread
```kotlin
viewModelScope.launch {
    val data = withContext(Dispatchers.IO) { database.query() }
    updateUI(data)
}
```

### Reduce Overdraw
- Remove redundant/stacked backgrounds — set background only once per view
- Use `canvas.clipRect()` in custom views; flatten layout hierarchy

### Fix Recomposition (Compose)
- **Strong Skipping Mode** (default in Kotlin 2.0.20+) auto-remembers lambdas
- Still annotate data classes with `@Immutable` / use immutable `List` instead of `MutableList`
```kotlin
@Immutable
data class User(val name: String, val list: List<String>)
```

### Avoid Allocations in `onDraw()`
```kotlin
private val paint = Paint() // allocate once at class level
override fun onDraw(canvas: Canvas) { canvas.drawCircle(cx, cy, radius, paint) }
```

### LazyColumn with `key` + `contentType`
```kotlin
LazyColumn {
    items(
        items = users,
        key = { it.id },               // stable identity → efficient reorder
        contentType = { "user_card" }  // same-type items share compositions
    ) { user -> UserCard(user) }
}
```
- Without `key`: items recompose on every list change
- Without `contentType`: type changes destroy & recreate compositions

### `derivedStateOf` for Computed State
```kotlin
val showButton by remember { derivedStateOf { listState.firstVisibleItemIndex > 0 } }
// recomposes only when result changes, not on every scroll event
```

### RecyclerView Optimization
- Use `ListAdapter` + `DiffUtil` — only rebinds changed items
- **Never use `notifyDataSetChanged()`** — rebinds all visible items
- `recyclerView.setHasFixedSize(true)` — skips `requestLayout()` when item size is fixed
- Load images with Coil/Glide in `onBindViewHolder` — async, cached, lifecycle-aware
- Inflate only in `onCreateViewHolder`, never in `onBindViewHolder`
- `recyclerView.setItemViewCacheSize(20)` — reduce rebinding on small scrolls
- `recyclerView.recycledViewPool.setMaxRecycledViews(0, 30)` — increase pool for view type 0

### Image Loading
```kotlin
// Never: BitmapFactory.decodeFile(path) — OOM + jank
AsyncImage(model = ImageRequest.Builder(context).data(url).crossfade(true).build(), ...)
```
- **`inSampleSize`**: power-of-2 downsampling (4x, 16x, 64x memory reduction)
- **`inBitmap`**: reuse existing bitmap allocation — avoids GC churn
- **`Bitmap.Config.RGB_565`**: 2 bytes/pixel vs 4 (no alpha) — halves memory
- **`Bitmap.recycle()`**: explicitly free native memory (rarely needed with modern GC)
- Always prefer **Coil/Glide** over manual bitmap decoding in production

### Room / Database
```kotlin
@Query("SELECT * FROM users WHERE name LIKE :search")
suspend fun searchUsers(search: String): List<User>  // never block main thread

@Entity(tableName = "users", indices = [Index(value = ["email"], unique = true)])
// index prevents full table scans on queried columns
```
- Use `@Transaction` for batch inserts — single disk write instead of N
- Enable WAL mode (default in Room 2.x) — concurrent reads/writes
- Use `Flow<List<T>>` for reactive, lifecycle-aware queries
- Avoid `SELECT *` — select only needed columns
- Use `LIMIT` / pagination with **Paging 3** — don't load entire table into memory

### `repeatOnLifecycle` for Flow Collection
```kotlin
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { updateUI(it) } // paused when backgrounded
    }
}
// In Compose: prefer collectAsStateWithLifecycle() over collectAsState()
```

### Startup Optimization
- Aim for a fast cold start, well under a second to first frame — Android Vitals flags cold startup as excessive above **5 seconds**
- Only critical init (e.g., crash reporter) in `Application.onCreate()` synchronously
- Defer analytics/image SDKs via `Handler.post {}` (runs after first frame)
- Background network/disk init via `Dispatchers.IO` coroutine
- Use `by lazy {}` for singletons initialized on first access
- **App Startup library** — consolidates multiple `ContentProvider` entries into one (~2ms savings each)
- **Baseline Profiles** — AOT-compile critical paths; reduces cold start **about 30%**

```kotlin
// SplashScreen API (Android 12+) — keep visible while loading
val splashScreen = installSplashScreen() // BEFORE super.onCreate()
splashScreen.setKeepOnScreenCondition { !isReady }
```

### Flatten Layout Hierarchy
- Use **`ConstraintLayout`** — replaces nested `LinearLayout`/`RelativeLayout`; single measure pass
- Use **`<merge>`** as root of included layouts — eliminates redundant wrapper view
- Use **`<ViewStub>`** for rarely-shown views (error/empty states) — zero cost until inflated
- Deep nesting with `match_parent`/`wrap_content` mixed → **double-taxation** (child measured twice)

### R8 Code Shrinking
```kotlin
release {
    isMinifyEnabled = true       // remove unused code → faster class loading
    isShrinkResources = true     // remove unused resources → smaller APK
    proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
}
```
- Always test release build — R8 may strip reflection/serialization code

---

## Frozen Frames & Google Play Vitals

**Core vitals (official Play bad-behavior thresholds):**

| Metric | Definition | Bad Behavior Threshold |
|---|---|---|
| **ANR rate** | Main thread blocked > timeout | > **0.47%** of daily active users (user-perceived ANRs) |
| **Crash rate** | — | > **1.09%** of daily active users (user-perceived crashes) |
| **Excessive partial wake locks** | Wake lock held too long | > **5%** of sessions |

**Rendering metrics (monitored vitals — no single official "bad behavior" threshold):**

| Metric | Definition | Aspirational Target |
|---|---|---|
| **Slow Frame** | Frame > **16ms** | Minimize slow rendering |
| **Frozen Frame** | Frame > **700ms** | < **0.1%** of sessions |

- Frozen frames caused by same root issues as ANR (but shorter duration)
- Monitor: `Play Console → Android Vitals` — ANR/crash/wake locks are **core vitals**; rendering (slow/frozen frames) is **monitored, not core**
- ADB: `adb shell dumpsys gfxinfo com.your.package` — check `Janky frames` %

---

## Tools Summary

| Tool | Finds | Location |
|---|---|---|
| **CPU Profiler (System Trace)** | Frame drops, slow methods | Android Studio |
| **Memory Profiler** | Leaks, GC pressure, heap dumps | Android Studio |
| **Layout Inspector** | Recompositions, hierarchy, overdraw | Android Studio |
| **LeakCanary** | Exact leak reference chains | `debugImplementation` |
| **Perfetto** | Deep system-level trace | ui.perfetto.dev |
| **GPU Overdraw** | Redundant pixel draws | Device Developer Options |
| **Profile GPU Rendering** | Frame timing bars vs 16ms line | Device Developer Options |
| **StrictMode** | Main thread violations | Code (debug builds) |
| **Compose Compiler Metrics** | Unstable/non-skippable composables | Build output |
| **Macrobenchmark** | Startup, scroll perf (CI) | Test module |
| **ADB gfxinfo / meminfo** | Janky frame %, heap usage | Terminal |
| **ANR Traces** | Deadlocks, blocked main thread | `/data/anr/traces.txt` |
| **Google Play Android Vitals** | ANR/crash/slow/frozen frame rates | Play Console |
| **App Startup Library** | ContentProvider init overhead | Jetpack Library |
| **R8 / ProGuard** | Unused code, large APK size | Build config |
| **Room Database Inspector** | Slow queries, missing indices | Android Studio |
| **SplashScreen API** | Cold start blank screen | AndroidX Library |

> Always profile on a **real device** in **release mode** — debug builds run significantly slower due to interpreter mode.
