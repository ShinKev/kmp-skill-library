---
name: kmp-expect-actual
description: Expert guidance on when and how to use Kotlin Multiplatform expect/actual declarations vs interface + Koin DI. Use when implementing platform-specific functionality, deciding where platform code belongs, or resolving compilation errors in KMP source sets.
---

# KMP expect/actual vs Interface + Koin DI

## The Two Patterns

### Pattern 1: expect/actual (thin value wrappers)
Use when the platform implementation is a thin wrapper over a platform API with minimal logic — nothing worth unit-testing or mocking independently.

```kotlin
// commonMain
expect fun createDatabaseDriver(name: String): SqlDriver

// androidMain
actual fun createDatabaseDriver(name: String): SqlDriver =
    AndroidSqliteDriver(AppDatabase.Schema, context, name)

// iosMain
actual fun createDatabaseDriver(name: String): SqlDriver =
    NativeSqliteDriver(AppDatabase.Schema, name)
```

### Pattern 2: interface + Koin DI (non-trivial implementations)
Use when the platform code has meaningful logic, needs to be mocked in `commonTest`, or when multiple implementations may exist.

```kotlin
// commonMain
interface PlatformCrypto {
    fun encrypt(data: ByteArray, key: ByteArray): ByteArray
    fun decrypt(data: ByteArray, key: ByteArray): ByteArray
}

// commonMain Koin module (expects binding from platform)
val cryptoModule = module {
    single<PlatformCrypto> { get() }
}

// androidMain Koin module
val androidCryptoModule = module {
    single<PlatformCrypto> { AndroidCryptoImpl() }
}

// iosMain Koin module
val iosCryptoModule = module {
    single<PlatformCrypto> { IosCryptoImpl() }
}
```

## Decision Table

| Feature | Pattern | Reason |
|---------|---------|--------|
| SQLite driver | `expect`/`actual` factory function | Thin, no behavior to test |
| HTTP engine | Koin platform module | Engine config is non-trivial |
| Date/Time | No platform code — use `kotlinx-datetime` | Full KMP support |
| Key-value prefs | No platform code — use MultiplatformSettings | Full KMP support |
| Cryptography | Interface + Koin DI | Non-trivial, needs mocking |
| File system | No platform code — use `okio` | Full KMP support for most paths |
| Push notification token | Interface + Koin DI | Lifecycle-tied, complex |
| Biometrics | Interface + Koin DI | Complex, platform-divergent |
| Clipboard | `expect`/`actual` | Thin wrapper |
| Logging | No platform code — use Kermit | Full KMP support |
| Permissions | Interface + Koin DI or Moko-Permissions | Async, complex |

## Rules

1. **`expect`/`actual` for**: functions, objects, and type aliases that are thin — minimal logic, not independently unit-tested.
2. **Interface + Koin for**: anything with a constructor that takes dependencies, anything that needs to be faked in `commonTest`, or anything with conditional logic.
3. **Never put `expect`/`actual` in domain models** — domain models must be pure Kotlin with zero platform specificity.
4. An `expect class` with an `actual typealias` to a platform type is valid for thin type bridges (e.g., `actual typealias AtomicInt = java.util.concurrent.atomic.AtomicInteger`).
5. Prefer `expect`/`actual` **functions** (factories) over `expect`/`actual` **classes** when possible — functions are simpler to `actual`-implement.

## Common Mistakes

```kotlin
// WRONG: Android Context in an expect function — Context is Android-only
expect fun createRepository(context: Context): UserRepository

// CORRECT: Inject context via Koin in androidMain; commonMain has no Context dependency
val androidModule = module {
    single { androidContext() } // Koin provides Android Context
    single<UserRepository> { UserRepositoryImpl(get(), get()) }
}
```

```kotlin
// WRONG: expect class with 10 methods — that's an interface
expect class UserStorage {
    fun save(user: User)
    fun load(id: String): User?
    fun delete(id: String)
    // ...7 more methods
}

// CORRECT: interface + Koin DI
interface UserStorage {
    fun save(user: User)
    fun load(id: String): User?
    fun delete(id: String)
}
```

```kotlin
// WRONG: Forgetting actual in iosMain causes compilation failure for the iOS target
// always add both androidMain AND iosMain actuals

// CORRECT
// commonMain
expect val platformName: String

// androidMain
actual val platformName: String = "Android"

// iosMain
actual val platformName: String = "iOS"
```
