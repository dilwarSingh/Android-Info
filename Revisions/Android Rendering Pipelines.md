# Android Rendering Pipelines — Cheat Sheet

## Overview

Both **View System** and **Jetpack Compose** ultimately render through **HWUI** (Hardware UI) — Android's GPU-accelerated 2D engine — into the same display stack.

- **HWUI** translates `Canvas`/`RenderNode` commands into GPU instructions.
- A `RenderNode` encapsulates a set of drawing commands plus transform properties (translation, scale, rotation, alpha, clip). Each `View` owns one.
- Rendering backend: **SkiaGL** (Android 9) → **SkiaVk** (Android 10, select devices; most stayed on SkiaGL) → **Vulkan grew as default on Pixel/some OEM flagships** (Android 12–13, SkiaGL still default on most devices).
- **Android 14+:** Vulkan is default on a **growing set** of new devices, but SkiaGL is **still common**; **ANGLE** provides OpenGL ES compatibility on top of Vulkan-only drivers.

---

## Part 1: View System Rendering Pipeline

**Pipeline:** `XML Inflation → Measure → Layout → Draw → RenderThread → GPU → Display`

### Stage Breakdown

- **XML Inflation:** `LayoutInflater` parses XML, reflectively creates `View` objects. Slow — reflection + deep hierarchies compound cost.
- **Measure Pass:** `onMeasure(widthSpec, heightSpec)` — top-down recursive; can be called **multiple times** (e.g., `wrap_content` in `LinearLayout`).
- **Layout Pass:** `onLayout(changed, left, top, right, bottom)` — top-down; positions children using measured sizes.
- **Draw Pass:** `onDraw(Canvas)` records commands into a **RenderNode** (a.k.a. DisplayList, renamed API 21). Each `View` owns one. `View.invalidate()` → re-records that `View`'s RenderNode.
  - Since **API 29**, `RenderNode` is a **public API** for standalone custom rendering without a View.
- **Hardware acceleration** introduced in **Android 3.0 (Honeycomb)**; became default in **Android 4.0 (ICS)**.
- **RenderThread** (since Android 5.0): dedicated thread replays recorded RenderNodes → HWUI → GPU. **Main Thread records; RenderThread executes.**
- **SurfaceFlinger:** composites all layers via **Client composition** (GPU) or **Device composition** (**HWC** — hardware overlays, more power-efficient).
- **Vsync** via **Choreographer** (Project Butter, Android 4.1): 16.6ms @ 60Hz / 11.1ms @ 90Hz / 8.3ms @ 120Hz. Triple buffering enabled.
- **Android 11+:** Variable refresh rate support — SurfaceFlinger dynamically switches refresh rates (e.g., 60Hz ↔ 120Hz) based on content demands.

```
Main Thread       RenderThread        SurfaceFlinger      Display
Measure ──────▶  Replay RenderNodes ▶  Composite Layers ▶  Vsync/Pixels
Layout            → HWUI               (HWC + GPU)
Draw/Record       → Vulkan/GL
```

### Key Pain Points

| Problem | Cause |
|---|---|
| Overdraw | Multiple Views paint same pixels |
| Slow inflation | XML parsing + reflection |
| Multi-pass measure | `wrap_content`, `RelativeLayout` |
| Full subtree redraws | `invalidate()` cascades |
| Deep hierarchy cost | One more traversal per level |

---

## Part 2: Jetpack Compose Rendering Pipeline

**Pipeline:** `Composition → SnapshotState Change → Recomposition → Layout → Drawing → RenderThread → GPU → Display`

### 1. Composition

- Executes `@Composable` functions → builds a **Slot Table** (gap buffer) + **LayoutNode** tree.
- No XML, no reflection — pure Kotlin.
- **Compose Compiler Plugin** (Kotlin 2.0+): bundled into Kotlin compiler (`org.jetbrains.kotlin.plugin.compose`). No separate `androidx.compose.compiler` dependency or version pinning needed.

### 2. SnapshotState System

- `mutableStateOf` / `remember` backed by the **Snapshot system**.
- State change → only **affected composable scopes** marked invalid — fine-grained reactivity, not full-tree re-evaluation.

### 3. Recomposition

