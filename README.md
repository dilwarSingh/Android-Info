# Android Infos

A collection of in-depth Android development reference guides. Each topic has a full guide here in the root; some also have a condensed cheat-sheet under `Revisions/` (see `AGENTS.md` for conventions).

---

## Document Inventory

| # | Document | Description | Lines |
|---|---|---|---|
| 1 | `Accessibility Guide.md` | TalkBack, Switch Access, contrast, touch targets, Compose semantics, accessibility testing | 1100 |
| 2 | `Android Animations Guide.md` | View-system vs Compose animation APIs, decision trees, do's/don'ts, durations | 876 |
| 3 | `Android App Architecture Guide.md` | Google's official layered architecture: UI/Domain/Data layers, state holders, UDF, events | 331 |
| 4 | `Android Data Storage Guide.md` | Room (entities/DAOs/migrations/testing) + DataStore (Preferences/Proto), SharedPreferences migration | 356 |
| 5 | `Android Dependency Injection Guide.md` | Hilt deep-dive: components/scopes, qualifiers, assisted injection, multi-module, testing | 341 |
| 6 | `Android Design Patterns.md` | MVC/MVP/MVVM/MVI/Clean Architecture/DDD compared, with a decision flowchart | 1284 |
| 7 | `Android Navigation Guide.md` | Navigation Compose vs Navigation 3, back stack, deep links, adaptive nav, multi-backstack | 349 |
| 8 | `Android Networking Guide.md` | Retrofit + OkHttp: endpoints, converters, interceptors, caching, error handling, testing | 344 |
| 9 | `Android Permissions Guide.md` | Permission types, runtime request flow, per-permission specifics, version-by-version changes | 272 |
| 10 | `Android Rendering Pipelines.md` | How the View system vs Compose render pixels internally (measure/layout/draw, RenderThread) | 438 |
| 11 | `Android Security Guide.md` | Keystore, biometric auth, Network Security Config, secure IPC, Play Integrity | 253 |
| 12 | `Android Service Guide.md` | Background vs Foreground Service vs WorkManager decision guide | 975 |
| 13 | `Android Testing Guide.md` | Testing pyramid, test doubles, coroutines/Compose/Espresso testing, Robolectric | 313 |
| 14 | `Android WorkManager Guide.md` | Worker types & threading, unique work, testing, debugging, legacy-scheduler migration | 301 |
| 15 | `Battery & Power Optimization.md` | Doze, App Standby Buckets, resource limits, radio/network efficiency | 220 |
| 16 | `Jetpack Compose Fundamentals.md` | Composables, layouts, Modifiers, state & side-effects, Material 3 theming | 290 |
| 17 | `Multithreading in Android with Java.md` | Threads, Handlers/Loopers, synchronization, Executors — Java concurrency fundamentals | 3547 |
| 18 | `Periodic Location Tracking.md` | WorkManager vs Foreground Service vs Geofencing for location tracking | 674 |
| 19 | `UI Performance & Memory Leaks.md` | Jank, memory leaks, ANRs — causes, diagnosis tools, fixes | 1655 |

**Total: 13,919 lines across 19 guides.**

---

## Importance Tiers

| Tier | Meaning | Documents |
|---|---|---|
| 🔴 **Must Know** | Core concepts every Android developer needs, regardless of app type or experience level | Jetpack Compose Fundamentals, Android App Architecture Guide, Android Design Patterns, Android Permissions Guide, Android Networking Guide, Android Data Storage Guide, Multithreading in Android with Java, UI Performance & Memory Leaks, Android Testing Guide |
| 🟡 **Good to Know** | Common, high-value topics for building polished, production-quality apps | Android Navigation Guide, Android Service Guide, Android WorkManager Guide, Accessibility Guide, Android Security Guide, Android Animations Guide |
| 🔵 **Reference / Deep-Dive** | Advanced internals and mechanics you consult when you need them, not everyday knowledge | Android Rendering Pipelines, Android Dependency Injection Guide |
| 🟢 **Nice to Have** | Situational or specialized topics tied to specific features/use cases | Battery & Power Optimization, Periodic Location Tracking |
