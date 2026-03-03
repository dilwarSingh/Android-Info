# 🎬 Android Animations — Complete Guide

---

## 📦 Android Views — Animation Types

### 1. View Animation (Legacy — `android.view.animation`)
Animates the **visual drawing** of a view only. The actual view position/size does **not** change.

```xml
<!-- res/anim/slide_in_left.xml -->
<translate xmlns:android="http://schemas.android.com/apk/res/android"
    android:fromXDelta="-100%"
    android:toXDelta="0%"
    android:duration="300"
    android:interpolator="@android:anim/decelerate_interpolator"/>
```

```java
Animation anim = AnimationUtils.loadAnimation(context, R.anim.slide_in_left);
myView.startAnimation(anim);
```

> ⚠️ **Limitation:** Touch/click areas remain at original position. Avoid for interactive elements.

---

### 2. `ObjectAnimator` — Animates Real Object Properties

Modifies actual properties via reflection (calls setters like `setAlpha()`, `setTranslationX()`).

```java
// Fade out
ObjectAnimator fadeOut = ObjectAnimator.ofFloat(myView, "alpha", 1f, 0f);
fadeOut.setDuration(400);
fadeOut.setInterpolator(new FastOutSlowInInterpolator());
fadeOut.start();

// Move horizontally
ObjectAnimator slide = ObjectAnimator.ofFloat(myView, "translationX", 0f, 300f);
slide.setDuration(300);
slide.start();

// Color change (requires ArgbEvaluator)
ObjectAnimator colorAnim = ObjectAnimator.ofArgb(myView, "backgroundColor", Color.RED, Color.BLUE);
colorAnim.setDuration(500);
colorAnim.start();
```

---

### 3. `ValueAnimator` — Animate Raw Values (Manual Control)

Does not target any view directly — you decide what to do with the value.

```java
ValueAnimator animator = ValueAnimator.ofFloat(0f, 1f);
animator.setDuration(500);
animator.addUpdateListener(anim -> {
    float value = (float) anim.getAnimatedValue();
    myView.setAlpha(value);
    myView.setScaleX(value);
    myView.setScaleY(value);
});
animator.start();
```

> ✅ **Use when:** You need full custom control — e.g., animating a canvas drawing, a custom progress bar, or multiple views from a single animator.

---

### 4. `AnimatorSet` — Combine Animators

Run animations **together**, **sequentially**, or **after a delay**.

```java
ObjectAnimator scaleX  = ObjectAnimator.ofFloat(view, "scaleX", 1f, 1.3f);
ObjectAnimator scaleY  = ObjectAnimator.ofFloat(view, "scaleY", 1f, 1.3f);
ObjectAnimator fadeOut = ObjectAnimator.ofFloat(view, "alpha", 1f, 0f);

AnimatorSet set = new AnimatorSet();
set.play(scaleX).with(scaleY);       // run together
set.play(fadeOut).after(scaleX);     // run after scale finishes
set.setDuration(400);
set.start();
```

---

### 5. `ViewPropertyAnimator` — Fluent API (Preferred for Simple View Anims)

The cleanest, most readable API. Automatically uses hardware layers.

```java
myView.animate()
      .translationY(-150f)
      .alpha(0f)
      .scaleX(0.9f)
      .setDuration(300)
      .setInterpolator(new FastOutSlowInInterpolator())
      .withStartAction(() -> myView.setVisibility(View.VISIBLE))
      .withEndAction(() -> myView.setVisibility(View.GONE))
      .start();
```

> ✅ **Use when:** Simple, one-off animations on a single view with no complex sequencing.

---

### 6. `SpringAnimation` — Physics-Based (DynamicAnimation)

Simulates real-world spring physics. Great for gesture responses.

```java
SpringAnimation spring = new SpringAnimation(myView, DynamicAnimation.TRANSLATION_Y, 0f);
spring.getSpring()
      .setDampingRatio(SpringForce.DAMPING_RATIO_MEDIUM_BOUNCY)
      .setStiffness(SpringForce.STIFFNESS_LOW);

spring.setStartValue(300f); // start 300px down
spring.start();             // bounces to 0
```

---

### 7. `TransitionManager` — Automatic Layout Transitions

Detects layout changes and animates them automatically.

```java
// 1. Signal start of transition
TransitionManager.beginDelayedTransition(rootLayout, new AutoTransition());

// 2. Make layout changes — framework animates them
myView.setVisibility(View.GONE);
textView.setText("Expanded!");
```

