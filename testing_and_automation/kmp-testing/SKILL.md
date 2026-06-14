---
name: kmp-testing
description: Comprehensive KMP testing strategy. Covers commonTest with kotlin.test + Turbine for shared logic, androidTest with Compose test rules and Roborazzi screenshot tests, and iOS snapshot testing guidance.
---

# KMP Testing Strategies

This skill provides expert guidance on testing KMP + CMP applications across three tiers.

## Testing Pyramid

```
          ┌─────────────────────┐
          │  UI / Screenshot    │  androidTest + iosApp Xcode tests
          ├─────────────────────┤
          │  Integration Tests  │  commonTest (SQLDelight queries, Ktor MockEngine)
          ├─────────────────────┤
          │    Unit Tests       │  commonTest (ViewModels, UseCases, Repos)
          └─────────────────────┘
```

## Where Tests Live

| Source Set | What Goes Here |
|------------|---------------|
| `commonTest` | ViewModel, UseCase, Repository, Flow tests — the majority of your tests |
| `androidTest` (or `androidUnitTest`) | Compose UI tests, Koin integration tests, Roborazzi screenshot tests |
| `iosTest` | Kotlin tests for iOS `actual` implementations (runs on simulator) |
| `iosApp/iosAppTests` (Xcode) | Swift/XCTest tests for the iOS app shell and Swift-side code |

---

## Tier 1: commonTest (kotlin.test + Turbine)

### Dependencies (`libs.versions.toml`)

```toml
[libraries]
kotlin-test              = { module = "org.jetbrains.kotlin:kotlin-test" }
kotlinx-coroutines-test  = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-test", version.ref = "kotlinxCoroutines" }
turbine                  = { group = "app.cash.turbine", name = "turbine", version.ref = "turbine" }
```

```kotlin
// shared/build.gradle.kts
commonTest.dependencies {
    implementation(libs.kotlin.test)
    implementation(libs.kotlinx.coroutines.test)
    implementation(libs.turbine)
}
```

### ViewModel Unit Test

```kotlin
// commonTest
class NewsViewModelTest {

    private val fakeRepository = FakeNewsRepository()
    private val viewModel = NewsViewModel(fakeRepository)

    @Test
    fun `initial state is Loading`() {
        assertEquals(NewsUiState.Loading, viewModel.uiState.value)
    }

    @Test
    fun `loading news emits Success`() = runTest {
        viewModel.uiState.test {
            assertEquals(NewsUiState.Loading, awaitItem())
            viewModel.load()
            assertEquals(NewsUiState.Success(fakeNews), awaitItem())
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `network error emits Error state`() = runTest {
        fakeRepository.shouldFail = true

        viewModel.uiState.test {
            skipItems(1) // Loading
            viewModel.load()
            val error = awaitItem()
            assertIs<NewsUiState.Error>(error)
            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

### Repository Flow Test

```kotlin
// commonTest
class NewsRepositoryTest {

    private val fakeDb = FakeNewsDatabase()
    private val fakeApi = FakeNewsApi()
    private val repository = OfflineFirstNewsRepository(fakeDb, fakeApi)

