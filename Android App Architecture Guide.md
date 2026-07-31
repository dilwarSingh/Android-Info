# Android App Architecture Guide

Concise reference for Google's official layered app architecture: UI layer (state holders, UDF, events), Data layer (repositories, sources of truth), Domain layer (use cases), and the official recommendations checklist.

> This complements `Android Design Patterns.md`, which compares MVC/MVP/MVVM/MVI/Clean Architecture/DDD as competing paradigms. This guide goes deeper into the **specific, official** layered architecture (UI/Domain/Data) Google recommends today — the two documents overlap somewhat by design.

---

## Table of Contents

1. [Layered Architecture Overview](#1-layered-architecture-overview)
2. [UI Layer: State](#2-ui-layer-state)
3. [UI Layer: State Holders](#3-ui-layer-state-holders)
4. [UI Layer: Events](#4-ui-layer-events)
5. [Data Layer: Repositories & Sources of Truth](#5-data-layer-repositories--sources-of-truth)
6. [Data Layer: Operation Types & Errors](#6-data-layer-operation-types--errors)
7. [Domain Layer: Use Cases](#7-domain-layer-use-cases)
8. [SavedStateHandle & Process Death](#8-savedstatehandle--process-death)
9. [Official Recommendations Summary](#9-official-recommendations-summary)
10. [Naming Conventions](#10-naming-conventions)
11. [Best Practices Checklist](#11-best-practices-checklist)
12. [Further Reading](#12-further-reading)

---

## 1. Layered Architecture Overview

```
UI layer (state holders + Compose UI)
        ↓ depends on
Domain layer (use cases) — optional
        ↓ depends on
Data layer (repositories → data sources)
```

| Layer | Responsibility | Depends on |
|---|---|---|
| **UI** | Display app data; primary point of user interaction | Domain (if present) or Data |
| **Domain** *(optional)* | Encapsulate business logic reused across multiple ViewModels | Data |
| **Data** | Business logic; owns and exposes app data | Nothing above it |

**Core principles:**
- **Separation of concerns** — don't put logic in `Activity`/`Fragment`; they're ephemeral (OS destroys/recreates them).
- **Drive UI from (persistent) data models** — survives process death, keeps working offline.
- **Single source of truth (SSOT)** — one owner per data type; only the SSOT mutates it; everyone else reads an immutable snapshot.
- **Unidirectional Data Flow (UDF)** — state flows down (SSOT → UI), events flow up (UI → SSOT).

---

## 2. UI Layer: State

| Type | Definition | Example |
|---|---|---|
| **Screen UI state** | *What* to display — app data shaped for the screen | `NewsUiState(newsItems, isSignedIn, ...)` |
| **UI element state** | Intrinsic rendering properties of a widget | `ScaffoldState`, expanded/collapsed flag |

```kotlin
data class NewsUiState(
    val isSignedIn: Boolean = false,
    val isPremium: Boolean = false,
    val newsItems: List<NewsItemUiState> = listOf(),
    val userMessages: List<Message> = listOf(),
)
```

- **Always immutable** — only the owning state holder mutates it (via `.copy()`), never the UI directly.
- **Single stream vs. multiple streams:** prefer **one** `uiState` object per screen (consistency, derived properties like `canBookmarkNews = isSignedIn && isPremium`); split into multiple streams only for genuinely unrelated data or very different update frequencies.
- Naming convention: `<Functionality>UiState` (e.g. `NewsUiState`, `NewsItemUiState`).

---

## 3. UI Layer: State Holders

| | Business logic state holder (**ViewModel**) | UI logic state holder (**plain class**) |
|---|---|---|
| Produces | Screen UI state (from Data/Domain layer) | UI element state / UI-only logic |
| Survives Activity recreation | ✅ Yes | ❌ No (recreated via `remember`) |
| Lifespan | As long as screen is on the back stack | Same as the Composition |
| Reusable across UIs | ❌ No — unique to one screen/function | ✅ Yes (e.g. a chip-group state holder) |
| Can hold `Context`/`Resources` | ❌ Avoid | ✅ Safe (same lifecycle as UI) |
| Typical impl | `ViewModel` (+ `@HiltViewModel`) | `remember { MyState(...) }` / `rememberSaveable` |

```kotlin
@HiltViewModel
class AuthorViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val authorsRepository: AuthorsRepository,
) : ViewModel() {
    val uiState: StateFlow<AuthorScreenUiState> = /* ... */
    fun followAuthor(followed: Boolean) { /* business logic */ }
}
```

```kotlin
// UI logic state holder — no business logic, no Activity-recreation survival needed
@Stable
class NiaAppState(val navController: NavHostController, val windowSizeClass: WindowSizeClass) {
    val shouldShowBottomBar: Boolean
        get() = windowSizeClass.widthSizeClass == WindowWidthSizeClass.Compact
    val shouldShowNavRail: Boolean get() = !shouldShowBottomBar
}
```

**Decision rule:** need business logic + must survive rotation + scoped to a nav destination → **ViewModel**. Shorter-lived, UI-only, potentially reusable → **plain state holder class**.

**Rules:**
- Never pass a `ViewModel` instance down to child composables — pass state + lambdas (`viewModel::doSomething`).
- State holders are **compoundable**: a shorter-lived holder can depend on a longer-lived one (UI logic holder ← screen ViewModel), never the reverse.
- Expose state via `StateFlow` (`stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), initial)`) or Compose `mutableStateOf` with `private set`.

---

## 4. UI Layer: Events

**Decision tree:**
```
Event originates in ViewModel?          → update UI state (always)
Event originates in UI, needs business logic?  → delegate to ViewModel
Event originates in UI, is pure UI logic?      → handle directly in the UI/state holder
```

| Logic type | Examples | Lives in |
|---|---|---|
| **Business logic** | Bookmarking an article, validating a payment | Domain/Data layer (ViewModel delegates) |
| **UI logic** | Expand/collapse, scroll-to-item, navigation call, which string resource to show | UI / plain state holder |

```kotlin
// UI logic handled directly (Compose)
var expanded by remember { mutableStateOf(false) }
Button(onClick = { expanded = !expanded }) { /* ... */ }

// Business logic delegated to ViewModel
Button(onClick = { viewModel.refreshNews() }) { /* ... */ }
```

**Golden rule: ViewModel events must always resolve to a UI state update — never send one-off events down to the UI via Channels/SharedFlow.** A producer (ViewModel) that outlives its consumer (UI) can't guarantee delivery through those APIs — the consumer might not be collecting when the event fires, silently dropping it.

```kotlin
// Model a one-off message AS STATE, and let the UI clear it once shown
data class NewsUiState(val userMessage: String? = null, /* ... */)

fun refreshNews() = viewModelScope.launch {
    if (!hasInternet()) _uiState.update { it.copy(userMessage = "No connection") }
}
fun userMessageShown() = _uiState.update { it.copy(userMessage = null) } // called after UI displays it
```

```kotlin
// Compose: show the message once, then tell the ViewModel it was consumed
viewModel.uiState.userMessage?.let { msg ->
    LaunchedEffect(msg) {
        snackbarHostState.showSnackbar(msg)
        viewModel.userMessageShown()
    }
}
```

**Navigation events:** if triggered by a plain UI tap → handle in the UI (`onClick = onHelp`). If it depends on business-logic validation (e.g. login success) → model as UI state (`isUserLoggedIn: Boolean`) and let a `LaunchedEffect` react to it. If the source screen stays on the back stack (so it would re-fire on recomposition), add a local `validationInProgress` flag (UI-owned) to gate the one-shot navigation.

**RecyclerView/list items:** never pass the `ViewModel` into an adapter. Expose a per-item UI state object with an embedded lambda instead:
```kotlin
data class NewsItemUiState(val title: String, val bookmarked: Boolean, val onBookmark: () -> Unit)
```

---

## 5. Data Layer: Repositories & Sources of Truth

```
Repository (1 per data type, e.g. NewsRepository)
   ↓ depends on
Data Source(s) (1 per source: Remote, Local — network, DB, file)
```

- **Repositories are the only entry point to the data layer** — ViewModels/use cases never touch a data source directly.
- Repository responsibilities: expose data, centralize changes, resolve multi-source conflicts, abstract sources away, hold business logic.
- **Source of truth**: usually the local DB (offline-first) or an in-memory cache; different repositories can pick different SSOTs.
- Repositories can depend on **other repositories** (aggregation) — sometimes called "managers" (`UserManager`).

```kotlin
class NewsRepository(
    private val remoteDataSource: NewsRemoteDataSource,
    private val localDataSource: NewsLocalDataSource,
) {
    val latestNews: Flow<List<Article>> = localDataSource.observeArticles() // SSOT = local DB
    suspend fun refresh() { localDataSource.save(remoteDataSource.fetchLatestNews()) }
}
```

**Exposing APIs:**
| Need | Expose |
|---|---|
| One-shot CRUD | `suspend fun` |
| Ongoing data changes | `Flow<T>` |

**Business models:** trim large API/DB models down to only what the app needs (`ArticleApiModel` → `Article`) — saves memory, decouples layers, lets teams work in parallel.

---

## 6. Data Layer: Operation Types & Errors

| Operation type | Canceled when | Example | Mechanism |
|---|---|---|---|
| **UI-oriented** | User leaves the screen | Fetch + display latest news | Follows ViewModel/`viewModelScope` lifecycle |
| **App-oriented** | App process is killed | Cache a network result for reuse | Repository-owned `CoroutineScope` (e.g. `SupervisorJob() + Dispatchers.Default`), started via `async`/`await` so it outlives the screen |
| **Business-oriented** | Never — must survive process death | Finish uploading a photo | **WorkManager** (see `Android WorkManager Guide.md`) |

```kotlin
class NewsRepository(
    private val remote: NewsRemoteDataSource,
    private val externalScope: CoroutineScope, // outlives the caller's screen scope
) {
    private val mutex = Mutex()
    private var cache: List<ArticleHeadline> = emptyList()

    suspend fun getLatestNews(refresh: Boolean = false) =
        if (refresh) externalScope.async {
            remote.fetchLatestNews().also { mutex.withLock { cache = it } }
        }.await() else mutex.withLock { cache }
}
```

**Errors:** use `try/catch` around `suspend` calls and `Flow.catch {}` for streams — let the **UI layer** handle/display the exception. Optionally wrap results in a `Result<T>` type to make expected failures explicit in the type system.

**Storage choice:** large/queryable/relational → **Room**; small key-value → **DataStore**; large blobs (JSON, bitmaps) → **File**. Full guidance → `Android Data Storage Guide.md`.

---

## 7. Domain Layer: Use Cases

Optional — add only when a ViewModel is too complex, or business logic must be **reused across multiple ViewModels**.

```kotlin
class GetTimeZoneUseCase(private val repository: SettingsRepository) {
    operator fun invoke(): TimeZone = repository.getTimeZone()
}

class NewsViewModel(
    private val getTimeZoneUseCase: GetTimeZoneUseCase,
    private val newsRepository: NewsRepository,
) : ViewModel() { /* ... */ }
```

- Naming: `<Action>UseCase` (e.g. `GetTimeZoneUseCase`), invoked with `operator fun invoke()` for call-site brevity.
- Each use case has **one** responsibility ("interactor" in some codebases).

---

## 8. SavedStateHandle & Process Death

- `ViewModel` survives **configuration changes** automatically, but not **process death**.
- Inject `SavedStateHandle` into the ViewModel constructor to persist small amounts of critical state (e.g. nav args, form input) across process death:

```kotlin
class AuthorViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    repo: AuthorsRepository,
) : ViewModel() {
    private val authorId: String = savedStateHandle.toRoute<AuthorRoute>().id
}
```

- For Compose-level UI element state, `rememberSaveable` is the equivalent mechanism (see `Jetpack Compose Fundamentals.md`).

---

## 9. Official Recommendations Summary

| Priority | Recommendation |
|---|---|
| Strongly recommended | Use a clearly defined Data layer (repositories, even single-source ones) and UI layer |
| Strongly recommended | UI layer never talks to a data source directly — always through a repository |
| Strongly recommended | Use coroutines + Flow to communicate between layers |
| Recommended (large apps) | Add a Domain layer when logic is reused across ViewModels |
| Strongly recommended | Follow UDF — ViewModel exposes state via observer pattern, UI sends events via method calls |
| Strongly recommended | Collect state with `collectAsStateWithLifecycle()` |
| Strongly recommended | Never send ViewModel → UI one-off events; always resolve to a state update |
| Strongly recommended | Single-Activity app, Navigation 3 for multi-screen apps |
| Strongly recommended | Keep ViewModels free of `Context`/`Activity`/`Resources`/lifecycle references |
| Strongly recommended | Use ViewModels only at screen level, not in reusable widgets (use plain state holders there) |
| Recommended | Don't use `AndroidViewModel` — move any `Application`-context need to UI/data layer |
| Recommended | Expose one `uiState: StateFlow` per screen (`stateIn` + `WhileSubscribed(5_000)` for stream-backed state; plain `MutableStateFlow` for simple cases) |
| Strongly recommended | Constructor injection + Hilt for DI in non-trivial apps |
| Strongly recommended | Test ViewModels, repositories/data sources, and add navigation regression tests; prefer fakes over mocks |

---

## 10. Naming Conventions

| Element | Convention | Example |
|---|---|---|
| Repository | `<Data>Repository` | `NewsRepository`, `PaymentsRepository` |
| Data source | `<Data><Remote|Local>DataSource` | `NewsRemoteDataSource`, `NewsLocalDataSource` |
| Use case | `<Action>UseCase` | `GetTimeZoneUseCase` |
| UI state | `<Functionality>UiState` | `NewsUiState`, `NewsItemUiState` |
| Data stream getter | `get<Model>Stream()` | `getAuthorStream(): Flow<Author>` |
| Methods | Verb phrase | `makePayment()` |
| Properties | Noun phrase | `inProgressTopicSelection` |
| Interface impl. | Meaningful name, `Default`/`Fake` prefix as fallback | `OfflineFirstNewsRepository`, `FakeAuthorsRepository` |

---

## 11. Best Practices Checklist

- [ ] Every screen has a clearly defined UI state class (`<Screen>UiState`), immutable, mutated only by its owner
- [ ] ViewModels never hold `Context`, `Activity`, `View`, or `Resources` references
- [ ] Business logic lives in Data/Domain layers; UI logic lives in the UI/plain state holders
- [ ] ViewModel → UI communication is always via state updates, never one-off Channels/events
- [ ] Repositories are the sole entry point into the data layer; no direct data-source access from ViewModels
- [ ] Pick one SSOT per data type (usually the local DB for offline-first apps)
- [ ] Match the operation to UI-oriented / app-oriented / business-oriented lifecycles (screen scope / external scope / WorkManager)
- [ ] Use `SavedStateHandle` for ViewModel state that must survive process death
- [ ] Add a Domain layer only once business logic is reused across multiple ViewModels
- [ ] Collect Flows with `collectAsStateWithLifecycle()`, never raw `collectAsState()` in production screens

---

## 12. Further Reading

| Resource | Link |
|---|---|
| Guide to app architecture | https://developer.android.com/topic/architecture |
| UI layer | https://developer.android.com/topic/architecture/ui-layer |
| UI events | https://developer.android.com/topic/architecture/ui-layer/events |
| State holders and UI state | https://developer.android.com/topic/architecture/ui-layer/stateholders |
| Data layer | https://developer.android.com/topic/architecture/data-layer |
| Recommendations for Android architecture | https://developer.android.com/topic/architecture/recommendations |

---

*Last Updated: July 2026.*