---

### 8. `MotionLayout` — Complex Scene Animations (XML-Driven)

Extends `ConstraintLayout`. Defines start/end constraint states and transitions.

```xml
<!-- layout/activity_main.xml -->
<MotionLayout
    android:id="@+id/motionLayout"
    app:layoutDescription="@xml/motion_scene">
    <ImageView android:id="@+id/avatar" .../>
</MotionLayout>
```

```xml
<!-- res/xml/motion_scene.xml -->
<MotionScene>
    <Transition
        app:constraintSetStart="@id/start"
        app:constraintSetEnd="@id/end"
        app:duration="500">
        <OnSwipe
            app:touchAnchorId="@id/avatar"
            app:dragDirection="dragUp"/>
    </Transition>
    <ConstraintSet android:id="@+id/start">
        <Constraint android:id="@+id/avatar"
            android:layout_width="64dp"
            android:layout_height="64dp"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintStart_toStartOf="parent"/>
    </ConstraintSet>
    <ConstraintSet android:id="@+id/end">
        <Constraint android:id="@+id/avatar"
            android:layout_width="match_parent"
            android:layout_height="200dp"
            app:layout_constraintTop_toTopOf="parent"/>
    </ConstraintSet>
</MotionScene>
```

---

### 9. `AnimationDrawable` — Frame-by-Frame Drawable Animation

```xml
<!-- res/drawable/loading_spinner.xml -->
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot="false">
    <item android:drawable="@drawable/frame1" android:duration="80"/>
    <item android:drawable="@drawable/frame2" android:duration="80"/>
    <item android:drawable="@drawable/frame3" android:duration="80"/>
</animation-list>
```

```java
imageView.setImageResource(R.drawable.loading_spinner);
AnimationDrawable spinner = (AnimationDrawable) imageView.getDrawable();
// Must start after view is attached
imageView.post(() -> spinner.start());
```

---

---

## 🎨 Jetpack Compose — Animation Types

### 1. `animate*AsState` — Simplest State-Driven Animation

Animates a single value whenever state changes. Fire and forget.

```kotlin
var expanded by remember { mutableStateOf(false) }

// Each animates independently
val alpha     by animateFloatAsState(if (expanded) 1f else 0f, label = "alpha")
val size      by animateDpAsState(if (expanded) 200.dp else 50.dp, label = "size")
val color     by animateColorAsState(if (expanded) Color.Green else Color.Red, label = "color")
val elevation by animateDpAsState(if (expanded) 8.dp else 2.dp, label = "elevation")

Box(
    modifier = Modifier
        .size(size)
        .graphicsLayer { this.alpha = alpha }
        .background(color)
        .clickable { expanded = !expanded }
)
```

**Available variants:**
| Function | Animates |
|---|---|
| `animateFloatAsState` | Float |
| `animateDpAsState` | Dp |
| `animateColorAsState` | Color |
| `animateIntAsState` | Int |
| `animateOffsetAsState` | Offset |
| `animateSizeAsState` | Size |
| `animateRectAsState` | Rect |

---

### 2. `AnimatedVisibility` — Animate Show/Hide of Composables

```kotlin
var visible by remember { mutableStateOf(true) }

AnimatedVisibility(
    visible = visible,
    enter = slideInVertically(initialOffsetY = { -it }) + fadeIn(animationSpec = tween(300)),
    exit  = slideOutVertically(targetOffsetY = { -it }) + fadeOut(animationSpec = tween(200))
) {
    Card(modifier = Modifier.fillMaxWidth().padding(16.dp)) {
        Text("I animate in and out!", modifier = Modifier.padding(16.dp))
    }
}

Button(onClick = { visible = !visible }) {
    Text(if (visible) "Hide" else "Show")
}
```

**Enter/Exit options:**
```
Enter:  fadeIn, slideIn, slideInHorizontally, slideInVertically, expandIn, expandHorizontally, expandVertically, scaleIn
Exit:   fadeOut, slideOut, slideOutHorizontally, slideOutVertically, shrinkOut, shrinkHorizontally, shrinkVertically, scaleOut
```

---

### 3. `AnimatedContent` — Animate Between Different Content

