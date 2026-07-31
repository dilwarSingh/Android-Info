# Android Testing Guide

Concise reference for building a test strategy: the testing pyramid, test doubles, unit-testing coroutines/ViewModels, Compose UI tests, Espresso, screenshot tests, and Robolectric.

---

## Table of Contents

1. [The Testing Pyramid](#1-the-testing-pyramid)
2. [Test Doubles](#2-test-doubles)
3. [Unit Testing Coroutines & ViewModels](#3-unit-testing-coroutines--viewmodels)
4. [Testing Flows](#4-testing-flows)
5. [Compose UI Testing](#5-compose-ui-testing)
6. [Espresso (View-Based UI)](#6-espresso-view-based-ui)
7. [Screenshot Testing](#7-screenshot-testing)
8. [Robolectric](#8-robolectric)
9. [Hilt Testing](#9-hilt-testing)
10. [Dependencies](#10-dependencies)
11. [Best Practices Checklist](#11-best-practices-checklist)
12. [Further Reading](#12-further-reading)

---

## 1. The Testing Pyramid

| Layer | Scope | Runs On | Network | Example |
|---|---|---|---|---|
| **Unit** | Single method/class, no Android deps | Local JVM | No | Form-validator regex logic |
| **Component** | Module/component | Local JVM (Robolectric) or emulator | No | Screenshot test of a custom button |
| **Feature** | 2+ components integrated | Local/Robolectric/emulator | Mocked | JVM test with fakes for an auth flow |
| **Application** | Whole app binary (debug) | Emulator/device | Mocked/staging | UI behavior test across a config change |
| **Release Candidate** | Minified release binary | Emulator/device | Prod | Critical user journey, performance test |

- Prefer **many small, fast tests**; few large/slow ones (a pyramid, not top-heavy).
- Bug cost scales with test layer: a unit test catches it in minutes; an E2E test can take days and delay a release.
- Rule of thumb: **test at the lowest layer that gives the team correct confidence.**

---

## 2. Test Doubles

| Type | Definition | Preference |
|---|---|---|
| **Fake** | Working implementation, unsuitable for production (e.g. in-memory DB) | ✅ Preferred — lightweight, no mocking framework |
| **Stub** | Returns canned answers, no interaction verification | Use if a fake isn't practical |
| **Mock** | Verifies interactions occurred as expected | Use for interaction-based assertions |
| **Dummy** | Passed but never used (e.g. empty click lambda) | Fine as filler |
| **Spy** | Wraps a real object, records calls | Avoid — adds complexity; prefer fake/mock |
| **Shadow** | Robolectric's fake mechanism | Robolectric-specific |

```kotlin
// Fake repository — lives in the test source set only
object FakeUserRepository : UserRepository {
    override fun getUsers() = listOf(User("Alice"), User("Bob"))
}

@Test
fun viewModel_loadsUsers_showsFirstUser() {
    val viewModel = UserViewModel(FakeUserRepository)
    assertEquals("Alice", viewModel.firstUserName)
}
```

Use **dependency injection** (Hilt) so production dependencies can be swapped for fakes at test time — see [Android Dependency Injection Guide.md](Android%20Dependency%20Injection%20Guide.md).

---

## 3. Unit Testing Coroutines & ViewModels

```kotlin
// build.gradle.kts
testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.11.0")
```

- `runTest {}` — coroutine builder for tests; auto-skips `delay()`.
- `StandardTestDispatcher` (default) — new coroutines are **queued**; call `advanceUntilIdle()` / `runCurrent()` / `advanceTimeBy()` to let them run.
- `UnconfinedTestDispatcher` — new coroutines run **eagerly** on the current thread; simpler but doesn't mirror production concurrency.
- **All `TestDispatcher`s in one test must share the same scheduler.**

```kotlin
@Test
fun standardTest() = runTest {
    val repo = UserRepository()
    launch { repo.register("Alice") }
    advanceUntilIdle()                       // let queued coroutines run
    assertEquals(listOf("Alice"), repo.getAllUsers())
}
```

### Replacing `Dispatchers.Main` (needed for `viewModelScope`)

```kotlin
class MainDispatcherRule(
    val dispatcher: TestDispatcher = UnconfinedTestDispatcher(),
) : TestWatcher() {
    override fun starting(d: Description) = Dispatchers.setMain(dispatcher)
    override fun finished(d: Description) = Dispatchers.resetMain()
}

class HomeViewModelTest {
    @get:Rule val mainDispatcherRule = MainDispatcherRule()

    @Test
    fun loadMessage_updatesState() = runTest {
        val viewModel = HomeViewModel()
        viewModel.loadMessage()
        assertEquals("Greetings!", viewModel.message.value)
    }
}
```

- Inject dispatchers (`CoroutineDispatcher`/`CoroutineContext`) into classes instead of hardcoding `Dispatchers.IO` — makes them swappable for `TestDispatcher`s.
- For classes that launch their own coroutines, inject a `CoroutineScope` and pass `this` (the `TestScope`) in tests; use `TestScope.backgroundScope` for coroutines that shouldn't block test completion.

---

## 4. Testing Flows

```kotlin
// build.gradle.kts
testImplementation("app.cash.turbine:turbine:1.2.1")
```

```kotlin
@Test
fun uiState_emitsLoadingThenSuccess() = runTest {
    viewModel.uiState.test {
        assertEquals(UiState.Loading, awaitItem())
        viewModel.load()
        assertEquals(UiState.Success(data), awaitItem())
        cancelAndIgnoreRemainingEvents()
    }
}
```

Turbine's `test {}` block collects a `Flow`/`StateFlow` and lets you assert emissions in order — avoids manual `toList()` collection + cancellation boilerplate.

---

## 5. Compose UI Testing

```kotlin
// build.gradle.kts
androidTestImplementation("androidx.compose.ui:ui-test-junit4:1.11.4")
debugImplementation("androidx.compose.ui:ui-test-manifest:1.11.4")
```

**Finders → Assertions → Actions**, matched against the semantics tree (not the View tree):

```kotlin
composeTestRule.onNodeWithText("Submit").assertIsDisplayed().performClick()
composeTestRule.onNodeWithContentDescription("Avatar").assertExists()
composeTestRule.onAllNodesWithText("Item").assertCountEquals(4)
```

| Category | Examples |
|---|---|
| Finders | `onNode`, `onAllNodes`, `onNodeWithText`, `onNodeWithContentDescription`, `onNodeWithTag` |
| Assertions | `assertExists`, `assertIsDisplayed`, `assertTextEquals`, `assertCountEquals`, `assertAny`, `assertAll` |
| Actions | `performClick()`, `performTextInput()`, `performSemanticsAction()`, `performGesture { swipeLeft() }` (call once per `perform...`, no chaining) |

**Unmerged tree:** composables that merge child semantics (e.g. a button wrapping two `Text`s) hide individual children by default:
```kotlin
composeTestRule.onNodeWithText("World", useUnmergedTree = true).assertIsDisplayed()
```

**Synchronization:** Compose tests run on a **virtual clock** and auto-sync on every assertion/action.
```kotlin
composeTestRule.mainClock.autoAdvance = false          // manual control (e.g. mid-animation screenshot)
composeTestRule.mainClock.advanceTimeBy(500)
composeTestRule.waitUntil(timeoutMs = 5_000) { condition }
composeTestRule.waitUntilExactlyOneExists(hasText("Done"), timeoutMs = 5_000)
```
Register an `IdlingResource` for background work Compose isn't aware of (e.g. a raw thread pool).

---

## 6. Espresso (View-Based UI)

```kotlin
@get:Rule
val activityRule = ActivityScenarioRule(MyActivity::class.java)

@Test
fun submitButton_click_showsSuccess() {
    onView(withId(R.id.submit)).perform(click())
    onView(withId(R.id.status)).check(matches(withText("Success")))
}
```

| Rule | Purpose |
|---|---|
| `ActivityScenarioRule<A>` | Launches/tears down an Activity per test |
| `ServiceTestRule` | Starts/binds a `Service`, auto-stops after (no `IntentService` support) |
| `FragmentScenario` (`fragment-testing` artifact) | Tests a Fragment in isolation |

Still relevant even in Compose-first codebases for legacy screens and some interop cases.

---

## 7. Screenshot Testing

Compares a rendered UI against an approved **reference/golden** image. Recommended way to verify Compose visual attributes (colors, spacing, fonts) — faster to write/maintain than equivalent behavior tests.

| | Pros | Cons |
|---|---|---|
| Screenshot tests | Multiple visual assertions per test; easy to write; catches regressions across screen sizes | Reference-image storage (Git LFS or cloud); platform rendering differences (Mac vs Linux CI); slower than behavior tests |

**Practical rules:**
- Minimize combinations — don't cross every theme × font-size × screen-size; pick the ones that give unique signal.
- Take reference screenshots **on CI/server**, not developer laptops, to avoid platform rendering drift.
- Tolerate small pixel diffs via a configurable threshold, or use a structural/semantic differ.
- Official tool: [Compose Preview Screenshot Testing](https://developer.android.com/studio/preview/compose-screenshot-testing) (host-side, uses Layoutlib).

---

## 8. Robolectric

```kotlin
// build.gradle.kts
testImplementation("org.robolectric:robolectric:4.16.1")
android { testOptions { unitTests.isIncludeAndroidResources = true } }
```

Runs Android-framework-dependent code on the local JVM (no emulator) — supports API 21+.

```kotlin
@RunWith(AndroidJUnit4::class)
class AddContactActivityTest {
    @Test
    fun textSurvivesRecreation() {
        val scenario = ActivityScenario.launchActivity<AddContactActivity>()
        onView(withId(R.id.name)).perform(typeText("Test User"))
        scenario.recreate()
        onView(withId(R.id.name)).check(matches(withText("Test User")))
    }
}
```

- **Use as a last resort for unit tests** — a well-isolated architecture usually shouldn't need Android framework fakes ("shadows") at all.
- Good fit for **behavior/UI tests** (state + interaction logic) — fidelity is high enough and it's far faster/cheaper than a device.
- Lower fidelity for **screenshot tests** (rendering differs slightly from real devices) and unsupported for system-UI features (edge-to-edge, PiP) or `WebView` — use real devices/emulators for those.

---

## 9. Hilt Testing

```kotlin
@HiltAndroidTest
class UserRepositoryTest {
    @get:Rule val hiltRule = HiltAndroidRule(this)

    @Inject lateinit var repository: UserRepository

    @Before fun init() = hiltRule.inject()
}
```

- `@UninstallModules(ProdModule::class)` + a `@TestInstallIn`-annotated fake module swaps real dependencies for fakes in instrumented tests.
- Full scoping/multibinding/testing depth → see `Android Dependency Injection Guide.md`.

---

## 10. Dependencies

```kotlin
// build.gradle.kts
testImplementation("junit:junit:4.13.2")
testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.11.0")
testImplementation("app.cash.turbine:turbine:1.2.1")
testImplementation("org.robolectric:robolectric:4.16.1")

androidTestImplementation("androidx.test.ext:junit:1.3.0")
androidTestImplementation("androidx.test:runner:1.7.0")
androidTestImplementation("androidx.test.espresso:espresso-core:3.7.0")
androidTestImplementation("androidx.compose.ui:ui-test-junit4:1.11.4")
debugImplementation("androidx.compose.ui:ui-test-manifest:1.11.4")
```

---

## 11. Best Practices Checklist

- [ ] Shape your test suite like a pyramid — many unit/component tests, few E2E tests
- [ ] Prefer fakes over mocks/spies; keep them in the test source set
- [ ] Inject dispatchers/scopes so `TestDispatcher`s can replace them in tests
- [ ] Replace `Dispatchers.Main` via a `MainDispatcherRule` for ViewModel tests
- [ ] Use `runTest` + `advanceUntilIdle()`/Turbine instead of `Thread.sleep()`
- [ ] Match Compose nodes via semantics (`onNodeWithText`, `useUnmergedTree` when needed)
- [ ] Keep screenshot test combinations minimal; generate goldens on CI, not locally
- [ ] Reach for Robolectric only when a device-free JVM test needs Android framework classes
- [ ] Write (and keep updated) a short testing-strategy doc so the team applies layers consistently

---

## 12. Further Reading

| Resource | Link |
|---|---|
| Testing strategies | https://developer.android.com/training/testing/fundamentals/strategies |
| Test doubles | https://developer.android.com/training/testing/fundamentals/test-doubles |
| Testing coroutines | https://developer.android.com/kotlin/coroutines/test |
| Testing flows | https://developer.android.com/kotlin/flow/test |
| Compose testing APIs | https://developer.android.com/develop/ui/compose/testing/apis |
| Compose test synchronization | https://developer.android.com/develop/ui/compose/testing/synchronization |
| Screenshot testing | https://developer.android.com/training/testing/ui-tests/screenshot |
| Robolectric strategies | https://developer.android.com/training/testing/local-tests/robolectric |
| AndroidX Test JUnit rules | https://developer.android.com/training/testing/instrumented-tests/androidx-test-libraries/rules |
| Hilt testing | https://developer.android.com/training/dependency-injection/hilt-testing |

---

*Last Updated: July 2026 · Coroutines-test 1.11.0, Robolectric 4.16.1, Espresso 3.7.0, Compose UI Test 1.11.4.*
