# Android Architecture & Design Patterns — Cheat Sheet

---

## 1. MVC – Model-View-Controller

- **Model**: data & business logic
- **View**: XML layouts
- **Controller**: Activity — receives input, updates Model, refreshes View
- **Problem**: Activity becomes both Controller *and* View → **God Activity**
- Tight coupling, untestable, does not scale.

---

## 2. MVP – Model-View-Presenter

- **Presenter**: plain Kotlin class, no Android framework dependency
- View communicates with Presenter via a **contract interface** — makes View passive (thin)
- 1:1 View ↔ Presenter mapping
- Must null out view reference in `onDestroy()` to prevent memory leaks

```kotlin
override fun onDestroy() { view = null }
```

- **Highly testable** — mock the `View` interface to unit test Presenter in isolation
- **Drawbacks**: contract boilerplate per screen; no native config-change handling

---

## 3. MVVM – Model-View-ViewModel

- **Google-recommended** pattern; default choice for modern Android
- **ViewModel** holds UI state as **`StateFlow`** / `LiveData`, survives config changes
- No View reference in ViewModel → no memory leak
- Follows **Unidirectional Data Flow (UDF)**: state flows **down**, events flow **up**

```kotlin
private val _uiState = MutableStateFlow(UserUiState())
val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()
```

- Collect safely with `repeatOnLifecycle(Lifecycle.State.STARTED)` (XML) or `collectAsStateWithLifecycle()` (Compose)
- **Drawbacks**: ViewModel can bloat without use cases; no built-in strict UDF enforcement; data binding (XML) can hide logic and make debugging harder

---

## 4. MVI – Model-View-Intent

- Enforces **strict UDF**: `Intent → Reducer → New State → View`
- All user actions modeled as a **`sealed interface`** (Intents)
- **Single immutable State** data class — single source of truth
- **Transient UI events** (toasts, navigation) modeled as nullable/boolean fields in State, cleared by the UI after consumption; a buffered `Channel` + `receiveAsFlow()` is also valid, but state-as-event survives config changes (tradeoff)

| Concept | Role |
|---|---|
| **Intent** | Sealed class for every user action |
| **State** | Immutable data class, entire screen state |
| **Reducer** | Pure function: `(state, intent) → newState` |
| **Transient UI event** | One-time events modeled as fields in State (nav, toast), consumed then cleared |

```kotlin
fun handleIntent(intent: UserIntent) {
    when (intent) {
        is UserIntent.LoadUser -> loadUser()
        is UserIntent.UpdateName -> _state.update { it.copy(name = intent.name) }
    }
}
```

- Reusable base: `abstract class MviViewModel<STATE, INTENT>` with `reduce {}` and `sendIntent()` — transient events live in STATE, not a separate `EFFECT`/`Channel`
- **Advantages**: predictable, traceable, time-travel debuggable, thread-safe (immutable state), perfect for Compose
- **Drawbacks**: more boilerplate than MVVM; overkill for simple screens

---

## 5. Clean Architecture

Proposed by **Robert C. Martin**. Concentric layers with one rule: **dependencies point inward only**.

| Layer | Contains | Depends On |
|---|---|---|
| **Domain** | Entities, Value Objects, Repository *interfaces* | Nothing |
| **Application** | Use Cases / Interactors | Domain |
| **Infrastructure** | Repository impls, APIs, DB | Domain, Application |
| **Presentation** | ViewModels, UI, Mappers | Application, Domain |

```kotlin
// Domain — repository interface (port)
interface UserRepository { suspend fun getUserById(id: String): User? }

// Application — use case
class GetUserUseCase(private val repository: UserRepository) {
    suspend operator fun invoke(id: String): Result<User> =
        runCatching { repository.getUserById(id) ?: throw NoSuchElementException() }
}
```

### DI: Hilt vs Koin

| | **Hilt** | **Koin** |
|---|---|---|
| Resolution | Compile-time (code gen) | Runtime (service locator) |
| Compile-time safety | Yes | No |
| Performance | Faster at runtime (pre-generated) | Slightly slower (runtime resolution) |
| Google recommended | Yes | No |
| KMP support | No | Yes |
| Setup | `@HiltAndroidApp`, `@HiltViewModel`, `@AndroidEntryPoint` | `startKoin { modules(...) }` |
| Best for | Production Android apps | Small apps, KMP, quick prototypes |

- **Hilt**: use `@Module @InstallIn(SingletonComponent::class)` for infra, `ViewModelComponent` for use cases
- **Koin**: `single {}` for singletons, `factory {}` for transient, `viewModel {}` for ViewModels

