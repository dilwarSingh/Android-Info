# Android Architecture & Design Patterns

A comprehensive guide to Android architecture patterns, Clean Architecture, and Domain-Driven Design (DDD).

---

## Table of Contents

1. [MVC – Model-View-Controller](#1-mvc--model-view-controller)
2. [MVP – Model-View-Presenter](#2-mvp--model-view-presenter)
3. [MVVM – Model-View-ViewModel](#3-mvvm--model-view-viewmodel)
4. [MVI – Model-View-Intent](#4-mvi--model-view-intent)
5. [Clean Architecture](#5-clean-architecture)
6. [Domain-Driven Design (DDD)](#6-domain-driven-design-ddd)
7. [Clean Architecture + DDD Combined](#7-clean-architecture--ddd-combined)
8. [Comparison Matrix](#8-comparison-matrix)
9. [When to Use What](#9-when-to-use-what)

---

## 1. MVC – Model-View-Controller

The oldest UI architecture pattern. The **Controller** acts as a mediator between the **Model** and the **View**.

```
┌───────────┐       ┌──────────────┐       ┌───────────┐
│   View    │──────▶│  Controller  │──────▶│   Model   │
│ (XML/UI)  │◀──────│  (Activity)  │◀──────│  (Data)   │
└───────────┘       └──────────────┘       └───────────┘
```

### How It Works

| Component  | Responsibility                                    |
|------------|---------------------------------------------------|
| **Model**  | Data & business logic                             |
| **View**   | UI rendering (XML layouts)                        |
| **Controller** | Receives user input, updates Model, refreshes View |

### Code Example

```kotlin
// --- Model ---
data class User(val name: String, val email: String)

class UserRepository {
    fun getUser(): User = User("Alice", "alice@example.com")
}

// --- Controller (Activity acts as both Controller and View in Android) ---
class UserActivity : AppCompatActivity() {

    private val repository = UserRepository()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_user)

        val user = repository.getUser()          // Controller fetches from Model
        findViewById<TextView>(R.id.tvName).text = user.name   // Controller updates View
        findViewById<TextView>(R.id.tvEmail).text = user.email
    }
}
```

### Pros

- ✅ Simple and easy to understand
- ✅ Quick prototyping
- ✅ Minimal boilerplate

### Cons

- ❌ **God Activity** — Activity becomes both Controller *and* View, leading to massive classes
- ❌ Tight coupling between View and Controller
- ❌ Extremely hard to unit test (UI + logic intertwined)
- ❌ Does not scale for complex apps

---

## 2. MVP – Model-View-Presenter

Decouples UI from business logic by introducing a **Presenter** that communicates with the View through an **interface (contract)**.

```
┌───────────┐       ┌──────────────┐       ┌───────────┐
│   View    │◀─────▶│  Presenter   │──────▶│   Model   │
│(Activity/ │       │ (Plain Class)│◀──────│  (Data)   │
│ Fragment) │       └──────────────┘       └───────────┘
└───────────┘
     ▲ implements
     │
┌────┴────────┐
│  View       │
│  Contract   │
│ (Interface) │
└─────────────┘
```

### How It Works

| Component   | Responsibility                              |
|-------------|---------------------------------------------|
| **Model**   | Data & business logic                       |
| **View**    | Renders UI, delegates user events to Presenter |
| **Presenter** | Fetches data from Model, formats it, calls View interface methods |

### Code Example

```kotlin
// --- Contract ---
interface UserContract {
    interface View {
        fun showUser(name: String, email: String)
        fun showError(message: String)
    }

    interface Presenter {
        fun loadUser()
        fun onDestroy()
    }
}

// --- Model ---
data class User(val name: String, val email: String)

class UserRepository {
    fun getUser(): User = User("Alice", "alice@example.com")
}

// --- Presenter ---
class UserPresenter(
    private var view: UserContract.View?,
    private val repository: UserRepository,
) : UserContract.Presenter {

    override fun loadUser() {
        try {
            val user = repository.getUser()
            view?.showUser(user.name, user.email)
        } catch (e: Exception) {
            view?.showError("Failed to load user")
        }
    }

    override fun onDestroy() {
        view = null  // Prevent memory leaks
    }
}

// --- View (Activity) ---
class UserActivity : AppCompatActivity(), UserContract.View {

    private lateinit var presenter: UserContract.Presenter

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_user)

        presenter = UserPresenter(this, UserRepository())
        presenter.loadUser()
    }

    override fun showUser(name: String, email: String) {
        findViewById<TextView>(R.id.tvName).text = name
        findViewById<TextView>(R.id.tvEmail).text = email
    }

    override fun showError(message: String) {
        Toast.makeText(this, message, Toast.LENGTH_SHORT).show()
    }

    override fun onDestroy() {
        presenter.onDestroy()
        super.onDestroy()
    }
}
```

### Pros

- ✅ Clear separation — Presenter has no Android framework dependency
- ✅ **Highly testable** — mock the View interface to unit test Presenter
- ✅ View is passive (thin) — just renders what Presenter tells it
- ✅ 1:1 mapping between View and Presenter

### Cons

- ❌ Lots of boilerplate (interfaces/contracts for every screen)
- ❌ Presenter can still become bloated for complex screens
- ❌ Manual lifecycle management (must null out view reference)
- ❌ Does not handle configuration changes (rotation) natively

---

## 3. MVVM – Model-View-ViewModel

The pattern most closely aligned with Google's official Android architecture guidance. Google's [Guide to app architecture](https://developer.android.com/topic/architecture) recommends a **layered architecture** (UI layer + Data layer + optional Domain layer) with **Unidirectional Data Flow (UDF)** and `ViewModel` as a state holder — MVVM is the closest named pattern to this recommendation. The ViewModel exposes **observable state** that the View reacts to, with state flowing down from ViewModel to View and events flowing up from View to ViewModel.

```
┌───────────┐  observes  ┌──────────────┐       ┌───────────┐
│   View    │───────────▶│  ViewModel   │──────▶│   Model   │
│(Activity/ │            │(StateFlow/   │◀──────│(Repository│
│ Fragment/ │            │ LiveData)    │       │ / UseCase)│
│ Compose)  │            └──────────────┘       └───────────┘
└───────────┘
      │  user actions (events flow up)
      └─────────────────▶ ViewModel

      UDF: State flows DOWN ↓  Events flow UP ↑
```

### How It Works

| Component    | Responsibility                                        |
|--------------|-------------------------------------------------------|
| **Model**    | Data layer (repositories, data sources)               |
| **View**     | Observes ViewModel state, renders UI, sends user actions |
| **ViewModel**| Holds UI state, exposes it as observable streams, survives config changes |

### Code Example

```kotlin
// --- Model ---
data class User(val name: String, val email: String)

class UserRepository {
    suspend fun getUser(): User = User("Alice", "alice@example.com")
}

// --- ViewModel ---
data class UserUiState(
    val name: String = "",
    val email: String = "",
    val isLoading: Boolean = false,
    val error: String? = null,
)

class UserViewModel(
    private val repository: UserRepository,
) : ViewModel() {

    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    fun loadUser() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            try {
                val user = repository.getUser()
                _uiState.update {
                    it.copy(
                        name = user.name,
                        email = user.email,
                        isLoading = false,
                    )
                }
            } catch (e: Exception) {
                _uiState.update {
                    it.copy(error = "Failed to load user", isLoading = false)
                }
            }
        }
    }
}

// --- ViewModel Factory (required when ViewModel has constructor parameters) ---
// For new code, prefer the lifecycle 2.5+ DSL: viewModelFactory { initializer { UserViewModel(repo) } }
class UserViewModelFactory(
    private val repository: UserRepository,
) : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        @Suppress("UNCHECKED_CAST")
        return UserViewModel(repository) as T
    }
}

// --- View (Activity with StateFlow collection) ---
class UserActivity : AppCompatActivity() {

    // Use factory when ViewModel has constructor dependencies.
    // With Hilt: simply annotate ViewModel with @HiltViewModel and use `by viewModels()`.
    private val viewModel: UserViewModel by viewModels {
        UserViewModelFactory(UserRepository())
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_user)

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    findViewById<TextView>(R.id.tvName).text = state.name
                    findViewById<TextView>(R.id.tvEmail).text = state.email
                    // Handle loading / error states ...
                }
            }
        }

        viewModel.loadUser()
    }
}

// --- View (Jetpack Compose) ---
@Composable
fun UserScreen(
    // UserViewModel needs a UserRepository — supply a factory (or use hiltViewModel())
    viewModel: UserViewModel = viewModel(factory = UserViewModelFactory(UserRepository()))
) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) { viewModel.loadUser() }

    Column {
        if (state.isLoading) {
            CircularProgressIndicator()
        } else {
            Text(text = state.name, style = MaterialTheme.typography.headlineMedium)
            Text(text = state.email, style = MaterialTheme.typography.bodyLarge)
        }
        state.error?.let { Text(text = it, color = Color.Red) }
    }
}
```

### Pros

- ✅ **Google-recommended** — first-class `ViewModel` and lifecycle support
- ✅ Survives configuration changes (screen rotation)
- ✅ Reactive UI — View automatically reacts to state changes
- ✅ No View reference in ViewModel — no memory leak risk
- ✅ Works perfectly with Jetpack Compose
- ✅ Highly testable (ViewModel is a plain class with observable state)
- ✅ Supports **Unidirectional Data Flow (UDF)** — Google's recommended data flow pattern

### Cons

- ❌ Steeper learning curve (Flows, coroutines, reactive programming)
- ❌ ViewModel can become bloated without further layering (use cases)
- ❌ Overkill for very simple screens
- ❌ Data binding (XML) can hide logic and make debugging harder
- ❌ No built-in mechanism to enforce strict UDF (events can be ad-hoc method calls)

---

## 4. MVI – Model-View-Intent

MVI enforces **strict Unidirectional Data Flow (UDF)** by modeling all user interactions as **Intents** (sealed classes), processing them through a **reducer**, and emitting a single immutable **State**. This makes state management predictable and debuggable.

```
┌───────────┐  Intent   ┌──────────────┐  Result  ┌───────────┐
│   View    │──────────▶│  ViewModel   │─────────▶│   Model   │
│(Activity/ │           │  (Reducer)   │◀─────────│(Repository│
│ Fragment/ │  State    │              │          │ / UseCase)│
│ Compose)  │◀──────────│  StateFlow   │          └───────────┘
└───────────┘           └──────────────┘

    ┌─────────────────────────────────────────┐
    │        Unidirectional Data Flow         │
    │                                         │
    │  Intent → Reducer → New State → View    │
    │     ↑                           │       │
    │     └───────── User Action ─────┘       │
    └─────────────────────────────────────────┘
```

### How It Works

| Component    | Responsibility                                              |
|--------------|-------------------------------------------------------------|
| **Model**    | The current immutable UI state (single source of truth)     |
| **View**     | Renders state, emits user Intents                           |
| **Intent**   | A sealed class representing every possible user action      |
| **ViewModel**| Receives Intents, processes them through a reducer, emits new State |

### Key Concepts

| Concept         | Description                                                      |
|-----------------|------------------------------------------------------------------|
| **Intent**      | Sealed class representing a user action (e.g., `LoadUser`, `Retry`) |
| **State**       | Single immutable data class representing the entire screen state |
| **Reducer**     | Pure function: `(currentState, intent) → newState`               |
| **Transient UI events** | One-time events (navigation, toasts) modeled as nullable/boolean fields in the State, cleared by the UI after consumption |

### Code Example

```kotlin
// --- Intents (all possible user actions) ---
sealed interface UserIntent {
    data object LoadUser : UserIntent
    data object Retry : UserIntent
    data class UpdateName(val name: String) : UserIntent
}

// --- State (single immutable source of truth) ---
// Transient events (toasts, navigation) are modeled directly in state here.
// This is one recommended approach per official Android architecture guidance.
// A buffered Channel + receiveAsFlow() is also valid for one-off events; the
// tradeoff is that state-as-event survives config changes, while an unbuffered
// channel can drop events if the UI isn't collecting. Pick per use case.
data class UserState(
    val name: String = "",
    val email: String = "",
    val isLoading: Boolean = false,
    val error: String? = null,
    val userMessage: String? = null,
    val navigateBack: Boolean = false,
)

// --- Model ---
data class User(val name: String, val email: String)

class UserRepository {
    suspend fun getUser(): User = User("Alice", "alice@example.com")
}

// --- ViewModel (processes intents, emits state) ---
class UserViewModel(
    private val repository: UserRepository,
) : ViewModel() {

    private val _state = MutableStateFlow(UserState())
    val state: StateFlow<UserState> = _state.asStateFlow()

    fun handleIntent(intent: UserIntent) {
        when (intent) {
            is UserIntent.LoadUser,
            is UserIntent.Retry -> loadUser()
            is UserIntent.UpdateName -> {
                _state.update { it.copy(name = intent.name) }
            }
        }
    }

    fun consumeUserMessage() {
        _state.update { it.copy(userMessage = null) }
    }

    fun consumeNavigateBack() {
        _state.update { it.copy(navigateBack = false) }
    }

    private fun loadUser() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            try {
                val user = repository.getUser()
                _state.update {
                    it.copy(
                        name = user.name,
                        email = user.email,
                        isLoading = false,
                    )
                }
            } catch (e: Exception) {
                _state.update {
                    it.copy(
                        error = "Failed to load user",
                        isLoading = false,
                        userMessage = "Network error",
                    )
                }
            }
        }
    }
}

// --- View (Jetpack Compose) ---
@Composable
fun UserScreen(
    // UserViewModel needs a UserRepository — supply a factory (or use hiltViewModel())
    viewModel: UserViewModel = viewModel(factory = UserViewModelFactory(UserRepository()))
) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    // Handle transient events from state — side effects belong in LaunchedEffect,
    // never in the composable body (which re-runs on every recomposition)
    val context = LocalContext.current
    LaunchedEffect(state.userMessage) {
        state.userMessage?.let { message ->
            Toast.makeText(context, message, Toast.LENGTH_SHORT).show()
            viewModel.consumeUserMessage()
        }
    }

    LaunchedEffect(state.navigateBack) {
        if (state.navigateBack) {
            viewModel.consumeNavigateBack()
            // perform navigation
        }
    }

    // Send intent on first composition
    LaunchedEffect(Unit) {
        viewModel.handleIntent(UserIntent.LoadUser)
    }

    Column {
        when {
            state.isLoading -> CircularProgressIndicator()
            state.error != null -> {
                Text(text = state.error!!, color = Color.Red)
                Button(onClick = { viewModel.handleIntent(UserIntent.Retry) }) {
                    Text("Retry")
                }
            }
            else -> {
                Text(text = state.name, style = MaterialTheme.typography.headlineMedium)
                Text(text = state.email, style = MaterialTheme.typography.bodyLarge)
            }
        }
    }
}
```

### MVI with a Reusable Base Class (No Third-Party Library)

```kotlin
// A lightweight, reusable MVI base class using only Kotlin coroutines + Android ViewModel.
// Can be placed in a :core module and reused across all features.
// Transient events are modeled as part of STATE (not via Channels).

abstract class MviViewModel<STATE, INTENT>(
    initialState: STATE,
) : ViewModel() {

    private val _state = MutableStateFlow(initialState)
    val state: StateFlow<STATE> = _state.asStateFlow()

    // Subclasses implement this to map intents → state changes
    protected abstract fun handleIntent(intent: INTENT)

    fun sendIntent(intent: INTENT) {
        handleIntent(intent)
    }

    protected fun reduce(block: STATE.() -> STATE) {
        _state.update { it.block() }
    }
}

// --- Usage: concrete ViewModel using the base class ---

class UserViewModel(
    private val repository: UserRepository,
) : MviViewModel<UserState, UserIntent>(UserState()) {

    override fun handleIntent(intent: UserIntent) {
        when (intent) {
            is UserIntent.LoadUser,
            is UserIntent.Retry -> loadUser()
            is UserIntent.UpdateName -> reduce { copy(name = intent.name) }
        }
    }

    fun consumeUserMessage() {
        reduce { copy(userMessage = null) }
    }

    fun consumeNavigateBack() {
        reduce { copy(navigateBack = false) }
    }

    private fun loadUser() {
        viewModelScope.launch {
            reduce { copy(isLoading = true, error = null) }
            try {
                val user = repository.getUser()
                reduce { copy(name = user.name, email = user.email, isLoading = false) }
            } catch (e: Exception) {
                reduce { copy(error = "Failed to load", isLoading = false, userMessage = "Network error") }
            }
        }
    }
}

// --- Compose UI collects state (including transient events) ---
@Composable
fun UserScreen(
    // UserViewModel needs a UserRepository — supply a factory (or use hiltViewModel())
    viewModel: UserViewModel = viewModel(factory = UserViewModelFactory(UserRepository()))
) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    // Handle transient events from state — keep side effects in LaunchedEffect,
    // not in the composable body (which re-runs on every recomposition)
    val context = LocalContext.current
    LaunchedEffect(state.userMessage) {
        state.userMessage?.let { message ->
            Toast.makeText(context, message, Toast.LENGTH_SHORT).show()
            viewModel.consumeUserMessage()
        }
    }

    LaunchedEffect(state.navigateBack) {
        if (state.navigateBack) {
            viewModel.consumeNavigateBack()
            // perform navigation
        }
    }

    LaunchedEffect(Unit) { viewModel.sendIntent(UserIntent.LoadUser) }

    Column {
        when {
            state.isLoading -> CircularProgressIndicator()
            state.error != null -> {
                Text(text = state.error!!, color = Color.Red)
                Button(onClick = { viewModel.sendIntent(UserIntent.Retry) }) {
                    Text("Retry")
                }
            }
            else -> {
                Text(text = state.name, style = MaterialTheme.typography.headlineMedium)
                Text(text = state.email, style = MaterialTheme.typography.bodyLarge)
            }
        }
    }
}
```

### Pros

- ✅ **Predictable state** — single source of truth, immutable state
- ✅ **Easy to debug** — every state change is traceable through intents
- ✅ **Time-travel debugging** — can replay intents to reproduce bugs
- ✅ **Thread-safe** — immutable state eliminates race conditions
- ✅ **Scales well** — adding features = adding new intents
- ✅ **Perfect for Compose** — Compose's declarative model aligns naturally with MVI
- ✅ **Strict UDF** — enforces unidirectional data flow by design

### Cons

- ❌ **More boilerplate** than MVVM (intents, state, reducer)
- ❌ **Steeper learning curve** — requires understanding of UDF and reducers
- ❌ **Overkill for simple screens** — a login form may not need sealed intents
- ❌ **Performance** — every action creates new state objects (usually negligible)
- ❌ **Transient event handling** requires discipline — must remember to clear consumed state fields

---

## 5. Clean Architecture

Proposed by **Robert C. Martin (Uncle Bob)**, Clean Architecture organizes code into concentric layers with strict **dependency rules**: outer layers depend on inner layers, never the reverse.

```
┌─────────────────────────────────────────────────────┐
│                  Presentation Layer                  │
│         (ViewModels, UI, Mappers, Adapters)          │
│  ┌───────────────────────────────────────────────┐  │
│  │              Application Layer                 │  │
│  │            (Use Cases / Interactors)           │  │
│  │  ┌───────────────────────────────────────┐    │  │
│  │  │           Domain Layer                 │    │  │
│  │  │   (Entities, Value Objects, Repo       │    │  │
│  │  │    Interfaces, Domain Services)        │    │  │
│  │  └───────────────────────────────────────┘    │  │
│  └───────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────┐  │
│  │          Infrastructure Layer                  │  │
│  │  (Repository Impls, APIs, DB, Framework)       │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘

            Dependency Rule: ───────▶ Inward only
```

### Layer Responsibilities

| Layer              | Contains                                      | Depends On       |
|--------------------|-----------------------------------------------|------------------|
| **Domain**         | Entities, Value Objects, Repository interfaces | Nothing          |
| **Application**    | Use Cases / Interactors                       | Domain           |
| **Infrastructure** | Repository implementations, APIs, DB, frameworks | Domain, Application |
| **Presentation**   | ViewModels, UI components, Mappers            | Application, Domain |

### Code Example

```kotlin
// ══════════════════════════════════════
//  DOMAIN LAYER (innermost — no dependencies)
// ══════════════════════════════════════

// Entity
data class User(
    val id: String,
    val name: String,
    val email: String,
)

// Repository Interface (port)
interface UserRepository {
    suspend fun getUserById(id: String): User?
    suspend fun saveUser(user: User)
}

// ══════════════════════════════════════
//  APPLICATION LAYER (Use Cases)
// ══════════════════════════════════════

class GetUserUseCase(private val repository: UserRepository) {
    suspend operator fun invoke(userId: String): Result<User> =
        runCatching {
            repository.getUserById(userId)
                ?: throw NoSuchElementException("User not found: $userId")
        }
}

// ══════════════════════════════════════
//  INFRASTRUCTURE LAYER (implementations)
// ══════════════════════════════════════

class UserRepositoryImpl(
    private val api: UserApi,
    private val dao: UserDao,
) : UserRepository {

    override suspend fun getUserById(id: String): User? {
        // Try cache first, then network
        return dao.findById(id) ?: api.fetchUser(id)?.also { dao.insert(it) }
    }

    override suspend fun saveUser(user: User) {
        dao.insert(user)
        api.updateUser(user)
    }
}

// ══════════════════════════════════════
//  PRESENTATION LAYER (ViewModel)
// ══════════════════════════════════════

class UserViewModel(
    private val getUserUseCase: GetUserUseCase,
) : ViewModel() {

    private val _uiState = MutableStateFlow<UserUiState>(UserUiState.Loading)
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    fun loadUser(userId: String) {
        viewModelScope.launch {
            _uiState.value = UserUiState.Loading
            getUserUseCase(userId)
                .onSuccess { user ->
                    _uiState.value = UserUiState.Success(user.name, user.email)
                }
                .onFailure { error ->
                    _uiState.value = UserUiState.Error(error.message ?: "Unknown error")
                }
        }
    }
}

sealed interface UserUiState {
    data object Loading : UserUiState
    data class Success(val name: String, val email: String) : UserUiState
    data class Error(val message: String) : UserUiState
}
```

### Dependency Injection

#### With Hilt (Google Recommended)

```kotlin
// Hilt is Google's recommended DI framework for Android, built on top of Dagger.
// It uses compile-time code generation for maximum performance.

// 1. Add Hilt Gradle plugin and dependencies:
//    plugins { id("com.google.dagger.hilt.android") }
//    dependencies {
//        implementation("com.google.dagger:hilt-android:<version>")
//        ksp("com.google.dagger:hilt-android-compiler:<version>")  // Kotlin DSL + KSP: use "hilt-android-compiler"
//        // Note: Groovy build scripts use "hilt-compiler" — Kotlin DSL requires "hilt-android-compiler"
//    }

// 2. Annotate your Application class:
@HiltAndroidApp
class MyApplication : Application()

// 3. Define modules to provide dependencies:
@Module
@InstallIn(SingletonComponent::class)
object InfrastructureModule {

    @Provides
    @Singleton
    fun provideUserApi(): UserApi = UserApi()

    @Provides
    @Singleton
    fun provideUserDao(): UserDao = UserDao()

    @Provides
    @Singleton
    fun provideUserRepository(api: UserApi, dao: UserDao): UserRepository =
        UserRepositoryImpl(api, dao)
}

@Module
@InstallIn(ViewModelComponent::class)
object DomainModule {

    @Provides
    fun provideGetUserUseCase(repository: UserRepository): GetUserUseCase =
        GetUserUseCase(repository)
}

// 4. Annotate ViewModel with @HiltViewModel — no manual factory needed:
@HiltViewModel
class UserViewModel @Inject constructor(
    private val getUserUseCase: GetUserUseCase,
) : ViewModel() {
    // ... same ViewModel code as above
}

// 5. Annotate Activity/Fragment with @AndroidEntryPoint:
@AndroidEntryPoint
class UserActivity : AppCompatActivity() {
    private val viewModel: UserViewModel by viewModels()  // Hilt handles injection
    // ...
}
```

#### With Koin (Lightweight Alternative)

```kotlin
// Koin is a lightweight, pure-Kotlin DI framework (no code generation).
// Easier to set up than Hilt, but uses runtime resolution (no compile-time safety).

// 1. Add Koin dependencies:
//    dependencies {
//        implementation("io.insert-koin:koin-android:<version>")
//        implementation("io.insert-koin:koin-androidx-compose:<version>")  // for Compose
//    }

// 2. Define modules:
val infrastructureModule = module {
    single { UserApi() }
    single { UserDao() }
    single<UserRepository> { UserRepositoryImpl(get(), get()) }
}

val domainModule = module {
    factory { GetUserUseCase(get()) }
}

val presentationModule = module {
    viewModel { UserViewModel(get()) }
}

// 3. Start Koin in your Application class:
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidContext(this@MyApplication)
            modules(infrastructureModule, domainModule, presentationModule)
        }
    }
}

// 4. Inject in Activity:
class UserActivity : AppCompatActivity() {
    private val viewModel: UserViewModel by viewModel()  // Koin resolves dependencies
    // ...
}

// 5. Inject in Compose:
@Composable
fun UserScreen(viewModel: UserViewModel = koinViewModel()) {
    // ...
}
```

#### Hilt vs Koin Comparison

| Criteria              | Hilt                              | Koin                             |
|-----------------------|-----------------------------------|----------------------------------|
| **Type**              | Compile-time (code generation)    | Runtime (service locator)        |
| **Setup complexity**  | Medium (Gradle plugin + annotations) | Low (pure Kotlin DSL)         |
| **Compile-time safety** | ✅ Yes — errors caught at build time | ❌ No — errors at runtime     |
| **Performance**       | Faster at runtime (pre-generated) | Slightly slower (reflection-free but runtime resolution) |
| **Learning curve**    | Steeper (Dagger concepts)         | Easier (simple DSL)              |
| **Google recommended**| ✅ Yes                             | ❌ No (community-driven)        |
| **Multiplatform (KMP)** | ❌ Android only                  | ✅ Supports KMP                  |
| **Best for**          | Production Android apps           | Small apps, KMP, quick prototypes |

### Pros

- ✅ **Framework independence** — Domain knows nothing about Android
- ✅ **Highly testable** — each layer tested in isolation
- ✅ **Flexible** — swap database, API, or UI framework without touching business logic
- ✅ **Scalable** — clear boundaries make large codebases manageable
- ✅ Enforces Single Responsibility Principle

### Cons

- ❌ **Significant boilerplate** — interfaces, mappers, and many small classes
- ❌ **Over-engineering risk** for small apps
- ❌ Steeper learning curve for new developers
- ❌ Mapping between layer models can feel repetitive

---

## 6. Domain-Driven Design (DDD)

DDD is a **strategic and tactical design approach** that places the **business domain** at the center of software architecture. It was introduced by **Eric Evans**.

### Strategic Patterns

```
┌─────────────────────────────────────────────────┐
│                Bounded Context A                │
│  ┌─────────┐  ┌─────────┐  ┌────────────────┐  │
│  │Aggregate│  │Aggregate│  │ Domain Service  │  │
│  │  Root   │  │  Root   │  │                 │  │
│  │ ┌─────┐ │  │ ┌─────┐ │  └────────────────┘  │
│  │ │Value│ │  │ │Value│ │                       │
│  │ │Obj. │ │  │ │Obj. │ │  ┌────────────────┐  │
│  │ └─────┘ │  │ └─────┘ │  │  Repository    │  │
│  └─────────┘  └─────────┘  │  (interface)   │  │
│                             └────────────────┘  │
└─────────────────────────────────────────────────┘
                      │
              Anti-Corruption Layer
                      │
┌─────────────────────────────────────────────────┐
│                Bounded Context B                │
└─────────────────────────────────────────────────┘
```

### Key Concepts

| Concept               | Description                                                        |
|-----------------------|--------------------------------------------------------------------|
| **Bounded Context**   | A boundary within which a domain model is defined and consistent   |
| **Entity**            | Object with a unique identity that persists over time              |
| **Value Object**      | Immutable object defined only by its attributes (no identity)      |
| **Aggregate**         | Cluster of entities/value objects treated as a single unit         |
| **Aggregate Root**    | The entry point entity for an Aggregate — all access goes through it |
| **Repository**        | Abstraction for persisting and retrieving Aggregates               |
| **Domain Service**    | Stateless operation that doesn't naturally belong to an Entity     |
| **Domain Event**      | Something meaningful that happened in the domain                   |
| **Ubiquitous Language** | Shared vocabulary between developers and domain experts          |

### Code Example

```kotlin
// ══════════════════════════════════════
//  VALUE OBJECTS (immutable, no identity)
// ══════════════════════════════════════

@JvmInline
value class UserId(val value: String) {
    init {
        require(value.isNotBlank()) { "UserId must not be blank" }
    }
}

@JvmInline
value class OrderId(val value: String) {
    init {
        require(value.isNotBlank()) { "OrderId must not be blank" }
    }
}

@JvmInline
value class Email(val value: String) {
    init {
        require(value.contains("@")) { "Invalid email format: $value" }
    }
}

data class Address(
    val street: String,
    val city: String,
    val zipCode: String,
    val country: String,
)

// ══════════════════════════════════════
//  VALUE OBJECT (no identity — equal by attributes)
// ══════════════════════════════════════

data class OrderItem(
    val productId: String,
    val quantity: Int,
    val pricePerUnit: BigDecimal,
) {
    val totalPrice: BigDecimal get() = pricePerUnit * quantity.toBigDecimal()
}

// ══════════════════════════════════════
//  AGGREGATE ROOT
// ══════════════════════════════════════

class Order private constructor(
    val id: OrderId,
    val customerId: UserId,
    private val _items: MutableList<OrderItem>,
    private var _status: OrderStatus,
) {
    val items: List<OrderItem> get() = _items.toList()
    val status: OrderStatus get() = _status
    val totalAmount: BigDecimal get() = _items.sumOf { it.totalPrice }

    fun addItem(item: OrderItem) {
        require(_status == OrderStatus.DRAFT) { "Cannot modify a ${_status} order" }
        _items.add(item)
    }

    fun submit(): Result<Unit> {
        if (_items.isEmpty()) return Result.failure(
            IllegalStateException("Cannot submit an empty order")
        )
        if (_status != OrderStatus.DRAFT) return Result.failure(
            IllegalStateException("Order is already ${_status}")
        )
        _status = OrderStatus.SUBMITTED
        return Result.success(Unit)
    }

    fun cancel(): Result<Unit> {
        if (_status == OrderStatus.DELIVERED) return Result.failure(
            IllegalStateException("Cannot cancel a delivered order")
        )
        _status = OrderStatus.CANCELLED
        return Result.success(Unit)
    }

    companion object {
        fun create(id: OrderId, customerId: UserId): Order =
            Order(id, customerId, mutableListOf(), OrderStatus.DRAFT)
    }
}

enum class OrderStatus { DRAFT, SUBMITTED, DELIVERED, CANCELLED }

// ══════════════════════════════════════
//  REPOSITORY INTERFACE
// ══════════════════════════════════════

interface OrderRepository {
    suspend fun findById(id: OrderId): Order?
    suspend fun save(order: Order)
    suspend fun findByCustomer(customerId: UserId): List<Order>
}

// ══════════════════════════════════════
//  DOMAIN SERVICE
// ══════════════════════════════════════

// Policy interface for calculating discounts
interface DiscountPolicy {
    fun calculateDiscount(order: Order): BigDecimal
}

// Example implementation
class VolumeDiscountPolicy : DiscountPolicy {
    override fun calculateDiscount(order: Order): BigDecimal {
        val itemCount = order.items.sumOf { it.quantity }
        val discountRate = when {
            itemCount >= 100 -> BigDecimal("0.15")  // 15% for 100+ items
            itemCount >= 50  -> BigDecimal("0.10")  // 10% for 50+ items
            itemCount >= 10  -> BigDecimal("0.05")  // 5% for 10+ items
            else             -> BigDecimal.ZERO
        }
        return order.totalAmount * discountRate
    }
}

class OrderPricingService(
    private val discountPolicy: DiscountPolicy,
) {
    fun calculateFinalPrice(order: Order): BigDecimal {
        val discount = discountPolicy.calculateDiscount(order)
        return order.totalAmount - discount
    }
}

// ══════════════════════════════════════
//  DOMAIN EVENT
// ══════════════════════════════════════

sealed interface OrderDomainEvent {
    val orderId: OrderId
    val occurredAt: Instant

    data class OrderSubmitted(
        override val orderId: OrderId,
        val customerId: UserId,
        val totalAmount: BigDecimal,
        override val occurredAt: Instant = Instant.now(),
    ) : OrderDomainEvent

    data class OrderCancelled(
        override val orderId: OrderId,
        val reason: String,
        override val occurredAt: Instant = Instant.now(),
    ) : OrderDomainEvent
}
```

### Pros

- ✅ **Business logic is explicit** — code mirrors the domain language
- ✅ **Rich domain model** — behavior lives with data, not in services
- ✅ **Bounded contexts** prevent model corruption across teams/modules
- ✅ Aggregate roots enforce consistency boundaries
- ✅ Scales well for complex business domains

### Cons

- ❌ **High complexity** — significant upfront investment in domain modeling
- ❌ Overkill for CRUD-heavy apps with little business logic
- ❌ Requires close collaboration with domain experts
- ❌ Steep learning curve for the full tactical pattern set
- ❌ Can lead to over-abstraction if misapplied

---

## 7. Clean Architecture + DDD Combined

This is the approach used in many production Android projects (including this project). Clean Architecture provides the **layering strategy**, while DDD provides the **domain modeling tactics**.

```
┌──────────────────────────────────────────────────────────┐
│  Presentation       ViewModels · Compose/XML · Mappers   │
├──────────────────────────────────────────────────────────┤
│  Application         Use Cases (orchestrate domain)       │
├──────────────────────────────────────────────────────────┤
│  Domain (DDD)        Entities · Value Objects · Aggregates│
│                      Repository Interfaces · Domain Svcs  │
├──────────────────────────────────────────────────────────┤
│  Infrastructure      Repo Impls · API · DB · Framework    │
└──────────────────────────────────────────────────────────┘
```

### How They Complement Each Other

| Concern                     | Clean Architecture           | DDD                            |
|-----------------------------|-----------------------------|--------------------------------|
| Layer structure             | ✅ Defines layers & rules    | —                              |
| Dependency direction        | ✅ Inward only               | —                              |
| Business logic modeling     | —                           | ✅ Entities, Aggregates, VOs    |
| Consistency boundaries      | —                           | ✅ Bounded Contexts, Aggregates |
| Use case orchestration      | ✅ Application layer          | —                              |
| Ubiquitous language         | —                           | ✅ Shared domain vocabulary     |
| Framework independence      | ✅ Domain has no framework deps | ✅ Pure domain model           |

### Module Structure Example

```
modules/
├── scoring/
│   └── src/main/kotlin/com/example/scoring/
│       ├── domain/               # DDD tactical patterns
│       │   ├── model/            #   Entities, Value Objects, Aggregates
│       │   ├── repository/       #   Repository interfaces
│       │   └── service/          #   Domain services
│       ├── application/          # Clean Architecture use cases
│       │   └── usecase/          #   Orchestrate domain logic
│       ├── infrastructure/       # Implementations
│       │   ├── repository/       #   Repository implementations
│       │   └── api/              #   External service adapters
│       └── presentation/         # UI layer
│           ├── viewmodel/
│           └── ui/
```

### Pros

- ✅ Best of both worlds — structural clarity + rich domain models
- ✅ Maximum testability at every layer
- ✅ Domain logic is portable and framework-independent
- ✅ Scales for large teams and complex domains

### Cons

- ❌ Most boilerplate of any approach
- ❌ Highest learning curve
- ❌ Requires discipline to maintain layer boundaries
- ❌ Not justified for simple apps

---

## 8. Comparison Matrix

| Criteria               | MVC   | MVP    | MVVM   | MVI    | Clean Arch | DDD    | Clean + DDD |
|------------------------|-------|--------|--------|--------|------------|--------|-------------|
| **Complexity**         | Low   | Medium | Medium | Medium-High | High  | High   | Very High   |
| **Testability**        | Poor  | Good   | Great  | Excellent | Excellent | Great | Excellent  |
| **Separation of Concerns** | Poor | Good | Great | Excellent | Excellent | Great | Excellent |
| **Boilerplate**        | None  | Medium | Low    | Medium | High       | Medium | Very High   |
| **Scalability**        | Poor  | Good   | Good   | Great  | Excellent  | Great  | Excellent   |
| **Learning Curve**     | Low   | Medium | Medium | Medium-High | High  | High   | Very High   |
| **Config Change Survival** | No | No   | Yes    | Yes    | Yes        | N/A    | Yes         |
| **State Predictability** | Poor | Fair | Good   | Excellent | Good    | Good   | Excellent   |
| **Android Ecosystem Fit** | Poor | Fair | Excellent | Excellent | Good | Fair  | Good        |
| **Suitable Team Size** | 1–2   | 2–5   | 2–10   | 2–10   | 5+         | 5+     | 5+          |
| **Compose Compatibility** | Poor | Poor | Excellent | Excellent | Good | Fair | Good       |

---

## 9. When to Use What

### 🟢 Use **MVC** when:
- Building a quick **prototype** or proof of concept
- The app has 1–3 simple screens with minimal logic
- You need something working in hours, not days

### 🔵 Use **MVP** when:
- You need **testable presenters** but aren't using Jetpack Compose
- Working on a legacy project that already uses MVP
- View logic is complex and needs explicit contract interfaces

### 🟣 Use **MVVM** when:
- Building a **modern Android app** (this should be your default)
- Using **Jetpack Compose** or **XML with data binding/ViewBinding**
- You need lifecycle-aware components that survive configuration changes
- Team is familiar with reactive programming (Flow/LiveData)

### 🟤 Use **MVI** when:
- The screen has **complex state interactions** (e.g., multi-step forms, real-time data)
- You need **predictable, reproducible state** for debugging
- Building with **Jetpack Compose** (MVI's declarative nature aligns perfectly)
- You want to enforce **strict Unidirectional Data Flow (UDF)**
- The team values **time-travel debugging** and state logging
- Multiple sources of events modify the same state (race condition risk)

### 🟠 Use **Clean Architecture** when:
- The app is **medium to large** with multiple feature modules
- You need to **swap implementations** (e.g., different data sources)
- Multiple teams work on different layers/features
- Long-term maintainability is a priority

### 🔴 Use **DDD** when:
- The domain has **complex business rules** (not simple CRUD)
- Domain experts are available for collaboration
- Business logic is the core competitive advantage
- You need explicit consistency boundaries (Aggregates)

### ⭐ Use **Clean Architecture + DDD** when:
- Building a **large-scale, long-lived** application
- The domain is complex with multiple bounded contexts
- Multiple teams need clear module boundaries
- You want maximum testability and framework independence
- Example: automotive dashboards, financial systems, healthcare apps

### Decision Flowchart

```
Is it a prototype or very simple app?
  ├─ YES → MVC or plain MVVM
  └─ NO
      │
      Does the screen have complex state / many user interactions?
        ├─ YES → MVI (or MVVM + strict UDF)
        └─ NO / MODERATE
            │
            Is the domain complex (beyond CRUD)?
              ├─ NO → MVVM + Repository Pattern
              └─ YES
                  │
                  Multiple modules / teams?
                    ├─ NO → MVVM/MVI + DDD tactical patterns
                    └─ YES → Clean Architecture + DDD ⭐
```

---

## Further Reading

- [Guide to app architecture — Android Developers](https://developer.android.com/topic/architecture)
- [UI Layer — Android Developers](https://developer.android.com/topic/architecture/ui-layer)
- [Domain Layer — Android Developers](https://developer.android.com/topic/architecture/domain-layer)
- [Data Layer — Android Developers](https://developer.android.com/topic/architecture/data-layer)
- [Clean Architecture — Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Domain-Driven Design — Eric Evans (Book)](https://www.domainlanguage.com/ddd/)
- [Kotlin Flow — Android Developers](https://developer.android.com/kotlin/flow)
- [ViewModel — Android Developers](https://developer.android.com/topic/libraries/architecture/viewmodel)
- [Hilt — Android Developers](https://developer.android.com/training/dependency-injection/hilt-android)
