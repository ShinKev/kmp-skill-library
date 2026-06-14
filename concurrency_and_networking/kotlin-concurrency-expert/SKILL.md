---
name: kotlin-concurrency-expert
description: Kotlin Coroutines review and remediation for KMP and Android codebases. Use when asked to review concurrency usage, fix coroutine-related bugs, improve thread safety, or resolve scoping issues in any Kotlin code (commonMain, androidMain, iosMain).
---

# Kotlin Concurrency Expert

## Overview

Review and fix Kotlin Coroutines issues in KMP and Android codebases by applying structured concurrency, lifecycle safety, proper scoping, and modern best practices with minimal behavior changes. Most rules apply uniformly to `commonMain`, `androidMain`, and `iosMain`; a small set of APIs (`lifecycleScope`, `repeatOnLifecycle`) is Android-only.

## Workflow

### 1. Triage the Issue

- Capture the exact error, crash, or symptom (ANR, memory leak, race condition, incorrect state).
- Identify the source set: `commonMain`, `androidMain`, or `iosMain`. This determines which scopes/APIs are available.
- Check project coroutines setup: `kotlinx-coroutines-core` (KMP) version; on Android also `kotlinx-coroutines-android` and `lifecycle-runtime-ktx`.
- Identify the current scope context (`viewModelScope` in KMP ViewModels, `lifecycleScope` on Android only, custom injected `CoroutineScope`, or none).
- Confirm whether the code is UI-bound (`Dispatchers.Main`) or intended to run off the main thread (`Dispatchers.IO`, `Dispatchers.Default`). Note: `Dispatchers.IO` is available in `commonMain` since `kotlinx-coroutines-core` 1.7.
- Verify Dispatcher injection patterns for testability.

### 2. Apply the Smallest Safe Fix

Prefer edits that preserve existing behavior while satisfying structured concurrency and lifecycle safety.

Common fixes:

- **ANR / Main thread blocking**: Move heavy work to `withContext(Dispatchers.IO)` or `Dispatchers.Default`; ensure suspend functions are main-safe.
- **Memory leaks / zombie coroutines**: Replace `GlobalScope` with a lifecycle-bound scope (`viewModelScope` in any source set; `lifecycleScope` Android-only; injected `applicationScope` in `commonMain`).
- **Lifecycle collection issues (Android-only)**: Replace deprecated `launchWhenStarted` with `repeatOnLifecycle(Lifecycle.State.STARTED)`. In CMP `commonMain` composables, use `collectAsState()` inside the composable and `LaunchedEffect` for side effects — not `lifecycleScope`.
- **State exposure**: Encapsulate `MutableStateFlow` / `MutableSharedFlow`; expose read-only `StateFlow` or `Flow`.
- **CancellationException swallowing**: Ensure generic `catch (e: Exception)` blocks rethrow `CancellationException`.
- **Non-cooperative cancellation**: Add `ensureActive()` or `yield()` in tight loops for cooperative cancellation.
- **Callback APIs**: Convert listeners to `callbackFlow` with proper `awaitClose` cleanup.
- **Hardcoded Dispatchers**: Inject `CoroutineDispatcher` via constructor for testability.

## Critical Rules

### Dispatcher Injection (Testability)

```kotlin
// CORRECT: Inject dispatcher
class UserRepository(
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) {
    suspend fun fetchUser() = withContext(ioDispatcher) { ... }
}

// INCORRECT: Hardcoded dispatcher
class UserRepository {
    suspend fun fetchUser() = withContext(Dispatchers.IO) { ... }
}
```

### Lifecycle-Aware Collection

**Android (`androidMain` / `androidApp` only):**
```kotlin
// CORRECT: Use repeatOnLifecycle
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state -> updateUI(state) }
    }
}

// INCORRECT: Direct collection (unsafe, deprecated)
lifecycleScope.launchWhenStarted {
    viewModel.uiState.collect { state -> updateUI(state) }
}
```

