# Android Navigation Guide

Concise reference for Jetpack Navigation Compose and Navigation 3: back stack management, type-safe routes, deep links, adaptive layouts, and common patterns (multi-backstack, conditional nav, results, dialogs).

---

## Table of Contents

1. [Navigation Compose vs Navigation 3](#1-navigation-compose-vs-navigation-3)
2. [Navigation Compose (Nav2) Basics](#2-navigation-compose-nav2-basics)
3. [Type-Safe Routes & Arguments](#3-type-safe-routes--arguments)
4. [Deep Links](#4-deep-links)
5. [Testing Navigation](#5-testing-navigation)
6. [Navigation 3 Basics](#6-navigation-3-basics)
7. [Navigation 3: Multiple Back Stacks](#7-navigation-3-multiple-back-stacks)
8. [Navigation 3: Conditional Navigation](#8-navigation-3-conditional-navigation)
9. [Navigation 3: Scenes (Dialogs & Adaptive Layouts)](#9-navigation-3-scenes-dialogs--adaptive-layouts)
10. [Navigation 3: Returning Results](#10-navigation-3-returning-results)
11. [Adaptive Navigation UI](#11-adaptive-navigation-ui)
12. [Best Practices Checklist](#12-best-practices-checklist)
13. [Further Reading](#13-further-reading)

---

## 1. Navigation Compose vs Navigation 3

| | Navigation Compose (Nav2) | Navigation 3 |
|---|---|---|
| Status | Stable, mature | Stable (1.1.5), Compose-first, actively recommended for new apps |
| Back stack control | Managed internally by `NavController` | **You own the back stack** — it's just a list you mutate |
| Adaptive / multi-pane | Limited (needs workarounds) | Built-in **Scenes** (dialog, list-detail, two-pane, supporting-pane) |
| Fragment interop | ✅ First-class | ❌ Compose-only |
| Setup complexity | Lower | Slightly higher (you write the back-stack/state classes) |
| Best for | Apps still using Fragments, or simple graphs | New Compose-only apps, adaptive/multi-pane UI, full back-stack control |

```kotlin
// build.gradle.kts
implementation("androidx.navigation:navigation-compose:2.9.8")      // Nav2
implementation("androidx.navigation3:navigation3-runtime:1.1.5")    // Nav3
implementation("androidx.navigation3:navigation3-ui:1.1.5")
implementation("androidx.lifecycle:lifecycle-viewmodel-navigation3:2.11.0")
```

---

## 2. Navigation Compose (Nav2) Basics

```kotlin
@Serializable object Home
@Serializable data class Profile(val id: String)

val navController = rememberNavController()

NavHost(navController, startDestination = Home) {
    composable<Home> { HomeScreen(onOpenProfile = { navController.navigate(Profile("user1234")) }) }
    composable<Profile> { backStackEntry ->
        val profile = backStackEntry.toRoute<Profile>()
        ProfileScreen(id = profile.id)
    }
}
```

- `NavHost` + `NavController` + a navigation graph = the three building blocks.
- **Don't pass the `NavController` down into every composable.** Pass navigation **lambdas** instead — keeps composables testable/previewable in isolation:

```kotlin
@Composable
fun ProfileScreen(userId: String, navigateToFriendProfile: (friendId: String) -> Unit) { /* ... */ }
```

---

## 3. Type-Safe Routes & Arguments

Requires Navigation 2.8.0+ and `kotlinx-serialization-json`.

| Before (string routes) | After (type-safe) |
|---|---|
| `"profile/{userId}"` | `@Serializable data class Profile(val userId: String)` |
| `navController.navigate("profile/$id")` | `navController.navigate(Profile(id))` |
| `backStackEntry.arguments?.getString("userId")` | `backStackEntry.toRoute<Profile>()` |

```kotlin
@Serializable data object Home                      // no args -> data object
@Serializable data class Profile(val userId: String) // with args -> data class
```

- **Never pass complex objects as arguments** — pass an ID, load the object from a single source of truth (repository) at the destination:

```kotlin
class UserViewModel(savedStateHandle: SavedStateHandle, repo: UserInfoRepository) : ViewModel() {
    private val profile = savedStateHandle.toRoute<Profile>()
    val userInfo = repo.getUserInfo(profile.userId)
}
```

- Custom types need a `NavType` + `typeMap` on `composable<T>(typeMap = mapOf(typeOf<X>() to XNavType))`.
- Group routes in a **sealed interface/class** for large graphs; use `object` (not `class`) for argument-free routes to avoid allocations.

---

## 4. Deep Links

```kotlin
composable<Profile>(
    deepLinks = listOf(navDeepLink<Profile>(basePath = "https://example.com/profile"))
) { backStackEntry -> ProfileScreen(id = backStackEntry.toRoute<Profile>().id) }
```

```xml
<!-- Manifest: expose the deep link externally -->
<activity ...>
    <intent-filter>
        <data android:scheme="https" android:host="www.example.com" />
    </intent-filter>
</activity>
```

Build a matching `PendingIntent` for notifications via `TaskStackBuilder.create(context).addNextIntentWithParentStack(deepLinkIntent)`.

---

## 5. Testing Navigation

```kotlin
// build.gradle.kts
androidTestImplementation("androidx.navigation:navigation-testing:2.9.8")
```

```kotlin
@Before
fun setup() {
    composeTestRule.setContent {
        navController = TestNavHostController(LocalContext.current)
        navController.navigatorProvider.addNavigator(ComposeNavigator())
        AppNavHost(navController)
    }
}

@Test
fun clickProfile_navigatesToProfile() {
    composeTestRule.onNodeWithContentDescription("Profile").performClick()
    assertTrue(navController.currentBackStackEntry?.destination?.hasRoute<Profile>() ?: false)
}
```

Wrap your graph in an `AppNavHost(navController)` composable so `TestNavHostController` can be injected — see `Android Testing Guide.md` for general Compose testing APIs.

---

## 6. Navigation 3 Basics

Core idea: **the back stack is just a `List` you mutate** — no hidden `NavController` state machine.

```kotlin
@Serializable data object Home : NavKey
@Serializable data class ChatDetail(val id: String) : NavKey

val backStack = rememberNavBackStack(Home)   // SnapshotStateList<NavKey>, saveable

NavDisplay(
    backStack = backStack,
    onBack = { backStack.removeLastOrNull() },
    entryProvider = entryProvider {
        entry<Home> { ContentScreen("Home") { Button(onClick = { backStack.add(ChatDetail("42")) }) { Text("Open chat") } } }
        entry<ChatDetail> { key -> ContentScreen("Chat ${key.id}") }
    }
)
```

| Piece | Role |
|---|---|
| `NavKey` | Marker interface for a route/destination key (`@Serializable` for state saving) |
| Back stack | `SnapshotStateList<NavKey>` — push with `.add()`, pop with `.removeLastOrNull()` |
| `entryProvider { entry<T> { } }` | Resolves a key to its `@Composable` content |
| `NavDisplay` | Renders the top of the back stack; auto-animates on stack changes |

---

## 7. Navigation 3: Multiple Back Stacks

Bottom-nav pattern where each tab keeps its own history — model with a small state holder that keeps one stack per top-level route and exposes a single **flattened** stack to `NavDisplay`:

```kotlin
class TopLevelBackStack<T : Any>(startKey: T) {
    private val topLevelStacks = linkedMapOf(startKey to mutableStateListOf(startKey))
    var topLevelKey by mutableStateOf(startKey); private set
    val backStack = mutableStateListOf(startKey)

    private fun updateBackStack() = backStack.apply { clear(); addAll(topLevelStacks.flatMap { it.value }) }

    fun addTopLevel(key: T) {
        topLevelStacks.getOrPut(key) { mutableStateListOf(key) }
        topLevelKey = key
        updateBackStack()
    }
    fun add(key: T) { topLevelStacks[topLevelKey]?.add(key); updateBackStack() }
    fun removeLast() {
        topLevelStacks[topLevelKey]?.removeLastOrNull()
        topLevelKey = topLevelStacks.keys.last()
        updateBackStack()
    }
}

// Usage
val topLevelBackStack = remember { TopLevelBackStack<Any>(Home) }
NavigationBar {
    TOP_LEVEL_ROUTES.forEach { route ->
        NavigationBarItem(
            selected = route == topLevelBackStack.topLevelKey,
            onClick = { topLevelBackStack.addTopLevel(route) },
            icon = { Icon(route.icon, null) }
        )
    }
}
NavDisplay(backStack = topLevelBackStack.backStack, onBack = { topLevelBackStack.removeLast() }, entryProvider = { /* ... */ })
```

For state that must survive **config change/process death**, back each tab's stack with `rememberSerializable` + `NavKeySerializer`/`MutableStateSerializer`, and decorate entries with `rememberSaveableStateHolderNavEntryDecorator()` so each tab retains scroll position/form state while inactive.

---

## 8. Navigation 3: Conditional Navigation

Gate destinations behind a condition (e.g. login) by centralizing the check in your `Navigator`/back-stack class:

```kotlin
sealed class ConditionalNavKey(val requiresLogin: Boolean = false) : NavKey
data object Profile : ConditionalNavKey(requiresLogin = true)
data class Login(val redirectToKey: ConditionalNavKey? = null) : ConditionalNavKey()

class Navigator(
    private val backStack: NavBackStack<ConditionalNavKey>,
    private val isLoggedIn: () -> Boolean,
) {
    fun navigate(key: ConditionalNavKey) {
        if (key.requiresLogin && !isLoggedIn()) backStack.add(Login(redirectToKey = key))
        else backStack.add(key)
    }
}
// On successful login: backStack.remove(loginEntry); navigate(redirectToKey)
```

Use the same pattern for onboarding-vs-main-app flows, feature flags, or entitlement checks.

---

## 9. Navigation 3: Scenes (Dialogs & Adaptive Layouts)

**Scenes** let `NavDisplay` render one or more back-stack entries in custom layouts (dialog, list-detail, two-pane) instead of a single full-screen destination.

```kotlin
val dialogStrategy = remember { DialogSceneStrategy<NavKey>() }

NavDisplay(
    backStack = backStack,
    sceneStrategies = listOf(dialogStrategy),
    entryProvider = entryProvider {
        entry<RouteA> { /* full screen */ }
        entry<RouteB>(metadata = DialogSceneStrategy.dialog(DialogProperties(windowTitle = "Details"))) { key ->
            /* rendered as a dialog instead of a full screen */
        }
    }
)
```

| Built-in / common Scene | Use case |
|---|---|
| `DialogSceneStrategy` | Show a destination as a dialog |
| List-Detail Scene (custom, or Material 3 Adaptive's `ListDetailPaneScaffold`) | Show list + detail side-by-side on large screens, single-pane on compact |
| Two-pane Scene | Generic side-by-side layout |
| Supporting-pane Scene | Main content + supporting panel |

For full adaptive window-size handling (compact/medium/expanded breakpoints, nav rail vs. bottom bar), see the `adaptive` skill.

---

## 10. Navigation 3: Returning Results

**State-based** (latest value only, doesn't survive config change/process death):

```kotlin
NavDisplay(
    entryDecorators = listOf(rememberResultEventBusNavEntryDecorator()),
    entryProvider = entryProvider {
        entry<Home> {
            val person by LocalResultEventBus.current.conflateAsState<Person?>(null)
            HomeScreen(person, onNext = { backStack.add(PersonDetailsForm()) })
        }
        entry<PersonDetailsForm> {
            val resultBus = LocalResultEventBus.current
            PersonDetailsScreen(onSubmit = { resultBus.sendResult(it); backStack.removeLastOrNull() })
        }
    }
)
```

**Event-based** (queued, delivered once) — use when every emitted result must be consumed (e.g. one-time confirmations); see the `results-event` recipe in the `navigation-3` skill for the `Channel`-backed variant.

---

## 11. Adaptive Navigation UI

```kotlin
NavigationSuiteScaffold(
    navigationSuiteItems = { /* items */ }
) {
    // Compact -> bottom NavigationBar, Medium/Expanded -> NavigationRail (auto-switches on WindowSizeClass)
}
```

| Screen width | Component |
|---|---|
| Compact (phones) | `NavigationBar` (bottom) |
| Medium (large phones / small tablets, landscape phones) | `NavigationRail` |
| Expanded (tablets, desktop) | `NavigationDrawer` (`PermanentNavigationDrawer` / `ModalNavigationDrawer`) |

`NavigationSuiteScaffold` auto-picks the right one from `WindowSizeClass`. Combine with `ListDetailPaneScaffold`/`SupportingPaneScaffold` for content panes — full breakpoint/window-size guidance lives in the `adaptive` skill.

---

## 12. Best Practices Checklist

- [ ] Use type-safe `@Serializable` routes, not string paths — compile-time argument safety
- [ ] Pass only IDs as nav arguments; load full objects from a repository at the destination
- [ ] Never thread `NavController`/back stack into leaf composables — pass lambdas
- [ ] For new Compose-only apps, default to **Navigation 3** for full back-stack control + Scenes
- [ ] Give each bottom-nav tab its own back stack if it needs independent history
- [ ] Decorate Nav3 entries with `rememberSaveableStateHolderNavEntryDecorator()` to retain per-tab UI state
- [ ] Centralize conditional-navigation logic (auth, entitlements) in one `Navigator`/back-stack class
- [ ] Use `DialogSceneStrategy`/list-detail Scenes instead of hand-rolled `if` branches for adaptive layouts
- [ ] Test navigation via `TestNavHostController` (Nav2) or by asserting on the back-stack list (Nav3), not by hardcoding UI flows

---

## 13. Further Reading

| Resource | Link |
|---|---|
| Navigation 3 overview | https://developer.android.com/guide/navigation/navigation-3 |
| Navigation Compose | https://developer.android.com/develop/ui/compose/navigation |
| Nav3 recipes (GitHub) | https://github.com/android/nav3-recipes |
| Navigation for responsive/adaptive UIs | https://developer.android.com/guide/topics/large-screens/navigation-for-responsive-uis |
| Local skill: `navigation-3` | Deep-links, scenes, multi-backstack, Hilt/Koin modular recipes |
| Local skill: `adaptive` | Window size classes, nav rail/bar switching, Grid/FlexBox |

---

*Last Updated: July 2026 · Navigation Compose 2.9.8, Navigation 3 (runtime/ui) 1.1.5.*
