---
id: project_bootstrapping
name: Project Bootstrapping
description: Android 新專案/新模組骨架建置 — Gradle Convention Plugins、Version Catalog、Feature-based package、Kotlin 2.1 + KSP2 + Compose BOM 2025.04 預設、CI 基礎 gate，產出可立即開發的標準化骨架
type: skill
---

# Project Bootstrapping

## Instructions

當需要從零建立 Android 專案，或為既有 multi-module repo 增添新 feature module 時載入本 skill。它只負責「骨架」與「Convention」；應用層 DI、資料層、UI 細節分別由對應 mastery skill 處理。版本與工具鏈一律對齊 `@platform_modernization_2026` 的 2026-04 基準。

## When to Use

- Scenario A：從零建立新專案
- 需要為現有 repo 加 feature module 並套上一致 Convention
- 想把零散 build script 收斂成 Convention Plugins
- 將 Version Catalog 標準化為單一依賴來源

## When NOT to Use

- DI 細節（Hilt module / Custom Component） → `@dependency_injection_mastery`
- Detekt/Ktlint 規則本身的設計 → `@coding_style_conventions`（本 skill 只做接線）
- 平台版本升級（KSP2 遷移、Gradle 9 升級）→ `@platform_modernization_2026`
- 把舊專案改造成新架構 → `@tech_stack_migration`

## Example Prompts

- 「從公司模板建一個新 App，含 build-logic、Compose、Hilt」
- 「幫我寫 AndroidFeatureConventionPlugin，套上之後 feature module 一行就能用」
- 「Version Catalog 怎麼安排 bundles？哪些該綁一起？」
- 「Package 結構用 feature-based，core/feature 怎麼切？」

## Workflow

1. **Template Repo** 或 `gh repo create --template`。
2. **build-logic** 掛 4 個 Convention Plugin（application/library/compose/feature）。
3. **gradle/libs.versions.toml** 建立，作為唯一依賴來源。
4. **core / feature 雙層** package 結構就位。
5. **CI Gate 基線**（lint / detekt / ktlint / unit test / assemble）；後續細化交給 `@release_automation`。

## Practical Notes (2026-04)

- 預設工具鏈：Kotlin 2.1.x、KSP 2.x、Gradle 9.0+、AGP 8.7+、JDK 21、compileSdk 35、minSdk 24。
- 預設 UI：Compose BOM 2025.04.x + Material 3（含 Expressive）。
- 預設 DI：Hilt 2.52+ via KSP（不用 kapt）。
- 預設 baseline：Macrobenchmark 模組 + 空白 Baseline Profile（後續 `@deep_performance_tuning` 接管）。
- 不在骨架塞 RxJava、Dagger 2、kapt、ViewBinding（除非明確需求）。

## Minimal Template

```
my-company-android-template/
├── app/
├── benchmark/                      # Macrobenchmark 模組（預埋）
├── build-logic/
│   ├── settings.gradle.kts
│   └── convention/
│       ├── build.gradle.kts
│       └── src/main/kotlin/
│           ├── AndroidApplicationConventionPlugin.kt
│           ├── AndroidLibraryConventionPlugin.kt
│           ├── AndroidComposeConventionPlugin.kt
│           ├── AndroidFeatureConventionPlugin.kt
│           └── AndroidHiltConventionPlugin.kt
├── core/
│   ├── common/         # Extensions / Utils（無 Android 依賴）
│   ├── data/           # Repository impl
│   ├── domain/         # UseCase, Entity
│   ├── network/        # Retrofit / Ktor
│   ├── database/       # Room
│   └── ui/             # Design System / Theme
├── feature/
│   └── sample/
├── gradle/libs.versions.toml
├── .editorconfig
├── config/detekt/detekt.yml
└── README.md
```

```bash
gh repo create my-new-app --template my-company/android-template --private
```

## Gradle Convention Plugins

### `build-logic/settings.gradle.kts`

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
    versionCatalogs {
        create("libs") { from(files("../gradle/libs.versions.toml")) }
    }
}
rootProject.name = "build-logic"
include(":convention")
```

### `build-logic/convention/build.gradle.kts`

```kotlin
plugins { `kotlin-dsl` }

dependencies {
    compileOnly(libs.android.gradle.plugin)
    compileOnly(libs.kotlin.gradle.plugin)
    compileOnly(libs.ksp.gradle.plugin)
    compileOnly(libs.compose.compiler.gradle.plugin)
}

gradlePlugin {
    plugins {
        register("androidApplication") {
            id = "mycompany.android.application"
            implementationClass = "AndroidApplicationConventionPlugin"
        }
        register("androidLibrary") {
            id = "mycompany.android.library"
            implementationClass = "AndroidLibraryConventionPlugin"
        }
        register("androidCompose") {
            id = "mycompany.android.compose"
            implementationClass = "AndroidComposeConventionPlugin"
        }
        register("androidFeature") {
            id = "mycompany.android.feature"
            implementationClass = "AndroidFeatureConventionPlugin"
        }
        register("androidHilt") {
            id = "mycompany.android.hilt"
            implementationClass = "AndroidHiltConventionPlugin"
        }
    }
}
```

### `AndroidLibraryConventionPlugin.kt`

```kotlin
class AndroidLibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) = with(target) {
        with(pluginManager) {
            apply("com.android.library")
            apply("org.jetbrains.kotlin.android")
        }