```kotlin
var count by remember { mutableStateOf(0) }

AnimatedContent(
    targetState = count,
    transitionSpec = {
        // Slide direction depends on whether count goes up or down
        if (targetState > initialState) {
            slideInVertically { -it } + fadeIn() togetherWith slideOutVertically { it } + fadeOut()
        } else {
            slideInVertically { it }  + fadeIn() togetherWith slideOutVertically { -it } + fadeOut()
        }
    },
    label = "counter"
) { target ->
    Text(text = "$target", fontSize = 48.sp, fontWeight = FontWeight.Bold)
}

Row {
    Button(onClick = { count-- }) { Text("−") }
    Spacer(Modifier.width(8.dp))
    Button(onClick = { count++ }) { Text("+") }
}
```

---

### 4. `Crossfade` — Crossfade Between Composables

Simple swap with a crossfade effect. Lighter than `AnimatedContent`.

```kotlin
var currentScreen by remember { mutableStateOf("home") }

Crossfade(targetState = currentScreen, animationSpec = tween(400), label = "screen") { screen ->
    when (screen) {
        "home"    -> HomeScreen()
        "profile" -> ProfileScreen()
        "settings"-> SettingsScreen()
    }
}
```

---

### 5. `updateTransition` — Coordinate Multiple Properties

Ideal when **multiple properties** must animate in sync from the same state.

```kotlin
enum class CardState { Normal, Selected }

var cardState by remember { mutableStateOf(CardState.Normal) }
val transition = updateTransition(targetState = cardState, label = "cardTransition")

val scale     by transition.animateFloat(label = "scale")       { if (it == CardState.Selected) 1.08f else 1f }
val elevation by transition.animateDp(label = "elevation")      { if (it == CardState.Selected) 12.dp else 2.dp }
val bgColor   by transition.animateColor(label = "bgColor")     { if (it == CardState.Selected) Color(0xFFE3F2FD) else Color.White }
val border    by transition.animateColor(label = "borderColor") { if (it == CardState.Selected) Color.Blue else Color.LightGray }

Card(
    modifier = Modifier
        .graphicsLayer { scaleX = scale; scaleY = scale }
        .clickable { cardState = if (cardState == CardState.Normal) CardState.Selected else CardState.Normal },
    colors = CardDefaults.cardColors(containerColor = bgColor),
    elevation = CardDefaults.cardElevation(elevation),
    border = BorderStroke(1.5.dp, border)
) { ... }
```

---

### 6. `Animatable` — Full Manual / Imperative Control

Used in coroutines. Gives you `animateTo`, `snapTo`, `stop`, and value clamping.

```kotlin
val offsetX = remember { Animatable(0f) }

// Coroutine-driven animation
LaunchedEffect(Unit) {
    offsetX.animateTo(
        targetValue = 300f,
        animationSpec = spring(
            dampingRatio = Spring.DampingRatioMediumBouncy,
            stiffness = Spring.StiffnessLow
        )
    )
}

// Gesture-driven with bounds
val scope = rememberCoroutineScope()
Box(
    modifier = Modifier
        .offset { IntOffset(offsetX.value.roundToInt(), 0) }
        .draggable(
            orientation = Orientation.Horizontal,
            state = rememberDraggableState { delta ->
                scope.launch { offsetX.snapTo(offsetX.value + delta) }
            },
            onDragStopped = {
                scope.launch {
                    offsetX.animateTo(0f, spring()) // snap back
                }
            }
        )
)
```

---

### 7. `rememberInfiniteTransition` — Looping Animations

```kotlin
val infiniteTransition = rememberInfiniteTransition(label = "pulse")

val scale by infiniteTransition.animateFloat(
    initialValue = 1f,
    targetValue  = 1.25f,
    animationSpec = infiniteRepeatable(
        animation  = tween(700, easing = FastOutSlowInEasing),
        repeatMode = RepeatMode.Reverse
    ),
    label = "scale"
)

val glowAlpha by infiniteTransition.animateFloat(
    initialValue = 0.3f,
    targetValue  = 1f,
    animationSpec = infiniteRepeatable(
        animation  = tween(700),
        repeatMode = RepeatMode.Reverse
    ),
    label = "glow"
)

// Pulsing button
Box(
    modifier = Modifier
        .graphicsLayer { scaleX = scale; scaleY = scale; alpha = glowAlpha }
        .size(80.dp)
        .background(Color.Red, CircleShape)
)
```

---

### 8. Animation Specs — Control the Feel

