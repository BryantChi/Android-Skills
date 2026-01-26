---
name: Project Bootstrapping
description: 快速建立專案骨架、Gradle Convention Plugins 與標準化架構
---

# Project Bootstrapping (專案快速建置)

**Related Scenarios**: A (新專案)

---

## One-Command Setup

### GitHub Template Repository

建立公司內部的 Template Repository，包含：

```
my-company-android-template/
├── app/
├── build-logic/
│   └── convention/           # Convention Plugins
├── core/
│   ├── common/
│   ├── data/
│   ├── domain/
│   ├── network/
│   └── ui/
├── feature/
│   └── sample/
├── gradle/
│   └── libs.versions.toml    # Version Catalog
├── .editorconfig
├── detekt.yml
└── README.md
```

### 使用方式

```bash
# GitHub Template → Use this template
# 或使用 gh cli
gh repo create my-new-app --template my-company/android-template
```

---

## Gradle Convention Plugins

### 目錄結構

```
build-logic/
├── convention/
│   ├── build.gradle.kts
│   └── src/main/kotlin/
│       ├── AndroidApplicationConventionPlugin.kt
│       ├── AndroidLibraryConventionPlugin.kt
│       ├── AndroidComposeConventionPlugin.kt
│       └── AndroidFeatureConventionPlugin.kt
└── settings.gradle.kts
```

### settings.gradle.kts (root)

```kotlin
pluginManagement {
    includeBuild("build-logic")
}
```

### AndroidLibraryConventionPlugin.kt

```kotlin
class AndroidLibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("com.android.library")
                apply("org.jetbrains.kotlin.android")
            }
            
            extensions.configure<LibraryExtension> {
                compileSdk = 34
                defaultConfig.minSdk = 24
                
                compileOptions {
                    sourceCompatibility = JavaVersion.VERSION_17
                    targetCompatibility = JavaVersion.VERSION_17
                }
            }
        }
    }
}
```

### 使用方式 (feature module)

```kotlin
// feature/login/build.gradle.kts
plugins {
    id("mycompany.android.feature")  // 一行搞定！
}

dependencies {
    implementation(projects.core.domain)
}
```

---

## Version Catalog (libs.versions.toml)

```toml
[versions]
kotlin = "1.9.22"
compose-bom = "2024.01.00"
hilt = "2.50"
room = "2.6.1"
retrofit = "2.9.0"

[libraries]
# Compose
compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
compose-ui = { group = "androidx.compose.ui", name = "ui" }
compose-material3 = { group = "androidx.compose.material3", name = "material3" }

# DI
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-compiler", version.ref = "hilt" }

[bundles]
compose = ["compose-ui", "compose-material3"]

[plugins]
android-application = { id = "com.android.application", version = "8.2.2" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
```

---

## Package Structure (Feature-based)

```
com.example.app/
├── core/
│   ├── common/          # 共用工具 (Extensions, Utils)
│   ├── data/            # Repository 實作
│   ├── domain/          # UseCase, Entity
│   ├── network/         # Retrofit, API
│   └── ui/              # Design System, Theme
├── feature/
│   ├── auth/
│   │   ├── data/        # Feature-specific data
│   │   ├── domain/      # Feature-specific use cases
│   │   └── ui/          # Screens, ViewModels
│   └── home/
└── app/                 # Application, DI, Navigation
```

---

## Quick Checklist

### 新專案建立
- [ ] 使用 Template Repository
- [ ] Convention Plugins 設定完成
- [ ] Version Catalog 配置
- [ ] Detekt/Ktlint 整合
- [ ] CI/CD 基礎 Pipeline

### 新模組建立
- [ ] 使用正確的 Convention Plugin
- [ ] 遵循 Package Structure
- [ ] 加入 Navigation Graph (如需要)
