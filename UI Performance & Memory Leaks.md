# 🚀 Android Performance Issues & UI Jank — Complete Developer Guide

---
## Best Steps
1. R8
2. Bitmaps
3. Memory Leaks
4. Profiling Manager (Perfetto {App based, Event based})
5. 


## 📑 Index

1. [📌 What is UI Jank?](#-what-is-ui-jank)
2. [❓ Why Does It Happen?](#-why-does-it-happen)
3. [❓ How It Happens — The Frame Pipeline](#-how-it-happens--the-frame-pipeline)
4. [🔍 How to Identify Bottlenecks — Tools & Techniques](#-how-to-identify-bottlenecks--tools--techniques)
   - [1. 🟢 Android Studio — Profiler](#1--android-studio--profiler-most-important-tool)
   - [2. 🟡 Systrace / Perfetto](#2--systrace--perfetto-deep-system-level-tracing)
   - [3. 🔵 GPU Overdraw Visualization](#3--gpu-overdraw-visualization)
   - [4. 🟠 Profile GPU Rendering (On-Device)](#4--profile-gpu-rendering-on-device)
   - [5. 🔴 StrictMode (Catch Issues in Code)](#5--strictmode-catch-issues-in-code)
   - [6. 🟣 Jetpack Compose — Recomposition Tracker](#6--jetpack-compose--recomposition-tracker)
   - [7. ⚡ Baseline Profiles (Measure & Improve Startup)](#7--baseline-profiles-measure--improve-startup)
   - [8. 🖥️ ADB Commands for Performance Analysis](#8-️-adb-commands-for-performance-analysis)
5. [🧠 Memory Leaks — Deep Dive](#-memory-leaks--deep-dive)
   - [📌 What is a Memory Leak?](#-what-is-a-memory-leak)
   - [❓ Why Does It Happen?](#-why-does-it-happen-1)
   - [❓ How It Happens — The Leak Chain](#-how-it-happens--the-leak-chain)
   - [🔍 How to Identify Memory Leaks](#-how-to-identify-memory-leaks)
     - [🔴 LeakCanary](#-leakcanary-best-tool--must-have)
     - [🟠 Android Studio Memory Profiler](#-android-studio-memory-profiler)
     - [🟡 ADB & dumpsys meminfo](#-adb--dumpsys-meminfo)
   - [🛠️ How to Avoid & Fix Memory Leaks](#️-how-to-avoid--fix-memory-leaks)
     - [Fix 1: Never Store Activity/View in Static Fields](#-fix-1-never-store-activityview-in-static-fields)
     - [Fix 2: Avoid Non-Static Inner Classes & Anonymous Classes](#-fix-2-avoid-non-static-inner-classes--anonymous-classes)
     - [Fix 3: Always Unregister Listeners & Callbacks](#-fix-3-always-unregister-listeners--callbacks)
     - [Fix 4: Scope Coroutines to Lifecycle](#-fix-4-scope-coroutines-to-lifecycle)
     - [Fix 5: Never Hold View References in ViewModel](#-fix-5-never-hold-view-references-in-viewmodel)
     - [Fix 6: Null Out ViewBinding in Fragments](#-fix-6-null-out-viewbinding-in-fragments)
     - [Fix 7: Always Close Closeable Resources](#-fix-7-always-close-closeable-resources)
     - [Fix 8: Compose — Avoid Capturing Stale State in Long-Lived Lambdas](#-fix-8-compose--avoid-capturing-stale-state-in-long-lived-lambdas)
   - [📊 Memory Leak Quick Reference](#-memory-leak-quick-reference)
6. [🛑 ANR (Application Not Responding) — Deep Dive](#-anr-application-not-responding--deep-dive)
   - [📌 What is an ANR?](#-what-is-an-anr)
   - [❓ Common ANR Causes](#-common-anr-causes)
   - [🔍 How to Diagnose ANRs](#-how-to-diagnose-anrs)
   - [🛠️ How to Fix ANRs](#️-how-to-fix-anrs)
7. [🛠️ How to Fix Common Issues](#️-how-to-fix-common-issues)
   - [Fix 1: Move Heavy Work Off Main Thread](#-fix-1-move-heavy-work-off-main-thread)
   - [Fix 2: Reduce Overdraw](#-fix-2-reduce-overdraw)
   - [Fix 3: Fix Recomposition (Compose)](#-fix-3-fix-recomposition-compose)
   - [Fix 4: Avoid Allocations in Draw](#-fix-4-avoid-allocations-in-draw)
   - [Fix 5: Use LazyColumn/LazyRow or RecyclerView](#-fix-5-use-lazycolumnlazyrow-compose-or-recyclerview-xml)
   - [Fix 6: Use `derivedStateOf` for Computed State](#-fix-6-use-derivedstateof-for-computed-state-compose)
   - [Fix 7: Image Loading — Use Coil/Glide](#-fix-7-image-loading--use-coilglide)
   - [Fix 8: RecyclerView Optimization (Views/XML)](#-fix-8-recyclerview-optimization-viewsxml)
   - [Fix 9: Use `repeatOnLifecycle` for Safe Flow Collection](#-fix-9-use-repeatonlifecycle-for-safe-flow-collection)
   - [Fix 10: Optimize App Startup with App Startup Library](#-fix-10-optimize-app-startup-with-app-startup-library)
   - [Fix 11: Enable R8 Code Shrinking & Optimization](#-fix-11-enable-r8-code-shrinking--optimization)
   - [Fix 12: Compose Lazy List Performance (`key`, `contentType`)](#-fix-12-compose-lazy-list-performance-key-contenttype)
   - [Fix 13: Flatten Layout Hierarchy](#-fix-13-flatten-layout-hierarchy-constraintlayout-merge-viewstub)
   - [Fix 14: Bitmap Sampling & Memory Management](#-fix-14-bitmap-sampling--memory-management)
   - [Fix 15: Database / Room Performance](#-fix-15-database--room-performance)
   - [Fix 16: Startup Optimization (Lazy Init, Deferred Init, Splash Screen)](#-fix-16-startup-optimization-lazy-init-deferred-init-splash-screen)
8. [📈 Frozen Frames & Slow Frames — Google Play Vitals](#-frozen-frames--slow-frames--google-play-vitals)
   - [📌 What Are They?](#-what-are-they)
   - [❓ Play Vitals Thresholds (Bad Behavior)](#-play-vitals-thresholds-bad-behavior)
   - [🔍 How to Monitor](#-how-to-monitor)
9. [📊 Performance Checklist](#-performance-checklist)
10. [🧰 Tools Summary](#-tools-summary)

---

## 📌 What is UI Jank?

**Jank** occurs when your app **misses frame rendering deadlines**, causing visible stuttering, freezing, or dropped frames in the UI.

- Android targets **60 FPS** → each frame must render in **≤ 16.6ms**
- With 90/120Hz displays → **≤ 11.1ms / 8.3ms** per frame
- If the main thread is **blocked or overloaded**, the GPU waits → frame is dropped → **JANK**

---

## ❓ Why Does It Happen?

| Root Cause | Description |
|---|---|
| **Heavy work on Main Thread** | Network calls, DB queries, file I/O blocking the UI thread |
| **Complex Layouts** | Deep view hierarchies, redundant measure/layout passes |
| **Overdraw** | Pixels drawn multiple times per frame (wasted GPU cycles) |
| **Memory Pressure / GC** | Heavy allocation makes garbage collection compete for CPU with rendering (plus brief pauses), dropping frames |
| **Bitmap/Image Misuse** | Loading large bitmaps on main thread, no caching |
| **Recomposition storms (Compose)** | Unstable lambdas/objects triggering unnecessary recompositions |
| **Animation on Main Thread** | Heavy animations not offloaded to RenderThread |
| **Slow RecyclerView** | Heavy `onBindViewHolder`, no `DiffUtil`, no view recycling |
| **Object Allocation in draw()** | Creating objects inside `onDraw()` triggers GC every frame |
| **Synchronous SharedPreferences** | `commit()` blocks main thread until disk write completes; use `apply()` |
| **Too many ContentProviders at startup** | Each `ContentProvider` adds overhead to cold start; use App Startup library |

---

## ❓ How It Happens — The Frame Pipeline

```
App Code → Main Thread → RenderThread → GPU → Display
              ↑
         If THIS is slow → JANK
```

Every frame:
1. `Choreographer` receives a VSYNC signal (emitted by the display/SurfaceFlinger) and posts its frame callback (`doFrame`)
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
# Capture a trace (use /data/local/tmp/ — writable without root on debug builds)
adb shell perfetto -o /data/local/tmp/trace.pb \
  -t 10s sched freq idle am wm gfx view binder_driver

# Pull the trace
adb pull /data/local/tmp/trace.pb ~/trace.pb
```
> **Note:** `/data/misc/perfetto-traces/` may require elevated permissions on some devices. Use `/data/local/tmp/` for broader compatibility on debug builds.

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
| ⬜ True color | No overdraw (perfect) |
| 🟦 Blue | Overdrawn 1 time — drawn 2× total (acceptable) |
| 🟩 Green | Overdrawn 2 times — drawn 3× total (warning) |
| 🟪 Pink | Overdrawn 3 times — drawn 4× total (fix this!) |
| 🟥 Red | Overdrawn 4+ times — drawn 5× or more (critical!) |

**Fix:** Remove unnecessary backgrounds, use `canvas.clipRect()`, flatten layouts.

---

### 4. 🟠 Profile GPU Rendering (On-Device)

> **Profile GPU Rendering** is a lightweight, always-on diagnostic tool built into Android Developer Options. It renders a scrolling bar chart directly over your app in real time, where each vertical bar represents the time taken to render a single frame. This gives you an immediate, visual indication of whether your app is consistently hitting the 16ms target or regularly dropping frames — no computer or IDE required. It's a great first check when testing on a physical device in the field.

```
Developer Options → Profile GPU Rendering → On screen as bars
```

- Each **bar** = one frame
- Bar height = render time
- **Green line** = **16.67ms** threshold (~60 FPS budget)
- Bars above green = dropped frames = **JANK**

**Bar segments explained (Android 6.0+, bottom to top in the stacked bar):**
```
Misc Time/VSync Delay | Input Handling & Animation | Measure/Layout | Draw | Sync & Upload | Command Issue | Swap Buffers
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

> ⚠️ Since **Kotlin 2.0+**, the Compose compiler is integrated as a Kotlin compiler plugin (`org.jetbrains.kotlin.plugin.compose`). The old `composeOptions { kotlinCompilerExtensionVersion }` block is no longer needed. Configure metrics via the `composeCompiler` block instead.

```kotlin
// In your module's build.gradle.kts (Kotlin 2.0+ / Compose Compiler Plugin)
plugins {
    id("org.jetbrains.kotlin.plugin.compose")
}

composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_metrics")
    metricsDestination = layout.buildDirectory.dir("compose_metrics")
}
```

<details>
<summary>Legacy approach (Kotlin < 2.0)</summary>

```kotlin
composeOptions {
    kotlinCompilerExtensionVersion = "1.5.x"
}

tasks.withType<org.jetbrains.kotlin.gradle.tasks.KotlinCompile>().configureEach {
    compilerOptions {
        freeCompilerArgs.addAll(
            "-P", "plugin:androidx.compose.compiler.plugins.kotlin:reportsDestination=${project.layout.buildDirectory.get()}/compose_metrics",
            "-P", "plugin:androidx.compose.compiler.plugins.kotlin:metricsDestination=${project.layout.buildDirectory.get()}/compose_metrics"
        )
    }
}
```
</details>

Then check `/build/compose_metrics/` for:
- `*-composables.txt` — which composables are restartable/skippable
- `*-classes.txt` — which classes are stable/unstable

---

### 7. ⚡ Baseline Profiles (Measure & Improve Startup)

> **Baseline Profiles** are a mechanism supported on Android 7+ (API 24+) that allow you to ship a pre-compiled list of critical code paths with your app, so the Android Runtime (ART) can ahead-of-time (AOT) compile them on device installation instead of interpreting them at runtime. This dramatically reduces **cold startup time** (often by **about 30%**) and eliminates early-launch jank. The **Macrobenchmark** library complements this by giving you a repeatable, automated way to measure startup time, scroll performance, and other user-perceived metrics in a realistic environment — on real devices, in release-mode builds — making it ideal for CI integration and regression detection.

```kotlin
// Add to your app
implementation("androidx.profileinstaller:profileinstaller:1.4.1")
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
Janky frames: 12 (2.40%)   ← aim for as low as possible (no official threshold; keep below 1% as a rule of thumb)
50th percentile: 6ms
90th percentile: 12ms
95th percentile: 18ms       ← above 16.67ms = jank!
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
Heap grows over time → more frequent GC competes with rendering for CPU → dropped frames → eventually OOM crash
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
| **Compose: stale state read in lambdas** | Capturing a `State<T>`/`StateFlow` wrapper in a long-lived lambda instead of reading the current value — a correctness/stale-read pitfall, not a leak (see Fix 8) |

---

### ❓ How It Happens — The Leak Chain

```
Activity (destroyed on rotation)
    ↑ referenced by
MyListener (registered but never unregistered)
    ↑ referenced by
SomeManager.instance (singleton — its static field is the GC root, so it lives forever)

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

> 💡 Check the [GitHub releases](https://github.com/square/leakcanary/releases) for the latest stable version — v3.0 was released as stable after v2.14. Update the version number in your build file accordingly.

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
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.text.collect { binding.textView.text = it }
            }
        }
    }
}
```

> ⚠️ **Important:** Always use `repeatOnLifecycle` when collecting `Flow` in Activities/Fragments. Without it, `lifecycleScope.launch { flow.collect }` continues collecting even when the app is in the **background**, wasting resources and potentially updating destroyed UI. `repeatOnLifecycle(STARTED)` automatically starts collection when the lifecycle reaches `STARTED` and cancels it when it drops below — safe by default.

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

#### ✅ Fix 8: Compose — Read the Current Value, Don't Capture the State Wrapper

```kotlin
// ❌ SUBOPTIMAL — passing the State/StateFlow wrapper to a callback means the
//                lambda reads through the wrapper, risking a stale read (not a leak)
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
    Button(onClick = { process(state) }) { // captures the current snapshot value — correct
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

## 🛑 ANR (Application Not Responding) — Deep Dive

---

### 📌 What is an ANR?

An **ANR (Application Not Responding)** dialog is shown to the user when the main thread is blocked for too long:

- **Input event not handled** within **5 seconds** (tap, key press)
- **BroadcastReceiver.onReceive()** not finished within **10 seconds** (foreground) or **60 seconds** (background) on **Android 13 and lower**. On **Android 14+**, timeouts dynamically scale to **10–20 seconds** (foreground priority intents) and **60–120 seconds** (background priority intents) depending on CPU starvation.
- **Service.onCreate() / onStartCommand()** not finished within **20 seconds** (foreground) or **200 seconds** (background, API 26+)

ANR is more severe than jank — it's a **hard system timeout** that gives the user the option to **force-close your app**.

```
User taps button
        ↓
Main thread is blocked (e.g., DB query, network call, deadlock)
        ↓
5 seconds pass with no response
        ↓
System shows "App isn't responding" dialog
        ↓
User taps "Close app" → your app is killed
```

---

### ❓ Common ANR Causes

| Cause | Example |
|---|---|
| **Network on main thread** | HTTP request in `onClick` handler |
| **Heavy DB query on main thread** | Room query without `suspend` / background thread |
| **Deadlock** | Two threads waiting on each other's locks |
| **Long BroadcastReceiver** | Doing heavy work in `onReceive()` |
| **Slow ContentProvider** | Blocking query in a `ContentProvider` called from main thread |
| **Binder call to blocked process** | IPC call where the remote process is stuck |
| **Disk I/O on main thread** | Reading/writing SharedPreferences synchronously with `commit()` |

---

### 🔍 How to Diagnose ANRs

#### 1. ANR Trace Files
```bash
# Pull ANR traces from device
adb pull /data/anr/traces.txt ~/anr_traces.txt

# Or for modern devices (Android 11+)
adb bugreport ~/bugreport.zip
```

#### 2. Google Play Console
- **Android Vitals** → **ANR rate** dashboard
- Shows stack traces from real user devices
- Target: **< 0.47%** ANR rate (bad behavior threshold)

#### 3. StrictMode (as described above)
Catches the violations that lead to ANRs before they happen.

---

### 🛠️ How to Fix ANRs

```kotlin
// ❌ BAD — SharedPreferences.commit() blocks main thread until write completes
fun savePreference(key: String, value: String) {
    prefs.edit().putString(key, value).commit() // synchronous write — can cause ANR!
}

// ✅ GOOD — apply() writes asynchronously
fun savePreference(key: String, value: String) {
    prefs.edit().putString(key, value).apply() // async — returns immediately
}

// ❌ BAD — BroadcastReceiver does heavy work in onReceive
class MyReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val data = downloadFromNetwork() // blocks for seconds → ANR!
        processData(data)
    }
}

// ✅ GOOD — delegate to a Worker or coroutine
class MyReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val pendingResult = goAsync() // extend BroadcastReceiver deadline
        CoroutineScope(Dispatchers.IO).launch {
            try {
                val data = downloadFromNetwork()
                processData(data)
            } finally {
                pendingResult.finish() // signal completion
            }
        }
    }
}
```

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

> 💡 **Note (Kotlin 2.0+):** **Strong Skipping Mode** is now enabled by default. It was introduced as an opt-in experimental feature in Compose Compiler 1.5.4, and became **enabled by default** starting with the Compose Compiler bundled in **Kotlin 2.0.20**. With strong skipping, the Compose compiler automatically remembers lambdas in composable functions, so manually wrapping lambdas with `remember` is largely **no longer necessary**. However, you should still ensure your data classes use **immutable types** to be considered stable. The examples below still apply to older versions or cases where strong skipping is disabled.

```kotlin
// ❌ BAD — lambda is unstable, triggers recomposition (relevant pre-Strong Skipping)
@Composable
fun Parent() {
    Child(onClick = { doSomething() }) // new lambda every recomposition!
}

// ✅ GOOD — stable reference (still recommended for clarity, but auto-handled with Strong Skipping)
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

### ✅ Fix 8: RecyclerView Optimization (Views/XML)

> **RecyclerView** is the standard component for displaying large scrollable lists in View-based Android apps. Poor RecyclerView usage — heavy `onBindViewHolder`, no `DiffUtil`, no `setHasFixedSize` — is one of the most frequent causes of scroll jank. Proper optimization ensures smooth 60fps scrolling even with thousands of items.

```kotlin
// ❌ BAD — inefficient adapter: no DiffUtil, recreates list on every update
class MyAdapter(var items: List<Item>) : RecyclerView.Adapter<MyViewHolder>() {
    fun updateItems(newItems: List<Item>) {
        items = newItems
        notifyDataSetChanged() // nuclear option — rebinds ALL visible items!
    }

    override fun onBindViewHolder(holder: MyViewHolder, position: Int) {
        val bitmap = BitmapFactory.decodeFile(items[position].imagePath) // heavy work in bind!
        holder.imageView.setImageBitmap(bitmap)
    }
    // ...
}

// ✅ GOOD — use ListAdapter with DiffUtil for efficient, animated updates
class MyAdapter : ListAdapter<Item, MyViewHolder>(ItemDiffCallback()) {

    override fun onBindViewHolder(holder: MyViewHolder, position: Int) {
        val item = getItem(position)
        holder.bind(item) // lightweight binding — image loading delegated to Glide/Coil
    }
    // ...
}

class ItemDiffCallback : DiffUtil.ItemCallback<Item>() {
    override fun areItemsTheSame(oldItem: Item, newItem: Item) = oldItem.id == newItem.id
    override fun areContentsTheSame(oldItem: Item, newItem: Item) = oldItem == newItem
}

// Additional optimizations:
recyclerView.setHasFixedSize(true)                // skip measure pass if item size doesn't change
recyclerView.setItemViewCacheSize(20)             // keep more off-screen views in cache
recyclerView.recycledViewPool.setMaxRecycledViews(0, 30) // increase pool size for view type 0
```

**RecyclerView Performance Checklist:**
| Optimization | Why |
|---|---|
| Use `ListAdapter` + `DiffUtil` | Only updates changed items, adds animations |
| `setHasFixedSize(true)` | Avoids unnecessary `requestLayout()` calls |
| Load images with Coil/Glide in `onBind` | Async, cached, lifecycle-aware |
| Avoid inflation in `onBindViewHolder` | Only inflate in `onCreateViewHolder` |
| Use `RecycledViewPool` for shared pools | Multiple RecyclerViews share recycled views |
| Use `setItemViewCacheSize()` | Reduces rebinding on small scrolls |

---

### ✅ Fix 9: Use `repeatOnLifecycle` for Safe Flow Collection

> When collecting `Flow` in Activities or Fragments, using `lifecycleScope.launch { flow.collect }` keeps the collection active even when the UI is in the background. This wastes resources, may cause crashes (updating destroyed views), and is a common mistake. The `repeatOnLifecycle` API solves this by automatically starting and stopping collection based on the lifecycle state.

```kotlin
// ❌ BAD — collects even when activity is in the background (not visible)
class MyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleScope.launch {
            viewModel.uiState.collect { state ->
                updateUI(state) // runs even when app is backgrounded!
            }
        }
    }
}

// ✅ GOOD — collect only when lifecycle is at least STARTED
class MyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    updateUI(state) // only runs when visible
                }
            }
        }
    }
}

// ✅ GOOD (Compose) — collectAsStateWithLifecycle handles this automatically
@Composable
fun MyScreen(viewModel: MyViewModel) {
    val state by viewModel.uiState.collectAsStateWithLifecycle() // lifecycle-safe
    // ...
}
```

> 💡 In **Compose**, prefer `collectAsStateWithLifecycle()` over `collectAsState()` — it automatically pauses collection when the app moves to the background.

---

### ✅ Fix 10: Optimize App Startup with App Startup Library

> The **Jetpack App Startup** library (`androidx.startup`) provides a performant way to initialize components at application startup. Instead of using multiple `ContentProvider` entries (each adds ~2ms to startup), it consolidates initialization into a single `ContentProvider` — reducing cold start time.

```kotlin
// build.gradle.kts
implementation("androidx.startup:startup-runtime:1.2.0")
```

```kotlin
// Define an initializer
class WorkManagerInitializer : Initializer<WorkManager> {
    override fun create(context: Context): WorkManager {
        val config = Configuration.Builder().build()
        WorkManager.initialize(context, config)
        return WorkManager.getInstance(context)
    }

    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}

// Chain dependencies — TimberInitializer runs first, then AnalyticsInitializer
class AnalyticsInitializer : Initializer<Analytics> {
    override fun create(context: Context): Analytics {
        return Analytics.init(context)
    }

    override fun dependencies(): List<Class<out Initializer<*>>> {
        return listOf(TimberInitializer::class.java) // runs after Timber
    }
}
```

```xml
<!-- AndroidManifest.xml — single ContentProvider replaces multiple -->
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data
        android:name="com.example.WorkManagerInitializer"
        android:value="androidx.startup" />
    <meta-data
        android:name="com.example.AnalyticsInitializer"
        android:value="androidx.startup" />
</provider>
```

---

### ✅ Fix 11: Enable R8 Code Shrinking & Optimization

> **R8** (replaced ProGuard as the default shrinker in AGP 3.4; **R8 full mode** became opt-in in AGP 7.0 and is enabled by default from AGP 8.0+) removes unused code, optimizes bytecode, and obfuscates class/method names. This reduces APK size, lowers memory usage, and can improve runtime performance by enabling more aggressive optimizations like class merging and inlining. It should **always** be enabled for release builds.

```kotlin
// build.gradle.kts (module-level)
android {
    buildTypes {
        release {
            isMinifyEnabled = true       // enable R8 code shrinking
            isShrinkResources = true     // remove unused resources
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

**Impact on performance:**
| Optimization | Effect |
|---|---|
| Code shrinking | Removes unused classes/methods → smaller DEX → faster class loading |
| Resource shrinking | Removes unused drawables, layouts → smaller APK → faster install |
| Obfuscation | Shorter class/method names → smaller DEX |
| Optimization | Inlining, dead code removal, class merging → faster bytecode |

> ⚠️ Always test your release build thoroughly — R8 may remove code it thinks is unused (reflection, serialization). Add ProGuard keep rules as needed.

---

### ✅ Fix 12: Compose Lazy List Performance (`key`, `contentType`)

> When using `LazyColumn` / `LazyRow` in Jetpack Compose, failing to provide stable **keys** and **content types** is one of the most common causes of scroll jank. Without keys, Compose cannot efficiently reuse or reorder items. Without content types, Compose cannot reuse compositions across different item types — it destroys and recreates them instead.

```kotlin
// ❌ BAD — no key, no contentType → Compose cannot efficiently reuse or reorder items
LazyColumn {
    items(users) { user ->
        UserCard(user)
    }
}

// ✅ GOOD — provide stable key + contentType for efficient reuse and reorder
LazyColumn {
    items(
        items = users,
        key = { it.id },                    // stable unique key → enables reordering without recomposition
        contentType = { "user_card" }       // items with same contentType share compositions
    ) { user ->
        UserCard(user)
    }
}

// ✅ GOOD — mixed content types for heterogeneous lists
LazyColumn {
    items(
        items = feedItems,
        key = { it.id },
        contentType = { item ->
            when (item) {
                is FeedItem.Post -> "post"
                is FeedItem.Ad -> "ad"
                is FeedItem.Header -> "header"
            }
        }
    ) { item ->
        when (item) {
            is FeedItem.Post -> PostCard(item)
            is FeedItem.Ad -> AdBanner(item)
            is FeedItem.Header -> SectionHeader(item)
        }
    }
}
```

**Why this matters:**
| Parameter | Without It | With It |
|---|---|---|
| `key` | Items recompose on every list change; reordering destroys & recreates | Items maintain identity; reordering is an efficient move operation |
| `contentType` | All items share one composition pool; type changes destroy & recreate | Items of same type reuse compositions; type changes are efficient |

> 💡 **Rule of thumb:** Always provide `key` if your items have a stable unique ID (e.g., database primary key, UUID). Always provide `contentType` if your list contains **more than one type** of item.

---

### ✅ Fix 13: Flatten Layout Hierarchy (ConstraintLayout, `<merge>`, `<ViewStub>`)

> Deep view hierarchies are one of the oldest and most impactful performance problems in Android. Each level of nesting adds another **measure and layout pass**, which runs on the main thread every frame. With deeply nested `LinearLayout`s inside `RelativeLayout`s inside `ScrollView`s, the measure/layout phase can easily exceed the 16ms budget — especially with `match_parent` and `wrap_content` mixed together, which can trigger **double-taxation** (measuring a child twice). Flattening your hierarchy to fewer levels dramatically reduces per-frame CPU time.

```xml
<!-- ❌ BAD — 4 levels deep, LinearLayout + RelativeLayout nesting causes double measure passes -->
<LinearLayout android:orientation="vertical">
    <RelativeLayout>
        <LinearLayout android:orientation="horizontal">
            <ImageView ... />
            <LinearLayout android:orientation="vertical">
                <TextView ... />
                <TextView ... />
            </LinearLayout>
        </LinearLayout>
    </RelativeLayout>
</LinearLayout>

<!-- ✅ GOOD — single ConstraintLayout, flat hierarchy, one measure pass -->
<androidx.constraintlayout.widget.ConstraintLayout>
    <ImageView
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" ... />
    <TextView
        app:layout_constraintStart_toEndOf="@id/image"
        app:layout_constraintTop_toTopOf="parent" ... />
    <TextView
        app:layout_constraintStart_toEndOf="@id/image"
        app:layout_constraintTop_toBottomOf="@id/title" ... />
</androidx.constraintlayout.widget.ConstraintLayout>
```

**Use `<merge>` to eliminate redundant root containers:**
```xml
<!-- ❌ BAD — include adds an extra FrameLayout wrapper -->
<!-- included_layout.xml -->
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android">
    <TextView android:id="@+id/title" ... />
    <TextView android:id="@+id/subtitle" ... />
</FrameLayout>

<!-- ✅ GOOD — <merge> eliminates the redundant wrapper when included -->
<!-- included_layout.xml -->
<merge xmlns:android="http://schemas.android.com/apk/res/android">
    <TextView android:id="@+id/title" ... />
    <TextView android:id="@+id/subtitle" ... />
</merge>
```

**Use `<ViewStub>` for rarely-shown views:**
```xml
<!-- ✅ GOOD — ViewStub has zero cost until inflated (error state, empty state, etc.) -->
<ViewStub
    android:id="@+id/error_stub"
    android:layout="@layout/error_view"
    android:layout_width="match_parent"
    android:layout_height="wrap_content" />
```
```kotlin
// Inflate only when needed
val errorView = binding.errorStub.inflate()
```

**Layout Optimization Rules:**
| Technique | When to Use |
|---|---|
| `ConstraintLayout` | Replace nested `LinearLayout` / `RelativeLayout` hierarchies |
| `<merge>` | Root of `<include>` layouts to avoid extra wrapper views |
| `<ViewStub>` | Views shown only on error / empty / rare states |
| Compose `Layout` | Custom measurement logic without nesting overhead |

---

### ✅ Fix 14: Bitmap Sampling & Memory Management

> Loading a full-resolution image into memory when it will be displayed at a fraction of its size is one of the most common causes of **OutOfMemoryError** and excessive GC pressure. A 4000×3000 photo occupies ~48MB of heap memory (ARGB_8888). If your `ImageView` is only 400×300, you're wasting **~47.5MB**. Android provides `BitmapFactory.Options` to **downsample** the image during decoding, loading only the pixels you need.

```kotlin
// ❌ BAD — loads full 48MB bitmap into memory for a 400×300 ImageView
val bitmap = BitmapFactory.decodeFile(path) // full resolution → OOM risk!
imageView.setImageBitmap(bitmap)

// ✅ GOOD — decode only the size you need using inSampleSize
fun decodeSampledBitmap(path: String, reqWidth: Int, reqHeight: Int): Bitmap {
    // Step 1: Read dimensions only (no pixel allocation)
    val options = BitmapFactory.Options().apply {
        inJustDecodeBounds = true
    }
    BitmapFactory.decodeFile(path, options)

    // Step 2: Calculate inSampleSize (power of 2)
    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight)

    // Step 3: Decode with downsampling
    options.inJustDecodeBounds = false
    return BitmapFactory.decodeFile(path, options)
}

fun calculateInSampleSize(options: BitmapFactory.Options, reqWidth: Int, reqHeight: Int): Int {
    val (height, width) = options.outHeight to options.outWidth
    var inSampleSize = 1
    if (height > reqHeight || width > reqWidth) {
        val halfHeight = height / 2
        val halfWidth = width / 2
        while (halfHeight / inSampleSize >= reqHeight && halfWidth / inSampleSize >= reqWidth) {
            inSampleSize *= 2
        }
    }
    return inSampleSize
}
```

**Bitmap memory management tips:**
| Technique | Benefit |
|---|---|
| `inSampleSize` | Reduces memory by 4x, 16x, 64x (power-of-2 downsampling) |
| `inBitmap` | Reuse existing bitmap memory allocation (avoids GC churn) |
| `Bitmap.Config.RGB_565` | 2 bytes/pixel instead of 4 (no alpha channel) — halves memory |
| `Bitmap.recycle()` | Explicitly free native pixel memory (rarely needed on modern Android: since API 26 pixel data lives on the native heap and is auto-released; reserve for legacy or very large bitmaps) |
| **Use Coil/Glide** | Handles all of this automatically — always prefer library over manual |

> 💡 **In production, always use Coil or Glide instead of manual bitmap decoding.** They handle sampling, caching (memory + disk), lifecycle awareness, and memory management automatically. Manual decoding is useful to understand the underlying concepts or for specialized use cases.

---

### ✅ Fix 15: Database / Room Performance

> Database operations are one of the most common sources of **main thread blocking** and **ANRs** in Android apps. Room — the recommended database layer — provides compile-time query verification and lifecycle integration, but you must still follow best practices to avoid performance pitfalls like unindexed queries, missing transactions, and synchronous access.

```kotlin
// ❌ BAD — synchronous query on main thread → jank / ANR
@Dao
interface UserDao {
    @Query("SELECT * FROM users WHERE name LIKE :search")
    fun searchUsers(search: String): List<User> // blocks main thread!
}

// ✅ GOOD — suspend function runs on background thread via coroutines
@Dao
interface UserDao {
    @Query("SELECT * FROM users WHERE name LIKE :search")
    suspend fun searchUsers(search: String): List<User>

    @Query("SELECT * FROM users WHERE name LIKE :search")
    fun searchUsersFlow(search: String): Flow<List<User>> // reactive, lifecycle-aware
}
```

**Add indices for frequently queried columns:**
```kotlin
// ❌ BAD — no index on 'email' → full table scan every query
@Entity(tableName = "users")
data class User(
    @PrimaryKey val id: Int,
    val name: String,
    val email: String
)

// ✅ GOOD — index on 'email' → fast lookups
@Entity(
    tableName = "users",
    indices = [Index(value = ["email"], unique = true)]
)
data class User(
    @PrimaryKey val id: Int,
    val name: String,
    val email: String
)
```

**Use transactions for batch operations:**
```kotlin
// ❌ BAD — 1000 individual INSERT operations → 1000 disk writes
suspend fun insertUsers(users: List<User>) {
    users.forEach { dao.insertUser(it) } // extremely slow!
}

// ✅ GOOD — single transaction wraps all inserts → one disk write
@Dao
interface UserDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(users: List<User>) // Room auto-wraps in transaction

    @Transaction
    suspend fun replaceAllUsers(newUsers: List<User>) {
        deleteAll()
        insertAll(newUsers)
    }
}
```

**Room Performance Checklist:**
| Optimization | Why |
|---|---|
| Use `suspend` functions or `Flow` | Never block main thread |
| Add `@Index` on queried columns | Avoid full table scans |
| Use `@Transaction` for batch ops | Single disk write instead of N |
| Enable WAL mode (default in Room 2.x) | Concurrent reads + writes |
| Avoid `SELECT *` — select only needed columns | Less data transferred from DB |
| Use `LIMIT` / pagination with Paging 3 | Don't load entire table into memory |

---

### ✅ Fix 16: Startup Optimization (Lazy Init, Deferred Init, Splash Screen)

> Cold startup time is the very first impression your app makes. Aim for a fast cold start (well under a second to first frame); Android Vitals flags **excessive cold startup as longer than 5 seconds**. Heavy initialization in `Application.onCreate()` — loading analytics SDKs, initializing DI frameworks, pre-fetching data — is the most common reason apps start slowly. The solution is to **defer, lazy-initialize, or background** non-critical work.

```kotlin
// ❌ BAD — everything initialized synchronously in Application.onCreate()
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        AnalyticsSDK.init(this)         // 80ms
        CrashReporter.init(this)        // 50ms
        ImageLoader.init(this)          // 40ms
        FeatureFlags.fetch(this)        // 120ms (network!)
        AdSDK.init(this)               // 100ms
        // Total: ~390ms BEFORE first Activity even starts!
    }
}

// ✅ GOOD — only critical items synchronous; rest deferred
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()

        // Critical — must be ready before first frame
        CrashReporter.init(this)        // 50ms — necessary for crash tracking

        // Deferred — initialize off the main thread after first frame
        val mainHandler = Handler(Looper.getMainLooper())
        mainHandler.post {
            // Runs after first frame is drawn
            AnalyticsSDK.init(this@MyApp)
            ImageLoader.init(this@MyApp)
        }

        // Background — no UI dependency, can run on IO thread
        CoroutineScope(Dispatchers.IO).launch {
            FeatureFlags.fetch(this@MyApp)
            AdSDK.init(this@MyApp)
        }
    }
}
```

**Lazy initialization pattern:**
```kotlin
// ✅ GOOD — initialize only when first used, not at startup
class MyApp : Application() {
    val imageLoader by lazy {
        ImageLoader.Builder(this)
            .memoryCache { MemoryCache.Builder(this).maxSizePercent(0.25).build() }
            .build()
    }

    val analytics by lazy {
        AnalyticsSDK.getInstance(this)
    }
}
```

**Use `SplashScreen` API (Android 12+) correctly:**
```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        // Install splash screen BEFORE super.onCreate()
        val splashScreen = installSplashScreen()

        // Keep splash screen visible while loading critical data
        var isReady = false
        splashScreen.setKeepOnScreenCondition { !isReady }

        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // Signal ready after critical data is loaded
        lifecycleScope.launch {
            loadCriticalData()
            isReady = true
        }
    }
}
```

**Startup Optimization Checklist:**
| Technique | What It Does |
|---|---|
| Defer non-critical init to `Handler.post{}` | Runs after first frame is drawn |
| Lazy init with `by lazy {}` | Initialize only when first accessed |
| Background init with `Dispatchers.IO` | Network/disk init off main thread |
| Use **App Startup library** | Consolidate `ContentProvider` init (see Fix 10) |
| Use **Baseline Profiles** | AOT-compile critical paths (see Section 4.7) |
| Use `SplashScreen` API | Keep splash while loading, avoid blank white screen |
| Measure with **Macrobenchmark** | Quantify cold/warm/hot start times in CI |

---

## 📈 Frozen Frames & Slow Frames — Google Play Vitals

---

### 📌 What Are They?

**Google Play Vitals** distinguishes between two types of rendering problems that affect your app's store rating and visibility:

| Metric | Definition | Threshold |
|---|---|---|
| **Slow Frame** | A frame that takes longer than **16ms** (60Hz) to render | Single frame over budget |
| **Frozen Frame** | A frame that takes longer than **700ms** to render | UI appears completely frozen to the user |

> **Frozen frames** are far more severe than slow frames — the UI is unresponsive for almost a full second. Google Play Vitals flags your app if frozen frames or excessive slow frames are detected.

---

### ❓ Play Vitals Thresholds (Bad Behavior)

**Core vitals** (affect Play Store visibility — official bad behavior thresholds):

| Metric | Bad Behavior Threshold |
|---|---|
| **ANR rate** | > **0.47%** of daily active users (user-perceived ANRs) |
| **Crash rate** | > **1.09%** of daily active users (user-perceived crashes) |
| **Excessive partial wake locks** | > **5%** of sessions |

**Rendering metrics** (tracked in Play Console → Android Vitals → Rendering — no single official "bad behavior" threshold):

| Metric | Definition | What to Target |
|---|---|---|
| **Slow Frame** | A frame taking longer than **16ms** to render | Keep janky frame % as low as possible |
| **Frozen Frame** | A frame taking longer than **700ms** to render | Zero tolerance — no frame should ever exceed 700ms |

> ⚠️ Unlike crash and ANR rates, slow rendering and frozen frames are **monitored vitals**, not core vitals with a defined bad behavior threshold that triggers store ranking penalties. However, frozen frames in particular are a strong signal of main-thread blocking and should be eliminated entirely.

---

### 🔍 How to Monitor

```bash
# Get slow/frozen frame stats via ADB
adb shell dumpsys gfxinfo com.your.package

# Look for:
# Total frames rendered: 1200
# Janky frames: 48 (4.00%)
# Number of Slow UI thread: 35
# Number of Slow bitmap uploads: 5
# Number of Slow issue draw commands: 8
# Number of frame deadline missed: 40
# Number MISSED_VSYNC: 12
```

**In Google Play Console:**
```
Play Console → Android Vitals → Rendering (slow/frozen frames — a monitored vital, not a core bad-behavior threshold)
→ "Slow rendering" and "Frozen frames" tabs
→ Drill down by device, OS version, app version
```

> 💡 **Tip:** Frozen frames are almost always caused by main thread blocking (same root causes as ANR, but shorter duration). If you fix your ANR causes, frozen frames typically disappear too.

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
| 📋 Lists | LazyColumn with `key` + `contentType`; RecyclerView with DiffUtil |
| 🖼️ Images | Loaded async with Coil/Glide (never manual `BitmapFactory` in production) |
| ⚡ Startup | Keep cold start fast (Vitals flags >5s); defer non-critical init; use `by lazy` |
| 📐 Layout | Flat hierarchy; ConstraintLayout; `<merge>` / `<ViewStub>` |
| 🔄 State | `derivedStateOf` for computed values |
| 📏 StrictMode | Enabled in DEBUG builds |
| 🔁 Flow Collection | `repeatOnLifecycle` / `collectAsStateWithLifecycle` |
| 🛑 ANR | No blocking calls on main thread, `apply()` over `commit()` |
| 📦 R8 | `isMinifyEnabled = true` and `isShrinkResources = true` in release |
| 📊 Baseline Profiles | Generated and shipped with release builds |
| 🗄️ Database | Room with `suspend` / `Flow`; indexed columns; batch `@Transaction` |
| 🖼️ Bitmaps | Downsampled with `inSampleSize` (or Coil/Glide) |
| 📈 Play Vitals | Frozen frames < 0.1%; slow rendering monitored |
| 🚀 Splash Screen | `SplashScreen` API; keep-on-screen until critical data ready |

---

## 🧰 Tools Summary

| Tool | What It Finds | Where |
|---|---|---|
| **CPU Profiler (System Trace)** | Frame drops, slow methods | Android Studio |
| **Memory Profiler** | Leaks, GC pressure, heap dumps | Android Studio |
| **Layout Inspector** | Recompositions, hierarchy, overdraw | Android Studio |
| **LeakCanary** | Exact leak reference chains | Library (debugImplementation) |
| **Perfetto / Systrace** | Deep system-level trace | ui.perfetto.dev |
| **GPU Overdraw** | Redundant pixel draws | Device Developer Options |
| **Profile GPU Rendering** | Frame timing bars | Device Developer Options |
| **StrictMode** | Main thread violations | Code |
| **Compose Compiler Metrics** | Unstable composables | Build output |
| **Macrobenchmark** | Startup, scroll perf | CI / Test |
| **ADB gfxinfo / meminfo** | Janky frame %, heap usage | Terminal |
| **ANR Traces** | Deadlocks, blocked main thread | `/data/anr/traces.txt` |
| **Google Play Android Vitals** | ANR rate, crash rate, slow/frozen frames | Play Console |
| **App Startup Library** | ContentProvider init overhead | Jetpack Library |
| **R8 / ProGuard** | Unused code, large APK size | Build config |
| **Room Database Inspector** | Slow queries, missing indices | Android Studio |
| **SplashScreen API** | Cold start blank screen | AndroidX Library |

---

> 💡 **Pro Tip:** Always profile on a **real device** in **release mode** (or at least with `debuggable false`). Debug builds are significantly slower due to interpreter mode and extra checks.

