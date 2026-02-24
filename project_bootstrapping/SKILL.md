---
id: project_bootstrapping
name: Project Bootstrapping
description: 快速建立專案骨架、Gradle Convention Plugins 與標準化架構
---

# Project Bootstrapping (專案快速建置)

## Instructions
- 僅在新專案或新模組起步時使用
- 依照下方章節順序建立骨架
- 一次只處理一個子系統（插件、版本、結構）
- 完成後對照 Quick Checklist

## When to Use
- Scenario A：從零建立新專案

## Example Prompts
- "請依照 One-Command Setup，建立公司模板的專案骨架"
- "依照 Gradle Convention Plugins 章節，建立 feature module 插件"
- "請根據 Package Structure 規劃模組與套件配置"

## Workflow
1. 先建立 Template 與目錄結構
2. 再落實 Convention Plugins 與 Version Catalog
3. 最後用 Quick Checklist 驗收

## Practical Notes (2026)
- 預設建立 CI Gate：lint、detekt、unit test、assemble
- 新專案先建立 Baseline Profile 量測框架
- Version Catalog 作為單一依賴來源

## Minimal Template
```
目標: 
模組範圍: 
Convention Plugins: 
CI Gate: 
驗收: Quick Checklist
```

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
