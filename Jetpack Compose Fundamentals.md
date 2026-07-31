# Jetpack Compose Fundamentals

Concise reference for building UI with Compose: composables, layouts, Modifiers, state & state hoisting, side-effects, and Material 3 theming.

> Scope note: internal rendering (recomposition/layout/draw phases, Slot Table) is covered in `Android Rendering Pipelines.md`; animation APIs are covered in `Android Animations Guide.md`. This guide is about the practical building blocks.

---

## Table of Contents

1. [Composition & Recomposition](#1-composition--recomposition)
2. [Layouts: Row, Column, Box](#2-layouts-row-column-box)
3. [Modifiers](#3-modifiers)
4. [State & State Hoisting](#4-state--state-hoisting)
5. [Side-Effects APIs](#5-side-effects-apis)
6. [Material 3 Theming](#6-material-3-theming)
7. [Interop with Views](#7-interop-with-views)
8. [Best Practices Checklist](#8-best-practices-checklist)
9. [Further Reading](#9-further-reading)

---

## 1. Composition & Recomposition

- Compose is **declarative** — the only way to update UI is calling the same composable with new arguments (no `findViewById` + mutate).
- **Composition** = the tree Compose builds by running `@Composable` functions. **Recomposition** = re-running composables whose inputs/state changed.
- A `TextField` won't update itself — it only reflects its `value` parameter. You must feed it new state.

```kotlin
@Composable
fun Greeting() {
    var name by remember { mutableStateOf("") }
    Column {
        if (name.isNotEmpty()) Text("Hello, $name!")
        OutlinedTextField(value = name, onValueChange = { name = it }, label = { Text("Name") })
    }
}
```

---

## 2. Layouts: Row, Column, Box

| Composable | Arranges children | Scoped modifiers |
|---|---|---|
| `Column` | Vertically | `verticalArrangement`, `horizontalAlignment`, `Modifier.weight()`, `Modifier.align()` |
| `Row` | Horizontally | `horizontalArrangement`, `verticalAlignment`, `Modifier.weight()`, `Modifier.align()` |
| `Box` | Stacked on top of each other | `contentAlignment`, `Modifier.align()`, `Modifier.matchParentSize()` |

**Layout model:** single-pass — parents measure before children but are sized/placed *after* children (bottom-up sizing, top-down constraints). No `RelativeLayout`-style multi-measure penalty; nest as deep as you need.

```kotlin
Row(verticalAlignment = Alignment.CenterVertically) {
    Image(bitmap = artist.image, contentDescription = null, modifier = Modifier.weight(2f))
    Column(Modifier.weight(1f)) {
        Text(artist.name)
        Text(artist.lastSeenOnline)
    }
}
```

**Scope safety:** modifiers like `weight` (Row/Column only) and `matchParentSize` (Box only) are only available inside their parent's scope — the compiler catches misuse, unlike View `LayoutParams`.

**Slot APIs:** components like `Scaffold`/`TopAppBar` expose named `@Composable` lambda parameters (`topBar`, `actions`, …) instead of dozens of primitive config params — pass your own composable into the "slot."

For scrollable/lazy lists (keys, `contentType`, prefetching) → see `UI Performance & Memory Leaks.md` (Fix 12) and `Android Rendering Pipelines.md` (LazyLayout section).

---

## 3. Modifiers

- Standard Kotlin objects; chain by calling functions on `Modifier`: `Modifier.padding(16.dp).fillMaxWidth()`.
- Every composable should **accept and pass through** a `modifier: Modifier = Modifier` parameter to its first child.
- **Order matters** — each function wraps the `Modifier` returned by the previous one:

```kotlin
// Whole area (incl. padding) is clickable — padding applied AFTER clickable
Modifier.clickable(onClick = onClick).padding(16.dp).fillMaxWidth()

// Padding area is NOT clickable — padding applied BEFORE clickable
Modifier.padding(16.dp).clickable(onClick = onClick).fillMaxWidth()
```

| Modifier | Effect |
|---|---|
| `padding(dp)` | Space inside the element's bounds |
| `size(w, h)` | Requests a size (parent constraints can override) |
| `requiredSize(dp)` | Forces a size regardless of parent constraints |
| `fillMaxWidth/Height/Size()` | Fill available space from parent |
| `offset(x, y)` | Shifts position **without** changing measured size (unlike `padding`) |
| `weight(f)` | Proportional space in `Row`/`Column` (scoped) |
| `matchParentSize()` | Same size as parent `Box`, without affecting its size (scoped) |

**Reuse modifier chains** as `val`s (they're immutable data-like objects) to avoid reallocating on every recomposition/frame — especially for `LazyColumn` item modifiers or frequently-animated values:

```kotlin
val cardModifier = Modifier.fillMaxWidth().padding(12.dp) // allocated once
LazyColumn { items(list) { ItemCard(it, modifier = cardModifier) } }
```

---

## 4. State & State Hoisting

```kotlin
var name by remember { mutableStateOf("") }        // survives recomposition
var name by rememberSaveable { mutableStateOf("") } // + survives config change/process death
```

| Concept | Rule |
|---|---|
| `remember` | Stores a value across recomposition; lost when the composable leaves Composition |
| `rememberSaveable` | Also survives Activity recreation / process death (via `Bundle`; use `@Parcelize`, `mapSaver`, or `listSaver` for non-bundleable types) |
| Stateful composable | Owns its state internally — simple, less reusable |
| Stateless composable | Takes `value` + `onValueChange` — reusable, testable |

**Hoisting = replace internal state with `value` + `onValueChange` params, moved to the caller:**

```kotlin
@Composable fun HelloScreen() {
    var name by rememberSaveable { mutableStateOf("") }
    HelloContent(name, onNameChange = { name = it })
}
@Composable fun HelloContent(name: String, onNameChange: (String) -> Unit) { /* stateless */ }
```

This is **Unidirectional Data Flow (UDF)**: state flows down, events flow up.

**Where to hoist — 3 rules:**
1. At least to the **lowest common parent** of all composables that *read* it.
2. At least to the **highest level that writes** it.
3. If two states **change together**, hoist them together.

| State type | Typical owner |
|---|---|
| UI element state (expanded/collapsed, scroll position) | The composable itself, or a **plain state holder class** (`remember { MyState() }`) if logic grows |
| Screen UI state (data from repositories, business logic) | **ViewModel** — lives outside the Composition, survives config changes |

```kotlin
class ConversationViewModel(repo: MessagesRepository) : ViewModel() {
    val messages = repo.getLatestMessages()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())
}

@Composable
fun ConversationScreen(viewModel: ConversationViewModel = viewModel()) {
    val messages by viewModel.messages.collectAsStateWithLifecycle() // lifecycle-aware collection
    MessagesList(messages)
}
```

- Prefer `collectAsStateWithLifecycle()` (needs `androidx.lifecycle:lifecycle-runtime-compose`) over `collectAsState()` — pauses collection when not `STARTED`.
- Never pass a `ViewModel` down through child composables — pass hoisted state + lambdas instead (avoids "property drilling" of the whole VM and keeps children testable/previewable).
- Don't use non-observable mutable objects (`ArrayList`, mutable `data class`) as state — Compose can't see the mutation. Use `State<List<T>>` + immutable `listOf()`.

---

## 5. Side-Effects APIs

A **side-effect** changes app state outside a composable's scope. Composables should otherwise be side-effect-free (they can recompose in any order, be skipped, or be discarded).

| API | Use for |
|---|---|
| `LaunchedEffect(keys)` | Run a suspend function scoped to the composable; cancelled on leaving Composition or key change |
| `rememberCoroutineScope()` | Launch a coroutine **from a callback** (e.g. `onClick`), scoped to the call site |
| `rememberUpdatedState(value)` | Reference the *latest* value inside a long-lived effect without restarting it |
| `DisposableEffect(keys)` | Effect that needs cleanup (`onDispose { }` required) — e.g. register/unregister a listener |
| `SideEffect { }` | Publish Compose state to non-Compose code on every successful recomposition |
| `produceState(initial, keys)` | Convert non-Compose async state (Flow/LiveData/callback) into a Compose `State` |
| `derivedStateOf { }` | Recompute a value only when its *result* changes, not every time an input changes (e.g. scroll-position threshold) |
| `snapshotFlow { }` | Convert Compose `State` reads into a cold `Flow` |

```kotlin
// LaunchedEffect — restart the pulse animation if pulseRateMs changes
LaunchedEffect(pulseRateMs) {
    while (isActive) { delay(pulseRateMs); alpha.animateTo(0f); alpha.animateTo(1f) }
}

// rememberCoroutineScope — launch from a click handler
val scope = rememberCoroutineScope()
Button(onClick = { scope.launch { snackbarHostState.showSnackbar("Done!") } }) { Text("Save") }

// DisposableEffect — register/unregister a lifecycle observer
DisposableEffect(lifecycleOwner) {
    val observer = LifecycleEventObserver { _, event -> /* ... */ }
    lifecycleOwner.lifecycle.addObserver(observer)
    onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
}

// derivedStateOf — only recompose when crossing the threshold, not on every scroll tick
val showButton by remember { derivedStateOf { listState.firstVisibleItemIndex > 0 } }
```

**Rules for restarting effects:** any mutable/immutable value read inside the effect block should be a **key**; values that shouldn't restart the effect go through `rememberUpdatedState` instead. `LaunchedEffect(true)`/`(Unit)` ties the effect to the call site's lifecycle only — treat it as suspiciously as a `while(true)`.

**Don't misuse `derivedStateOf`:** combining two states that should always update together (e.g. `"$firstName $lastName"`) needs no `derivedStateOf` — that's just a plain computed value.

---

## 6. Material 3 Theming

```kotlin
// build.gradle.kts
implementation("androidx.compose.material3:material3:<BOM-managed version>")
```

```kotlin
MaterialTheme(
    colorScheme = if (isSystemInDarkTheme()) DarkColorScheme else LightColorScheme,
    typography = AppTypography,
    shapes = AppShapes,
) { /* app content */ }
```

| Subsystem | Access | Notes |
|---|---|---|
| **Color** | `MaterialTheme.colorScheme.primary` | 5 key colors → tonal palettes; roles: primary/secondary/tertiary + `on*`/`*Container` variants |
| **Typography** | `MaterialTheme.typography.titleLarge` | Scale: display/headline/title/body/label × large/medium/small |
| **Shapes** | `MaterialTheme.shapes.medium` | extraSmall → extraLarge corner radii |

**Dynamic color** (Android 12+ / API 31+, derives palette from wallpaper):
```kotlin
val colors = when {
    Build.VERSION.SDK_INT >= 31 && darkTheme -> dynamicDarkColorScheme(LocalContext.current)
    Build.VERSION.SDK_INT >= 31 -> dynamicLightColorScheme(LocalContext.current)
    darkTheme -> DarkColorScheme
    else -> LightColorScheme
}
```

- Generate a starting `ColorScheme` + `Color.kt`/`Theme.kt` from brand colors via the [Material Theme Builder](https://material.io/material-theme-builder).
- Accessibility: always pair `on*` colors with their base (`onPrimary` on `primary`, `onPrimaryContainer` on `primaryContainer`) — mismatched pairs (e.g. `tertiaryContainer` + `primaryContainer`) break contrast.
- Navigation surfaces: `NavigationBar` (compact/phones), `NavigationRail` (medium width), `NavigationDrawer` (expanded/tablets) — see `Android Navigation Guide.md` for wiring these to a nav graph.

---

## 7. Interop with Views

Use only for SDK components with no Compose equivalent yet, or during incremental migration — see `migrate-xml-views-to-jetpack-compose` for the full migration workflow.

```kotlin
// Embed a legacy View
AndroidView(
    factory = { context -> MyView(context).apply { setOnClickListener { /* View -> Compose */ } } },
    update = { view -> view.selectedItem = selectedItem } // Compose -> View, recomposes on state read
)

// Embed an XML layout via ViewBinding
AndroidViewBinding(ExampleLayoutBinding::inflate) { exampleView.setBackgroundColor(Color.GRAY) }

// Embed a Fragment (transitional only — prefer single-Activity + Navigation 3 for new code)
AndroidFragment<MyFragment>()
```

- In `LazyColumn`/`LazyRow`, use the `AndroidView` overload with `onReset`/`onRelease` so the underlying `View` is reused instead of recreated when items recycle.
- **`CompositionLocal`** propagates ambient values implicitly (`LocalContext.current`, `LocalConfiguration.current`, `LocalView.current`) — read Android framework types without threading them through every parameter.

---

## 8. Best Practices Checklist

- [ ] Every composable accepts a `modifier: Modifier = Modifier` param and forwards it to its root child
- [ ] Chain modifiers in the order that produces the intended clickable/padding/layout behavior
- [ ] Hoist state to the lowest common ancestor — no higher, no lower
- [ ] Use `rememberSaveable` for state that must survive rotation/process death
- [ ] Never pass a `ViewModel` to non-screen-level composables — pass state + lambdas
- [ ] Use `collectAsStateWithLifecycle()` for Flows, not raw `collectAsState()`
- [ ] Wrap callback-triggered coroutines in `rememberCoroutineScope()`; long-lived ones in `LaunchedEffect`
- [ ] Always pair `DisposableEffect` with `onDispose {}`
- [ ] Reach for `derivedStateOf` only when inputs change more often than the UI needs to react
- [ ] Reuse `Modifier` chains as `val`s for frequently-recomposed/animated items
- [ ] Pull colors/type/shape from `MaterialTheme.*`, never hardcode values in components

---

## 9. Further Reading

| Resource | Link |
|---|---|
| State in Compose | https://developer.android.com/develop/ui/compose/state |
| State hoisting | https://developer.android.com/develop/ui/compose/state-hoisting |
| Compose layout basics | https://developer.android.com/develop/ui/compose/layouts/basics |
| Compose modifiers | https://developer.android.com/develop/ui/compose/modifiers |
| Side-effects in Compose | https://developer.android.com/develop/ui/compose/side-effects |
| Material Design 3 in Compose | https://developer.android.com/develop/ui/compose/designsystems/material3 |
| Views in Compose (interop) | https://developer.android.com/develop/ui/compose/migrate/interoperability-apis/views-in-compose |

---

*Last Updated: July 2026 · Compose BOM 2026.06.01.*
