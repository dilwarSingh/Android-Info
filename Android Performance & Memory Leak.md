# 🚀 Android Performance Issues & UI Jank — Complete Developer Guide

---

## 📌 What is UI Jank?

**Jank** occurs when your app **misses frame rendering deadlines**, causing visible stuttering, freezing, or dropped frames in the UI.

- Android targets **60 FPS** → each frame must render in **≤ 16ms**
- With 90/120Hz displays → **≤ 11ms / 8ms** per frame
- If the main thread is **blocked or overloaded**, the GPU waits → frame is dropped → **JANK**

---

## ❓ Why Does It Happen?

| Root Cause | Description |
|---|---|
| **Heavy work on Main Thread** | Network calls, DB queries, file I/O blocking the UI thread |
| **Complex Layouts** | Deep view hierarchies, redundant measure/layout passes |
| **Overdraw** | Pixels drawn multiple times per frame (wasted GPU cycles) |
| **Memory Pressure / GC** | Frequent garbage collection pauses freeze the UI |
| **Bitmap/Image Misuse** | Loading large bitmaps on main thread, no caching |
| **Recomposition storms (Compose)** | Unstable lambdas/objects triggering unnecessary recompositions |
| **Animation on Main Thread** | Heavy animations not offloaded to RenderThread |
| **Slow RecyclerView** | Heavy `onBindViewHolder`, no `DiffUtil`, no view recycling |
| **Object Allocation in draw()** | Creating objects inside `onDraw()` triggers GC every frame |

---

## ❓ How It Happens — The Frame Pipeline

```
App Code → Main Thread → RenderThread → GPU → Display
              ↑
         If THIS is slow → JANK
```

Every frame:
1. `Choreographer` fires a VSYNC signal
2. Your app has **16ms** to: handle input → run logic → layout → draw
3. If step 2 takes > 16ms → the frame is **skipped**
4. User sees a **stutter**

---

## 🔍 How to Identify Bottlenecks — Tools & Techniques

---

### 1. 🟢 Android Studio — Profiler (Most Important Tool)

> Android Studio's built-in **Profiler** is your first stop for diagnosing performance issues. It gives you real-time visibility into CPU usage, memory allocation, network activity, and frame rendering — all without leaving your IDE. It is the most accessible and comprehensive tool available to Android developers and should be the default starting point for any performance investigation.

**Path:** `View → Tool Windows → Profiler`

#### CPU Profiler
- **System Trace** — Best for finding jank. Shows frame timings, Choreographer signals, RenderThread, Main Thread work
- **Java/Kotlin Method Trace** — Shows exact method call durations
- **Callstack Sample** — Low-overhead sampling profiler

**What to look for:**
```
- Long frames in the "Frames" row
- Main thread work exceeding 16ms
- "RenderThread" delays
- Frequent GC events in Memory Profiler
```

#### Memory Profiler
- Detect **memory leaks**
- Watch for frequent **GC pauses** (causes jank)
- Heap dumps to find **retained objects**

#### Layout Inspector
- **Path:** `Tools → Layout Inspector`
- See live view hierarchy
- Detect **overdraw** and **deep nesting**

---

### 2. 🟡 Systrace / Perfetto (Deep System-Level Tracing)

> **Perfetto** is a production-grade, open-source tracing tool and the modern replacement for the older Systrace. It captures a detailed timeline of everything happening on your device — CPU scheduling, memory, GPU activity, Binder IPC calls, and app-level events — all in one unified trace. It is especially powerful when Android Studio's profiler isn't enough detail, or when you need to understand interactions between your app and the Android framework at a system level. Traces are visualized at **https://ui.perfetto.dev**.

**Perfetto** (modern replacement for Systrace):
```bash
# Capture a trace
adb shell perfetto -o /data/misc/perfetto-traces/trace.pb \
  -t 10s sched freq idle am wm gfx view binder_driver

# Pull the trace
adb pull /data/misc/perfetto-traces/trace.pb ~/trace.pb
```
Open at: **https://ui.perfetto.dev**

**What to look for:**
- `Choreographer#doFrame` duration
- `View#draw`, `measure`, `layout` timings
- **Binder calls** on the main thread
- **RenderThread** busy time

---

### 3. 🔵 GPU Overdraw Visualization

