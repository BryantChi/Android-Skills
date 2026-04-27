---
id: platform_modernization_2026
name: Platform Modernization 2026
description: Android 平台升級任務集（KSP2、Gradle 9 / AGP 8.7+、compileSdk 35、Edge-to-Edge、Predictive Back、Foreground Service Types、Privacy Sandbox、Photo Picker），產出可逐步推進的升級檢核表
type: skill
next_review: 2027-04
---

# Platform Modernization 2026

## Instructions

當專案需要追上 Android / Kotlin / Gradle 年度升級時載入本 skill。它聚焦「平台層」的強制與行為變更（compileSdk、targetSdk、API 強制執行、工具鏈版本），不取代各領域 skill。**本 skill 為短壽命設計，每年汰換一版**（檔名與 description 都標年度）。

## When to Use

- Scenario M：版本/平台升級（Android 14 → 15、AGP 8.x → 8.7+、Gradle 8 → 9、Kotlin 1.9/2.0 → 2.1）
- 升 targetSdk 35 前的影響面評估
- 從 kapt 全量遷移到 KSP2
- 處理 Android 15 強制 Edge-to-Edge、Predictive Back 後的 layout/動畫破圖
- Foreground Service Types 強制要求後 service 啟動失敗

## When NOT to Use

- 純架構/領域問題（DI、資料層、UI）→ 看對應 mastery skill
- 既有 Compose/RxJava/View 等技術棧的「程式語意」遷移 → 看 `tech_stack_migration`
- 全新專案從 0 建立 → 看 `project_bootstrapping`（會直接套用本 skill 的版本基準）

## Example Prompts

- 「我們專案要升 targetSdk 到 35，幫我列升級檢核表」
- 「kapt 還在用，要遷到 KSP2 怎麼推？哪些套件已支援？」
- 「升 Android 15 後 Activity 內容被 system bar 蓋住，怎麼正確處理 Edge-to-Edge？」
- 「Foreground Service 啟動報 ForegroundServiceTypeException，怎麼宣告 type？」
- 「Gradle 9 Configuration Cache 開不起來，要怎麼修？」

## Workflow

1. **盤點現況**：跑 `./gradlew dependencyUpdates`（versions plugin）+ `./gradlew :app:dependencies` 列出舊版工具鏈與 library。
2. **分批升級**：先升 Gradle/AGP（建構鏈）→ 再升 Kotlin/KSP（編譯鏈）→ 最後升 compileSdk/targetSdk（執行時行為）。每批一個 PR，CI 綠燈再下一批。
3. **行為變更回歸**：對照 §「Android 15 行為變更」「Predictive Back」「Foreground Service Types」逐項驗證。
4. **更新 Baseline Profile**：升級後重跑 Macrobenchmark 產生新 Baseline Profile，避免效能退化。

## Practical Notes (2026-04)

| 元件 | 推薦版本 | 備註 |
|------|----------|------|
| Kotlin | 2.1.x | K2 編譯器穩定 |
| KSP | 2.x | kapt 已不建議 |
| Gradle | 9.0.x | Configuration Cache 預設啟用 |
| AGP | 8.7+ | Variant API 穩定 |
| compileSdk / targetSdk | 35 (Android 15) | Play 上架要求 |
| minSdk（範例） | 24 | 涵蓋率 > 97% |
| JDK（建構） | 21 | AGP 8.7 要求 |

## Minimal Template

`gradle/libs.versions.toml`（升級基準）：

```toml
[versions]
agp = "8.7.2"
kotlin = "2.1.0"
ksp = "2.1.0-1.0.29"
composeBom = "2025.04.00"
hilt = "2.52"

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
```

`app/build.gradle.kts`：

```kotlin
android {
    compileSdk = 35
    defaultConfig {
        minSdk = 24
        targetSdk = 35
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_21
        targetCompatibility = JavaVersion.VERSION_21
    }
    kotlinOptions { jvmTarget = "21" }
}
```

