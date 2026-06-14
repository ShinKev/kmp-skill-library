# kmp-skill-library

> **19 Claude AI skills for Kotlin Multiplatform (KMP) and Compose Multiplatform (CMP) development.**  
> A curated library of expert-level system prompts covering the full Android and iOS development lifecycle — architecture, concurrency, networking, UI, performance, testing, migration, and Gradle build tooling.

[![Skills](https://img.shields.io/badge/skills-19-brightgreen)](./skills)
[![Platform](https://img.shields.io/badge/platform-KMP%20%7C%20CMP%20%7C%20Android%20%7C%20iOS-blue)](./skills)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

---

## What is a skill?

A **skill** is a focused, self-contained system prompt you load into an AI assistant (Claude, GPT-4, Gemini, Cursor, Continue, or any LLM) to turn it into a specialist for a specific domain. Instead of explaining your tech stack on every prompt, you load the relevant skill once and the AI already knows your architecture, your patterns, and your conventions.

Each skill in this library is a `SKILL.md` file containing:
- Expert-level best practices and decision tables
- Code patterns ready to copy into a KMP or CMP project
- Common pitfalls and how to avoid them
- Migration guides (RxJava→Coroutines, XML→Compose, Retrofit→Ktor, Hilt→Koin)

---

## Quick Start

### Claude (claude.ai)

1. Open a [Claude Project](https://claude.ai) and go to **Project Instructions**
2. Copy the contents of the skill(s) you need (e.g. `skills/kmp-architecture/SKILL.md`)
3. Paste into the Project Instructions field
4. Start prompting — Claude now knows your KMP/CMP stack

### Cursor / Continue / Copilot

Add the skill file as context in your IDE's AI chat, or reference it in your `.cursorrules` / `AGENTS.md` file at the repo root.

### Any LLM

Paste the `SKILL.md` content as a system prompt before your first message.

> **Tip:** Combine multiple skills in one project for full-stack coverage — e.g. `kmp-architecture` + `kmp-di-koin` + `compose-ui` gives you a complete KMP feature assistant.

---

## Skills

### 🏗️ Architecture

| Skill | Description |
|---|---|
| [`kmp-architecture`](skills/kmp-architecture/SKILL.md) | KMP Clean Architecture — inward-flowing layers (UI → Domain → Data → Platform), ViewModel in `commonMain`, Koin DI wiring, `:shared` / `:androidApp` / `:iosApp` module structure |
| [`kmp-data-layer`](skills/kmp-data-layer/SKILL.md) | Data layer — Repository as single source of truth, SQLDelight persistence, MultiplatformSettings K/V storage, Ktor remote fetching, offline-first sync (stale-while-revalidate, optimistic writes) |
| [`android-viewmodel`](skills/android-viewmodel/SKILL.md) | `StateFlow` / `SharedFlow` best practices in KMP ViewModels — `collectAsState()` vs `collectAsStateWithLifecycle()`, lifecycle awareness |

### ⚡ Concurrency & Networking

| Skill | Description |
|---|---|
| [`kotlin-concurrency-expert`](skills/kotlin-concurrency-expert/SKILL.md) | Coroutine review and remediation — ANR bugs, memory leaks, race conditions, dispatcher injection, exception handling, `callbackFlow`, scope guidelines |
| [`android-coroutines`](skills/android-coroutines/SKILL.md) | Android-specific coroutine patterns — `repeatOnLifecycle`, `viewModelScope`, cooperative cancellation, `runTest` |
| [`kmp-networking-ktor`](skills/kmp-networking-ktor/SKILL.md) | Ktor Client in `commonMain` — OkHttp (Android) and Darwin (iOS) engines, `ContentNegotiation` with `kotlinx.serialization`, Bearer auth with token refresh, Retrofit→Ktor migration table |

### 🔌 KMP Specifics

| Skill | Description |
|---|---|
| [`kmp-expect-actual`](skills/kmp-expect-actual/SKILL.md) | `expect`/`actual` vs interface + Koin DI — decision table for 11+ common platform features (DB drivers, file system, crypto, clipboard, etc.) |
| [`kmp-di-koin`](skills/kmp-di-koin/SKILL.md) | Full Koin setup for KMP — shared modules in `commonMain`, platform modules in `androidMain`/`iosMain`, `initKoin()` for Swift interop, `koinViewModel()` in composables, Hilt→Koin migration table |

### 🎨 UI

| Skill | Description |
|---|---|
| [`compose-ui`](skills/compose-ui/SKILL.md) | Compose Multiplatform UI best practices — state hoisting, modifier ordering, `remember` / `derivedStateOf`, lambda stability, `MaterialTheme` theming, preview annotations |
| [`cmp-navigation`](skills/cmp-navigation/SKILL.md) | Type-safe navigation — `@Serializable` route data classes, `NavHost` in `commonMain`, `NavigationSuiteScaffold`, `SavedStateHandle` for ViewModel args, deep links |
| [`coil-compose`](skills/coil-compose/SKILL.md) | Coil 3 image loading for KMP — `AsyncImage`, `rememberAsyncImagePainter`, singleton `ImageLoader` via Koin, lazy list performance rules |
| [`android-accessibility`](skills/android-accessibility/SKILL.md) | Accessibility audit for Compose — content descriptions, 48×48dp touch targets, WCAG AA contrast, focus order, semantic headings, state descriptions |

### 🚀 Performance

| Skill | Description |
|---|---|
| [`compose-performance-audit`](skills/compose-performance-audit/SKILL.md) | Recomposition audit — unstable keys, heavy composition work, Layout Inspector, Perfetto, Instruments profiling, type stability checklist |
| [`gradle-build-performance`](skills/gradle-build-performance/SKILL.md) | Gradle optimization — configuration cache, build cache, KSP migration from kapt, KMP hierarchy templates, CI remote cache setup |

### 🧪 Testing & Automation

| Skill | Description |
|---|---|
| [`kmp-testing`](skills/kmp-testing/SKILL.md) | Three-tier testing pyramid — `commonTest` (kotlin.test + Turbine), `androidTest` (Compose rules + Roborazzi screenshots), iOS (XCTest + swift-snapshot-testing) |
| [`android-emulator-skill`](skills/android-emulator-skill/SKILL.md) | Android automation suite — health checks, build & run scripts, semantic UI navigation by text/resource-id, gesture simulation, AVD management |

### 🔄 Migration

| Skill | Description |
|---|---|
| [`xml-to-compose-migration`](skills/xml-to-compose-migration/SKILL.md) | XML→Compose — layout/widget/attribute mapping tables, incremental migration via `ComposeView`/`AndroidView` interop, LiveData→StateFlow |
| [`rxjava-to-coroutines-migration`](skills/rxjava-to-coroutines-migration/SKILL.md) | RxJava→Coroutines — type mappings (`Single`→`suspend`, `Observable`→`Flow`), Subject→hot flow, operator equivalents, 6-step migration workflow |

### 🔧 Build & Tooling

| Skill | Description |
|---|---|
| [`android-gradle-logic`](skills/android-gradle-logic/SKILL.md) | Convention plugins — `build-logic` composite build, Version Catalog (`libs.versions.toml`), `AndroidApplicationConventionPlugin`, `KmpLibraryConventionPlugin`, `applyHierarchyTemplate` |

---

## Stack coverage

These skills cover the following libraries and tools:

**Language & Runtime:** Kotlin Multiplatform, Kotlin Coroutines, Kotlin Flow, kotlinx.serialization  
**UI:** Jetpack Compose, Compose Multiplatform, Coil 3, Navigation Compose  
**Architecture:** Clean Architecture, MVVM, Repository pattern, ViewModel  
**DI:** Koin, (migration from Hilt)  
**Networking:** Ktor Client, (migration from Retrofit / OkHttp)  
**Persistence:** SQLDelight, MultiplatformSettings  
**Testing:** kotlin.test, Turbine, Compose Testing, Roborazzi, XCTest, swift-snapshot-testing  
**Build:** Gradle, KSP, kapt, Version Catalog, Convention Plugins  
**Platform targets:** Android, iOS (via Kotlin/Native and Swift interop)

---

## Contributing

New skills, corrections, and improvements are welcome.

1. Fork the repo
2. Create a folder under `skills/your-skill-name/`
3. Add a `SKILL.md` following the structure of an existing skill
4. Open a pull request with a short description of what the skill covers

Please open an issue first for large additions or structural changes.

---

## Credits

Based on [awesome-android-agent-skills](https://github.com/new-silvermoon/awesome-android-agent-skills) by [@new-silvermoon](https://github.com/new-silvermoon), licensed under Apache 2.0.

The original Android skills (architecture, UI, concurrency, migration, testing, build tooling) have been adapted and extended for Kotlin Multiplatform (KMP) and Compose Multiplatform (CMP), with new skills added for `expect`/`actual`, Koin DI, Ktor networking, SQLDelight, and KMP-specific testing.

---

## License

[Apache 2.0](LICENSE) — free to use, modify, and distribute with attribution. See [NOTICE](NOTICE) for third-party credits.
