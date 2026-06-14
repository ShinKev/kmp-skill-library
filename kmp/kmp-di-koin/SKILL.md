---
name: kmp-di-koin
description: Expert guidance on setting up and using Koin for dependency injection in KMP projects. Covers commonMain module definitions, platform-specific modules, ViewModel injection, iOS initialization, and migration from Hilt to Koin.
---

# Koin for KMP Dependency Injection

## Setup

### libs.versions.toml
```toml
[versions]
koin = "4.0.0"

[libraries]
koin-core           = { group = "io.insert-koin", name = "koin-core",           version.ref = "koin" }
koin-android        = { group = "io.insert-koin", name = "koin-android",        version.ref = "koin" }
koin-compose        = { group = "io.insert-koin", name = "koin-compose",        version.ref = "koin" }
koin-compose-viewmodel = { group = "io.insert-koin", name = "koin-compose-viewmodel", version.ref = "koin" }
```

### shared/build.gradle.kts
```kotlin
kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation(libs.koin.core)
            implementation(libs.koin.compose)
            implementation(libs.koin.compose.viewmodel)
        }
        androidMain.dependencies {
            implementation(libs.koin.android)
        }
    }
}
```

---

## Module Structure

### commonMain — Shared Modules
```kotlin
// shared/src/commonMain/kotlin/di/SharedModules.kt
val networkModule = module {
    single { createHttpClient(get()) } // engine provided by platform module
}

val repositoryModule = module {
    single<NewsRepository> { OfflineFirstNewsRepository(get(), get()) }
    single<UserRepository> { UserRepositoryImpl(get(), get()) }
}

val viewModelModule = module {
    viewModel { NewsViewModel(get()) }
    viewModel { parameters -> ProfileViewModel(get(), parameters.get()) }
}

val sharedModules = listOf(networkModule, repositoryModule, viewModelModule)
```

### androidMain — Android-Specific Modules
```kotlin
// shared/src/androidMain/kotlin/di/AndroidModules.kt
val androidModule = module {
    single<HttpClientEngine> { OkHttp.create() }
    single<SqlDriver> { AndroidSqliteDriver(AppDatabase.Schema, androidContext(), "app.db") }
    single<Settings> { SharedPreferencesSettings(androidContext().getSharedPreferences("prefs", Context.MODE_PRIVATE)) }
}

val androidModules = listOf(androidModule)
```

### iosMain — iOS-Specific Modules
```kotlin
// shared/src/iosMain/kotlin/di/IosModules.kt
val iosModule = module {
    single<HttpClientEngine> { Darwin.create() }
    single<SqlDriver> { NativeSqliteDriver(AppDatabase.Schema, "app.db") }
    single<Settings> { NSUserDefaultsSettings(NSUserDefaults.standardUserDefaults) }
}

val iosModules = listOf(iosModule)
```

---

## Initializing Koin

### Android (Application.onCreate)
```kotlin
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidContext(this@MyApp)
            modules(sharedModules + androidModules)
        }
    }
}
```

### iOS (Swift @main)
```kotlin
// shared/src/iosMain/kotlin/KoinInit.kt — exposed to Swift
fun initKoin() {
    startKoin {
        modules(sharedModules + iosModules)
    }
}
```

```swift
// iosApp/iosApp/iOSApp.swift
@main
struct iOSApp: App {
    init() {
        // Swift mangles Kotlin function names that start with `init` by prepending `do`
        // (because `init` is a reserved Swift initializer keyword).
        // So Kotlin `initKoin()` in file KoinInit.kt becomes `KoinInitKt.doInitKoin()` here.
        KoinInitKt.doInitKoin()
    }
    var body: some Scene { WindowGroup { ContentView() } }
}
```

---

## Injecting in Composables (CMP)
```kotlin
// commonMain composable — koinViewModel() works on all platforms
@Composable
fun NewsScreen() {
    val viewModel: NewsViewModel = koinViewModel()
    val state by viewModel.uiState.collectAsState()
    // ...
}
```

## Injecting in ViewModel
```kotlin
// No annotation needed — Koin provides constructor injection automatically
class NewsViewModel(
    private val repository: NewsRepository
) : ViewModel() {
    private val _uiState = MutableStateFlow<NewsUiState>(NewsUiState.Loading)
    val uiState: StateFlow<NewsUiState> = _uiState.asStateFlow()
}
```

## ViewModel with Navigation Arguments
```kotlin
// Declare with parameters factory in the module
val viewModelModule = module {
    viewModel { parameters -> ProfileViewModel(get(), parameters.get<String>()) }
}

// Inject with argument at the call site
@Composable
fun ProfileScreen(userId: String) {
    val viewModel: ProfileViewModel = koinViewModel(parameters = { parametersOf(userId) })
}
```

---

## Testing with Koin
```kotlin
// commonTest
class NewsViewModelTest : KoinTest {

    @get:Rule
    val koinTestRule = KoinTestRule.create {
        modules(module {
            single<NewsRepository> { FakeNewsRepository() }
            viewModel { NewsViewModel(get()) }
        })
    }

    @Test
    fun `loading emits success state`() = runTest {
        val viewModel: NewsViewModel by inject()
        viewModel.load()

        viewModel.uiState.test {
            assertEquals(NewsUiState.Loading, awaitItem())
            assertEquals(NewsUiState.Success(fakeNews), awaitItem())
            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

---

## Hilt → Koin Migration Table

| Hilt | Koin Equivalent |
|------|----------------|
| `@HiltAndroidApp` on Application | `startKoin { androidContext(this) }` in `onCreate` |
| `@HiltViewModel` + `@Inject constructor` | `viewModel { MyViewModel(get()) }` in module; no annotation |
| `@AndroidEntryPoint` on Activity | No annotation; use `KoinAndroidActivity` or `inject()` delegate |
| `@Module @InstallIn(SingletonComponent)` | `module { single { ... } }` in `sharedModules`/`androidModules` |
| `@Binds abstract fun bind...` | `single<Interface> { Impl(get()) }` |
| `@Provides fun provide...` | `single { ... }` or `factory { ... }` |
| `hiltViewModel()` in Compose | `koinViewModel()` in Compose |
| `@HiltAndroidTest` | `KoinTestRule.create { modules(...) }` |
| `SavedStateHandle` injection | ViewModel constructor receives it via `viewModel { params -> Vm(get(), params.get()) }` |
