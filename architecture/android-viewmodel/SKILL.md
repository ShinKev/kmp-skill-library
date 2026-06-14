---
name: android-viewmodel
description: Best practices for implementing ViewModels in KMP/CMP. ViewModel is now KMP-compatible via org.jetbrains.androidx.lifecycle. Covers StateFlow for UI state, SharedFlow for events, and platform differences in lifecycle-aware collection.
---

# ViewModel & State Management (KMP + Android)

> **KMP Note:** As of `lifecycle-viewmodel` 2.8+, `ViewModel` is KMP-compatible and lives in `commonMain` using `org.jetbrains.androidx.lifecycle:lifecycle-viewmodel`. No `@HiltViewModel` annotation — use Koin constructor injection instead. Use `koinViewModel()` in composables.

## Instructions

Use `ViewModel` to hold state and business logic. It must outlive configuration changes.

### 1. UI State (StateFlow)
*   **What**: Represents the persistent state of the UI (e.g., `Loading`, `Success(data)`, `Error`).
*   **Type**: `StateFlow<UiState>`.
*   **Initialization**: Must have an initial value.
*   **Exposure**: Expose as a read-only `StateFlow` backing a private `MutableStateFlow`.
    ```kotlin
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    ```
*   **Updates**: Update state using `.update { oldState -> ... }` for thread safety.

### 2. One-Off Events (SharedFlow)
*   **What**: Transient events like "Show Toast", "Navigate to Screen", "Show Snackbar".
*   **Type**: `SharedFlow<UiEvent>`.
*   **Configuration**: Must use `replay = 0` to prevent events from re-triggering on screen rotation.
    ```kotlin
    private val _uiEvent = MutableSharedFlow<UiEvent>(replay = 0)
    val uiEvent: SharedFlow<UiEvent> = _uiEvent.asSharedFlow()
    ```
*   **Sending**: Use `.emit(event)` (suspend) or `.tryEmit(event)`.

### 3. Collecting in UI

**In `commonMain` CMP composables** — use `collectAsState()`:
```kotlin
val state by viewModel.uiState.collectAsState()
```

**In Android-only composables (`androidMain` / `androidApp`)** — use `collectAsStateWithLifecycle()` for lifecycle-aware collection that pauses when the app is backgrounded:
```kotlin
val state by viewModel.uiState.collectAsStateWithLifecycle()
```
For `SharedFlow` on Android, use `LaunchedEffect` with `LocalLifecycleOwner`.

**In XML Views (Android-only)** — use `repeatOnLifecycle(Lifecycle.State.STARTED)`:
```kotlin
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { /* update UI */ }
    }
}
```

> `collectAsStateWithLifecycle()`, `lifecycleScope`, and `repeatOnLifecycle` are **Android-only APIs**. Never use them in `commonMain`.

### 4. Scope
*   Use `viewModelScope` for all coroutines started by the ViewModel. It is available in `commonMain` via the KMP-compatible ViewModel.
*   `lifecycleScope` is Android-only — never use it in `commonMain`.
*   Delegate specific operations to UseCases or Repositories; keep ViewModels thin.