`gradle.properties`：

```properties
org.gradle.configuration-cache=true
org.gradle.caching=true
org.gradle.parallel=true
kotlin.code.style=official
```

## kapt → KSP2 遷移

```kotlin
// Before
plugins { id("kotlin-kapt") }
dependencies {
    kapt("com.google.dagger:hilt-compiler:2.52")
    kapt("androidx.room:room-compiler:2.7.0")
}

// After
plugins { alias(libs.plugins.ksp) }
dependencies {
    ksp("com.google.dagger:hilt-compiler:2.52")
    ksp("androidx.room:room-compiler:2.7.0")
}
```

**遷移檢核**：
- [ ] Hilt 2.48+、Room 2.6+ 支援 KSP（ok）
- [ ] Glide / Moshi-codegen / DataBinding 是否已支援 KSP；DataBinding 仍依賴 kapt → 改用 ViewBinding 或 Compose
- [ ] `kapt.use.k2 = true` 已不需，整個移除 kapt block
- [ ] 量測：建構時間應下降 30-50%

## Gradle 9 / AGP 8.7+ 升級

- **Configuration Cache**：Gradle 9 預設開啟。task action 不可捕捉 `Project` reference；改用 `Provider<T>` 與 `@Input/@OutputFile`。
- **Build features 最小化**：`buildFeatures { viewBinding = false; dataBinding = false }`（除非真的用到）。
- **Repository 集中宣告**：`settings.gradle.kts` 用 `dependencyResolutionManagement { repositoriesMode.set(FAIL_ON_PROJECT_REPOS) }`。
- **JDK 21**：CI 改 `actions/setup-java@v4` + `distribution: 'temurin'`、`java-version: '21'`。

## Android 15 行為變更（targetSdk 35）

### Edge-to-Edge 強制執行

App 一律 edge-to-edge 顯示，system bar 變透明且覆蓋內容。需主動處理 inset。

```kotlin
// Activity.onCreate
WindowCompat.setDecorFitsSystemWindows(window, false)

// Compose
Scaffold(
    contentWindowInsets = WindowInsets.safeDrawing,
) { padding -> /* ... */ }

// 自訂 Composable
Box(
    Modifier
        .fillMaxSize()
        .windowInsetsPadding(WindowInsets.systemBars)
)
```

詳細視覺規範請見 `@ui_ux_engineering` 的「Edge-to-Edge」章節。

### Predictive Back 強制執行

```xml
<!-- AndroidManifest.xml -->
<application android:enableOnBackInvokedCallback="true">
```

```kotlin
// 自訂 back handling
val callback = object : OnBackPressedCallback(true) {
    override fun handleOnBackProgressed(backEvent: BackEventCompat) { /* 動畫進度 */ }
    override fun handleOnBackPressed() { /* 完成 */ }
    override fun handleOnBackCancelled() { /* 取消 */ }
}
onBackPressedDispatcher.addCallback(this, callback)
```

Compose 端用 `PredictiveBackHandler { progress -> ... }`。導航整合請見 `@navigation_patterns`。

### Foreground Service Types

啟動 foreground service 必須宣告 type，否則拋 `ForegroundServiceTypeException`。

```xml
<service
    android:name=".LocationService"
    android:foregroundServiceType="location"
    android:permission="android.permission.FOREGROUND_SERVICE_LOCATION" />
```

```kotlin
ServiceCompat.startForeground(
    this,
    NOTIFICATION_ID,
    notification,
    ServiceInfo.FOREGROUND_SERVICE_TYPE_LOCATION,
)
```

支援 type：`camera`、`connectedDevice`、`dataSync`、`health`、`location`、`mediaPlayback`、`mediaProjection`、`microphone`、`phoneCall`、`remoteMessaging`、`shortService`、`specialUse`、`systemExempted`。

### 16 KB Page Size 相容

Android 15+ 部分裝置使用 16 KB page。NDK / 含 native lib 的第三方套件需重編。檢查：

