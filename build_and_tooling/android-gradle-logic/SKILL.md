---
name: android-gradle-logic
description: Expert guidance on setting up scalable Gradle build logic for KMP + CMP projects using Convention Plugins and Version Catalogs.
---

# KMP + Android Gradle Build Logic & Convention Plugins

This skill helps you configure a scalable, maintainable build system for **KMP + Android apps** using **Gradle Convention Plugins** and **Version Catalogs**.

## Goal
Stop copy-pasting code between `build.gradle.kts` files. Centralize build logic (KMP target setup, CMP plugin, Compose compiler, Koin, Kotlin options, hierarchy template) in reusable plugins.

## Project Structure

Ensure your project has a `build-logic` directory included in `settings.gradle.kts` as a composite build.

```text
root/
├── build-logic/
│   ├── convention/
│   │   ├── src/main/kotlin/
│   │   │   ├── AndroidApplicationConventionPlugin.kt
│   │   │   └── KmpLibraryConventionPlugin.kt
│   │   └── build.gradle.kts
│   ├── build.gradle.kts
│   └── settings.gradle.kts
├── gradle/
│   └── libs.versions.toml
├── shared/                 ← KMP module (commonMain/androidMain/iosMain)
│   └── build.gradle.kts
├── androidApp/             ← Android entry point
│   └── build.gradle.kts
├── iosApp/                 ← Xcode project
└── settings.gradle.kts
```

## Step 1: Configure `settings.gradle.kts`

Include the `build-logic` as a plugin management source.

```kotlin
// settings.gradle.kts
pluginManagement {
    includeBuild("build-logic")
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}
```

## Step 2: Define Dependencies in `libs.versions.toml`

Use the Version Catalog for both libraries *and* plugins.

```toml
[versions]
androidGradlePlugin    = "8.5.0"
kotlin                 = "2.0.21"
composeMultiplatform   = "1.8.0"

[libraries]
# ...

[plugins]
android-application      = { id = "com.android.application",                 version.ref = "androidGradlePlugin" }
android-library          = { id = "com.android.library",                      version.ref = "androidGradlePlugin" }
kotlin-android           = { id = "org.jetbrains.kotlin.android",             version.ref = "kotlin" }
kotlin-multiplatform     = { id = "org.jetbrains.kotlin.multiplatform",       version.ref = "kotlin" }
kotlin-serialization     = { id = "org.jetbrains.kotlin.plugin.serialization",version.ref = "kotlin" }
compose-multiplatform    = { id = "org.jetbrains.compose",                    version.ref = "composeMultiplatform" }
compose-compiler         = { id = "org.jetbrains.kotlin.plugin.compose",      version.ref = "kotlin" }
# Convention plugins
myapp-android-application = { id = "myapp.android.application", version = "unspecified" }
myapp-kmp-library         = { id = "myapp.kmp.library",         version = "unspecified" }
```

## Step 3: Create a Convention Plugin

Inside `build-logic/convention/src/main/kotlin/AndroidApplicationConventionPlugin.kt`:

```kotlin
import com.android.build.api.dsl.ApplicationExtension
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure

class AndroidApplicationConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("com.android.application")
                apply("org.jetbrains.kotlin.android")
            }

            extensions.configure<ApplicationExtension> {
                defaultConfig.targetSdk = 34
                // Configure common options here
            }
        }
    }
}
```

Don't forget to register it in `build-logic/convention/build.gradle.kts`:

```kotlin
gradlePlugin {
    plugins {
        register("androidApplication") {
            id = "myapp.android.application"
            implementationClass = "AndroidApplicationConventionPlugin"
        }
    }
}
```

## KMP Convention Plugin Example

Add a `KmpLibraryConventionPlugin` for shared KMP modules:

```kotlin
// build-logic/convention/src/main/kotlin/KmpLibraryConventionPlugin.kt
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.configure
import org.jetbrains.kotlin.gradle.dsl.KotlinMultiplatformExtension

class KmpLibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("org.jetbrains.kotlin.multiplatform")
                apply("com.android.library")
            }
            extensions.configure<KotlinMultiplatformExtension> {
                androidTarget()
                iosX64()
                iosArm64()
                iosSimulatorArm64()

                // Explicit hierarchy: only your actual targets
                applyHierarchyTemplate {
                    common {
                        withAndroidTarget()
                        group("ios") {
                            withIosX64()
                            withIosArm64()
                            withIosSimulatorArm64()
                        }
                    }
                }
            }
        }
    }
}
```

Register in `build-logic/convention/build.gradle.kts`:
```kotlin
gradlePlugin {
    plugins {
        register("kmpLibrary") {
            id = "myapp.kmp.library"
            implementationClass = "KmpLibraryConventionPlugin"
        }
    }
}
```

## Usage

```kotlin
// shared/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.kmp.library)
}

// androidApp/build.gradle.kts
plugins {
    alias(libs.plugins.myapp.android.application)
}
```

Convention plugins drastically clean up module-level build files and enforce consistent configuration across all modules.