> **Overdraw** happens when the same pixel is drawn more than once in a single frame — for example, a child view's background completely covers and hides the parent's background, yet both are still painted. This wastes GPU cycles and can degrade rendering performance, especially on mid/low-end devices. Android provides a built-in **color-coded overlay** directly on your device screen so you can instantly see which areas of your UI are suffering from overdraw without any code changes.

**Enable on Device:**
```
Developer Options → Debug GPU Overdraw → Show overdraw areas
```

| Color | Meaning |
|---|---|
| ⬜ No color | 1x draw (perfect) |
| 🟦 Blue | 2x overdraw (acceptable) |
| 🟩 Green | 3x overdraw (warning) |
| 🟥 Pink/Red | 4x+ overdraw (fix this!) |

**Fix:** Remove unnecessary backgrounds, use `canvas.clipRect()`, flatten layouts.

---

### 4. 🟠 Profile GPU Rendering (On-Device)

> **Profile GPU Rendering** is a lightweight, always-on diagnostic tool built into Android Developer Options. It renders a scrolling bar chart directly over your app in real time, where each vertical bar represents the time taken to render a single frame. This gives you an immediate, visual indication of whether your app is consistently hitting the 16ms target or regularly dropping frames — no computer or IDE required. It's a great first check when testing on a physical device in the field.

```
Developer Options → Profile GPU Rendering → On screen as bars
```

- Each **bar** = one frame
- Bar height = render time
- **Green line** = 16ms threshold
- Bars above green = dropped frames = **JANK**

**Bar segments explained:**
```
Input Handling | Animation | Measure/Layout | Draw | Sync | Command Issue | Swap Buffers
```

---

### 5. 🔴 StrictMode (Catch Issues in Code)

> **StrictMode** is a developer-only diagnostic tool built into the Android SDK that helps you detect accidental policy violations in your code — such as disk reads, disk writes, or network calls happening on the main (UI) thread. These violations are among the most common hidden causes of UI jank and ANRs (Application Not Responding). StrictMode can alert you via Logcat warnings, a visible screen flash, or even crash the app in debug builds, making it easy to catch these issues early during development before they ever reach users. It requires zero third-party dependencies — just a few lines of code in your `Application` or `Activity`.

Add to your `Application.onCreate()` or `Activity.onCreate()`:

```kotlin
if (BuildConfig.DEBUG) {
    StrictMode.setThreadPolicy(
        StrictMode.ThreadPolicy.Builder()
            .detectDiskReads()
            .detectDiskWrites()
            .detectNetwork()        // Catches network on main thread!
            .detectCustomSlowCalls()
            .penaltyLog()           // Log to Logcat
            .penaltyFlashScreen()   // Visual flash when violation occurs
            .build()
    )
    StrictMode.setVmPolicy(
        StrictMode.VmPolicy.Builder()
            .detectLeakedSqlLiteObjects()
            .detectLeakedClosableObjects()
            .detectActivityLeaks()
            .penaltyLog()
            .build()
    )
}
```

---

### 6. 🟣 Jetpack Compose — Recomposition Tracker

> In Jetpack Compose, **recomposition** is the process of re-running composable functions to reflect state changes. While recomposition is expected and normal, **excessive or unnecessary recomposition** is one of the most common performance pitfalls in Compose apps. It happens when composables receive unstable parameters (like lambdas recreated on every pass, or mutable data classes) that Compose cannot skip. The Recomposition Tracker — accessible via Android Studio's Layout Inspector — highlights exactly which composables are recomposing and how frequently, helping you pinpoint and eliminate wasted work. The Compose Compiler can also generate detailed metrics reports to flag unstable classes and non-skippable composables at build time.

**Enable Composition Highlighting:**
```
Android Studio → Layout Inspector → Enable "Highlight recompositions"
```

Or add logging manually:
```kotlin
@Composable
fun MyComposable(data: String) {
    SideEffect {
        Log.d("Recomposition", "MyComposable recomposed with: $data")
    }
    // ...
}
```

