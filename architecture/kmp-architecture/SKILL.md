---
name: kmp-architecture
description: Expert guidance on setting up and maintaining a modern Kotlin Multiplatform (KMP) + Compose Multiplatform (CMP) application architecture. Use when asked about project structure, KMP module setup, Koin DI, or cross-platform clean architecture.
---

# KMP + CMP Modern Architecture

## Instructions

Structure the application with KMP-first Clean Architecture. All layers except platform adapters live in `commonMain`.

### 1. High-Level Layers

Dependencies flow strictly **inward** to the core logic.

- **UI Layer** (`commonMain` + platform overrides):
  - Composables, ViewModels.
  - `ViewModel` lives in `commonMain` via `org.jetbrains.androidx.lifecycle:lifecycle-viewmodel`.
  - Never depends on Data Layer implementation details directly.

- **Domain Layer** (`commonMain`, no platform imports):
  - Use Cases (e.g., `GetLatestNewsUseCase`), Domain Models (pure Kotlin data classes).
  - **Zero platform imports** — no `android.*`, no `platform.*`, no `kotlinx.coroutines.android`.
  - Depends only on Repository interfaces.

- **Data Layer** (`commonMain` + platform drivers):
  - Repository implementations, Ktor client calls, SQLDelight queries, MultiplatformSettings.
  - Platform-specific drivers (SQLite, HTTP engine) injected via Koin platform modules.

### 2. Dependency Injection with Koin

Use **Koin** for all DI. No Hilt in shared code.

```kotlin
// commonMain — shared modules
val repositoryModule = module {
    single<NewsRepository> { OfflineFirstNewsRepository(get(), get()) }
}

val viewModelModule = module {
    viewModel { NewsViewModel(get()) }
}

// androidMain — platform module
val androidPlatformModule = module {
    single<SqlDriver> { AndroidSqliteDriver(AppDatabase.Schema, androidContext(), "app.db") }
    single<HttpClientEngine> { OkHttp.create() }
}

// iosMain — platform module
val iosPlatformModule = module {
    single<SqlDriver> { NativeSqliteDriver(AppDatabase.Schema, "app.db") }
    single<HttpClientEngine> { Darwin.create() }
}
```

Android startup (`Application.onCreate`):
```kotlin
startKoin {
    androidContext(this@MyApp)
    modules(sharedModules + androidPlatformModules)
}
```

iOS startup (called from Swift `@main`):
```kotlin
fun initKoin() = startKoin {
    modules(sharedModules + iosPlatformModules)
}
```

See the `kmp-di-koin` skill for complete setup details.

### 3. Module Structure

```
:shared                         ← KMP module with commonMain/androidMain/iosMain
  ├── src/commonMain/kotlin/
  │   ├── domain/               ← Use Cases, interfaces, domain models
  │   ├── data/                 ← Repository implementations, Ktor, SQLDelight
  │   └── presentation/         ← ViewModels, UI state models
  ├── src/androidMain/kotlin/
  │   └── di/                   ← Android Koin platform modules
  └── src/iosMain/kotlin/
      └── di/                   ← iOS Koin platform modules

:androidApp                     ← Android entry point (Activity, Application, Manifest)
:iosApp                         ← Xcode project / iOS entry point

:feature:[name]                 ← Optional: standalone KMP feature modules
```

Core dependency rules:
- `:feature:*` depends on `:shared`
- `:androidApp` depends on `:shared`; applies Android-specific plugins
- `:iosApp` consumes the compiled Kotlin/Native framework from `:shared`

### 4. Checklist

- [ ] Domain layer has zero platform imports (`android.*`, `platform.*`, `UIKit.*`)
- [ ] `ViewModel` is in `commonMain` using `org.jetbrains.androidx.lifecycle:lifecycle-viewmodel`
- [ ] DI uses Koin; no `@HiltViewModel`, `@Inject`, or `@AndroidEntryPoint` in `commonMain`
- [ ] Repositories expose main-safe `suspend` functions and `Flow` streams
- [ ] Platform drivers (SQLite, HTTP engine) injected via `androidMain`/`iosMain` Koin modules
- [ ] `applyHierarchyTemplate { }` explicitly lists only your actual targets (see CLAUDE.md §9)
