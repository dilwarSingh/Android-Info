# Android Rendering Pipelines: View System vs Jetpack Compose

---

## 🖼️ Overview

Yes — the rendering pipelines of the **Android View System** and **Jetpack Compose** are fundamentally different, though both ultimately produce pixels on screen via the same GPU/display hardware.

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

### 4. Draw Pass (Record DisplayList)
- Each `View` calls `onDraw(Canvas)`.
- Canvas commands (drawText, drawRect, etc.) are recorded into a **DisplayList** (not executed immediately).
- `View.invalidate()` → marks the view dirty → triggers a **full re-draw** of that branch.

### 5. RenderThread (Hardware Acceleration)
- Since Android 5.0, a dedicated **RenderThread** replays the DisplayList.
- Converts Canvas commands → **OpenGL ES / Vulkan** draw calls.
- The **Main Thread** records; the **RenderThread** executes → parallelism.

### 6. SurfaceFlinger & Display
- RenderThread renders into a **Surface buffer**.
- **SurfaceFlinger** composites all app layers + system UI.
- **Hardware Composer (HWC)** overlays layers directly on display hardware.
- Vsync signal from display triggers each frame (16ms @ 60Hz).

```
┌──────────────┐    ┌────────────────┐    ┌─────────────────┐    ┌─────────┐
│  Main Thread │    │  RenderThread  │    │  SurfaceFlinger │    │ Display │
│              │    │                │    │                 │    │         │
│  Measure     │───▶│  Replay        │───▶│  Composite      │───▶│  Vsync  │
│  Layout      │    │  DisplayList   │    │  All Layers     │    │  Pixels │
│  Draw/Record │    │  → GL/Vulkan   │    │                 │    │         │
└──────────────┘    └────────────────┘    └─────────────────┘    └─────────┘
```

### Key Pain Points
| Problem | Cause |
|---|---|
| Overdraw | Multiple `View`s paint the same pixels |
| Slow inflation | XML parsing + reflection |
| Multiple measure passes | `wrap_content`, `RelativeLayout` |
| Full subtree redraws | `invalidate()` cascades |
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

### 2. SnapshotState System
- State variables (`mutableStateOf`, `remember`) are backed by Compose's **Snapshot system**.
- When state changes → Compose marks only the **affected composable scopes** as invalid.
- This is **fine-grained reactivity** — not full-tree re-evaluation.

### 3. Recomposition (Smart Re-execution)
- Only the **invalidated composable lambdas** re-execute.
- Compose **skips** composables whose inputs haven't changed (`@Stable`, `@Immutable` help here).
- Updates the **Slot Table** with new values.
- **No measure/layout/draw happens yet** — just Kotlin function execution.

### 4. Layout Phase

Compose's Layout phase has **two sub-steps**:

#### 4a. Measure
- Each `LayoutNode` calls `measure(constraints)` on its children **exactly once**.
- Calling `measure()` more than once throws an exception at runtime — single-pass is **enforced**.
- `Constraints` (minWidth, maxWidth, minHeight, maxHeight) flow **top-down**.
- Children report their chosen size **bottom-up**.

#### 4b. Place
- After sizes are known, the parent calls `Placeable.placeAt(x, y)` to position each child.
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
                placeable.placeAt(x = 0, y = 0)
            }
        }
    }
)
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
- **`graphicsLayer` block** — transforms like `scaleX`, `translationY`, `rotationZ` applied via `graphicsLayer {}` run **entirely on the RenderThread**, skipping Composition, Layout, AND Drawing on the Main Thread.
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
- Records drawing commands into a **DisplayList** (same concept as View system).
- `DrawScope` provides the composable drawing API.
- Each `LayoutNode` can have its own display list.

### 6. RenderThread → GPU → Display (Same as View)
- From here, the pipeline **merges with the View pipeline**:
  - `ComposeView` is a regular Android `View`.
  - Its `Surface` is handed to the **RenderThread**.
  - RenderThread replays DisplayList → OpenGL ES / Vulkan.
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
│  DisplayList recorded (Drawing)           │
└───────────────┬───────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────┐
│             RenderThread                  │
│  Replay DisplayList → OpenGL ES / Vulkan  │
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
| **Measurement** | Multi-pass possible | Single-pass (enforced) |
| **Layout Skipping** | Not possible — always runs on `requestLayout()` | ✅ Can be skipped if size/position unchanged |
| **Reuse/Skipping** | No built-in skipping | Smart recomposition skipping |
| **Modifiers** | `setPadding()`, `setBackground()` etc. | `Modifier` chain (composable, ordered) |
| **Drawing** | `onDraw(Canvas)` | `DrawScope` → DisplayList |
| **Animation** | `ObjectAnimator`, `ValueAnimator` | `animate*AsState`, `Transition`, `Animatable` |
| **Threading** | Main thread bottleneck | Recomposition can be parallelized (future) |
| **Interop** | — | `ComposeView` embeds Compose in View hierarchy |

---

## 🧵 Threading Model

### View System
```
Main Thread:  Measure → Layout → Draw (record) → [Vsync]
RenderThread: Replay DisplayList → GPU commands
GPU:          Rasterize → Frame Buffer
SurfaceFlinger: Composite → HWC → Screen
```

### Compose
```
Main Thread:  Composition → Recomposition → Layout (Measure+Place) → Draw (record) → [Vsync]
RenderThread: Replay DisplayList + graphicsLayer transforms → GPU commands
GPU:          Rasterize → Frame Buffer
SurfaceFlinger: Composite → HWC → Screen
```

> 🔑 **Key Insight:** Compose's advantage is in the **CPU work on the Main Thread** — it does far less work per frame update thanks to smart recomposition, single-pass layout, phase skipping, and no XML/reflection overhead.

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
- **Compose:** Declarative, Kotlin-only, single-pass layout, SnapshotState-driven fine-grained recomposition, modern and more efficient.
- **Compose Layout can be skipped** when only visual properties (color, alpha) change — only the Drawing phase reruns. `graphicsLayer` transforms skip even Drawing on the Main Thread, running purely on the RenderThread.
- **Both share** the same lower-level rendering: RenderThread → OpenGL ES/Vulkan → SurfaceFlinger → HWC → Display.
- Compose is **more efficient on the CPU side** (less work per update), which directly reduces **UI jank** and improves frame rates.