**Enable Compose compiler metrics** (build.gradle.kts):
```kotlin
composeOptions {
    kotlinCompilerExtensionVersion = "..."
}

// In your module's build.gradle.kts
tasks.withType<org.jetbrains.kotlin.gradle.tasks.KotlinCompile>().configureEach {
    compilerOptions {
        freeCompilerArgs.addAll(
            "-P", "plugin:androidx.compose.compiler.plugins.kotlin:reportsDestination=${project.buildDir}/compose_metrics",
            "-P", "plugin:androidx.compose.compiler.plugins.kotlin:metricsDestination=${project.buildDir}/compose_metrics"
        )
    }
}
```

Then check `/build/compose_metrics/` for:
- `*-composables.txt` — which composables are restartable/skippable
- `*-classes.txt` — which classes are stable/unstable

---

### 7. ⚡ Baseline Profiles (Measure & Improve Startup)

> **Baseline Profiles** are a mechanism introduced in Android 9 (API 28) that allow you to ship a pre-compiled list of critical code paths with your app, so the Android Runtime (ART) can ahead-of-time (AOT) compile them on device installation instead of interpreting them at runtime. This dramatically reduces **cold startup time** (often by 30–40%) and eliminates early-launch jank. The **Macrobenchmark** library complements this by giving you a repeatable, automated way to measure startup time, scroll performance, and other user-perceived metrics in a realistic environment — on real devices, in release-mode builds — making it ideal for CI integration and regression detection.

```kotlin
// Add to your app
implementation("androidx.profileinstaller:profileinstaller:1.3.1")
```

Use **Macrobenchmark** library to measure:
```kotlin
@RunWith(AndroidJUnit4::class)
class StartupBenchmark {
    @get:Rule
    val benchmarkRule = MacrobenchmarkRule()

    @Test
    fun startup() = benchmarkRule.measureRepeated(
        packageName = "com.your.app",
        metrics = listOf(StartupTimingMetric()),
        iterations = 5,
        startupMode = StartupMode.COLD
    ) {
        pressHome()
        startActivityAndWait()
    }
}
```

---

### 8. 🖥️ ADB Commands for Performance Analysis

> **ADB (Android Debug Bridge)** shell commands give you direct, scriptable access to low-level performance data from your device via the terminal — no GUI required. This makes them perfect for **CI pipelines, automated testing, and quick spot-checks** on physical devices. The `dumpsys gfxinfo` command is particularly valuable: it reports the total number of frames rendered, the percentage that were janky, and detailed frame time percentiles, letting you quantify exactly how smooth (or rough) your app's rendering is during a specific interaction.

```bash
# Check frame stats (dumpsys gfxinfo)
adb shell dumpsys gfxinfo com.your.package

# Check janky frames
adb shell dumpsys gfxinfo com.your.package framestats

# Reset stats before testing
adb shell dumpsys gfxinfo com.your.package reset

# Check memory info
adb shell dumpsys meminfo com.your.package

# Monitor CPU usage live
adb shell top -p $(adb shell pidof com.your.package)

# Check battery/doze impact
adb shell dumpsys batterystats com.your.package
```

**Reading gfxinfo output:**
```
Total frames rendered: 500
Janky frames: 12 (2.40%)   ← should be < 1%
50th percentile: 6ms
90th percentile: 12ms
95th percentile: 18ms       ← above 16ms = jank!
99th percentile: 34ms
```

---

## 🧠 Memory Leaks — Deep Dive

---

### 📌 What is a Memory Leak?

A **memory leak** occurs when your app holds a reference to an object that is **no longer needed**, preventing the Garbage Collector (GC) from reclaiming that memory. Over time, leaked objects accumulate and consume increasing amounts of heap memory until the app throws an **OutOfMemoryError (OOM)** or becomes so slow from constant GC pressure that the user experiences severe jank or crashes.

Unlike a crash that happens immediately, memory leaks are **silent and gradual** — they are often only noticed after extended use of the app.

```
Object no longer needed
        ↓
Still referenced by a long-lived object (e.g., static field, singleton)
        ↓
GC cannot collect it
        ↓
Heap grows over time → GC runs more often → UI pauses → OOM crash
```

---

### ❓ Why Does It Happen?

Android's lifecycle model is the primary culprit. Activities, Fragments, and Views are **created and destroyed frequently** (on rotation, navigation, etc.), but if anything outside their lifecycle holds a reference to them, they cannot be garbage collected.

