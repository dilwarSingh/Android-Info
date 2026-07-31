# Android Animations — Last-Minute Revision

---

## Android Views — Animation Types

### View Animation (Legacy — `android.view.animation`)
- Animates **visual drawing only** — actual position/size does **not** change.
- Touch/click areas remain at original position.
- **Avoid for interactive/clickable elements.**

### `ObjectAnimator`
- Modifies **real object properties** via reflection (calls setters: `setAlpha()`, `setTranslationX()`).
- Use `ofArgb()` with **`ArgbEvaluator`** for color transitions.
- Use `setRepeatCount(INFINITE)` for looping.

```java
ObjectAnimator.ofFloat(myView, "alpha", 1f, 0f).setDuration(400).start();
ObjectAnimator.ofArgb(myView, "backgroundColor", Color.RED, Color.BLUE).start();
```

### `ValueAnimator`
- Animates **raw values** — no view target. You apply the value manually in `addUpdateListener`.
- Use for: custom canvas drawing, multi-view sync from one animator, custom progress bars.

### `AnimatorSet`
- Combines animators: **`play().with()`** (parallel), **`play().after()`** (sequential).

```java
set.play(scaleX).with(scaleY);
set.play(fadeOut).after(scaleX);
```

### `ViewPropertyAnimator`
- **Preferred API** for simple single-view animations. Fluent, readable.
- Call **`.withLayer()`** to render on a hardware layer for the animation's duration.
- `withStartAction()` / `withEndAction()` for lifecycle hooks.

```java
myView.animate().translationY(-150f).alpha(0f).setDuration(300).withEndAction(() -> myView.setVisibility(View.GONE)).start();
```

### `SpringAnimation` (DynamicAnimation)
- **Physics-based** spring simulation. Ideal for gesture responses.
- Key params: **`DAMPING_RATIO`** (bounciness), **`STIFFNESS`** (speed).

```java
new SpringAnimation(myView, DynamicAnimation.TRANSLATION_Y, 0f)
    .getSpring().setDampingRatio(SpringForce.DAMPING_RATIO_MEDIUM_BOUNCY).setStiffness(SpringForce.STIFFNESS_LOW);
```

### `TransitionManager`
- Detects layout changes and animates them **automatically**.

```java
TransitionManager.beginDelayedTransition(rootLayout, new AutoTransition());
myView.setVisibility(View.GONE); // framework animates the change
```

### `MotionLayout`
- Extends **`ConstraintLayout`**. Defines start/end `ConstraintSet` states in a `MotionScene` XML.
- Supports gesture-driven transitions via `<OnSwipe>` with `touchAnchorId` and `dragDirection`.

### `AnimationDrawable`
- Frame-by-frame drawable animation via XML `<animation-list>`.
- **Must call `start()` via `imageView.post()`** — not in `onCreate()` (view not yet attached).

### `AnimatedVectorDrawable` (AVD)
- Animates **vector asset properties** (path morphing, rotation) tied to `ObjectAnimator` XMLs.
- Use `Animatable` cast check before calling `.start()`.

### Activity & Fragment Transitions (Shared Elements)
- Use **`ActivityOptionsCompat.makeSceneTransitionAnimation()`** with a `transitionName`.

---

## Jetpack Compose — Animation Types

### `animate*AsState`
- Animates a **single value** on state change. Fire-and-forget.
- Variants: `animateFloatAsState`, `animateDpAsState`, `animateColorAsState`, `animateIntAsState`, `animateOffsetAsState`, `animateSizeAsState`, `animateRectAsState`.

```kotlin
val alpha by animateFloatAsState(if (expanded) 1f else 0f, label = "alpha")
```

### `AnimatedVisibility`
- Animates **show/hide** of composables with composable enter/exit specs.
- Enter: `fadeIn`, `slideIn*`, `expandIn`, `scaleIn`. Exit: `fadeOut`, `slideOut*`, `shrinkOut`, `scaleOut`.
- Specs are combinable with `+`.

```kotlin
AnimatedVisibility(visible, enter = slideInVertically { -it } + fadeIn(), exit = fadeOut()) { ... }
```

### `AnimatedContent`
- Animates **between different content** for the same state slot.
- `transitionSpec` receives `initialState` and `targetState` — use for directional slides.
- Uses `togetherWith` (`ContentTransform`) to pair enter + exit specs.

### `Crossfade`
- Simpler than `AnimatedContent`. Swaps composables with a **crossfade effect** only.

```kotlin
Crossfade(targetState = currentScreen, animationSpec = tween(400), label = "screen") { screen -> ... }
```

### `Modifier.animateContentSize()`
- Automatically animates **size changes** when composable content changes (e.g., expanding text).

```kotlin
Modifier.animateContentSize(animationSpec = spring(stiffness = Spring.StiffnessLow))
```

### Shared Element Transitions (Compose)
- Requires **`SharedTransitionLayout`** wrapping `AnimatedContent` or Navigation Compose.
- Use **`Modifier.sharedElement()`** with `rememberSharedContentState(key = ...)` and `animatedVisibilityScope`.

### `updateTransition`
- Coordinates **multiple properties** animating in sync from the same state enum/sealed class.
- Each property derived via `transition.animateFloat`, `transition.animateDp`, etc.

