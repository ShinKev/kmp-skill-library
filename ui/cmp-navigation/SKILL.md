---
name: cmp-navigation
description: Implement type-safe navigation in Compose Multiplatform using the JetBrains Navigation Compose Multiplatform library. Use when setting up navigation, passing arguments, handling deep links, or structuring multi-screen KMP apps.
---

# CMP Navigation (JetBrains Navigation Compose Multiplatform)

## Overview

Implement type-safe navigation in Compose Multiplatform applications using the JetBrains fork of Navigation Compose. The API is nearly identical to AndroidX Navigation Compose, making migration straightforward. The `NavHost` and route definitions live in `commonMain`.

## Setup

```kotlin
// libs.versions.toml
[libraries]
navigation-compose = { group = "org.jetbrains.androidx.navigation", name = "navigation-compose", version = "2.8.0-alpha10" }
kotlinx-serialization-json = { group = "org.jetbrains.kotlinx", name = "kotlinx-serialization-json", version.ref = "kotlinxSerialization" }

[plugins]
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```

```kotlin
// shared/build.gradle.kts
plugins {
    alias(libs.plugins.kotlin.serialization)
}
commonMain.dependencies {
    implementation(libs.navigation.compose)
    implementation(libs.kotlinx.serialization.json)
}
```

---

## Core Concepts

### 1. Define Routes (Type-Safe, in `commonMain`)

```kotlin
import kotlinx.serialization.Serializable

@Serializable object Home
@Serializable data class Profile(val userId: String)
@Serializable data class Settings(val section: String? = null)
@Serializable object AuthGraph
@Serializable object Login
```

### 2. Create NavHost (in `commonMain`)

```kotlin
@Composable
fun AppNavHost(
    navController: NavHostController = rememberNavController(),
    modifier: Modifier = Modifier
) {
    NavHost(
        navController = navController,
        startDestination = Home,
        modifier = modifier
    ) {
        composable<Home> {
            HomeScreen(onNavigateToProfile = { userId ->
                navController.navigate(Profile(userId))
            })
        }

        composable<Profile> { backStackEntry ->
            val profile: Profile = backStackEntry.toRoute()
            ProfileScreen(userId = profile.userId)
        }

        composable<Settings> { backStackEntry ->
            val settings: Settings = backStackEntry.toRoute()
            SettingsScreen(section = settings.section)
        }

        navigation<AuthGraph>(startDestination = Login) {
            composable<Login> {
                LoginScreen(onLoginSuccess = {
                    navController.navigate(Home) {
                        popUpTo<AuthGraph> { inclusive = true }
                    }
                })
            }
        }
    }
}
```

---

## Navigation Patterns

### Basic Navigation

```kotlin
navController.navigate(Profile(userId = "user123"))
navController.popBackStack()
navController.popBackStack<Home>(inclusive = false)
```

### Navigate with Options

```kotlin
navController.navigate(Home) {
    popUpTo<Home> { inclusive = true; saveState = true }
    launchSingleTop = true
    restoreState = true
}
```

### Bottom Navigation (Adaptive)

```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()

    // NavigationSuiteScaffold automatically switches between bottom bar / rail / drawer
    NavigationSuiteScaffold(
        navigationSuiteItems = {
            val current = navController.currentBackStackEntryAsState().value?.destination
            item(
                icon = { Icon(Icons.Default.Home, null) },
                label = { Text("Home") },
                selected = current?.hasRoute<Home>() == true,
                onClick = {
                    navController.navigate(Home) {
                        popUpTo(navController.graph.findStartDestination().id) { saveState = true }
                        launchSingleTop = true
                        restoreState = true
                    }
                }
            )
        }
    ) {
        AppNavHost(navController = navController)
    }
}
```

---

## ViewModel with Navigation Arguments

**Default pattern: read route arguments from `SavedStateHandle`.** This survives process death, integrates naturally with deep links, and keeps the composable signature clean. The VM is reconstructed automatically with its original args after backgrounding.

```kotlin
// commonMain — no annotations
class ProfileViewModel(
    savedStateHandle: SavedStateHandle,
    private val userRepository: UserRepository
) : ViewModel() {

    private val profile: Profile = savedStateHandle.toRoute<Profile>()

    val user: StateFlow<User?> = userRepository
        .getUser(profile.userId)
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), null)
}
```

```kotlin
// Koin module — SavedStateHandle is provided automatically by koin-compose-viewmodel
viewModel { ProfileViewModel(get(), get()) }

// Composable — no params needed; userId comes from the route
composable<Profile> {
    val viewModel: ProfileViewModel = koinViewModel()
    ProfileScreen(viewModel)
}
```

### When to use `parametersOf` instead

For VMs that are **not** tied to a navigation destination (a dialog VM, a list-item VM, an embedded composable), pass primitives via Koin's parameters:

```kotlin
// VM has no SavedStateHandle dependency
class ItemRowViewModel(
    private val itemId: String,
    private val repo: ItemRepository
) : ViewModel()

// Koin
viewModel { params -> ItemRowViewModel(params.get(), get()) }

// Composable
@Composable
fun ItemRow(itemId: String) {
    val vm: ItemRowViewModel = koinViewModel(parameters = { parametersOf(itemId) })
}
```

**Do not mix the two patterns on the same VM** — pick one. Prefer `SavedStateHandle` for any VM scoped to a `NavBackStackEntry`.

---

## Deep Links

Deep link URI patterns are defined in `commonMain`. Manifest `<intent-filter>` goes in `androidApp`.

```kotlin
// commonMain
composable<Profile>(
    deepLinks = listOf(
        navDeepLink<Profile>(basePath = "https://example.com/profile")
    )
) { backStackEntry ->
    ProfileScreen(userId = backStackEntry.toRoute<Profile>().userId)
}
```

```xml
<!-- androidApp/src/main/AndroidManifest.xml — Android-specific -->
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" android:host="example.com" />
    </intent-filter>
</activity>
```

---

## Critical Rules

### DO
- Use `@Serializable` routes for type safety
- Pass only primitive IDs as navigation arguments, fetch full objects in ViewModel
- Define `NavHost` in `commonMain`
- Place manifest `<intent-filter>` in `androidApp` only

### DON'T
- Pass complex objects as navigation arguments
- Use `hiltViewModel()` — use `koinViewModel()` instead
- Use string-based routes (legacy pattern)
- Put Android-specific deep link setup in `commonMain`

---

## References

- [JetBrains Navigation Compose](https://www.jetbrains.com/help/kotlin-multiplatform-dev/compose-navigation-routing.html)
- [Type-Safe Navigation](https://developer.android.com/guide/navigation/design#compose)
