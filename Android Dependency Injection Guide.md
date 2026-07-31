# Android Dependency Injection Guide

Concise reference for **Hilt** (Google's recommended DI library, built on Dagger): setup, bindings, qualifiers, component scopes, assisted injection, entry points, multi-module apps, and testing.

> For a basic Hilt vs. Koin comparison and a first working example, see `Android Design Patterns.md` (Clean Architecture section). This guide goes deeper into Hilt's mechanics.

---

## Table of Contents

1. [Setup](#1-setup)
2. [Core Concepts](#2-core-concepts)
3. [Providing Bindings: @Binds vs @Provides](#3-providing-bindings-binds-vs-provides)
4. [Qualifiers](#4-qualifiers)
5. [Components, Scopes & Lifetimes](#5-components-scopes--lifetimes)
6. [Assisted Injection](#6-assisted-injection)
7. [Entry Points (Unsupported Classes)](#7-entry-points-unsupported-classes)
8. [Multi-Module / Feature Modules](#8-multi-module--feature-modules)
9. [Jetpack Integrations](#9-jetpack-integrations)
10. [Testing with Hilt](#10-testing-with-hilt)
11. [Best Practices Checklist](#11-best-practices-checklist)
12. [Further Reading](#12-further-reading)

---

## 1. Setup

```kotlin
// Project-level build.gradle.kts
plugins { id("com.google.dagger.hilt.android") version "2.60.1" apply false }

// app/build.gradle.kts
plugins {
    id("com.google.devtools.ksp")
    id("com.google.dagger.hilt.android")
}
dependencies {
    implementation("com.google.dagger:hilt-android:2.60.1")
    ksp("com.google.dagger:hilt-android-compiler:2.60.1") // Kotlin DSL + KSP
}
android {
    compileOptions { // Required by Hilt + Compose
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
}
```

```kotlin
@HiltAndroidApp
class MyApplication : Application()   // required — bootstraps the app-level container
```

---

## 2. Core Concepts

| Annotation | Applies to | Purpose |
|---|---|---|
| `@HiltAndroidApp` | `Application` | Triggers Hilt code gen; creates the app-level container |
| `@AndroidEntryPoint` | `Activity`, `Fragment`, `View`, `Service`, `BroadcastReceiver` | Generates a Hilt component for that class; enables field injection |
| `@Inject` (constructor) | Any class you own | Tells Hilt how to construct it + its dependencies |
| `@Inject` (field) | Inside `@AndroidEntryPoint` classes | Requests a dependency (field **cannot** be `private`) |

```kotlin
class AnalyticsAdapter @Inject constructor(private val service: AnalyticsService) { /* ... */ }

@AndroidEntryPoint
class ExampleActivity : ComponentActivity() {
    @Inject lateinit var analytics: AnalyticsAdapter // Hilt populates this
}
```

**Compose:** only the root `ComponentActivity` needs `@AndroidEntryPoint` — it's the single DI entry point for the whole UI tree; use `hiltViewModel()` inside composables (see §9).

Hilt supports: `Application`, `ViewModel`, `Activity`, `Fragment`, `View`, `Service`, `BroadcastReceiver` (injected directly from `SingletonComponent`, no dedicated component).

---

## 3. Providing Bindings: @Binds vs @Provides

Use a **Hilt module** (`@Module` + `@InstallIn(SomeComponent::class)`) when a type **can't** be constructor-injected — interfaces, or classes you don't own (Retrofit, OkHttpClient, Room).

| | `@Binds` | `@Provides` |
|---|---|---|
| Use for | Interface → implementation | Classes from external libraries, builder-pattern construction |
| Function body | None — abstract function | Contains construction logic |
| Module type | `abstract class` | `object` (or class with instance methods) |

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class AnalyticsModule {
    @Binds
    abstract fun bindAnalyticsService(impl: AnalyticsServiceImpl): AnalyticsService
}

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient = OkHttpClient.Builder().build()
}
```

---

## 4. Qualifiers

Disambiguate **multiple bindings of the same type**:

```kotlin
@Qualifier @Retention(AnnotationRetention.BINARY) annotation class AuthInterceptorOkHttpClient
@Qualifier @Retention(AnnotationRetention.BINARY) annotation class OtherInterceptorOkHttpClient

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @AuthInterceptorOkHttpClient @Provides
    fun provideAuthClient(interceptor: AuthInterceptor): OkHttpClient =
        OkHttpClient.Builder().addInterceptor(interceptor).build()

    @OtherInterceptorOkHttpClient @Provides
    fun provideOtherClient(interceptor: OtherInterceptor): OkHttpClient =
        OkHttpClient.Builder().addInterceptor(interceptor).build()
}

class Service @Inject constructor(@AuthInterceptorOkHttpClient private val client: OkHttpClient)
```

**Predefined qualifiers:** `@ApplicationContext` / `@ActivityContext` — resolve `Context` unambiguously. `Application`/`Activity` are also injectable directly, unqualified.

> Best practice: if you qualify one binding of a type, qualify **all** bindings of that type — an unqualified "default" is error-prone.

---

## 5. Components, Scopes & Lifetimes

```
SingletonComponent (Application)
├── ActivityRetainedComponent (survives config changes)
│   ├── ViewModelComponent (ViewModel)
│   └── ActivityComponent (Activity)
│       └── ViewComponent / FragmentComponent
└── ServiceComponent (Service)
```

| Android class | Component | Scope annotation | Created at | Destroyed at |
|---|---|---|---|---|
| `Application` | `SingletonComponent` | `@Singleton` | `Application#onCreate()` | App destroyed |
| `Activity` (retained) | `ActivityRetainedComponent` | `@ActivityRetainedScoped` | First `onCreate()` | Last `onDestroy()` (survives rotation) |
| `ViewModel` | `ViewModelComponent` | `@ViewModelScoped` | ViewModel created | ViewModel cleared |
| `Activity` | `ActivityComponent` | `@ActivityScoped` | `onCreate()` | `onDestroy()` |
| `Service` | `ServiceComponent` | `@ServiceScoped` | `onCreate()` | `onDestroy()` |

- **Unscoped (default):** a new instance every injection.
- **Scoped:** one instance per component instance — install the module in the **matching** component (a binding's scope must match where it's installed).
- A module installed in a component is visible to that component **and all its children** (e.g. `SingletonComponent` bindings are available everywhere).
- Minimize scoping — a scoped object stays in memory for its component's entire lifetime. Reserve it for stateful/expensive/synchronization-requiring bindings.

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class AnalyticsModule {
    @Singleton @Binds
    abstract fun bindAnalyticsService(impl: AnalyticsServiceImpl): AnalyticsService
}
```

---

## 6. Assisted Injection

For dependencies needing a **runtime argument** alongside Hilt-managed ones — e.g. a `ViewModel` that needs a nav-arg `userId` plus a repository.

```kotlin
@HiltViewModel(assistedFactory = MyViewModel.Factory::class)
class MyViewModel @AssistedInject constructor(
    @Assisted val userId: String,
    private val repository: MyRepository,
) : ViewModel() {
    @AssistedFactory interface Factory { fun create(userId: String): MyViewModel }
}
```

Same pattern powers **`@HiltWorker`** (WorkManager — see §9): `@Assisted` marks `Context`/`WorkerParameters`, everything else is regular Hilt injection.

---

## 7. Entry Points (Unsupported Classes)

For classes Hilt doesn't support directly (e.g. `ContentProvider`), define an `@EntryPoint` boundary:

```kotlin
class ExampleContentProvider : ContentProvider() {
    @EntryPoint
    @InstallIn(SingletonComponent::class)
    interface Entry { fun analyticsService(): AnalyticsService }

    override fun query(...): Cursor {
        val entry = EntryPointAccessors.fromApplication(context!!.applicationContext, Entry::class.java)
        val service = entry.analyticsService()
        // ...
    }
}
```

Match the `EntryPointAccessors.from*()` call (`fromApplication`/`fromActivity`) to the component the `@EntryPoint` is `@InstallIn`.

---

## 8. Multi-Module / Feature Modules

Regular Gradle modules work with Hilt as-is — **feature modules** (dynamic-delivery, inverted dependency direction) cannot be processed by Hilt directly and need a Dagger bridge:

1. Declare an `@EntryPoint` in the `app` module exposing what the feature needs.
2. Create a plain **Dagger** `@Component(dependencies = [EntryPointInterface::class])` inside the feature module.
3. Use constructor injection as normal inside the feature module; build the Dagger component manually at the feature's entry Activity/Fragment.

```kotlin
// app module
@EntryPoint
@InstallIn(SingletonComponent::class)
interface LoginModuleDependencies { @AuthInterceptorOkHttpClient fun okHttpClient(): OkHttpClient }

// login feature module
@Component(dependencies = [LoginModuleDependencies::class])
interface LoginComponent {
    fun inject(activity: LoginActivity)
    @Component.Builder interface Builder {
        fun context(@BindsInstance context: Context): Builder
        fun appDependencies(deps: LoginModuleDependencies): Builder
        fun build(): LoginComponent
    }
}
```

For deep multi-module graphs, consider `enableExperimentalClasspathAggregation` in Gradle.

---

## 9. Jetpack Integrations

| Library | Pattern |
|---|---|
| **ViewModel** | `@HiltViewModel` + `@Inject constructor` → `by viewModels()` (Views) or `hiltViewModel()` (Compose) |
| **Navigation Compose** | `hiltViewModel()` auto-scopes to the current back-stack entry |
| **Navigation 3** | Add `rememberViewModelStoreNavEntryDecorator()` to `NavDisplay.entryDecorators`, then call `hiltViewModel()` inside the entry provider |
| **WorkManager** | `@HiltWorker` + `@AssistedInject` (see §6); `Application` implements `Configuration.Provider`, injects `HiltWorkerFactory` |

```kotlin
@HiltWorker
class SyncWorker @AssistedInject constructor(
    @Assisted appContext: Context,
    @Assisted params: WorkerParameters,
    private val repo: SyncRepository,
) : CoroutineWorker(appContext, params) { /* ... */ }

@HiltAndroidApp
class MyApplication : Application(), Configuration.Provider {
    @Inject lateinit var workerFactory: HiltWorkerFactory
    override val workManagerConfiguration
        get() = Configuration.Builder().setWorkerFactory(workerFactory).build()
}
```
Remove WorkManager's default `androidx.startup` initializer from the manifest when using a custom `Configuration.Provider` — see `Android WorkManager Guide.md`.

---

## 10. Testing with Hilt

```kotlin
// build.gradle.kts
testImplementation("com.google.dagger:hilt-android-testing:2.60.1")
kspTest("com.google.dagger:hilt-android-compiler:2.60.1")
androidTestImplementation("com.google.dagger:hilt-android-testing:2.60.1")
kspAndroidTest("com.google.dagger:hilt-android-compiler:2.60.1")
```

- **Unit tests don't need Hilt at all** — constructor-injected classes can be instantiated directly with fakes: `AnalyticsAdapter(fakeService)`.
- **Instrumented/UI tests:**

```kotlin
@HiltAndroidTest
class SettingsActivityTest {
    @get:Rule val hiltRule = HiltAndroidRule(this)
    @Inject lateinit var analyticsAdapter: AnalyticsAdapter

    @Before fun init() = hiltRule.inject()
}
```

- Requires a custom `AndroidJUnitRunner` that swaps in `HiltTestApplication`:
```kotlin
class CustomTestRunner : AndroidJUnitRunner() {
    override fun newApplication(cl: ClassLoader?, name: String?, ctx: Context?) =
        super.newApplication(cl, HiltTestApplication::class.java.name, ctx)
}
```

| Replace a binding | Scope | How |
|---|---|---|
| **All tests** | Project-wide | `@Module @TestInstallIn(components=[...], replaces=[ProdModule::class])` in `test`/`androidTest` |
| **Single test class** | Local | `@UninstallModules(ProdModule::class)` on the test + a nested `@Module` inside it |
| **Single field, quick** | Local | `@BindValue @JvmField val service: AnalyticsService = FakeAnalyticsService()` (supports qualifiers; `@BindValueIntoSet`/`@BindValueIntoMap` for multibindings) |

> `@UninstallModules` generates new components per test — faster iteration comes from preferring `@TestInstallIn` when the fake should apply everywhere.

Robolectric: set `application = dagger.hilt.android.testing.HiltTestApplication` in `robolectric.properties`, or use `@Config(application = HiltTestApplication::class)` per test.

---

## 11. Best Practices Checklist

- [ ] Prefer constructor injection (`@Inject constructor`) over field injection everywhere possible
- [ ] Use `@Binds` for interfaces, `@Provides` only for types you don't own / need builder logic
- [ ] Qualify **all** bindings of a type once any one of them needs a qualifier
- [ ] Scope sparingly — only for stateful, expensive, or synchronization-sensitive bindings
- [ ] Install modules in the **lowest** component that needs them, not always `SingletonComponent`
- [ ] Use `@AssistedInject`/`@Assisted` for ViewModels/Workers that need runtime args
- [ ] Bridge feature (dynamic-delivery) modules with `@EntryPoint` + a plain Dagger `@Component`
- [ ] In tests, prefer `@TestInstallIn`/`@BindValue` over ad-hoc `@UninstallModules` where the fake applies broadly
- [ ] Never instantiate Hilt-managed classes manually with `new`/constructor calls in production code — let Hilt provide them

---

## 12. Further Reading

| Resource | Link |
|---|---|
| Hilt and Android DI | https://developer.android.com/training/dependency-injection/hilt-android |
| Hilt and Jetpack integrations | https://developer.android.com/training/dependency-injection/hilt-jetpack |
| Hilt in multi-module projects | https://developer.android.com/training/dependency-injection/hilt-multi-module |
| Hilt testing | https://developer.android.com/training/dependency-injection/hilt-testing |
| Hilt & Dagger annotations cheat sheet | https://developer.android.com/training/dependency-injection/hilt-cheatsheet |
| Dagger multibindings (dagger.dev) | https://dagger.dev/dev-guide/multibindings |

---

*Last Updated: July 2026 · Hilt 2.60.1.*