- Only **invalidated composable lambdas** re-execute; unchanged composables are **skipped**.
- `@Stable` / `@Immutable` types use `equals()` comparison to enable skipping — Strong Skipping Mode reduces (but doesn't eliminate) the need for these annotations.
- **Strong Skipping Mode** (default since Compose 1.7):
  - All composables skippable by default, even with **unstable parameters**.
  - Unstable params compared by **instance equality** (`===`); stable by `equals()`.
  - **Lambdas auto-remembered** — no manual `remember { }` wrapping needed.

### 4. Layout Phase

**Single-pass by convention** — re-measuring is discouraged but not forbidden; `SubcomposeLayout`, `BoxWithConstraints`, intrinsics, and lookahead legitimately measure more than once.

- **Measure:** `Constraints` (min/max width+height) flow **top-down**; children report size **bottom-up**.
- **Intrinsic measurements** (`minIntrinsicWidth`, etc.) allow querying preferred size without a full measure pass.
- **Place:** Parent calls `Placeable.place(x, y)` / `placeRelative(x, y)` (RTL-aware) — top-down.
- `Modifier` chains (e.g., `padding`, `fillMaxSize`, `offset`) participate in both measure and place.

```kotlin
Layout(content = { /* children */ }, measurePolicy = { measurables, constraints ->
    val placeables = measurables.map { it.measure(constraints) }  // measure once
    layout(constraints.maxWidth, constraints.maxHeight) {
        placeables.forEach { it.place(x = 0, y = 0) }             // then place
    }
})
```

#### Modifier.Node API (since Compose 1.3; `ModifierNodeElement` public API stabilized around 1.4)

- Replaces legacy `composed {}` and `Modifier.Element` patterns.
- **`Modifier.Node` instances are retained across recompositions** — no recreation, fewer allocations.
- Lives in a **linked list** on each `LayoutNode`.
- Key node types: `LayoutModifierNode`, `DrawModifierNode`, `SemanticsModifierNode`, `PointerInputModifierNode`, `GlobalPositionAwareModifierNode`.

```kotlin
class CircleModifierNode : Modifier.Node(), DrawModifierNode {
    override fun ContentDrawScope.draw() { drawCircle(Color.Red); drawContent() }
}
data object CircleElement : ModifierNodeElement<CircleModifierNode>() {
    override fun create() = CircleModifierNode()
    override fun update(node: CircleModifierNode) {}
}
```

#### Phase Skipping Rules

| Scenario | Composition | Layout | Drawing |
|---|---|---|---|
| State read in `@Composable` body changed | Runs | Runs | Runs |
| Only size/position changed | Skipped | Runs | Runs |
| Only visual property (color, alpha) changed | Skipped | **Skipped** | Runs |
| `graphicsLayer` transform changed | Skipped | **Skipped** | Skipped (RenderThread only) |

```kotlin
// Layout SKIPPED — only Drawing reruns on color change
val color by animateColorAsState(if (selected) Color.Blue else Color.Gray)
Box(Modifier.background(color))

// Layout AND Drawing SKIPPED — runs on RenderThread only
val scale by animateFloatAsState(if (pressed) 0.95f else 1f)
Box(Modifier.graphicsLayer { scaleX = scale; scaleY = scale })
```

**Rule:** Size/position change → Layout must run. Visual-only change → Layout skipped. GPU-level transform (`graphicsLayer`) → even Drawing skipped on Main Thread.

### 5. Drawing Phase

- `DrawScope` API records commands into a **RenderNode** (same mechanism as View system).
- Each `LayoutNode` can own its own RenderNode.
- **AGSL (Android Graphics Shading Language)** — Android 13 (API 33+): write custom GPU shaders in GLSL ES-like syntax; applied via `RuntimeShader` + `ShaderBrush`.

```kotlin
val runtimeShader = remember { RuntimeShader(agslCode) }
Box(Modifier.drawWithCache {
    runtimeShader.setFloatUniform("resolution", size.width, size.height)
    onDrawBehind { drawRect(ShaderBrush(runtimeShader)) }
})
```

### 6. RenderThread → GPU → Display

- `ComposeView` is a regular `View`; its `Surface` is handed to the **RenderThread**.
- RenderThread replays RenderNodes → HWUI → Vulkan/GL → **SurfaceFlinger** → **HWC** → pixels.

---

## Side-by-Side Comparison

| Aspect | View System | Jetpack Compose |
|---|---|---|
| **UI Description** | XML + Java/Kotlin imperative | Pure Kotlin declarative |
| **Tree Structure** | View hierarchy | LayoutNode tree + Slot Table |
| **Inflation** | XML parsing + reflection | Kotlin function calls |
| **State Reactivity** | `invalidate()` → subtree redraw | SnapshotState → affected scope only |
| **Measurement** | Multi-pass possible | Single-pass by convention + intrinsics |
| **Layout Skipping** | No automatic skipping; `requestLayout()` always re-runs layout, but `invalidate()`-only change skips it (draw only) | Skipped if size/position unchanged |
| **Reuse/Skipping** | None built-in | Smart recomposition + Strong Skipping |
| **Modifiers** | `setPadding()`, `setBackground()`, etc. | `Modifier` chain + `Modifier.Node` |
| **Drawing** | `onDraw(Canvas)` | `DrawScope` → RenderNode |
| **Animation** | `ObjectAnimator`, `ValueAnimator` | `animate*AsState`, `Transition`, `Animatable` |
| **Custom Shaders** | `RuntimeShader` (API 33+) | `ShaderBrush` + AGSL |
| **Compiler Integration** | N/A | Compose Compiler bundled in Kotlin 2.0+ |
| **Threading** | Main thread bottleneck | Recomposition can be parallelized (future) |
| **Interop** | — | `ComposeView` embeds Compose in View hierarchy |

---

## Threading Model

**View System:**
```
Main Thread:    Measure → Layout → Draw (record RenderNodes)
RenderThread:   Replay RenderNodes → HWUI → Vulkan/GL
GPU:            Rasterize → Frame Buffer
SurfaceFlinger: Composite (HWC + GPU) → Screen
```

**Compose:**
```
Main Thread:    Composition → Recomposition → Layout → Draw (record)
RenderThread:   Replay RenderNodes + graphicsLayer transforms → HWUI → Vulkan/GL
GPU:            Rasterize → Frame Buffer
SurfaceFlinger: Composite (HWC + GPU) → Screen
```

Compose's advantage is entirely on the **Main Thread CPU side** — less work per frame via smart recomposition, strong skipping, single-pass layout, phase skipping, and no XML/reflection overhead.

---

## LazyLayout Rendering Optimizations

`LazyColumn`, `LazyRow`, `LazyGrid` — all built on **`LazyLayout`**.

| Aspect | Regular Compose | LazyLayout |
|---|---|---|
| **Composition** | All items upfront | Only **visible items** |
| **Off-screen items** | In Slot Table | Disposed or cached in pool |
| **Recycling** | None | Via `key` + `contentType` |

- **`key = { it.id }`** — stable identity; enables reorder/move without recomposing all subsequent items.
- **`contentType = { it.type }`** — items with matching type reuse **LayoutNode + RenderNode pools** (equivalent to RecyclerView `ViewType` recycling).
- **Prefetching** — pre-composes and pre-measures items just outside visible bounds during idle frames.
- **`animateItem()`** — animates additions, removals, reorders using `graphicsLayer` internally (GPU-accelerated).

```kotlin
LazyColumn {
    items(items = dataList, key = { it.id }, contentType = { it.type }) { item ->
        ItemRow(item, Modifier.animateItem())
    }
}
```

---

## Performance Profiling & Tools

| Tool | Measures |
|---|---|
| **Profile GPU Rendering** (on-device) | Per-frame bars: input, animation, measure/layout, draw, sync, command issue |
| **GPU Overdraw visualization** (on-device) | Pixel overdraw: blue=1x, green=2x, pink=3x, red=4x+ |
| **FrameMetrics API** (`Window.OnFrameMetricsAvailableListener`, API 24+) | Programmatic per-frame timing: `UNKNOWN`, `INPUT`, `ANIMATION`, `LAYOUT_MEASURE`, `DRAW`, `SYNC`, `COMMAND_ISSUE`, `SWAP_BUFFERS`, `TOTAL`, `FIRST_DRAW_FRAME` |
| **Perfetto / System Trace** | Choreographer, RenderThread, SurfaceFlinger, GPU work system-wide |
| **Layout Inspector** | Live 3D View hierarchy or Compose LayoutNode tree |
| **Compose Compiler Metrics** | Reports which composables are restartable, skippable, and why |
| **Compose Composition Tracing** | Trace recomposition counts and skips per composable in Android Studio Profiler |
| **Recomposition Highlighter** | Visual highlight of recomposing composables in Layout Inspector |

- **`@NonRestartableComposable`** — opt-out for composables that should never restart independently.
- **Stability Configuration File** — mark third-party classes as stable without annotating source.

### Baseline Profiles

- Provide **AOT compilation hints** for critical rendering paths to ART.
- Compose BOM ships built-in profiles for core Compose libraries.
- **About 30% faster** code execution from first launch, and reduced first-run jank.

```kotlin
@get:Rule val rule = BaselineProfileRule()
@Test fun generateProfile() = rule.collect("com.example.app") {
    startActivityAndWait()
    // Navigate critical user journeys
}
```

---

## Final Pixel Path (Both Pipelines)

1. App renders into a **`Surface`** buffer (shared memory).
2. **RenderThread** submits GPU commands.
3. **GPU rasterizes** into frame buffer.
4. **SurfaceFlinger** composites all layers (status bar, nav bar, other apps in split screen).
5. **Hardware Composer (HWC)** overlays layers directly on display hardware (no GPU needed for eligible layers).
6. **Display controller** reads final buffer at **Vsync** → physical pixels lit.