```kotlin
// 1. tween — fixed duration with easing curve
tween(durationMillis = 400, easing = FastOutSlowInEasing)

// 2. spring — physics-based, no fixed duration
spring(
    dampingRatio = Spring.DampingRatioMediumBouncy,  // bounciness
    stiffness = Spring.StiffnessMedium               // speed
)

// 3. keyframes — precise multi-step control
keyframes {
    durationMillis = 600
    0f   at 0   with LinearEasing        // start at 0
    1.3f at 300 with FastOutSlowInEasing // overshoot at 300ms
    1f   at 600                          // settle at 1.0
}

// 4. infiniteRepeatable — loop forever
infiniteRepeatable(
    animation = tween(800),
    repeatMode = RepeatMode.Reverse  // or Restart
)

// 5. snap — instant, no animation
snap(delayMillis = 0)
```

---

---

## 🤔 Which One to Use & When?

### Android Views Decision Tree

```
Do you need animation in Views?
│
├─ Simple show/hide/move on a single view?
│   └─ ✅ ViewPropertyAnimator  →  view.animate().alpha(0f).translationY(100f).start()
│
├─ Need to animate a raw value / custom drawing?
│   └─ ✅ ValueAnimator
│
├─ Animating a real property (alpha, rotation, color)?
│   └─ ✅ ObjectAnimator
│
├─ Multiple animators in sequence or parallel?
│   └─ ✅ AnimatorSet
│
├─ Automatic layout change animation?
│   └─ ✅ TransitionManager.beginDelayedTransition()
│
├─ Gesture-driven, physics feel?
│   └─ ✅ SpringAnimation (DynamicAnimation)
│
└─ Complex multi-state scene with constraints?
    └─ ✅ MotionLayout
```

---

### Jetpack Compose Decision Tree

```
Do you need animation in Compose?
│
├─ Single value changes with state?
│   └─ ✅ animate*AsState  →  animateFloatAsState, animateDpAsState, etc.
│
├─ Show/hide a composable?
│   └─ ✅ AnimatedVisibility
│
├─ Swap between two composables with crossfade?
│   └─ ✅ Crossfade
│
├─ Complex content swap with custom transitions?
│   └─ ✅ AnimatedContent
│
├─ Multiple properties animating from same state?
│   └─ ✅ updateTransition
│
├─ Gesture-driven or manual coroutine control?
│   └─ ✅ Animatable
│
└─ Infinite looping animation (shimmer, pulse)?
    └─ ✅ rememberInfiniteTransition
```

---

### Views vs Compose — Side-by-Side

| Use Case | Android Views | Jetpack Compose |
|---|---|---|
| Fade in/out | `view.animate().alpha()` | `AnimatedVisibility` + `fadeIn/fadeOut` |
| Slide in/out | `ViewPropertyAnimator.translationX/Y` | `AnimatedVisibility` + `slideIn*` |
| Show/hide | `setVisibility` + `TransitionManager` | `AnimatedVisibility` |
| Color transition | `ObjectAnimator.ofArgb` | `animateColorAsState` |
| Content swap | `ViewFlipper` / `ViewSwitcher` | `Crossfade` / `AnimatedContent` |
| Multiple props sync | `AnimatorSet` | `updateTransition` |
| Looping animation | `ObjectAnimator.setRepeatCount(INFINITE)` | `rememberInfiniteTransition` |
| Physics/spring | `SpringAnimation` | `spring()` animationSpec |
| Gesture-driven | `VelocityTracker` + `SpringAnimation` | `Animatable` in coroutine |
| Custom value | `ValueAnimator` | `Animatable` / `animate*AsState` |
| Complex scenes | `MotionLayout` | `updateTransition` + `AnimatedContent` |

---

---

## ✅ Do's and ❌ Don'ts

### ✅ DO: Enable Hardware Layer During Animation

```java
// Views — toggle hardware layer for GPU-accelerated animation
myView.setLayerType(View.LAYER_TYPE_HARDWARE, null);
ObjectAnimator anim = ObjectAnimator.ofFloat(myView, "alpha", 0f, 1f);
anim.addListener(new AnimatorListenerAdapter() {
    @Override public void onAnimationEnd(Animator animation) {
        myView.setLayerType(View.LAYER_TYPE_NONE, null); // ← always restore!
    }
});
anim.start();
```

> Compose handles this automatically via `graphicsLayer`.

---

### ✅ DO: Animate Only Cheap Properties

```
✅ CHEAP  (GPU composited — no layout pass):
   alpha, translationX/Y, scaleX/Y, rotation, rotationX/Y

❌ EXPENSIVE (triggers layout + measure + draw):
   width, height, padding, margin
```

