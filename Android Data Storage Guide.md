# Android Data Storage Guide

Concise reference for local persistence: Room (structured/relational data) and DataStore (key-value / typed objects), including migrations, testing, and the SharedPreferences migration path.

---

## Table of Contents

1. [Choosing a Storage Solution](#1-choosing-a-storage-solution)
2. [Room: Setup & Core Components](#2-room-setup--core-components)
3. [Room: DAOs](#3-room-daos)
4. [Room: Migrations](#4-room-migrations)
5. [Room: Testing](#5-room-testing)
6. [DataStore: Preferences vs Typed](#6-datastore-preferences-vs-typed)
7. [DataStore: Read & Write](#7-datastore-read--write)
8. [DataStore: Compose Usage & Multi-Process](#8-datastore-compose-usage--multi-process)
9. [Migrating from SharedPreferences](#9-migrating-from-sharedpreferences)
10. [Best Practices Checklist](#10-best-practices-checklist)
11. [Further Reading](#11-further-reading)

---

## 1. Choosing a Storage Solution

| Need | Use |
|---|---|
| Large/queryable/relational data, partial updates, referential integrity | **Room** |
| Small key-value pairs (settings, flags, preferences) | **DataStore (Preferences)** |
| Typed custom objects, schema evolution, no partial updates | **DataStore (Proto/JSON)** |
| Legacy key-value store | `SharedPreferences` — **migrate to DataStore** |
| Large blobs (JSON dumps, bitmaps) | Plain **File** |
| Cross-app file sharing | `FileProvider` / Storage Access Framework |

Data layer placement of these classes → see `Android App Architecture Guide.md` (repositories are the only entry point; data sources wrap Room DAOs / DataStore instances).

---

## 2. Room: Setup & Core Components

```kotlin
// build.gradle.kts
dependencies {
    implementation("androidx.room:room-runtime:2.8.4")
    ksp("androidx.room:room-compiler:2.8.4")           // Kotlin: KSP
    implementation("androidx.room:room-ktx:2.8.4")     // coroutines/Flow support
    testImplementation("androidx.room:room-testing:2.8.4")
    implementation("androidx.room:room-paging:2.8.4")  // optional: Paging 3
}
```

Three components:

```kotlin
// 1. Entity — represents a table
@Entity
data class User(
    @PrimaryKey val uid: Int,
    @ColumnInfo(name = "first_name") val firstName: String?,
    @ColumnInfo(name = "last_name") val lastName: String?,
)

// 2. DAO — query surface
@Dao
interface UserDao {
    @Query("SELECT * FROM user") fun getAll(): List<User>
    @Insert fun insertAll(vararg users: User)
    @Delete fun delete(user: User)
}

// 3. Database — access point
@Database(entities = [User::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}

// Usage — one instance per process (singleton)
val db = Room.databaseBuilder(context, AppDatabase::class.java, "app-db").build()
```

- **Singleton per process** — `RoomDatabase` instances are expensive; don't create multiple.
- Multi-process apps: add `.enableMultiInstanceInvalidation()` so invalidation propagates across processes.
- Room validates every `@Query` **at compile time** — malformed SQL fails the build, not at runtime.

---

## 3. Room: DAOs

| Annotation | Purpose | Returns |
|---|---|---|
| `@Insert` | Insert entity/entities | `Unit`, or `Long`/`Long[]` row IDs |
| `@Update` | Update by primary key match | Optional `Int` (rows updated) |
| `@Delete` | Delete by primary key match | Optional `Int` (rows deleted) |
| `@Query` | Custom SQL (compile-time checked) | Entity, projection object, `Flow<T>`, `PagingSource<Int, T>`, `Cursor` (avoid) |

```kotlin
@Dao
interface UserDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    fun insertUsers(vararg users: User)

    @Query("SELECT * FROM user WHERE age > :minAge")
    fun loadAllUsersOlderThan(minAge: Int): List<User>

    @Query("SELECT * FROM user WHERE region IN (:regions)")
    fun loadUsersFromRegions(regions: List<String>): List<User> // collection param auto-expands

    // Return only the columns you need — maps onto a projection data class
    @Query("SELECT first_name, last_name FROM user")
    fun loadFullName(): List<NameTuple>

    // Reactive stream — recomposes/re-collects on every table change
    @Query("SELECT * FROM user")
    fun observeAll(): Flow<List<User>>

    // Paging 3 integration
    @Query("SELECT * FROM users WHERE label LIKE :query")
    fun pagingSource(query: String): PagingSource<Int, User>
}
```

- Prefer a **projection data class** over full entities when you only need a subset of columns (saves memory/parsing).
- **Multimap returns** (Room 2.4+): `Map<User, List<Book>>` directly from a `JOIN` query, avoiding a manual wrapper class; use `@MapInfo(keyColumn=..., valueColumn=...)` for column-only mappings.
- Avoid the raw `Cursor` return type — no guarantee about row existence or contents.

---

## 4. Room: Migrations

### Automated (schema changes Room can infer)

```kotlin
@Database(version = 2, entities = [User::class], autoMigrations = [AutoMigration(from = 1, to = 2)])
abstract class AppDatabase : RoomDatabase()
```

- Requires `exportSchema = true` (default) and a compiled schema JSON for **both** versions.
- Ambiguous changes (dropped/renamed table or column) need an `AutoMigrationSpec`:

```kotlin
@Database(version = 2, autoMigrations = [AutoMigration(from = 1, to = 2, spec = AppDatabase.Spec::class)])
abstract class AppDatabase : RoomDatabase() {
    @RenameTable(fromTableName = "User", toTableName = "AppUser")
    class Spec : AutoMigrationSpec
}
```

### Manual (complex changes — splitting tables, data transforms)

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE Book ADD COLUMN pub_year INTEGER")
    }
}
Room.databaseBuilder(context, AppDatabase::class.java, "db").addMigrations(MIGRATION_1_2).build()
```

- If both an `@AutoMigration` and a manual `Migration` exist for the same version pair, **the manual one wins**.
- **Never** call `.fallbackToDestructiveMigration()` casually — it **permanently deletes all user data** when no migration path is found. Prefer `fallbackToDestructiveMigrationFrom(specificVersions)` or `...OnDowngrade()` to scope the blast radius.

### Schema export (required for automated migrations + migration tests)

```kotlin
// build.gradle.kts (Room Gradle Plugin, 2.6.0+)
plugins { id("androidx.room") }
room { schemaDirectory("$projectDir/schemas") }
```
Commit the exported JSON schema files to version control.

---

## 5. Room: Testing

```kotlin
// build.gradle.kts
androidTestImplementation("androidx.room:room-testing:2.8.4")
```

**In-memory DB for DAO tests (on-device, not host JVM — SQLite versions can differ):**
```kotlin
@RunWith(AndroidJUnit4::class)
class UserDaoTest {
    private lateinit var db: TestDatabase
    private lateinit var userDao: UserDao

    @Before fun createDb() {
        db = Room.inMemoryDatabaseBuilder(ApplicationProvider.getApplicationContext(), TestDatabase::class.java).build()
        userDao = db.userDao()
    }
    @After fun closeDb() { db.close() }

    @Test fun writeAndReadUser() {
        userDao.insert(User(1, "George", "Bot"))
        assertThat(userDao.getAll()).contains(User(1, "George", "Bot"))
    }
}
```

**Migration tests** (verify each incremental step, plus a full-chain test):
```kotlin
@get:Rule
val helper = MigrationTestHelper(InstrumentationRegistry.getInstrumentation(), AppDatabase::class.java.canonicalName, FrameworkSQLiteOpenHelperFactory())

@Test
fun migrate1To2() {
    helper.createDatabase(TEST_DB, 1).apply { execSQL(/* seed v1 data */); close() }
    helper.runMigrationsAndValidate(TEST_DB, 2, true, MIGRATION_1_2) // validates schema; assert data separately
}
```

**Debugging:** Android Studio's **Database Inspector** (live query/edit while running) or `adb shell sqlite3 /data/data/<pkg>/databases/<db>` for `.dump`/`.schema`.

---

## 6. DataStore: Preferences vs Typed

| | Preferences DataStore | Proto / JSON DataStore |
|---|---|---|
| Schema | None (SharedPreferences-like keys) | Predefined (`.proto` file or `@Serializable` class) |
| Type safety | ❌ No | ✅ Yes |
| Best for | Simple flags/settings | Structured settings objects |

```kotlin
// build.gradle.kts
implementation("androidx.datastore:datastore-preferences:1.2.1")   // Preferences
implementation("androidx.datastore:datastore:1.2.1")                // Typed (Proto/JSON)
implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.9.0") // if using JSON
```

**Preferences:**
```kotlin
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "settings")
val EXAMPLE_COUNTER = intPreferencesKey("example_counter")
```

**JSON-serialized typed object:**
```kotlin
@Serializable data class Settings(val exampleCounter: Int)

object SettingsSerializer : Serializer<Settings> {
    override val defaultValue = Settings(exampleCounter = 0)
    override suspend fun readFrom(input: InputStream): Settings =
        try { Json.decodeFromString(input.readBytes().decodeToString()) }
        catch (e: SerializationException) { throw CorruptionException("Unable to read Settings", e) }
    override suspend fun writeTo(t: Settings, output: OutputStream) { output.write(Json.encodeToString(t).encodeToByteArray()) }
}

val Context.dataStore: DataStore<Settings> by dataStore(fileName = "settings.json", serializer = SettingsSerializer)
```

**Golden rules:**
- **Never create more than one `DataStore` instance for the same file in a process** — throws `IllegalStateException`.
- The generic type of `DataStore<T>` **must be immutable** (protocol buffers or immutable data classes).
- Don't mix `SingleProcessDataStore` and `MultiProcessDataStore` for the same file.

---

## 7. DataStore: Read & Write

```kotlin
// Read — exposed as a Flow
fun counterFlow(): Flow<Int> = context.dataStore.data.map { it[EXAMPLE_COUNTER] ?: 0 }       // Preferences
fun counterFlow(): Flow<Int> = context.dataStore.data.map { it.exampleCounter }               // Typed

// Write — atomic read-modify-write transaction
suspend fun incrementCounter() = context.dataStore.updateData {
    it.toMutablePreferences().apply { this[EXAMPLE_COUNTER] = (this[EXAMPLE_COUNTER] ?: 0) + 1 }
}
suspend fun incrementCounter() = context.dataStore.updateData { it.copy(exampleCounter = it.exampleCounter + 1) } // Typed
```

**Corruption handling** — replace a corrupted file with a default instead of crashing:
```kotlin
val dataStore = DataStoreFactory.create(
    serializer = SettingsSerializer,
    produceFile = { File("${context.filesDir.path}/settings.pb") },
    corruptionHandler = ReplaceFileCorruptionHandler { Settings(exampleCounter = 0) },
)
```

---

## 8. DataStore: Compose Usage & Multi-Process

**Keep DataStore in the data layer; expose via ViewModel + StateFlow — never read/write DataStore directly in a composable:**

```kotlin
class SettingsViewModel(private val repo: UserPreferencesRepository) : ViewModel() {
    val userSettings: StateFlow<UserSettings> = repo.userSettingsFlow
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), UserSettings.getDefaultInstance())

    fun updateCounter(newValue: Int) = viewModelScope.launch { repo.updateCounter(newValue) }
}

@Composable
fun SettingsScreen(viewModel: SettingsViewModel = viewModel()) {
    val settings by viewModel.userSettings.collectAsStateWithLifecycle()
    Button(onClick = { viewModel.updateCounter(settings.counter + 1) }) { Text("Increment") }
}
```

**Multi-process access** (e.g. a `Service` in `:my_process` writing, the app reading):
```kotlin
val dataStore = MultiProcessDataStoreFactory.create(
    serializer = TimeSerializer,
    produceFile = { File("${context.filesDir.path}/time.pb") },
)
```
```xml
<service android:name=".TimestampUpdateService" android:process=":my_process_id" />
```
Guarantees across processes: reads return only persisted data, read-after-write consistency, serialized writes, reads never blocked by writes. Scope the `MultiProcessDataStore` instance as a Hilt `@Singleton` per process.

---

## 9. Migrating from SharedPreferences

| SharedPreferences | DataStore equivalent |
|---|---|
| `getSharedPreferences(name, MODE_PRIVATE)` | `preferencesDataStore(name)` |
| `.edit().putInt(key, value).apply()` (async, but no completion signal, no error handling) | `updateData { it[intPreferencesKey(key)] = value }` (suspend, transactional, error-aware) |
| `.getInt(key, default)` (synchronous, can jank the main thread on large files) | `data.map { it[intPreferencesKey(key)] ?: default }` (async `Flow`, never blocks main thread) |
| No built-in migration tooling | `SharedPreferencesMigration` can seed a new DataStore from existing `SharedPreferences` on first read |

Reasons to migrate: DataStore is fully async (no main-thread reads), transactional (no partial-write corruption), and exposes changes reactively as `Flow` instead of requiring a manually-registered `OnSharedPreferenceChangeListener`.

---

## 10. Best Practices Checklist

- [ ] Room `RoomDatabase` is a singleton per process (or multi-instance-invalidation enabled)
- [ ] Every `@Query` returning a stream uses `Flow<T>`, one-shot calls use `suspend fun`
- [ ] Export Room schemas and commit them to version control for automated migrations + migration tests
- [ ] Test every migration path (individual + full chain) before shipping a schema change
- [ ] Avoid `fallbackToDestructiveMigration()` in production without a very deliberate, scoped reason
- [ ] Exactly one `DataStore` instance per file per process; immutable generic type
- [ ] Small key-value settings → DataStore Preferences; structured/typed settings → DataStore Proto/JSON
- [ ] Keep Room/DataStore access inside repositories — never call DAOs/DataStore directly from ViewModels or Composables
- [ ] Migrate legacy `SharedPreferences` usage to DataStore for main-thread safety and reactive updates

---

## 11. Further Reading

| Resource | Link |
|---|---|
| Save data using Room | https://developer.android.com/training/data-storage/room |
| Accessing data using Room DAOs | https://developer.android.com/training/data-storage/room/accessing-data |
| Migrate your Room database | https://developer.android.com/training/data-storage/room/migrating-db-versions |
| Test and debug your database | https://developer.android.com/training/data-storage/room/testing-db |
| DataStore | https://developer.android.com/topic/libraries/architecture/datastore |
| Data and file storage overview | https://developer.android.com/training/data-storage |

---

*Last Updated: July 2026 · Room 2.8.4, DataStore 1.2.1.*