    @Test
    fun `newsStream emits from local source`() = runTest {
        fakeDb.emit(listOf(testNews))

        repository.newsStream.test {
            assertEquals(listOf(testNews), awaitItem())
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `refresh writes API data to DB`() = runTest {
        fakeApi.news = listOf(remoteNews)
        repository.refresh()
        assertEquals(listOf(remoteNews.toDomain()), fakeDb.stored)
    }
}
```

### Testing Dispatcher Injection

```kotlin
// commonTest
class SearchViewModelTest {

    private val testDispatcher = StandardTestDispatcher()

    @Test
    fun `search debounces input`() = runTest(testDispatcher) {
        val viewModel = SearchViewModel(FakeSearchRepo(), testDispatcher)

        viewModel.onSearchQuery("ko")
        advanceTimeBy(200) // Less than debounce threshold
        assertEquals(0, viewModel.results.value.size)

        advanceTimeBy(400) // Past debounce threshold
        advanceUntilIdle()
        assertTrue(viewModel.results.value.isNotEmpty())
    }
}
```

---

## Tier 2: androidTest (Compose + Roborazzi)

### Dependencies

```toml
[libraries]
junit4                   = { module = "junit:junit", version = "4.13.2" }
kotlinx-coroutines-test  = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-test", version.ref = "kotlinxCoroutines" }
androidx-test-ext-junit  = { group = "androidx.test.ext", name = "junit", version = "1.1.5" }
compose-ui-test          = { group = "androidx.compose.ui", name = "ui-test-junit4" }
roborazzi                = { group = "io.github.takahirom.roborazzi", name = "roborazzi", version.ref = "roborazzi" }
```

### Compose UI Test

```kotlin
@RunWith(AndroidJUnit4::class)
class NewsScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun newsScreen_showsLoadingIndicator() {
        composeTestRule.setContent {
            NewsScreen(uiState = NewsUiState.Loading, onRefresh = {})
        }
        composeTestRule.onNodeWithTag("loading_indicator").assertIsDisplayed()
    }

    @Test
    fun newsScreen_showsNewsList_whenSuccess() {
        composeTestRule.setContent {
            NewsScreen(uiState = NewsUiState.Success(fakeNews), onRefresh = {})
        }
        composeTestRule.onNodeWithText(fakeNews.first().title).assertIsDisplayed()
    }
}
```

### Screenshot Testing with Roborazzi

```kotlin
@RunWith(AndroidJUnit4::class)
@GraphicsMode(GraphicsMode.Mode.NATIVE)
@Config(sdk = [33], qualifiers = RobolectricDeviceQualifiers.Pixel5)
class NewsScreenScreenshotTest {

    @get:Rule
    val composeTestRule = createAndroidComposeRule<ComponentActivity>()

    @Test
    fun capture_loading_state() {
        composeTestRule.setContent {
            AppTheme { NewsScreen(uiState = NewsUiState.Loading, onRefresh = {}) }
        }
        composeTestRule.onRoot().captureRoboImage()
    }
}
```

Run commands:
```bash
./gradlew recordRoborazziDebug   # Record golden images
./gradlew verifyRoborazziDebug   # Compare against golden images
```

---

## Tier 3: iOS Testing

There are two separate test surfaces on iOS — do not confuse them:

### 3a. KMP `iosTest` source set (Kotlin)

Lives in `shared/src/iosTest/kotlin/`. Tests Kotlin `actual` implementations that live in `iosMain`. Runs on the iOS simulator via Gradle.

```kotlin
// shared/src/iosTest/kotlin/CryptoImplTest.kt
class IosCryptoImplTest {
    @Test
    fun encryptDecryptRoundTrip() {
        val crypto = IosCryptoImpl()
        val data = "hello".encodeToByteArray()
        val key = ByteArray(32)
        val encrypted = crypto.encrypt(data, key)
        val decrypted = crypto.decrypt(encrypted, key)
        assertContentEquals(data, decrypted)
    }
}
```

Run with: `./gradlew :shared:iosSimulatorArm64Test`

### 3b. Xcode test target (Swift)

Lives in the Xcode project at `iosApp/iosAppTests/`. Tests Swift app-shell code and integrates with the Kotlin/Native framework from the Swift side.

```swift
// iosApp/iosAppTests/ProfileViewTests.swift
import XCTest
@testable import iosApp
import shared

class ProfileViewTests: XCTestCase {
    func testViewModelFromSwift() {
        let vm = ProfileViewModel(savedStateHandle: ..., userRepository: FakeUserRepository())
        // Exercise the KMP VM from Swift
    }
}
```

For Swift snapshot tests, use [swift-snapshot-testing](https://github.com/pointfreeco/swift-snapshot-testing).

Run with: `xcodebuild test -project iosApp/iosApp.xcodeproj -scheme iosApp -destination 'platform=iOS Simulator,name=iPhone 15'`

**Default strategy:** test shared logic in `commonTest` (it's faster, runs on JVM, no simulator needed). Reserve `iosTest` for `actual` implementations, and the Xcode test target for Swift app-shell code.

---

## Running Tests

```bash
# All shared tests (runs on JVM)
./gradlew :shared:allTests

# Android unit tests only
./gradlew :androidApp:test

# Android instrumented tests
./gradlew :androidApp:connectedAndroidTest

# iOS tests (requires Xcode / xcodebuild)
xcodebuild test -project iosApp/iosApp.xcodeproj -scheme iosApp -destination 'platform=iOS Simulator,name=iPhone 15'
```