```java
// ❌ Bad — triggers relayout on every frame
ValueAnimator.ofInt(100, 300).addUpdateListener(a -> {
    view.getLayoutParams().width = (int) a.getAnimatedValue();
    view.requestLayout(); // layout pass every frame!
});

// ✅ Good — GPU only
view.animate().scaleX(1.5f).scaleY(1.5f).start();
```

---

### ✅ DO: Respect Accessibility (Reduced Motion)

```kotlin
// Compose
@Composable
fun AccessibleAnimation() {
    val reduceMotion = LocalReduceMotion.current

    val duration = if (reduceMotion) 0 else 400

    val alpha by animateFloatAsState(
        targetValue = 1f,
        animationSpec = tween(duration),
        label = "alpha"
    )
}
```

```java
// Views
boolean reduceMotion = Settings.Global.getFloat(
    context.getContentResolver(),
    Settings.Global.ANIMATOR_DURATION_SCALE, 1f
) == 0f;

int duration = reduceMotion ? 0 : 400;
```

---

### ✅ DO: Cancel Animations on View Recycle / Destroy

```java
// In RecyclerView.Adapter — cancel before rebinding
@Override
public void onBindViewHolder(MyViewHolder holder, int position) {
    holder.itemView.animate().cancel(); // ← cancel pending anim
    // ... bind new data
}

// In Activity/Fragment
@Override
protected void onDestroy() {
    myView.animate().cancel();
    animatorSet.cancel();
    mainHandler.removeCallbacksAndMessages(null);
    super.onDestroy();
}
```

---

### ✅ DO: Use Staggered Entry for Lists

```kotlin
// Compose — staggered list entry
items.forEachIndexed { index, item ->
    val visible by produceState(false) {
        delay(index * 60L) // stagger by 60ms per item
        value = true
    }
    AnimatedVisibility(visible = visible, enter = fadeIn() + slideInVertically { it }) {
        ItemCard(item)
    }
}
```

```java
// Views — staggered entry
for (int i = 0; i < views.size(); i++) {
    views.get(i).animate()
        .alpha(1f)
        .translationY(0f)
        .setStartDelay(i * 60L)
        .setDuration(250)
        .start();
}
```

---

### ❌ DON'T: Use View Animation for Interactive/Clickable Elements

```java
// ❌ Click area stays at original position — user gets confused!
Animation translate = new TranslateAnimation(0, 300, 0, 0);
translate.setDuration(300);
translate.setFillAfter(true);
myButton.startAnimation(translate); // button looks moved, but clicks work at original spot

// ✅ Actual position changes
myButton.animate().translationX(300f).start(); // click area moves too
```

---

### ❌ DON'T: Start AnimationDrawable in onCreate

```java
// ❌ Crashes — view not yet attached
@Override
protected void onCreate(Bundle savedInstanceState) {
    imageView.setImageResource(R.drawable.animation);
    ((AnimationDrawable) imageView.getDrawable()).start(); // NPE or no-op!
}

// ✅ Use post() to wait for attachment
imageView.post(() -> ((AnimationDrawable) imageView.getDrawable()).start());
```

---

### ❌ DON'T: Create Animatables in Composition Without `remember`

```kotlin
// ❌ New Animatable created on every recomposition — broken animation!
val offsetX = Animatable(0f)  // missing remember!

// ✅ Survive recompositions
val offsetX = remember { Animatable(0f) }
```

---

### ❌ DON'T: Block the Main Thread During Animations

```java
// ❌ Jank — blocks UI thread
new Handler().postDelayed(() -> {
    Thread.sleep(500); // NEVER do this on main thread
    view.animate().alpha(0f).start();
}, 100);

// ✅ Compose — use coroutines
LaunchedEffect(Unit) {
    delay(500)            // suspends, doesn't block
    animatable.animateTo(0f)
}
```

---

### ❌ DON'T: Overuse Animations

- Too many simultaneous animations → frame drops (target **60fps / 16ms per frame**)
- Long durations feel sluggish — **keep most UI animations between 150ms–400ms**
- Never animate just to show off — animations should **communicate** state changes, not distract

---

## ⏱ Recommended Durations

| Animation Type | Duration |
|---|---|
| Button press / ripple | 100–150ms |
| Fade in / fade out | 150–250ms |
| Slide in / slide out | 200–350ms |
| Expand / collapse | 250–400ms |
| Complex transitions | 300–500ms |
| Never exceed | 500ms for UI, 1s for hero animations |