| Common Cause | Explanation |
|---|---|
| **Static references to Context/Activity** | A static field holding an `Activity` context keeps the entire Activity in memory forever |
| **Inner classes & anonymous classes** | Non-static inner classes implicitly hold a reference to the outer class (e.g., `Activity`) |
| **Unregistered listeners/callbacks** | Registering a listener (sensor, location, broadcast) and forgetting to unregister it in `onStop`/`onDestroy` |
| **Long-lived coroutines/threads** | A coroutine or thread started in an Activity that outlives the Activity's lifecycle |
| **Handler & Runnable references** | `Handler.postDelayed()` with a `Runnable` that captures `this` (Activity) keeps it alive |
| **Singleton holding Context** | A singleton initialized with `Activity` context instead of `Application` context |
| **Bitmap not recycled** | Large `Bitmap` objects held in memory after they are no longer displayed |
| **ViewModel holding View refs** | ViewModel storing a `View` or `Activity` reference that survives configuration changes |
| **Closeable resources not closed** | `Cursor`, `InputStream`, `SQLiteDatabase` left open leaks both memory and file descriptors |
| **Compose: captured State in lambdas** | Lambdas in Compose capturing `State<T>` directly instead of reading its `.value` |

---

### ❓ How It Happens — The Leak Chain

```
Activity (destroyed on rotation)
    ↑ referenced by
MyListener (registered but never unregistered)
    ↑ referenced by
SomeManager.instance (singleton — lives forever)
    ↑ referenced by
Application (never destroyed)

Result: Activity is never GC'd → LEAK
```

Every rotation = another leaked Activity instance. After 10 rotations = 10 Activity instances still in memory.

---

### 🔍 How to Identify Memory Leaks

---

#### 🔴 LeakCanary (Best Tool — Must Have)

> **LeakCanary** is an open-source memory leak detection library by Square. It automatically watches destroyed objects (Activities, Fragments, ViewModels, Views), takes a heap dump when they are not GC'd, analyzes the heap, and shows you the **exact reference chain** causing the leak — with a notification directly on your device. It requires almost zero setup and should be part of every Android project's debug dependencies.

Add to your `build.gradle.kts`:
```kotlin
dependencies {
    debugImplementation("com.squareup.leakcanary:leakcanary-android:2.14")
}
```

That's it — no code changes needed. LeakCanary automatically hooks into the Activity/Fragment lifecycle.

**What you get — a readable leak trace:**
```
┬───
│ GC Root: static field MyManager.instance
│
├─ MyManager instance
│    Leaking: NO
│    ↓ MyManager.listener
│
├─ MyActivity instance
│    Leaking: YES (Activity#mDestroyed is true)
│
╰→ Reference that caused the leak: MyManager.listener → MyActivity
```

---

#### 🟠 Android Studio Memory Profiler

> The **Memory Profiler** in Android Studio gives you a real-time graph of your app's heap usage and lets you take **heap dumps** to inspect every object in memory. It shows you object counts, retained sizes, and allows you to filter by class name to spot suspiciously high instance counts — like seeing 15 instances of `MainActivity` when there should only ever be 1.

**Path:** `View → Tool Windows → Profiler → Memory`

**Steps to find a leak:**
1. Navigate through your app (especially rotate the screen several times)
2. Click **"Dump Java Heap"** (camera icon)
3. Filter by your package name
4. Look for **Activity / Fragment classes with high instance counts**
5. Click an instance → inspect the **Reference tree** to find what's holding it

**Signs of a leak in the profiler:**
```
- Heap size grows continuously and never drops even after GC
- Multiple instances of Activity/Fragment visible in the heap dump
- Retained size of a class is unexpectedly large
- Frequent GC events (sawtooth / shark-fin pattern on the memory graph)
```

---

#### 🟡 ADB & dumpsys meminfo

```bash
# Check overall heap usage for your app
adb shell dumpsys meminfo com.your.package

# Watch memory grow in real time (updates every 2 seconds)
watch -n 2 "adb shell dumpsys meminfo com.your.package | grep 'TOTAL'"

# Force GC then capture a heap dump
adb shell am dumpheap com.your.package /data/local/tmp/heap.hprof
adb pull /data/local/tmp/heap.hprof ~/heap.hprof
```

