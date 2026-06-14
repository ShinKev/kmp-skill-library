---
name: coil-compose
description: Expert guidance on using Coil 3 for image loading in Compose Multiplatform. Use when asked about loading images from URLs, handling image states, or optimizing image performance in CMP (Android + iOS).
---

# Coil 3 for Compose Multiplatform

## Instructions

Use **Coil 3** (`coil3`) for image loading in CMP. Coil 3 is fully KMP-compatible — it works in `commonMain` composables without `LocalContext.current`.

> **Migration note:** Coil 2.x (`coil` package) required `LocalContext.current` in `ImageRequest.Builder`. Coil 3 (`coil3` package) removes this requirement. If you see `LocalContext.current` in image loading code, update to Coil 3.

### Setup (`libs.versions.toml`)

```toml
[versions]
coil = "3.1.0"

[libraries]
coil-compose  = { group = "io.coil-kt.coil3", name = "coil-compose",        version.ref = "coil" }
coil-network  = { group = "io.coil-kt.coil3", name = "coil-network-ktor",   version.ref = "coil" }
```

```kotlin
// shared/build.gradle.kts
commonMain.dependencies {
    implementation(libs.coil.compose)
    implementation(libs.coil.network) // Uses Ktor under the hood
}
```

### 1. Primary Composable: `AsyncImage`

Use `AsyncImage` for most cases. No `LocalContext` needed in `commonMain`:

```kotlin
// coil3 — works in commonMain
AsyncImage(
    model = ImageRequest.Builder()
        .data("https://example.com/image.jpg")
        .crossfade(true)
        .build(),
    contentDescription = "Profile picture",
    contentScale = ContentScale.Crop,
    placeholder = painterResource(Res.drawable.placeholder),
    error = painterResource(Res.drawable.error_image),
    modifier = Modifier
        .size(64.dp)
        .clip(CircleShape)
)
```

### 2. Low-Level Control: `rememberAsyncImagePainter`

Use only when you need a `Painter` (e.g., for `Canvas` or `Icon`).

> **Warning:** `rememberAsyncImagePainter` does not auto-detect the display size and loads the image at its original dimensions by default. Always set an explicit size or prefer `AsyncImage`.

```kotlin
val painter = rememberAsyncImagePainter(
    model = ImageRequest.Builder()
        .data("https://example.com/image.jpg")
        .size(Size.ORIGINAL)
        .build()
)
```

### 3. Custom State Slots: `SubcomposeAsyncImage`

Use when you need custom loading/error UI with access to the `AsyncImagePainter.State`.

> **Caution:** Subcomposition is slower than regular composition. Avoid in `LazyColumn` or `LazyRow`.

```kotlin
SubcomposeAsyncImage(
    model = "https://example.com/image.jpg",
    contentDescription = null,
    loading = { CircularProgressIndicator(modifier = Modifier.size(24.dp)) },
    error = { Icon(Icons.Default.BrokenImage, contentDescription = null) }
)
```

### 4. Singleton ImageLoader via Koin

Provide a single `ImageLoader` instance for the whole app to share cache:

```kotlin
// commonMain Koin module
val imageModule = module {
    single {
        ImageLoader.Builder()
            .components { add(KtorNetworkFetcherFactory(get())) } // reuse app HttpClient
            .crossfade(true)
            .build()
    }
}
```

### 5. Performance & Best Practices

- **Singleton**: Always share one `ImageLoader` — it owns the disk/memory cache.
- **Crossfade**: Enable `crossfade(true)` for smooth placeholder → image transitions.
- **Sizing**: Set `contentScale` appropriately — Coil 3 uses it to avoid loading images larger than needed.
- **Lists**: Prefer `AsyncImage` over `SubcomposeAsyncImage` inside `LazyColumn`/`LazyRow`.
- **Transformations**: Apply via `ImageRequest.Builder().transformations(CircleCropTransformation())`.

### 6. Checklist

- [ ] Using `coil3` package (not `coil` — that's Coil 2)
- [ ] No `LocalContext.current` in `commonMain` image loading code
- [ ] `ImageLoader` provided as a Koin singleton
- [ ] `AsyncImage` preferred over other variants
- [ ] Meaningful `contentDescription` on all images (or `null` for decorative)
- [ ] `crossfade(true)` enabled for better UX
- [ ] `SubcomposeAsyncImage` avoided in lazy lists
