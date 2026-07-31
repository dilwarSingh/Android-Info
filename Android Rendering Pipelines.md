# Android Rendering Pipelines: View System vs Jetpack Compose

---

## 🖼️ Overview

Yes — the rendering pipelines of the **Android View System** and **Jetpack Compose** are fundamentally different, though both ultimately produce pixels on screen via the same GPU/display hardware.

Both pipelines share the same low-level renderer: **HWUI (Hardware UI)**, which is Android's GPU-accelerated 2D rendering engine. HWUI translates Canvas drawing commands into GPU instructions. Since Android 9, HWUI uses **Skia** as its rendering backend — initially via OpenGL ES (**SkiaGL**), with a Vulkan backend (**SkiaVk**) added in Android 10 and increasingly enabled on newer devices.

---

## 📌 Part 1: Android View System Rendering Pipeline

The traditional View system uses an **imperative, multi-phase pipeline** involving the CPU and GPU working together.

### Pipeline Stages

```
XML Inflation → Measure → Layout → Draw → RenderThread → GPU → Display
```

### 1. XML Inflation
- `LayoutInflater` parses XML files.
- Reflectively creates `View` objects (`TextView`, `LinearLayout`, etc.).
- **Cost:** Reflection is slow; deep hierarchies multiply the cost.

### 2. Measure Pass
- Each `View` calls `onMeasure(widthSpec, heightSpec)`.
- Parent asks children: *"How big do you want to be?"*
- Can be called **multiple times** (e.g., `wrap_content` in `LinearLayout`).
- Traversal: **Top-down**, recursive.

### 3. Layout Pass
- Each `View` calls `onLayout(changed, left, top, right, bottom)`.
- Parent positions its children using the sizes from measure.
- Traversal: **Top-down**, recursive.

### 4. Draw Pass (Record RenderNode / DisplayList)
- Each `View` calls `onDraw(Canvas)`.
- Canvas commands (drawText, drawRect, etc.) are recorded into a **RenderNode** (internally renamed from "DisplayList" in Android 5.0 / API 21).
- A `RenderNode` encapsulates a set of drawing commands plus transform properties (translation, scale, rotation, alpha, clip). Each `View` owns a RenderNode.
- `View.invalidate()` → marks the view dirty → triggers a **re-record of that View's RenderNode** (and its children if needed).
- Since **API 29 (Android 10)**, `RenderNode` became a **public API** — developers can create standalone RenderNodes for custom high-performance rendering without needing a View. (Before API 29, RenderNode existed internally but was not part of the public SDK.)

### 5. HWUI & RenderThread (Hardware Acceleration)
- Hardware-accelerated rendering was introduced in **Android 3.0 (Honeycomb)** and became the default in **Android 4.0 (Ice Cream Sandwich)**.
- Since **Android 5.0 (Lollipop)**, a dedicated **RenderThread** replays the recorded RenderNodes.
- The rendering engine is called **HWUI** (Hardware UI), which translates Canvas/RenderNode commands into GPU draw calls.
- **Graphics backend evolution:**
  - **Android 4.0–8.x:** OpenGL ES 2.0/3.x via HWUI's custom OpenGL renderer.
  - **Android 9 (Pie):** HWUI switched to **Skia** as its rendering backend, using OpenGL ES underneath (**SkiaGL**). This was a major internal architectural change.
  - **Android 10:** Introduced **SkiaVk** (Skia + Vulkan) as an alternative backend, initially on select Pixel/flagship devices; most devices remained on SkiaGL.
  - **Android 12–13:** Vulkan adoption grew; it became the default on Pixel and some OEM flagships, while **SkiaGL remained the default on most devices**.
  - **Android 14+:** Vulkan is the default on a growing set of new devices, but SkiaGL is still common. ANGLE (OpenGL ES → Vulkan translation layer) is available on some devices to provide OpenGL ES compatibility on top of Vulkan-only drivers.
- The **Main Thread** records; the **RenderThread** executes → parallelism.