Open the `.hprof` file in **Android Studio** (`File → Open → heap.hprof`) to analyze the object graph.

---

### 🛠️ How to Avoid & Fix Memory Leaks

---

#### ✅ Fix 1: Never Store Activity/View in Static Fields

```kotlin
// ❌ BAD — Activity lives in a static field forever
object MyManager {
    var activity: Activity? = null // LEAK!
}

// ✅ GOOD — use WeakReference if you truly need it
object MyManager {
    var activityRef: WeakReference<Activity>? = null
    val activity get() = activityRef?.get()
}

// ✅ BEST — redesign to not need the reference at all
// Use Application context for singletons — it is safe and lives as long as the process
object MyManager {
    lateinit var appContext: Context

    fun init(context: Context) {
        appContext = context.applicationContext
    }
}
```

---

#### ✅ Fix 2: Avoid Non-Static Inner Classes & Anonymous Classes

```kotlin
// ❌ BAD — anonymous Runnable implicitly captures the outer Activity
class MyActivity : AppCompatActivity() {
    val handler = Handler(Looper.getMainLooper())

    override fun onStart() {
        super.onStart()
        handler.postDelayed({
            updateUI() // 'this' Activity is captured by the lambda → LEAK
        }, 5000)
    }
}

// ✅ GOOD — extract the Runnable and always remove it in the paired callback
class MyActivity : AppCompatActivity() {
    private val handler = Handler(Looper.getMainLooper())
    private val updateRunnable = Runnable { updateUI() }

    override fun onStart() {
        super.onStart()
        handler.postDelayed(updateRunnable, 5000)
    }

    override fun onStop() {
        super.onStop()
        handler.removeCallbacks(updateRunnable) // always clean up!
    }
}
```

---

#### ✅ Fix 3: Always Unregister Listeners & Callbacks

```kotlin
// ❌ BAD — listener registered, never removed → SensorManager holds Activity forever
class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        sensorManager.registerListener(this, sensor, SENSOR_DELAY_UI)
    }
    // Fragment destroyed but SensorManager still holds reference → LEAK
}

// ✅ GOOD — unregister in the symmetric lifecycle callback
class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        sensorManager.registerListener(this, sensor, SENSOR_DELAY_UI)
    }

    override fun onDestroyView() {
        super.onDestroyView()
        sensorManager.unregisterListener(this) // matched pair → no leak
    }
}
```

---

#### ✅ Fix 4: Scope Coroutines to Lifecycle

```kotlin
// ❌ BAD — GlobalScope outlives the Activity entirely
class MyActivity : AppCompatActivity() {
    fun loadData() {
        GlobalScope.launch { // still running after Activity is destroyed!
            val data = fetchData()
            withContext(Dispatchers.Main) { updateUI(data) }
        }
    }
}

// ✅ GOOD — lifecycleScope is automatically cancelled when Activity is destroyed
class MyActivity : AppCompatActivity() {
    fun loadData() {
        lifecycleScope.launch {
            val data = withContext(Dispatchers.IO) { fetchData() }
            updateUI(data) // safe — scope is tied to the lifecycle
        }
    }
}

// ✅ GOOD — viewModelScope is cancelled when ViewModel is cleared
class MyViewModel : ViewModel() {
    fun loadData() {
        viewModelScope.launch {
            val data = withContext(Dispatchers.IO) { fetchData() }
            _uiState.value = data
        }
    }
}
```

---

#### ✅ Fix 5: Never Hold View References in ViewModel

```kotlin
// ❌ BAD — ViewModel survives configuration changes, but View does not
class MyViewModel : ViewModel() {
    var textView: TextView? = null // LEAK on every rotation!
}

// ✅ GOOD — ViewModel exposes state; the UI observes it and updates itself
class MyViewModel : ViewModel() {
    private val _text = MutableStateFlow("")
    val text: StateFlow<String> = _text

    fun updateText(value: String) { _text.value = value }
}

class MyActivity : AppCompatActivity() {
    private val viewModel: MyViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleScope.launch {
            viewModel.text.collect { binding.textView.text = it }
        }
    }
}
```

---

#### ✅ Fix 6: Null Out ViewBinding in Fragments