```kotlin
val transition = updateTransition(targetState = cardState, label = "cardTransition")
val scale by transition.animateFloat(label = "scale") { if (it == CardState.Selected) 1.08f else 1f }
```

### `Animatable`
- **Full manual / imperative control** via coroutines: `animateTo`, `snapTo`, `stop`.
- Supports value clamping (bounds).
- **Must be wrapped in `remember {}`** to survive recompositions.

```kotlin
val offsetX = remember { Animatable(0f) }
LaunchedEffect(Unit) { offsetX.animateTo(300f, spring(dampingRatio = Spring.DampingRatioMediumBouncy)) }
```

### `rememberInfiniteTransition`
- Runs **looping animations**. Uses `infiniteRepeatable` with `RepeatMode.Reverse` or `RepeatMode.Restart`.

### Animation Specs
| Spec | Behavior |
|---|---|
| `tween(durationMillis, easing)` | Fixed duration + easing curve |
| `spring(dampingRatio, stiffness)` | Physics-based, no fixed duration |
| `keyframes { }` | Multi-step precise control with per-keyframe easing |
| `infiniteRepeatable(animation, repeatMode)` | Loops forever |
| `snap(delayMillis)` | Instant, no animation |

---

## Decision Trees

### Views — Quick Reference
| Scenario | Use |
|---|---|
| Simple show/hide/move | `ViewPropertyAnimator` |
| Raw value / custom drawing | `ValueAnimator` |
| Real property (alpha, color, rotation) | `ObjectAnimator` |
| Vector icon morph | `AnimatedVectorDrawable` |
| Sequential/parallel animators | `AnimatorSet` |
| Shared element across Activities | `ActivityOptionsCompat` |
| Auto layout change animation | `TransitionManager.beginDelayedTransition()` |
| Physics/gesture feel | `SpringAnimation` |
| Complex multi-state with constraints | `MotionLayout` |

### Compose — Quick Reference
| Scenario | Use |
|---|---|
| Single value on state change | `animate*AsState` |
| Size change from content | `Modifier.animateContentSize()` |
| Show/hide composable | `AnimatedVisibility` |
| Simple composable swap | `Crossfade` |
| Shared element across screens | `SharedTransitionLayout` + `Modifier.sharedElement` |
| Complex content swap + custom transitions | `AnimatedContent` |
| Multiple props from same state | `updateTransition` |
| Gesture-driven / coroutine control | `Animatable` |
| Infinite loop (shimmer, pulse) | `rememberInfiniteTransition` |

---

## Do's and Don'ts

### DO: Animate Only Cheap Properties
- **Cheap (GPU composited — no layout pass):** `alpha`, `translationX/Y`, `scaleX/Y`, `rotation`, `rotationX/Y`
- **Expensive (triggers layout + measure + draw):** `width`, `height`, `padding`, `margin`

### DO: Enable Hardware Layer During Animation (Views)
- Set `LAYER_TYPE_HARDWARE` before animation, **restore `LAYER_TYPE_NONE` in `onAnimationEnd`**.
- Compose handles this automatically via `graphicsLayer`.

### DO: Respect Reduced Motion
- **Compose (standard):** no built-in `LocalReduceMotion` — `animate*AsState`/`AnimatedVisibility` already respect `Settings.Global.ANIMATOR_DURATION_SCALE` automatically; for custom animation logic, query it via `LocalContext.current` and use `Settings.Global.getFloat(...)`.
- **Compose (Wear OS only):** `LocalReduceMotion` (from `androidx.wear.compose.foundation`) is available — not present in standard Compose.
- **Views:** Check `Settings.Global.ANIMATOR_DURATION_SCALE == 0f`.

### DO: Cancel Animations on Recycle / Destroy
- RecyclerView: `holder.itemView.animate().cancel()` in `onBindViewHolder`.
- Activity/Fragment: cancel in `onDestroy()` — `animatorSet.cancel()`, `view.animate().cancel()`.

### DO: Stagger List Entry Animations
- **Compose:** `delay(index * 60L)` inside `produceState`, wrap item in `AnimatedVisibility`.
- **Views:** `.setStartDelay(i * 60L)` per item in loop.

### DON'T: Use View Animation for Interactive Elements
- Click areas stay at **original position** — `TranslateAnimation` moves visuals only.
- Use `ViewPropertyAnimator.translationX/Y` instead.

### DON'T: Start `AnimationDrawable` in `onCreate`
- View not yet attached → NPE or no-op. Always use **`imageView.post(() -> spinner.start())`**.

### DON'T: Create `Animatable` Without `remember`
- Without `remember`, a new `Animatable` is created on every recomposition → broken animation.

### DON'T: Block Main Thread
- Never `Thread.sleep()` on UI thread during animations.
- **Compose:** use `delay()` in `LaunchedEffect` (suspends, doesn't block).

### DON'T: Overuse Animations
- Target **60fps / 16ms per frame**. Too many simultaneous animations cause jank.
- Keep UI animations **150ms–400ms**. Never exceed **500ms** for UI, **1s** for hero animations.

---

## Recommended Durations

| Animation Type | Duration |
|---|---|
| Button press / ripple | 100–150ms |
| Fade in / fade out | 150–250ms |
| Slide in / slide out | 200–350ms |
| Expand / collapse | 250–400ms |
| Complex transitions | 300–500ms |
| Hard limit | 500ms UI · 1s hero |
