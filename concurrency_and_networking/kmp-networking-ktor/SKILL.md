---
name: kmp-networking-ktor
description: Expert guidance on setting up and using Ktor Client for type-safe HTTP networking in KMP. Covers client configuration, platform engines, ContentNegotiation with kotlinx.serialization, authentication, error handling, and Koin integration.
---

# KMP Networking with Ktor Client

## Instructions

When implementing network layers in KMP, use **Ktor Client** in `commonMain`. Platform-specific engines are provided via Koin platform modules.

### 1. Setup (`libs.versions.toml`)

```toml
[versions]
ktor = "3.1.3"

[libraries]
ktor-client-core        = { group = "io.ktor", name = "ktor-client-core",              version.ref = "ktor" }
ktor-client-okhttp      = { group = "io.ktor", name = "ktor-client-okhttp",            version.ref = "ktor" }
ktor-client-darwin      = { group = "io.ktor", name = "ktor-client-darwin",            version.ref = "ktor" }
ktor-client-logging     = { group = "io.ktor", name = "ktor-client-logging",           version.ref = "ktor" }
ktor-serialization-json = { group = "io.ktor", name = "ktor-serialization-kotlinx-json", version.ref = "ktor" }
ktor-content-negotiation = { group = "io.ktor", name = "ktor-client-content-negotiation", version.ref = "ktor" }
```

`shared/build.gradle.kts` source sets:
```kotlin
commonMain.dependencies {
    implementation(libs.ktor.client.core)
    implementation(libs.ktor.client.logging)
    implementation(libs.ktor.serialization.json)
    implementation(libs.ktor.content.negotiation)
}
androidMain.dependencies {
    implementation(libs.ktor.client.okhttp)
}
iosMain.dependencies {
    implementation(libs.ktor.client.darwin)
}
```

### 2. Client Configuration (`commonMain`)

```kotlin
fun createHttpClient(engine: HttpClientEngine): HttpClient = HttpClient(engine) {
    install(ContentNegotiation) {
        json(Json {
            ignoreUnknownKeys = true
            coerceInputValues = true
        })
    }
    install(Logging) {
        logger = Logger.DEFAULT
        level = LogLevel.HEADERS // Use LogLevel.BODY during development only
    }
    install(HttpTimeout) {
        requestTimeoutMillis = 30_000
        connectTimeoutMillis = 10_000
        socketTimeoutMillis = 30_000
    }
    defaultRequest {
        url("https://api.example.com/")
        contentType(ContentType.Application.Json)
    }
}
```

### 3. Platform Engines via Koin

```kotlin
// androidMain
val androidNetworkModule = module {
    single<HttpClientEngine> {
        OkHttp.create {
            // OkHttp-specific config
        }
    }
}

// iosMain
val iosNetworkModule = module {
    single<HttpClientEngine> { Darwin.create() }
}

// commonMain
val networkModule = module {
    single { createHttpClient(get()) }
}
```

### 4. API Service Pattern

Plain classes — no annotations, no interface magic:

```kotlin
// commonMain
class UserApi(private val client: HttpClient) {

    suspend fun getUsers(): List<UserDto> =
        client.get("users").body()

    suspend fun getUser(id: String): UserDto =
        client.get("users/$id").body()

    suspend fun createUser(request: CreateUserRequest): UserDto =
        client.post("users") {
            setBody(request)
        }.body()

    suspend fun updateUser(id: String, request: UpdateUserRequest): UserDto =
        client.put("users/$id") {
            setBody(request)
        }.body()
}
```

### 5. URL & Query Building

```kotlin
// Path parameters — use string templates
client.get("group/$groupId/users")

// Query parameters
client.get("users") {
    parameter("sort", "name")
    parameter("limit", 20)
}

// Dynamic headers
client.get("protected/resource") {
    header("Authorization", "Bearer $token")
}
```

### 6. Authentication

Use the `Auth` plugin for Bearer tokens:

```kotlin
install(Auth) {
    bearer {
        loadTokens {
            BearerTokens(
                accessToken = authPreferences.authToken ?: "",
                refreshToken = authPreferences.refreshToken ?: ""
            )
        }
        refreshTokens {
            val refreshed = authApi.refresh(oldTokens?.refreshToken ?: "")
            authPreferences.authToken = refreshed.accessToken
            BearerTokens(refreshed.accessToken, refreshed.refreshToken)
        }
    }
}
```

### 7. Error Handling in Repositories

Always handle network exceptions at the Repository layer:

```kotlin
class UserRepository(private val api: UserApi) {

    suspend fun getUser(id: String): Result<User> = runCatching {
        api.getUser(id).toDomain()
    }.onFailure { cause ->
        when (cause) {
            is ResponseException -> { /* HTTP 4xx/5xx */ }
            is UnresolvedAddressException -> { /* No network */ }
            is SerializationException -> { /* Malformed response */ }
        }
    }
}
```

### 8. Retrofit → Ktor Migration Table

| Retrofit | Ktor Equivalent |
|----------|----------------|
| `@GET("path")` | `client.get("path")` |
| `@POST("path") @Body` | `client.post("path") { setBody(obj) }` |
| `@PUT("path") @Body` | `client.put("path") { setBody(obj) }` |
| `@DELETE("path")` | `client.delete("path")` |
| `@Path("id")` | String template `"resource/$id"` |
| `@Query("key")` | `parameter("key", value)` |
| `@Header("key")` | `header("key", value)` |
| `Response<T>` | `client.get(...).let { it.status to it.body<T>() }` |
| `HttpLoggingInterceptor` | `install(Logging)` plugin |
| `OkHttpClient` interceptor | `HttpSend` plugin or `Auth` plugin |
| Hilt `@Provides fun provideRetrofit` | Koin `single { createHttpClient(get()) }` |

### 9. Checklist

- [ ] `HttpClient` configured in `commonMain`; engine injected from platform Koin module
- [ ] `ContentNegotiation` installed with `kotlinx.serialization` JSON
- [ ] All DTOs annotated `@Serializable`
- [ ] `HttpTimeout` configured (don't rely on defaults)
- [ ] Network exceptions caught in Repository layer, not in ViewModel or UI
- [ ] `LogLevel.BODY` only in debug builds; use `LogLevel.HEADERS` in release