```kotlin
// ❌ BAD — binding holds a reference to the View tree after the Fragment's
//          view is destroyed (e.g., when navigating away but Fragment is kept in back stack)
class MyFragment : Fragment(R.layout.my_fragment) {
    private val binding: MyFragmentBinding by lazy {
        MyFragmentBinding.inflate(layoutInflater)
    }
    // binding is never nulled out → entire View tree is leaked
}

// ✅ GOOD — null out binding in onDestroyView so the View tree can be GC'd
class MyFragment : Fragment(R.layout.my_fragment) {
    private var _binding: MyFragmentBinding? = null
    private val binding get() = _binding!!

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        _binding = MyFragmentBinding.bind(view)
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null // allow GC to collect the View tree
    }
}
```

---

#### ✅ Fix 7: Always Close Closeable Resources

```kotlin
// ❌ BAD — Cursor never closed → database connection + memory both leaked
fun getUser(id: Int): User? {
    val cursor = db.rawQuery("SELECT * FROM users WHERE id=?", arrayOf("$id"))
    return cursor.toUser() // if toUser() throws, cursor is never closed!
}

// ✅ GOOD — use Kotlin's 'use' extension: auto-closes on exit, even on exception
fun getUser(id: Int): User? {
    return db.rawQuery("SELECT * FROM users WHERE id=?", arrayOf("$id")).use { cursor ->
        if (cursor.moveToFirst()) cursor.toUser() else null
    }
}
```

---

#### ✅ Fix 8: Compose — Avoid Capturing Stale State in Long-Lived Lambdas

```kotlin
// ❌ BAD — passing the State object itself to a non-composable callback
//          can cause it to hold onto a stale or leaked reference
@Composable
fun MyScreen(viewModel: MyViewModel) {
    val state = viewModel.uiState        // this is a StateFlow / State<T>
    Button(onClick = { process(state) }) { // captures the State wrapper, not its current value
        Text("Click")
    }
}

// ✅ GOOD — collect state as a value and capture the value in the lambda
@Composable
fun MyScreen(viewModel: MyViewModel) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    Button(onClick = { process(state) }) { // captures the current snapshot value — safe
        Text("Click")
    }
}
```

---

### 📊 Memory Leak Quick Reference

| Scenario | Root Cause | Fix |
|---|---|---|
| Activity leaked on rotation | Static/singleton holds Activity ref | Use `WeakReference` or `applicationContext` |
| Multiple Activity instances in heap | Anonymous inner class holds outer ref | Extract Runnable / use top-level class |
| Fragment view not released | Binding not nulled in `onDestroyView` | Set `_binding = null` in `onDestroyView` |
| ViewModel crash after rotation | ViewModel holds View/Activity ref | Expose `StateFlow`, never store Views |
| App slows down over long session | Coroutine running after destroy | Use `lifecycleScope` / `viewModelScope` |
| OOM after heavy list scrolling | Bitmaps not released | Use Coil/Glide with lifecycle awareness |
| Sensor/GPS keeps running in background | Listener not unregistered | Unregister in `onDestroyView` / `onStop` |
| DB/file descriptor exhaustion | Cursor/stream not closed | Use `.use { }` or try-finally |

---

## 🛠️ How to Fix Common Issues

---

### ✅ Fix 1: Move Heavy Work Off Main Thread

```kotlin
// ❌ BAD — blocks UI
fun loadData() {
    val data = database.query() // on main thread!
    updateUI(data)
}

// ✅ GOOD — use coroutines
fun loadData() {
    viewModelScope.launch {
        val data = withContext(Dispatchers.IO) {
            database.query()
        }
        updateUI(data) // back on Main
    }
}
```

---

### ✅ Fix 2: Reduce Overdraw

```kotlin
// ❌ BAD — each parent has its own background
Column(modifier = Modifier.background(Color.White)) {
    Box(modifier = Modifier.background(Color.White)) { // redundant!
        Text("Hello")
    }
}

// ✅ GOOD — set background only once
Column(modifier = Modifier.background(Color.White)) {
    Box { Text("Hello") }
}
```

---

### ✅ Fix 3: Fix Recomposition (Compose)