```bash
./gradlew :app:assembleRelease
# 檢視 .so 是否 16 KB aligned
unzip -p app/build/outputs/apk/release/app-release.apk lib/arm64-v8a/*.so > /tmp/lib.so
readelf -lW /tmp/lib.so | grep LOAD
```

對齊值需為 `0x4000`（16 KB）或更大。

## Photo Picker / Selected Photos Access

避免 `READ_MEDIA_IMAGES` 權限引發的權限疲勞。

```kotlin
val pickMedia = registerForActivityResult(
    PickVisualMedia()
) { uri -> /* 使用者授權的單張圖 */ }

pickMedia.launch(PickVisualMediaRequest(PickVisualMedia.ImageOnly))
```

對於需多張且持續存取的相簿類 App，使用 `PickMultipleVisualMedia` 或 `READ_MEDIA_VISUAL_USER_SELECTED`（Android 14+）的部分授權。

## Privacy Sandbox（Beta → 生產過渡）

- **SDK Runtime**：第三方廣告 SDK 需打包為 SDK Bundle，分離程序執行；自家 App 不直接受影響但 SDK 升級需對齊。
- **Topics / Protected Audience**：替代 cross-app advertising ID，需在 manifest 宣告 `<uses-permission android:name="android.permission.ACCESS_ADSERVICES_TOPICS" />` 與相關 ad-services-config XML。
- 評估清單：是否使用 `AdvertisingIdClient` → 規劃替代信號。

## 常見升級失敗排查

| 症狀 | 根因 | 解法 |
|------|------|------|
| `Configuration cache problems found` | task 用了 Project reference | 改用 `providers.gradleProperty(...)` |
| `Could not find :hilt-compiler` | KSP 版本與 Kotlin 版本不匹配 | 對齊 `ksp = "<kotlin>-1.0.x"` |
| Activity 內容被 status bar 蓋住 | edge-to-edge 強制 + 未處理 inset | `WindowInsets.safeDrawing` padding |
| `ForegroundServiceTypeException` | 未宣告 type | manifest + ServiceCompat.startForeground |
| Native crash on 16 KB device | .so 未 16 KB align | NDK r27+ 重編、第三方升級 |
| `kapt` build 仍跑 | 殘留 plugin 或 dataBinding | 移除 kotlin-kapt、改 ViewBinding |

## Cross-Skill References

- `@project_bootstrapping`：新專案直接套用本 skill 的版本基準與 Convention Plugin。
- `@tech_stack_migration`：技術棧語意層遷移（View→Compose、RxJava→Flow）由它負責，本 skill 只管平台版本。
- `@ui_ux_engineering`：Edge-to-Edge 視覺規範、Material 3 Expressive 細節。
- `@navigation_patterns`：Predictive Back 與 Compose Navigation 3 整合。
- `@release_automation`：CI 端 JDK/Gradle 版本切換、Configuration Cache 設定。
- `@supply_chain_security`：升級伴隨的依賴審查與 SBOM 重生。

## Quick Checklist

- [ ] Gradle 9.0+ / AGP 8.7+ / JDK 21 已升級且 CI 綠燈
- [ ] kapt 全數移除，改用 KSP2，建構時間下降 ≥ 30%
- [ ] compileSdk = 35、targetSdk = 35
- [ ] Edge-to-Edge：所有畫面用 `WindowInsets.safeDrawing` 或對應 padding
- [ ] Predictive Back：`enableOnBackInvokedCallback="true"`，自訂 back handling 已遷移
- [ ] Foreground Service：每個 service 宣告正確 `foregroundServiceType` 與權限
- [ ] 16 KB page size：檢查所有 native .so 對齊
- [ ] Photo Picker 已替換 `READ_MEDIA_IMAGES`（如適用）
- [ ] Baseline Profile 已重新生成，效能無回歸
- [ ] `next_review: 2027-04` 到期前重新審視本 skill