**CMP `commonMain` composables:**
```kotlin
// CORRECT: collectAsState() inside the composable
@Composable
fun NewsScreen(viewModel: NewsViewModel = koinViewModel()) {
    val state by viewModel.uiState.collectAsState()
    // For side effects keyed to a value, use LaunchedEffect
    LaunchedEffect(state.snackbarMessage) {
        state.snackbarMessage?.let { showSnackbar(it) }
    }
}

// INCORRECT: lifecycleScope / repeatOnLifecycle do not exist in commonMain
// Never reference Android lifecycle APIs from shared code
```

### State Encapsulation

```kotlin
// CORRECT: Expose read-only StateFlow
class MyViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(UiState())
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
}

// INCORRECT: Exposed mutable state
class MyViewModel : ViewModel() {
    val uiState = MutableStateFlow(UiState()) // Leaks mutability
}
```

### Exception Handling

```kotlin
// CORRECT: Rethrow CancellationException
try {
    doSuspendWork()
} catch (e: CancellationException) {
    throw e // Must rethrow!
} catch (e: Exception) {
    handleError(e)
}

// INCORRECT: Swallows cancellation
try {
    doSuspendWork()
} catch (e: Exception) {
    handleError(e) // CancellationException swallowed!
}
```

### Cooperative Cancellation

```kotlin
// CORRECT: Check for cancellation in tight loops
suspend fun processLargeList(items: List<Item>) {
    items.forEach { item ->
        ensureActive() // Check cancellation
        processItem(item)
    }
}

// INCORRECT: Non-cooperative (ignores cancellation)
suspend fun processLargeList(items: List<Item>) {
    items.forEach { item ->
        processItem(item) // Never checks cancellation
    }
}
```

### Callback Conversion

```kotlin
// CORRECT: callbackFlow with awaitClose
fun locationUpdates(): Flow<Location> = callbackFlow {
    val listener = LocationListener { location ->
        trySend(location)
    }
    locationManager.requestLocationUpdates(listener)
    
    awaitClose { locationManager.removeUpdates(listener) }
}
```

## Scope Guidelines

| Scope | Use When | Where |
|-------|----------|-------|
| `viewModelScope` | ViewModel operations | `commonMain` (KMP-compatible) |
| `CoroutineScope` (injected) | Repositories, services with a defined owner | `commonMain` |
| `lifecycleScope` | UI operations in Activity/Fragment | `androidMain` only |
| `repeatOnLifecycle` | Flow collection tied to Android lifecycle | `androidMain` only |
| `applicationScope` (injected) | App-wide background work | `commonMain` or `androidMain` |
| `GlobalScope` | **NEVER USE** | Breaks structured concurrency |

> `lifecycleScope` and `repeatOnLifecycle` are Android-only. In `commonMain` composables, collect flows with `collectAsState()` and start coroutines inside `LaunchedEffect`.

## Testing Pattern

```kotlin
@Test
fun `loading data updates state`() = runTest {
    val testDispatcher = StandardTestDispatcher(testScheduler)
    val repository = FakeRepository()
    val viewModel = MyViewModel(repository, testDispatcher)
    
    viewModel.loadData()
    advanceUntilIdle()
    
    assertEquals(UiState.Success(data), viewModel.uiState.value)
}
```

## Reference Material

- [Kotlin Coroutines Guide](https://kotlinlang.org/docs/coroutines-guide.html) (KMP)
- [Asynchronous Flow](https://kotlinlang.org/docs/flow.html) (KMP)
- [Kotlin Coroutines Best Practices](https://developer.android.com/kotlin/coroutines/coroutines-best-practices) (Android)
- [StateFlow and SharedFlow](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow) (Android-flavored, concepts apply to KMP)
- [repeatOnLifecycle API](https://developer.android.com/topic/libraries/architecture/coroutines#repeatOnLifecycle) (Android-only)