```kotlin
// ❌ BAD — lambda is unstable, triggers recomposition
@Composable
fun Parent() {
    Child(onClick = { doSomething() }) // new lambda every recomposition!
}

// ✅ GOOD — stable reference
@Composable
fun Parent() {
    val onClick = remember { { doSomething() } }
    Child(onClick = onClick)
}
```

```kotlin
// ❌ BAD — unstable data class
data class User(val name: String, val list: MutableList<String>)

// ✅ GOOD — use @Stable or immutable types
@Immutable
data class User(val name: String, val list: List<String>)
```

---

### ✅ Fix 4: Avoid Allocations in Draw

```kotlin
// ❌ BAD — allocates on every frame
override fun onDraw(canvas: Canvas) {
    val paint = Paint() // allocation every frame!
    canvas.drawCircle(cx, cy, radius, paint)
}

// ✅ GOOD — allocate once
private val paint = Paint()

override fun onDraw(canvas: Canvas) {
    canvas.drawCircle(cx, cy, radius, paint)
}
```

---

### ✅ Fix 5: Use LazyColumn/LazyRow (Compose) or RecyclerView (XML)

```kotlin
// ❌ BAD — renders ALL items at once
Column {
    items.forEach { item -> ItemCard(item) }
}

// ✅ GOOD — only renders visible items
LazyColumn {
    items(items) { item -> ItemCard(item) }
}
```

---

### ✅ Fix 6: Use `derivedStateOf` for Computed State (Compose)

```kotlin
// ❌ BAD — recomposes on every scroll
@Composable
fun List(listState: LazyListState) {
    val showButton = listState.firstVisibleItemIndex > 0 // recomposes always!
}

// ✅ GOOD — only recomposes when result changes
@Composable
fun List(listState: LazyListState) {
    val showButton by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 0 }
    }
}
```

---

### ✅ Fix 7: Image Loading — Use Coil/Glide

```kotlin
// ❌ BAD — loads full bitmap on main thread
val bitmap = BitmapFactory.decodeFile(path) // OOM + jank!

// ✅ GOOD — use Coil (Compose-friendly)
AsyncImage(
    model = ImageRequest.Builder(context)
        .data(imageUrl)
        .crossfade(true)
        .build(),
    contentDescription = null
)
```

---

## 📊 Performance Checklist

| Area | Check |
|---|---|
| 🧵 Threading | No network/DB on main thread |
| 🖼️ Rendering | No overdraw (max 2x) |
| ♻️ Recomposition | All composables skippable |
| 🗑️ Memory | No leaks, minimal GC |
| 🧠 Memory Leaks | LeakCanary integrated in debug builds |
| 🔗 Listeners | All listeners/callbacks unregistered |
| 🔄 Coroutines | All coroutines scoped to lifecycle |
| 📦 ViewBinding | `_binding` nulled in `onDestroyView` |
| 📋 Lists | LazyColumn/RecyclerView for long lists |
| 🖼️ Images | Loaded async with Coil/Glide |
| ⚡ Startup | Cold start < 500ms |
| 📐 Layout | Flat hierarchy, ConstraintLayout |
| 🔄 State | `derivedStateOf` for computed values |
| 📏 StrictMode | Enabled in DEBUG builds |

---

## 🧰 Tools Summary

| Tool | What It Finds | Where |
|---|---|---|
| **CPU Profiler (System Trace)** | Frame drops, slow methods | Android Studio |
| **Memory Profiler** | Leaks, GC pressure, heap dumps | Android Studio |
| **Layout Inspector** | Recompositions, hierarchy | Android Studio |
| **LeakCanary** | Exact leak reference chains | Library (debugImplementation) |
| **Perfetto / Systrace** | Deep system-level trace | ui.perfetto.dev |
| **GPU Overdraw** | Redundant pixel draws | Device Developer Options |
| **Profile GPU Rendering** | Frame timing bars | Device Developer Options |
| **StrictMode** | Main thread violations | Code |
| **Compose Compiler Metrics** | Unstable composables | Build output |
| **Macrobenchmark** | Startup, scroll perf | CI / Test |
| **ADB gfxinfo / meminfo** | Janky frame %, heap usage | Terminal |

---

> 💡 **Pro Tip:** Always profile on a **real device** in **release mode** (or at least with `debuggable false`). Debug builds are significantly slower due to interpreter mode and extra checks.

