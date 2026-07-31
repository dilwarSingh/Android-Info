# Android Networking Guide

Concise reference for HTTP networking on Android using **Retrofit** + **OkHttp**: setup, API declarations, converters, interceptors, timeouts, caching, error handling, and testing.

> **Note:** Retrofit and OkHttp are Square-originated libraries; their current docs/source live under the `lysine-dev` GitHub org / `lysine.dev` (Maven coordinates are unchanged: `com.squareup.retrofit2:*`, `com.squareup.okhttp3:*`). Verify this yourself if it matters for your supply-chain policies — don't assume `square.github.io` links still resolve.

---

## Table of Contents

1. [Stack Overview](#1-stack-overview)
2. [Setup](#2-setup)
3. [Defining API Endpoints](#3-defining-api-endpoints)
4. [Converters](#4-converters)
5. [Repository Pattern](#5-repository-pattern)
6. [OkHttp Interceptors](#6-okhttp-interceptors)
7. [Timeouts, Retries & Dispatch](#7-timeouts-retries--dispatch)
8. [Caching](#8-caching)
9. [Error Handling](#9-error-handling)
10. [Testing with MockWebServer](#10-testing-with-mockwebserver)
11. [Security Considerations](#11-security-considerations)
12. [Best Practices Checklist](#12-best-practices-checklist)
13. [Further Reading](#13-further-reading)

---

## 1. Stack Overview

```
Your ApiService interface (Retrofit)
        ↓ generates implementation backed by
     OkHttpClient (connection pooling, interceptors, caching, TLS)
        ↓
     Network
```

| Component | Role |
|---|---|
| **Retrofit** | Turns an HTTP API into a type-safe Kotlin/Java interface |
| **OkHttp** | The actual HTTP client underneath — connections, interceptors, cache, TLS |
| **Converter** | (De)serializes request/response bodies (JSON ↔ Kotlin objects) |
| **Kotlin `suspend`** | Retrofit 3.x calls suspend functions directly — no separate call adapter needed |

Platform alternative: `Ktor` (JetBrains, coroutine-native, multiplatform) — a valid alternative to Retrofit for KMP projects.

---

## 2. Setup

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.squareup.retrofit2:retrofit:3.0.0")
    implementation("com.squareup.retrofit2:converter-kotlinx-serialization:3.0.0") // or converter-moshi / converter-gson

    implementation(platform("com.squareup.okhttp3:okhttp-bom:5.4.0"))
    implementation("com.squareup.okhttp3:okhttp")
    implementation("com.squareup.okhttp3:logging-interceptor")

    testImplementation("com.squareup.okhttp3:mockwebserver3:5.4.0")
}
```

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```
Both are **normal** (install-time) permissions — no runtime prompt needed. See `Android Permissions Guide.md`.

```kotlin
val okHttpClient = OkHttpClient.Builder().build()

val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .client(okHttpClient)
    .addConverterFactory(Json.asConverterFactory("application/json".toMediaType()))
    .build()

val apiService = retrofit.create(ApiService::class.java)
```

---

## 3. Defining API Endpoints

```kotlin
interface ApiService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): User          // body directly, throws HttpException on non-2xx

    @GET("users/{id}")
    suspend fun getUserResponse(@Path("id") id: String): Response<User> // wraps code/headers/body, no throw

    @GET("users")
    suspend fun listUsers(@Query("sort") sort: String, @QueryMap filters: Map<String, String>): List<User>

    @POST("users")
    suspend fun createUser(@Body user: User): User

    @FormUrlEncoded
    @POST("login")
    suspend fun login(@Field("username") user: String, @Field("password") pass: String): AuthToken

    @Multipart
    @PUT("users/{id}/photo")
    suspend fun uploadPhoto(@Path("id") id: String, @Part photo: MultipartBody.Part): User

    @Headers("Cache-Control: max-age=640000")
    @GET("config")
    suspend fun getConfig(): Config

    @GET("secure")
    suspend fun getSecure(@Header("Authorization") token: String): SecureData
}
```

| Annotation | Purpose |
|---|---|
| `@GET`/`@POST`/`@PUT`/`@PATCH`/`@DELETE`/`@HEAD`/`@OPTIONS`/`@HTTP` | Request method + relative URL |
| `@Path` | Replace a `{placeholder}` in the URL |
| `@Query` / `@QueryMap` | Query string parameter(s) |
| `@Body` | Serialize an object as the request body (via converter) |
| `@FormUrlEncoded` + `@Field` | `application/x-www-form-urlencoded` body |
| `@Multipart` + `@Part` | Multipart/file-upload body |
| `@Headers` (static) / `@Header` (dynamic) / `@HeaderMap` | Request headers — duplicates are **not** overwritten, all are sent |

**Kotlin support (built-in, no extra call-adapter dependency):**
- `suspend fun foo(): T` — returns the body directly; **throws `HttpException`** on non-2xx.
- `suspend fun foo(): Response<T>` — never throws for HTTP-level errors; inspect `.isSuccessful`/`.code()`/`.body()` yourself.

---

## 4. Converters

| Library | Artifact |
|---|---|
| Kotlin serialization | `com.squareup.retrofit2:converter-kotlinx-serialization` |
| Moshi | `com.squareup.retrofit2:converter-moshi` |
| Gson | `com.squareup.retrofit2:converter-gson` |
| Jackson | `com.squareup.retrofit2:converter-jackson` |
| Protobuf | `com.squareup.retrofit2:converter-protobuf` |
| Scalars (String/primitives) | `com.squareup.retrofit2:converter-scalars` |

```kotlin
// kotlinx.serialization example
@Serializable data class User(val id: String, val name: String)

val json = Json { ignoreUnknownKeys = true } // tolerate server fields you don't model
Retrofit.Builder()
    .baseUrl(BASE_URL)
    .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
    .build()
```

Without any converter, Retrofit can only accept/return OkHttp's raw `RequestBody`/`ResponseBody`.

---

## 5. Repository Pattern

Keep networking behind a repository — never call the Retrofit service directly from a ViewModel (see `Android App Architecture Guide.md`):

```kotlin
class UserRepository(private val api: ApiService) {
    suspend fun getUserById(id: String): User = api.getUser(id) // main-safe; suspend fns run off Dispatchers.Main by default in Retrofit
}

class MainViewModel(
    savedStateHandle: SavedStateHandle,
    private val repo: UserRepository,
) : ViewModel() {
    private val userId: String = savedStateHandle["uid"]!!
    private val _user = MutableStateFlow<User?>(null)
    val user: StateFlow<User?> = _user.asStateFlow()

    init {
        viewModelScope.launch {
            try { _user.value = repo.getUserById(userId) }
            catch (e: Exception) { /* surface error via UI state, see Architecture Guide §4 */ }
        }
    }
}
```

Retrofit suspend calls execute on an OkHttp dispatcher thread internally — safe to call from `viewModelScope.launch` without an explicit `withContext(Dispatchers.IO)`, but that's still fine to add for clarity/consistency with other repository methods.

---

## 6. OkHttp Interceptors

Two kinds — choose based on what you need to see/do:

| | Application interceptor (`addInterceptor`) | Network interceptor (`addNetworkInterceptor`) |
|---|---|---|
| Invoked | Exactly once per call, even for cached responses | Once per physical request (including redirects/retries) |
| Sees | The app's original request/response intent | The literal bytes on the wire (incl. OkHttp-added headers like `Accept-Encoding`) |
| Can short-circuit / retry | ✅ Yes (skip or repeat `chain.proceed()`) | Limited — tied to the real connection |
| Typical use | Auth header injection, logging, retry logic | Compression, low-level diagnostics |

```kotlin
class AuthInterceptor(private val tokenProvider: () -> String?) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request().newBuilder()
            .apply { tokenProvider()?.let { addHeader("Authorization", "Bearer $it") } }
            .build()
        return chain.proceed(request)
    }
}

val client = OkHttpClient.Builder()
    .addInterceptor(AuthInterceptor { tokenStore.currentToken })
    .addInterceptor(HttpLoggingInterceptor().apply { level = HttpLoggingInterceptor.Level.BODY }) // debug builds only!
    .build()
```

`chain.proceed(request)` is where the actual HTTP work happens — if you call it more than once (retry logic), close prior response bodies first.

---

## 7. Timeouts, Retries & Dispatch

```kotlin
val client = OkHttpClient.Builder()
    .connectTimeout(10, TimeUnit.SECONDS)
    .readTimeout(30, TimeUnit.SECONDS)
    .writeTimeout(30, TimeUnit.SECONDS)
    .retryOnConnectionFailure(true) // default; retries with an alternate route on stale/failed pooled connections
    .build()
```

- OkHttp automatically follows redirects and retries on a different route if a pooled connection was stale — no manual retry loop needed for that class of failure.
- `Call`s can be canceled from any thread (`call.cancel()`); in-flight body reads/writes then throw `IOException`.
- **`Dispatcher`** controls concurrency: default max **5 requests per host**, **64 total** — tune via `Dispatcher.maxRequestsPerHost`/`maxRequests` for high-throughput scenarios.

---

## 8. Caching

```kotlin
val client = OkHttpClient.Builder()
    .cache(Cache(directory = File(context.cacheDir, "http_cache"), maxSize = 50L * 1024 * 1024)) // 50 MiB
    .build()
```

- Disabled by default; RFC-correct, browser-like behavior once enabled.
- Default freshness heuristic (no explicit `Cache-Control`): **10% of the document's age** since `Last-Modified` — not applied to URLs with a query string.
- Only **one process** may own a given cache directory at a time.
- Responses must be **read fully** (not canceled/stalled) to be written to the cache.
- Clear/prune with `cache.evictAll()` (all) or iterate `cache.urls()` and `.remove()` (selective, e.g. pull-to-refresh).

---

## 9. Error Handling

```kotlin
sealed interface ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>
    data class Error(val code: Int?, val message: String) : ApiResult<Nothing>
}

suspend fun <T> safeApiCall(block: suspend () -> T): ApiResult<T> = try {
    ApiResult.Success(block())
} catch (e: HttpException) {           // non-2xx from a suspend fun returning the body directly
    ApiResult.Error(e.code(), e.message())
} catch (e: IOException) {             // no connectivity / timeout / cancelled
    ApiResult.Error(null, "Network error")
}
```

- `HttpException` — thrown by suspend functions that return the body directly, for any non-2xx response.
- `IOException` (and subclasses) — connectivity failures, timeouts.
- Prefer returning `Response<T>` when you need to branch on specific HTTP status codes without exceptions.
- Let the **UI layer** render the final error message — the repository/ViewModel should expose a typed result, not a raw exception (see `Android App Architecture Guide.md` §6).

---

## 10. Testing with MockWebServer

```kotlin
// build.gradle.kts
testImplementation("com.squareup.okhttp3:mockwebserver3:5.4.0")
```

```kotlin
class UserApiTest {
    private lateinit var server: MockWebServer
    private lateinit var api: ApiService

    @Before fun setup() {
        server = MockWebServer().apply { start() }
        api = Retrofit.Builder().baseUrl(server.url("/")).addConverterFactory(/* ... */).build()
            .create(ApiService::class.java)
    }

    @Test fun getUser_parsesResponse() = runTest {
        server.enqueue(MockResponse().setBody("""{"id":"1","name":"Ada"}""").setResponseCode(200))
        val user = api.getUser("1")
        assertEquals("Ada", user.name)
    }

    @After fun tearDown() = server.shutdown()
}
```

MockWebServer is intentionally minimal (not actively growing new features) — for advanced contract/stub testing needs, consider a dedicated tool like MockServer.

---

## 11. Security Considerations

- Send all traffic over **TLS/HTTPS** — never plaintext HTTP for anything sensitive.
- Use a **Network Security Configuration** to pin certificates or restrict trusted CAs.
- Minimize personal/sensitive data transmitted.
- Full coverage (cert pinning, Network Security Config XML, TLS best practices) → `Android Security Guide.md`.

---

## 12. Best Practices Checklist

- [ ] All network calls are `suspend fun`s behind a repository — never called directly from UI/ViewModel
- [ ] Use `Response<T>` when you need explicit status-code handling; plain body return + `HttpException` otherwise
- [ ] Auth/logging via **application** interceptors; low-level diagnostics via **network** interceptors
- [ ] `HttpLoggingInterceptor` body logging enabled in debug builds only — never in release
- [ ] Explicit connect/read/write timeouts set — don't rely purely on defaults for production apps
- [ ] Enable OkHttp response caching where appropriate; respect server `Cache-Control`
- [ ] Test API layers with `MockWebServer`, not live endpoints
- [ ] Route all traffic over TLS; add a Network Security Config for cert pinning if required
- [ ] Don't assume stale training/documentation about "square.github.io" URLs — verify current doc/maintainer locations before citing them

---

## 13. Further Reading

| Resource | Link |
|---|---|
| Retrofit docs | https://lysine.dev/retrofit/ |
| OkHttp docs | https://lysine.dev/okhttp/ |
| Connect to the network (Android) | https://developer.android.com/develop/connectivity/network-ops/connecting |
| Network Security Config | https://developer.android.com/privacy-and-security/security-config |

---

*Last Updated: July 2026 · Retrofit 3.0.0, OkHttp 5.4.0.*