        extensions.configure<LibraryExtension> {
            compileSdk = 35
            defaultConfig {
                minSdk = 24
                consumerProguardFiles("consumer-rules.pro")
            }
            compileOptions {
                sourceCompatibility = JavaVersion.VERSION_21
                targetCompatibility = JavaVersion.VERSION_21
            }
        }
        extensions.configure<KotlinAndroidProjectExtension> {
            jvmToolchain(21)
        }
    }
}
```

### `AndroidHiltConventionPlugin.kt`（KSP-only，無 kapt）

```kotlin
class AndroidHiltConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) = with(target) {
        with(pluginManager) {
            apply("com.google.devtools.ksp")
            apply("com.google.dagger.hilt.android")
        }

        dependencies {
            "implementation"(libs.findLibrary("hilt.android").get())
            "ksp"(libs.findLibrary("hilt.compiler").get())
        }
    }
}
```

### `AndroidFeatureConventionPlugin.kt`（一行套滿 feature module）

```kotlin
class AndroidFeatureConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) = with(target) {
        with(pluginManager) {
            apply("mycompany.android.library")
            apply("mycompany.android.compose")
            apply("mycompany.android.hilt")
        }
        dependencies {
            "implementation"(project(":core:domain"))
            "implementation"(project(":core:ui"))
            "implementation"(libs.findLibrary("androidx.lifecycle.runtimeCompose").get())
        }
    }
}
```

使用：

```kotlin
// feature/login/build.gradle.kts
plugins { id("mycompany.android.feature") }
android { namespace = "com.mycompany.feature.login" }
```

## Version Catalog（2026-04 基準）

```toml
# gradle/libs.versions.toml
[versions]
agp = "8.7.2"
kotlin = "2.1.0"
ksp = "2.1.0-1.0.29"
composeBom = "2025.04.00"
hilt = "2.52"
room = "2.7.0"
retrofit = "2.11.0"
okhttp = "4.12.0"
coroutines = "1.9.0"
lifecycle = "2.8.7"
navigation = "2.8.5"           # 升至 3.x 後改 navigation3
junit = "4.13.2"
truth = "1.4.4"

[libraries]
android-gradle-plugin = { group = "com.android.tools.build", name = "gradle", version.ref = "agp" }
kotlin-gradle-plugin = { group = "org.jetbrains.kotlin", name = "kotlin-gradle-plugin", version.ref = "kotlin" }
ksp-gradle-plugin = { group = "com.google.devtools.ksp", name = "com.google.devtools.ksp.gradle.plugin", version.ref = "ksp" }
compose-compiler-gradle-plugin = { group = "org.jetbrains.kotlin", name = "compose-compiler-gradle-plugin", version.ref = "kotlin" }

compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "composeBom" }
compose-ui = { group = "androidx.compose.ui", name = "ui" }
compose-material3 = { group = "androidx.compose.material3", name = "material3" }
compose-ui-tooling-preview = { group = "androidx.compose.ui", name = "ui-tooling-preview" }

androidx-lifecycle-runtimeCompose = { group = "androidx.lifecycle", name = "lifecycle-runtime-compose", version.ref = "lifecycle" }

hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-android-compiler", version.ref = "hilt" }

room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }

retrofit = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
okhttp-bom = { group = "com.squareup.okhttp3", name = "okhttp-bom", version.ref = "okhttp" }

[bundles]
compose = ["compose-ui", "compose-material3", "compose-ui-tooling-preview"]
room = ["room-runtime", "room-ktx"]

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
android-library = { id = "com.android.library", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
```

## Package Structure（Feature-based）

```
com.example.app/
├── core/
│   ├── common/          # KMP-ready，不依 Android
│   ├── data/            # Repository impl + DataSource
│   ├── domain/          # UseCase, Model
│   ├── network/         # Retrofit/Ktor + Interceptor
│   ├── database/        # Room
│   └── ui/              # Design System, Theme（Material 3 Expressive）
├── feature/
│   ├── auth/
│   │   ├── data/  domain/  ui/
│   └── home/
│       ├── data/  domain/  ui/
└── app/                 # Application, NavHost, top-level DI
```

**邊界規則**：
- `core:*` 不可依 `feature:*`
- `feature:a` 不可依 `feature:b`（跨 feature 走 navigation event 或 core domain）
- `app` 是唯一可依所有 feature 的模組

## Cross-Skill References

- `@platform_modernization_2026`：版本基準與行為變更（Edge-to-Edge、Predictive Back）。本 skill 套用其 catalog。
- `@dependency_injection_mastery`：Hilt module 設計、Custom Component；本 skill 只套 Hilt plugin。
- `@coding_style_conventions`：Detekt/Ktlint 規則檔；本 skill 只串到 build。
- `@release_automation`：CI workflow 設計；本 skill 只列基線 gate。
- `@navigation_patterns`、`@ui_ux_engineering`、`@data_layer_mastery`：對應領域細節。

## Quick Checklist

- [ ] Template Repo 或 `gh repo create --template` 建立完成
- [ ] `build-logic` 包含 5 個 Convention Plugin（application/library/compose/feature/hilt）
- [ ] `gradle/libs.versions.toml` 為唯一依賴來源
- [ ] compileSdk = 35、JDK 21、Kotlin 2.1、KSP 2.x
- [ ] core / feature 邊界規則寫入 README
- [ ] benchmark 模組已預埋
- [ ] CI 基線 gate（lint/detekt/ktlint/unit/assemble）綠燈
- [ ] Detekt + Ktlint 規則檔位於 `config/`