### 6. SurfaceFlinger & Display
- RenderThread renders into a **Surface buffer** (a `BufferQueue` producer/consumer pair).
- **SurfaceFlinger** composites all app layers + system UI using:
  - **Client composition:** GPU-based compositing (when hardware overlays aren't sufficient).
  - **Device composition:** Hardware Composer (HWC) overlays layers directly on display hardware (more power-efficient).
- **Hardware Composer (HWC)** handles as many layers as possible directly in hardware.
- **Vsync** signal from display triggers each frame (16.6ms @ 60Hz, 11.1ms @ 90Hz, 8.3ms @ 120Hz).
- Since **Android 4.1 (Jelly Bean)** — **Project Butter** — the system uses **Vsync synchronization**, **triple buffering**, and **Choreographer** to coordinate frame production between the app, RenderThread, and SurfaceFlinger via phase offsets.
- Since **Android 11 (API 30)**, apps can hint their preferred frame rate to the system via `Surface.setFrameRate()`, allowing SurfaceFlinger to choose an appropriate refresh rate for the content. Note: the full system-level **Adaptive Refresh Rate (ARR)** feature — where the display refresh rate dynamically tracks the actual render rate — requires **Android 15-QPR1+** and specific HAL support.

```
┌──────────────┐    ┌────────────────┐    ┌─────────────────┐    ┌─────────┐
│  Main Thread │    │  RenderThread  │    │  SurfaceFlinger │    │ Display │
│              │    │                │    │                 │    │         │
│  Measure     │───▶│  Replay        │───▶│  Composite      │───▶│  Vsync  │
│  Layout      │    │  RenderNodes   │    │  All Layers     │    │  Pixels │
│  Draw/Record │    │  → HWUI        │    │  (HWC + GPU)    │    │         │
│              │    │  → Vulkan/GL   │    │                 │    │         │
└──────────────┘    └────────────────┘    └─────────────────┘    └─────────┘
```

### Key Pain Points
| Problem | Cause |
|---|---|
| Overdraw | Multiple `View`s paint the same pixels |
| Slow inflation | XML parsing + reflection |
| Multiple measure passes | `wrap_content`, `RelativeLayout` |
| Re-record on invalidate | `invalidate()` re-records the dirty view's RenderNode; invalidation propagates up to schedule a traversal |
| Deep hierarchy cost | Each level = one more traversal |

---

## ⚡ Part 2: Jetpack Compose Rendering Pipeline

Compose uses a **declarative, reactive, slot-table based pipeline** — a completely reimagined approach.

### Pipeline Stages

```
Composition → SnapshotState Change → Recomposition → Layout → Drawing → RenderThread → GPU → Display
```

### 1. Composition (First Run)
- Compose executes your `@Composable` functions.
- Builds a **Slot Table** (a gap buffer data structure) storing the UI tree in memory.
- Creates a **LayoutNode** tree (Compose's equivalent of View tree).
- **No XML, no reflection** — pure Kotlin function calls.

#### Compose Compiler Plugin (Kotlin 2.0+)
- The **Compose Compiler Plugin** transforms `@Composable` functions at compile time, inserting code for recomposition tracking, group management, and state observation.
- Since **Kotlin 2.0 (May 2024)**, the Compose Compiler Plugin is **bundled directly into the Kotlin compiler** (`org.jetbrains.kotlin.plugin.compose` Gradle plugin). You no longer need a separate `androidx.compose.compiler:compiler` dependency or worry about Kotlin-Compose compiler version compatibility.

### 2. SnapshotState System
- State variables (`mutableStateOf`, `remember`) are backed by Compose's **Snapshot system**.
- When state changes → Compose marks only the **affected composable scopes** as invalid.
- This is **fine-grained reactivity** — not full-tree re-evaluation.

### 3. Recomposition (Smart Re-execution)
- Only the **invalidated composable lambdas** re-execute.
- Compose **skips** composables whose inputs haven't changed (`@Stable`, `@Immutable` help here).
- Updates the **Slot Table** with new values.
- **No measure/layout/draw happens yet** — just Kotlin function execution.

#### Strong Skipping Mode (Default since Kotlin 2.0.20)
- **Strong Skipping Mode** was introduced experimentally in Compose Compiler 1.5.4 and became the **default behavior in Kotlin 2.0.20** (not "Compose 1.7" — the gate is the Kotlin compiler version, not the Compose runtime version).
- With strong skipping:
  - **All restartable composable functions are skippable by default**, even those with unstable parameters. (Non-restartable composables remain unskippable regardless of this mode.)
  - Unstable parameters are compared using **instance equality** (`===`) instead of structural equality.
  - Stable parameters continue to use **structural equality** (`equals()`).
  - **Lambdas are automatically remembered** — no need to manually wrap them in `remember { }`.
- This significantly **reduces the need** for `@Stable` and `@Immutable` annotations, though they still provide optimization benefits (enabling `equals()` comparison instead of `===` for your types).

### 4. Layout Phase

Compose's Layout phase has **two sub-steps**:

#### 4a. Measure
- Each `LayoutNode` calls `measure(constraints)` on its children — typically **exactly once** per layout pass.
- Measuring is single-pass by convention, which keeps layout fast; re-measuring a child repeatedly is discouraged but not forbidden. APIs like `SubcomposeLayout`, `BoxWithConstraints`, intrinsics, and lookahead legitimately measure more than once.
- `Constraints` (minWidth, maxWidth, minHeight, maxHeight) flow **top-down**.
- Children report their chosen size **bottom-up**.
- **Intrinsic measurements** (`minIntrinsicWidth`, `maxIntrinsicWidth`, etc.) allow querying a child's preferred size without performing a full measure — this is how Compose achieves multi-pass-like behavior within the single-pass constraint.

#### 4b. Place
- After sizes are known, the parent calls `Placeable.place(x, y)` (or `placeRelative(x, y)` for RTL-aware positioning) to position each child.
- Position is set **top-down**.
- `Modifier` chains participate in both measure and place (e.g., `padding`, `fillMaxSize`, `offset`).

```kotlin
Layout(
    content = { /* children */ },
    measurePolicy = { measurables, constraints ->
        // MEASURE step — children measured exactly once
        val placeables = measurables.map { it.measure(constraints) }

        layout(constraints.maxWidth, constraints.maxHeight) {
            // PLACE step — children positioned
            placeables.forEach { placeable ->
                placeable.place(x = 0, y = 0)
            }
        }
    }
)
```

#### Modifier.Node API (Modern Modifier System)

Since **Compose 1.3**, the `Modifier.Node` API is the recommended way to create custom modifiers (the public `ModifierNodeElement` API stabilized around 1.4). It replaces the older `composed {}` and `Modifier.Element` patterns:

- **`Modifier.Node`** instances are **retained across recompositions** — they are not recreated, reducing allocations.
- Each node lives in a **linked list** attached to a `LayoutNode`, enabling efficient traversal.
- Node types: `LayoutModifierNode`, `DrawModifierNode`, `SemanticsModifierNode`, `PointerInputModifierNode`, `GlobalPositionAwareModifierNode`, etc.
- **`Modifier.Element`-based modifiers** (the old API) are now considered legacy.

```kotlin
// Modern Modifier.Node approach
class CircleModifierNode : Modifier.Node(), DrawModifierNode {
    override fun ContentDrawScope.draw() {
        drawCircle(Color.Red)
        drawContent()
    }
}

data object CircleElement : ModifierNodeElement<CircleModifierNode>() {
    override fun create() = CircleModifierNode()
    override fun update(node: CircleModifierNode) {}
}

fun Modifier.drawRedCircle() = this then CircleElement
```

#### ⏭️ When Can the Layout Phase Be Skipped?

Compose tracks which phases actually need to re-run. The Layout phase can be **skipped entirely** if:

| Scenario | Composition | Layout | Drawing |
|---|---|---|---|
| State read inside `@Composable` body changed | ✅ Runs | ✅ Runs | ✅ Runs |
| Only size/position changed (e.g., `offset` animated) | ❌ Skipped | ✅ Runs | ✅ Runs |
| Only visual property changed (e.g., color, alpha) | ❌ Skipped | ❌ **Skipped** | ✅ Runs |
| `graphicsLayer` transform changed (scale, rotation) | ❌ Skipped | ❌ **Skipped** | ❌ Skipped (RenderThread only) |
| Nothing changed | ❌ Skipped | ❌ **Skipped** | ❌ Skipped |

**Concrete examples of Layout being skipped:**

- **Color/alpha animation** — `animateColorAsState`, `alpha` modifier change → only Drawing reruns.
- **`graphicsLayer` block** — transforms like `scaleX`, `translationY`, `rotationZ` applied via `graphicsLayer {}` are applied by the **RenderThread** when it replays the layer, skipping Composition, Layout, AND the Drawing re-record on the Main Thread. (The small property-setting lambda itself still runs on the Main Thread; the expensive transform/composite work is what the RenderThread handles.)
- **`Modifier.drawWithContent`** — custom drawing without changing size skips Layout.

```kotlin
// ✅ Layout SKIPPED — only Drawing reruns when color changes
val color by animateColorAsState(if (selected) Color.Blue else Color.Gray)
Box(Modifier.background(color))  // color change → draw only

// ✅ Layout AND Drawing SKIPPED — runs on RenderThread only
val scale by animateFloatAsState(if (pressed) 0.95f else 1f)
Box(Modifier.graphicsLayer { scaleX = scale; scaleY = scale })
```

> 🔑 **Key Rule:** If the **size or position** of any `LayoutNode` could change, Layout must run. If only **how it looks** changes (color, alpha, draw content), Layout is skipped. If only **GPU-level transforms** change (`graphicsLayer`), even Drawing is skipped on the Main Thread.

### 5. Drawing Phase
- Compose draws using a `Canvas` (backed by `androidx.compose.ui.graphics.Canvas`).
- Records drawing commands into a **RenderNode** (same mechanism as the View system).
- `DrawScope` provides the composable drawing API.
- Each `LayoutNode` can have its own RenderNode.
- **AGSL (Android Graphics Shading Language):** Since **Android 13 (API 33)**, custom GPU shaders can be written in AGSL (a GLSL ES-like language) and applied via `RuntimeShader`. Compose exposes this through `ShaderBrush`, enabling custom per-pixel effects (gradients, noise, distortion) that execute on the GPU.

```kotlin
// AGSL shader example in Compose
val shaderCode = """
    uniform float2 resolution;
    half4 main(float2 fragCoord) {
        float2 uv = fragCoord / resolution;
        return half4(uv.x, uv.y, 0.5, 1.0);
    }
"""
val runtimeShader = remember { RuntimeShader(shaderCode) }
Box(Modifier.drawWithCache {
    runtimeShader.setFloatUniform("resolution", size.width, size.height)
    val brush = ShaderBrush(runtimeShader)
    onDrawBehind { drawRect(brush) }
})
```

### 6. RenderThread → GPU → Display (Same as View)
- From here, the pipeline **merges with the View pipeline**:
  - `ComposeView` is a regular Android `View`.
  - Its `Surface` is handed to the **RenderThread**.
  - RenderThread replays RenderNodes → HWUI → Vulkan / OpenGL ES.
  - **SurfaceFlinger** composites → **HWC** → pixels on screen.

```
┌───────────────────────────────────────────┐
│              Main Thread                  │
│                                           │
│  @Composable functions execute            │
│       ↓                                   │
│  Slot Table updated (Composition)         │
│       ↓                                   │
│  LayoutNode tree (Measure + Place)        │
│       ↓                                   │
│  RenderNodes recorded (Drawing)           │
└───────────────┬───────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────┐
│             RenderThread                  │
│  Replay RenderNodes → HWUI → Vulkan/GL   │
│  (graphicsLayer transforms applied here)  │
└───────────────┬───────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────┐
│  SurfaceFlinger → HWC → Display           │
└───────────────────────────────────────────┘
```

---

## 🔁 Side-by-Side Comparison

| Aspect | View System | Jetpack Compose |
|---|---|---|
| **UI Description** | XML (declarative) + Java/Kotlin (imperative) | Pure Kotlin (declarative) |
| **Tree Structure** | View hierarchy | LayoutNode tree + Slot Table |
| **Inflation** | XML parsing + reflection | Kotlin function calls |
| **State Reactivity** | `invalidate()` → redraw subtree | SnapshotState → recompose only affected scope |
| **Measurement** | Multi-pass possible | Single-pass by convention + intrinsic measurements |
| **Layout Skipping** | No *automatic* skipping; `requestLayout()` always re-runs layout, but an `invalidate()`-only change skips it (draw only) | ✅ Can be skipped if size/position unchanged |
| **Reuse/Skipping** | No built-in skipping | Smart recomposition skipping + Strong Skipping Mode |
| **Modifiers** | `setPadding()`, `setBackground()` etc. | `Modifier` chain (composable, ordered) + `Modifier.Node` |
| **Drawing** | `onDraw(Canvas)` | `DrawScope` → RenderNode |
| **Custom Shaders** | `RuntimeShader` (API 33+) | `ShaderBrush` + AGSL |
| **Animation** | `ObjectAnimator`, `ValueAnimator` | `animate*AsState`, `Transition`, `Animatable` |
| **Threading** | Main thread bottleneck | Recomposition can be parallelized (future) |
| **Compiler Integration** | N/A | Compose Compiler in Kotlin 2.0+ |
| **Interop** | — | `ComposeView` embeds Compose in View hierarchy |

---

## 🧵 Threading Model

### View System
```
Main Thread:  Measure → Layout → Draw (record RenderNodes) → [Vsync]
RenderThread: Replay RenderNodes → HWUI → Vulkan/GL GPU commands
GPU:          Rasterize → Frame Buffer
SurfaceFlinger: Composite (HWC + GPU) → Screen
```

### Compose
```
Main Thread:  Composition → Recomposition → Layout (Measure+Place) → Draw (record) → [Vsync]
RenderThread: Replay RenderNodes + graphicsLayer transforms → HWUI → Vulkan/GL GPU commands
GPU:          Rasterize → Frame Buffer
SurfaceFlinger: Composite (HWC + GPU) → Screen
```

> 🔑 **Key Insight:** Compose's advantage is in the **CPU work on the Main Thread** — it does far less work per frame update thanks to smart recomposition, strong skipping mode, single-pass layout, phase skipping, and no XML/reflection overhead.

---

## 📊 LazyLayout Rendering Optimizations

`LazyColumn`, `LazyRow`, and `LazyGrid` (all built on `LazyLayout`) have specialized rendering behavior:

### How LazyLayout Differs from Regular Compose
| Aspect | Regular Compose | LazyLayout |
|---|---|---|
| **Composition** | All items composed upfront | Only **visible items** composed |
| **Off-screen items** | Still in Slot Table | **Disposed** or **cached** in pool |
| **Recycling** | No concept | Items can be reused via `key` + `contentType` |

### Key Optimizations
1. **Item keys (`key = { ... }`):** Uniquely identify items so Compose can **reorder/move** without recomposing. Without keys, any insertion/removal causes recomposition of all subsequent items.
2. **Content types (`contentType = { ... }`):** Items with the same content type can **reuse LayoutNode and RenderNode pools**. This is the closest equivalent to RecyclerView's ViewType-based recycling.
3. **Prefetching:** LazyLayout **pre-composes and pre-measures** items just outside the visible bounds during idle frames, so they're ready when the user scrolls.
4. **`animateItem()`:** Animate item additions, removals, and reordering. Uses `graphicsLayer` transforms internally for GPU-accelerated animation.

```kotlin
LazyColumn {
    items(
        items = dataList,
        key = { it.id },                    // Stable identity
        contentType = { it.type }            // Recycling type
    ) { item ->
        ItemRow(item, Modifier.animateItem()) // Animated placement
    }
}
```

---

## 🛠️ Performance Profiling & Tools

### Frame Rendering Performance
| Tool | What It Measures |
|---|---|
| **Profile GPU Rendering** (on-device bars) | Per-frame breakdown: input, animation, measure/layout, draw, sync, command issue |
| **GPU Overdraw visualization** (on-device) | Shows how many times each pixel is drawn (blue=1x, green=2x, pink=3x, red=4+) |
| **FrameMetrics API** (`Window.OnFrameMetricsAvailableListener`) | Programmatic per-frame timing (API 24+). Measures: UNKNOWN, INPUT, ANIMATION, LAYOUT_MEASURE, DRAW, SYNC, COMMAND_ISSUE, SWAP_BUFFERS, TOTAL, FIRST_DRAW_FRAME |
| **Perfetto / System Trace** | System-wide trace showing Choreographer, RenderThread, SurfaceFlinger, GPU work |
| **Layout Inspector** (Android Studio) | Live 3D inspection of View hierarchy or Compose LayoutNode tree |
| **Compose Composition Tracing** | Trace recomposition counts and skips per composable in Android Studio Profiler |
| **Recomposition Highlighter** | Visual highlight of recomposing composables in Layout Inspector |

### Compose-Specific Performance Tools
| Tool | Purpose |
|---|---|
| **Stability Configuration File** | Define classes as stable without annotating source (useful for third-party types) |
| **Compose Compiler Metrics** | Generate reports showing which composables are restartable, skippable, and why |
| **`@NonSkippableComposable`** | Opt-out annotation for restartable composables that should remain non-skippable under Strong Skipping Mode |

### Baseline Profiles
- **Baseline Profiles** provide ahead-of-time (AOT) compilation hints for critical rendering paths.
- The `androidx.benchmark` library generates profiles that include Compose layout, draw, and recomposition paths.
- The **Compose Bill of Materials (BOM)** ships with built-in baseline profiles for core Compose libraries.
- On first install, ART uses these profiles for **partial AOT compilation**, significantly reducing jank during first-run rendering (about **30% faster** code execution from first launch) and smoother initial frames.
- Custom baseline profiles can be generated using `BaselineProfileRule` in an `androidTest`:

```kotlin
@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {
    @get:Rule
    val rule = BaselineProfileRule()

    @Test
    fun generateProfile() = rule.collect("com.example.app") {
        startActivityAndWait()
        // Navigate through critical user journeys
    }
}
```

---

## 🎯 How Pixels Actually Appear on Screen

Regardless of View or Compose, the final path is the same:

1. **App renders** into a `Surface` (a buffer in shared memory).
2. **RenderThread** submits GPU commands.
3. **GPU rasterizes** into the frame buffer.
4. **SurfaceFlinger** receives the buffer, composites with other layers (status bar, nav bar, other apps in split screen).
5. **Hardware Composer (HWC)** overlays layers that can be handled directly by display hardware (no GPU needed).
6. **Display controller** reads the final buffer at **Vsync** and drives the physical pixels.

```
App Surface Buffer
       ↓
  RenderThread
       ↓
    GPU Rasterize
       ↓
  Frame Buffer
       ↓
  SurfaceFlinger
       ↓
  Hardware Composer (HWC)
       ↓
  Display Panel (Vsync-synchronized)
       ↓
  💡 Pixels Light Up!
```

---

## 🏁 Summary

- **View System:** Imperative, XML-inflated, multi-pass measure/layout, `invalidate()` causes branch redraws, older but mature.
- **Compose:** Declarative, Kotlin-only, single-pass layout, SnapshotState-driven fine-grained recomposition, Strong Skipping Mode enabled by default (Kotlin 2.0.20+), modern and more efficient.
- **Compose Compiler** is now part of Kotlin 2.0+ — no separate compiler version management needed.
- **Modifier.Node** is the modern modifier API, reducing allocations by retaining modifier instances across recompositions.
- **Compose Layout can be skipped** when only visual properties (color, alpha) change — only the Drawing phase reruns. `graphicsLayer` transforms skip even Drawing on the Main Thread, running purely on the RenderThread.
- **Both share** the same lower-level rendering: HWUI → RenderThread → Vulkan/OpenGL ES → SurfaceFlinger → HWC → Display.
- **AGSL** (Android 13+) enables custom GPU shaders for advanced visual effects in both View and Compose.
- **Baseline Profiles** significantly improve first-run rendering performance for Compose apps.
- **LazyLayout** provides RecyclerView-like efficiency with item recycling via `contentType` and prefetching.
- Compose is **more efficient on the CPU side** (less work per update), which directly reduces **UI jank** and improves frame rates.

