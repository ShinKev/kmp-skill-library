---
name: kmp-data-layer
description: Guidance on implementing the KMP Data Layer using the Repository pattern, SQLDelight (relational), MultiplatformSettings (key-value), and Ktor Client (remote) with offline-first synchronization.
---

# KMP Data Layer

## Instructions

The Data Layer coordinates data from multiple sources. All of it lives in `commonMain`; platform drivers are injected via Koin platform modules.

### 1. Repository Pattern

Single Source of Truth (SSOT). The repository decides whether to return cached data or fetch fresh data.

```kotlin
// commonMain
class OfflineFirstNewsRepository(
    private val newsQueries: NewsQueries,   // SQLDelight
    private val newsApi: NewsApi            // Ktor
) : NewsRepository {

    override val newsStream: Flow<List<News>> =
        newsQueries.selectAll().asFlow().mapToList(Dispatchers.Default)

    override suspend fun refresh() {
        val remote = newsApi.fetchLatest()
        newsQueries.transaction {
            remote.forEach { newsQueries.insert(it.toEntity()) }
        }
    }
}
```

### 2. Relational Persistence: SQLDelight

Define `.sq` schema files in `commonMain/sqldelight/`:

```sql
-- commonMain/sqldelight/com/example/db/News.sq
CREATE TABLE News (
    id    TEXT NOT NULL PRIMARY KEY,
    title TEXT NOT NULL,
    body  TEXT NOT NULL
);

selectAll:
SELECT * FROM News;

insert:
INSERT OR REPLACE INTO News VALUES (?, ?, ?);
```

Provide the platform driver via Koin platform modules:

```kotlin
// androidMain
val androidDbModule = module {
    single<SqlDriver> { AndroidSqliteDriver(AppDatabase.Schema, androidContext(), "app.db") }
}

// iosMain
val iosDbModule = module {
    single<SqlDriver> { NativeSqliteDriver(AppDatabase.Schema, "app.db") }
}

// commonMain — uses injected SqlDriver
val databaseModule = module {
    single { AppDatabase(get()) }
    single { get<AppDatabase>().newsQueries }
}
```

### 3. Key-Value Storage: MultiplatformSettings

Use `com.russhwolf:multiplatform-settings` for tokens, user preferences, and feature flags. No `expect`/`actual` needed — the library wraps `SharedPreferences` (Android) and `NSUserDefaults` (iOS) internally.

```kotlin
// commonMain
class AuthPreferences(private val settings: Settings) {
    var authToken: String?
        get() = settings.getStringOrNull("auth_token")
        set(value) {
            if (value != null) settings.putString("auth_token", value)
            else settings.remove("auth_token")
        }

    var isOnboardingComplete: Boolean
        get() = settings.getBoolean("onboarding_complete", false)
        set(value) = settings.putBoolean("onboarding_complete", value)
}
```

Provide `Settings` via Koin platform modules:

```kotlin
// androidMain
val androidPrefsModule = module {
    single<Settings> {
        SharedPreferencesSettings(
            androidContext().getSharedPreferences("prefs", Context.MODE_PRIVATE)
        )
    }
}

// iosMain
val iosPrefsModule = module {
    single<Settings> { NSUserDefaultsSettings(NSUserDefaults.standardUserDefaults) }
}
```

### 4. Remote Data: Ktor Client

```kotlin
// commonMain
class NewsApi(private val client: HttpClient) {
    suspend fun fetchLatest(): List<NewsDto> = client.get("/api/news").body()
    suspend fun getById(id: String): NewsDto = client.get("/api/news/$id").body()
}
```

All DTOs must be `@Serializable`:
```kotlin
@Serializable
data class NewsDto(
    val id: String,
    val title: String,
    val body: String
)
```

See the `kmp-networking-ktor` skill for full client configuration.

### 5. Synchronization Strategies

- **Read (Stale-While-Revalidate):** Emit local SQLDelight `Flow` immediately, then trigger a background `refresh()` that writes back to the DB — the stream updates automatically.
- **Write (Optimistic):** Save locally first, emit updated state, sync to server in background. On failure, rollback and emit error state.

### 6. DI Wiring

```kotlin
// commonMain
val dataModule = module {
    single { AppDatabase(get()) }
    single { get<AppDatabase>().newsQueries }
    single { NewsApi(get()) }
    single { AuthPreferences(get()) }
    single<NewsRepository> { OfflineFirstNewsRepository(get(), get()) }
}
```

### 7. Checklist

- [ ] No `android.*` or platform imports in `commonMain` repository classes
- [ ] SQLDelight `.sq` files in `commonMain/sqldelight/`
- [ ] `SqlDriver` provided via Koin platform modules (not hardcoded)
- [ ] `Settings` instance provided via Koin platform modules
- [ ] All DTOs annotated with `@Serializable`
- [ ] Repository interfaces in Domain layer; implementations in Data layer
- [ ] API DTOs mapped to Domain models to keep layers decoupled