---

## 6. Domain-Driven Design (DDD)

Introduced by **Eric Evans**. Places the **business domain** at the center.

| Concept | Description |
|---|---|
| **Bounded Context** | Boundary within which a domain model is defined and consistent |
| **Entity** | Has unique identity persisting over time |
| **Value Object** | Immutable, defined by attributes only (no identity); use `@JvmInline value class` |
| **Aggregate** | Cluster of entities/VOs treated as one unit |
| **Aggregate Root** | Single entry point to an Aggregate — all access goes through it |
| **Repository** | Interface for persisting/retrieving Aggregates |
| **Domain Service** | Stateless operation that doesn't belong to an Entity |
| **Domain Event** | Sealed interface for meaningful occurrences (`OrderSubmitted`, `OrderCancelled`) |
| **Ubiquitous Language** | Shared vocabulary between devs and domain experts |

```kotlin
@JvmInline value class Email(val value: String) {
    init { require(value.contains("@")) { "Invalid email" } }
}

// Aggregate Root enforces invariants (methods inside class — not extension functions)
class Order private constructor(...) {
    fun submit(): Result<Unit> {
        if (_items.isEmpty()) return Result.failure(IllegalStateException("Empty order"))
        if (_status != OrderStatus.DRAFT) return Result.failure(IllegalStateException("Already $_status"))
        _status = OrderStatus.SUBMITTED
        return Result.success(Unit)
    }
    companion object { fun create(id: OrderId, customerId: UserId): Order = Order(...) }
}
```

- **Anti-Corruption Layer** isolates Bounded Contexts from each other
- Best for complex business rules; overkill for CRUD apps

---

## 7. Clean Architecture + DDD Combined

Clean Architecture → **layer structure**. DDD → **domain modeling tactics**.

```
Presentation   →  ViewModels, Compose/XML, Mappers
Application    →  Use Cases (orchestrate domain)
Domain (DDD)   →  Entities, Value Objects, Aggregates, Repo Interfaces, Domain Services
Infrastructure →  Repo Impls, API, DB, Framework
```

Module layout per feature:
```
feature/
├── domain/        # model/, repository/, service/
├── application/   # usecase/
├── infrastructure/# repository/, api/
└── presentation/  # viewmodel/, ui/
```

- Maximum testability at every layer; domain is fully framework-independent
- **Drawbacks**: most boilerplate; highest learning curve; requires team discipline

---

## 8. Comparison Matrix

| Criteria | MVC | MVP | MVVM | MVI | Clean Arch | DDD | Clean+DDD |
|---|---|---|---|---|---|---|---|
| Complexity | Low | Medium | Medium | Med-High | High | High | Very High |
| Testability | Poor | Good | Great | Excellent | Excellent | Great | Excellent |
| Boilerplate | None | Medium | Low | Medium | High | Medium | Very High |
| Scalability | Poor | Good | Good | Great | Excellent | Great | Excellent |
| Learning Curve | Low | Medium | Medium | Med-High | High | High | Very High |
| Config Change | No | No | Yes | Yes | Yes | N/A | Yes |
| State Predictability | Poor | Fair | Good | Excellent | Good | Good | Excellent |
| Android Ecosystem Fit | Poor | Fair | Excellent | Excellent | Good | Fair | Good |
| Compose Fit | Poor | Poor | Excellent | Excellent | Good | Fair | Good |
| Team Size | 1–2 | 2–5 | 2–10 | 2–10 | 5+ | 5+ | 5+ |

---

## 9. When to Use What

- **MVC** — prototype, 1–3 trivial screens, hours not days
- **MVP** — legacy projects, XML-only, need testable presenters without Compose
- **MVVM** — default for any modern Android app; Compose or XML with ViewBinding; lifecycle-aware state
- **MVI** — complex state, multi-step flows, real-time data, strict UDF enforcement, Compose
- **Clean Architecture** — medium-large app, multiple teams/modules, swappable implementations
- **DDD** — complex business rules, domain experts available, explicit consistency boundaries needed
- **Clean + DDD** — large-scale long-lived apps, multiple bounded contexts, financial/healthcare/automotive

### Decision Flowchart

```
Prototype / very simple?
  YES → MVC or plain MVVM
  NO
  └─ Complex state / many interactions?
       YES → MVI (or MVVM + strict UDF)
       NO/MODERATE
       └─ Domain beyond CRUD?
            NO  → MVVM + Repository Pattern
            YES
            └─ Multiple modules / teams?
                 NO  → MVVM/MVI + DDD tactical patterns
                 YES → Clean Architecture + DDD ⭐
```
