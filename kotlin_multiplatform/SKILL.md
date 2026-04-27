---
id: kotlin_multiplatform
name: Kotlin Multiplatform
description: KMP 跨平台架構（2026-04） — 共享邊界決策、Kotlin 2.1 + Ktor 3.x + SQLDelight 2.x、iOS Interop（SPM、Shared ViewModel）、Compose Multiplatform 策略、平台 Dispatcher 差異、Serialization 選型、Room→SQLDelight 遷移路徑
type: skill
---

# Kotlin Multiplatform

## Instructions

當需要跨平台共享邏輯（特別是 Android + iOS）時載入。本 skill 專注 KMP 共享層；Android 端的 Room/Compose/Hilt 細節由對應 mastery skill 處理，本 skill 只提供 KMP 對接點。

## When to Use

- Scenario F：跨平台共享邏輯
- 評估專案要不要走 KMP（vs 純 Android + 純 iOS）
- 設計 commonMain 邊界、處理 expect/actual
- iOS Interop（SPM、Swift Package、Shared ViewModel）
- 選擇 Ktor / SQLDelight / kotlinx-serialization
- Compose Multiplatform vs SwiftUI 決策

## When NOT to Use

- Android-only 資料層細節（Room 進階）→ `@data_layer_mastery`
- Android-only DI 設計 → `@dependency_injection_mastery`
- 純 iOS Swift/SwiftUI 細節 → 不在本 skill 範圍
- 平台升級（Kotlin 2.1、KSP2 在 Android）→ `@platform_modernization_2026`

## Example Prompts

- 「我們有 Android + iOS，該不該開 KMP？怎麼盤點共享比例？」
- 「Shared ViewModel 在 iOS 端怎麼接 SwiftUI？」
- 「Room 換 SQLDelight 的遷移步驟」
- 「iOS Framework 用 SPM 包，Xcode 怎麼接？」
- 「Compose Multiplatform 和 SwiftUI 各做一份，怎麼決策？」

## Workflow

1. **共享邊界決策**：列模組對 ROI 矩陣（程式碼量 × 平台差異 × 維護成本），UI 一律不共享除非用 CMP。
2. **Project Setup**：Kotlin 2.1 + Gradle 9 + targets（android、iosX64/Arm64/SimulatorArm64）。
3. **Network/DB/Serialization**：commonMain 寫業務，platform main 提供 driver/engine。
4. **Shared ViewModel**：用 `androidx.lifecycle.viewmodel:viewmodel-compose` 的 KMP 版（2.8+）。
5. **iOS Distribution**：Cocoapods（過渡）或 SPM（推薦）。
6. **Test**：commonTest 覆蓋業務邏輯，platform test 驗 driver。

## Practical Notes (2026-04)

| 元件 | 版本 | 備註 |
|------|------|------|
| Kotlin | 2.1.x | K2 + KMP stable |
| Gradle | 9.0+ | |
| Ktor | 3.x | 取代 2.x，coroutines/serialization 內建升級 |
| SQLDelight | 2.x | 已支援 Kotlin 2.x |
| kotlinx-coroutines | 1.9+ | Native dispatcher 改為 default `Dispatchers.IO` |
| kotlinx-serialization | 1.7+ | json + protobuf |
| Compose Multiplatform | 1.7.x | iOS 為 stable beta |
| AndroidX Lifecycle ViewModel | 2.8.x（KMP） | commonMain 可用 |
| Room | 2.7.x（KMP，iOS 仍 alpha） | 評估後再用 |
| Cocoapods | 過渡 | 新專案首選 SPM |
| Swift Package Manager | 預設整合 | `xcframework` 產出可直接放 Swift Package |

## Minimal Template

```
共享邊界:
  - shared:domain（純 Kotlin，無平台依賴）
  - shared:data（Ktor + SQLDelight + Settings）
  - shared:viewmodel（AndroidX Lifecycle KMP，可選）
不共享:
  - androidApp/Compose UI 或 shared CMP UI
  - iosApp/SwiftUI（若不用 CMP）
  - 平台 SDK（HealthKit / Wear / WorkManager）
產出:
  - .aar for Android
  - .xcframework + Swift Package for iOS
測試: commonTest（業務邏輯）+ androidUnitTest / iosTest（driver）
驗收: Quick Checklist
```

## 共享邊界決策

| 屬性 | 共享 | 不共享 |
|------|------|--------|
| 純業務邏輯（formula、validation、policy） | ✅ | |
| Repository 介面 + DTO | ✅ | |
| Network 呼叫（Ktor） | ✅ | |
| Database 操作（SQLDelight） | ✅ | |
| 通用 ViewModel（state machine） | ✅（用 KMP Lifecycle） | |
| UI（Compose/SwiftUI） | 🟡 視 CMP 決策 | ✅ 預設不共享 |
| 平台 SDK（Push、Sensor、Wear） | | ✅ |
| 設計 Token | 🟡 純 token 可，UI 元件分開 | |

**共享比例 30-70% 為甜蜜點**：太低無 ROI，太高 UI/平台差異會把 commonMain 弄成 if-else 大會。

## Project Setup

### shared/build.gradle.kts

```kotlin
plugins {
    alias(libs.plugins.kotlin.multiplatform)
    alias(libs.plugins.android.library)
    alias(libs.plugins.kotlin.serialization)
    alias(libs.plugins.sqldelight)
}

kotlin {
    androidTarget {
        compilerOptions { jvmTarget.set(JvmTarget.JVM_21) }
    }

    listOf(iosX64(), iosArm64(), iosSimulatorArm64()).forEach { target ->
        target.binaries.framework {
            baseName = "Shared"
            isStatic = true
            export(libs.androidx.lifecycle.viewmodel)   // 暴露給 Swift 端
        }
    }

    // Swift Package Manager 友善的 xcframework
    targets.withType<KotlinNativeTarget>().configureEach {
        compilations.configureEach { kotlinOptions.freeCompilerArgs += "-Xexpect-actual-classes" }
    }

    sourceSets {
        commonMain.dependencies {
            implementation(libs.ktor.client.core)
            implementation(libs.ktor.client.content.negotiation)
            implementation(libs.ktor.serialization.kotlinx.json)
            implementation(libs.kotlinx.coroutines.core)
            implementation(libs.kotlinx.serialization.json)
            implementation(libs.sqldelight.runtime)
            implementation(libs.sqldelight.coroutines.extensions)
            api(libs.androidx.lifecycle.viewmodel)
        }
        commonTest.dependencies {
            implementation(libs.kotlin.test)
            implementation(libs.kotlinx.coroutines.test)
            implementation(libs.turbine)
        }
        androidMain.dependencies {
            implementation(libs.ktor.client.okhttp)
            implementation(libs.sqldelight.android.driver)
        }
        iosMain.dependencies {
            implementation(libs.ktor.client.darwin)
            implementation(libs.sqldelight.native.driver)
        }
    }
}
```

## Ktor 3.x 跨平台 Client

```kotlin
// commonMain
class ApiClient(private val client: HttpClient) {
    suspend fun getUser(id: String): User =
        client.get("https://api.example.com/users/$id").body()
}

expect fun httpClientEngine(): HttpClientEngineFactory<*>

fun apiClient(): ApiClient = ApiClient(
    HttpClient(httpClientEngine()) {
        install(ContentNegotiation) { json(Json { ignoreUnknownKeys = true }) }
        install(HttpTimeout) { requestTimeoutMillis = 15_000 }
        install(Logging) { level = LogLevel.INFO }
        defaultRequest { header(HttpHeaders.Accept, "application/json") }
    }
)

// androidMain
actual fun httpClientEngine(): HttpClientEngineFactory<*> = OkHttp

// iosMain
actual fun httpClientEngine(): HttpClientEngineFactory<*> = Darwin
```

## SQLDelight 2.x 跨平台 DB

```sql
-- shared/src/commonMain/sqldelight/com/example/db/User.sq
CREATE TABLE userEntity (
    id TEXT PRIMARY KEY NOT NULL,
    name TEXT NOT NULL,
    email TEXT NOT NULL
);

selectAll:
SELECT * FROM userEntity;

upsert:
INSERT OR REPLACE INTO userEntity(id, name, email) VALUES (?, ?, ?);
```

```kotlin
// commonMain
expect class DriverFactory {
    fun create(): SqlDriver
}

class UserDataSource(driverFactory: DriverFactory) {
    private val db = AppDatabase(driverFactory.create())
    fun observeAll(): Flow<List<UserEntity>> = db.userQueries.selectAll().asFlow().mapToList(Dispatchers.IO)
}

// androidMain
actual class DriverFactory(private val context: Context) {
    actual fun create(): SqlDriver = AndroidSqliteDriver(AppDatabase.Schema, context, "app.db")
}

// iosMain
actual class DriverFactory {
    actual fun create(): SqlDriver = NativeSqliteDriver(AppDatabase.Schema, "app.db")
}
```

### Room → SQLDelight 遷移路徑

1. 把 Room schema 的所有 `@Entity` 改成等價 `.sq` `CREATE TABLE`。
2. 把 `@Dao` 的 query 寫成 `.sq` 的 named query。
3. 改用 SQLDelight 的 `asFlow().mapToList(IO)` 取代 Room 的 `Flow<List<Entity>>`。
4. Migration 檔：用 SQLDelight `migration { version = N; ... }` 對應 Room Migration。
5. 維持 `core:database` 介面不變，逐 entity 切換、A/B 對拍。

## Shared ViewModel（AndroidX Lifecycle KMP）

```kotlin
// commonMain
class CounterViewModel : ViewModel() {
    private val _state = MutableStateFlow(0)
    val state: StateFlow<Int> = _state.asStateFlow()
    fun increment() { _state.update { it + 1 } }
}
```

### Android 端

```kotlin
@Composable
fun CounterScreen(vm: CounterViewModel = viewModel()) {
    val v by vm.state.collectAsStateWithLifecycle()
    Button(onClick = vm::increment) { Text("Count: $v") }
}
```

### iOS 端（SwiftUI）

```swift
import Shared

@MainActor
class CounterObservable: ObservableObject {
    let vm = CounterViewModel()
    @Published var value: Int32 = 0
    private var task: Task<Void, Never>?

    init() {
        task = Task {
            for await v in asyncSequence(for: vm.state) {
                value = v.int32Value
            }
        }
    }
    deinit { task?.cancel() }
}

struct CounterView: View {
    @StateObject var observable = CounterObservable()
    var body: some View {
        Button("Count: \(observable.value)") { observable.vm.increment() }
    }
}
```

`asyncSequence(for:)` 來自 `KMP-NativeCoroutines` 或自家 helper（需處理 cancellation）。

## iOS Distribution（SPM 為主）

`shared/build.gradle.kts` 用 `XCFramework` task：

```kotlin
val xcf = XCFramework("Shared")
listOf(iosX64(), iosArm64(), iosSimulatorArm64()).forEach { target ->
    target.binaries.framework {
        baseName = "Shared"
        xcf.add(this)
    }
}
```

```bash
./gradlew :shared:assembleSharedXCFramework
# 產出 shared/build/XCFrameworks/release/Shared.xcframework
```

放到獨立 Swift Package repo：

```swift
// Package.swift
let package = Package(
    name: "Shared",
    platforms: [.iOS(.v15)],
    products: [.library(name: "Shared", targets: ["Shared"])],
    targets: [
        .binaryTarget(name: "Shared", path: "Shared.xcframework"),
    ]
)
```

iOS 專案 `File → Add Package Dependencies` 加 GitHub URL 即可。

## Coroutines / Dispatcher 差異

| 平台 | Main | IO | Default |
|------|------|----|---------|
| Android | `Dispatchers.Main.immediate`（UI 執行緒） | `Dispatchers.IO` | `Dispatchers.Default` |
| iOS（Native） | `Dispatchers.Main`（main queue） | `Dispatchers.IO`（kotlinx-coroutines 1.9+ 提供） | `Dispatchers.Default` |
| 注意 | Native 端 Dispatcher 切換成本較高，避免頻繁切換 | iOS 上 `IO` 為 thread pool（不同於 Android） |

```kotlin
// commonMain
suspend fun loadAndProcess(): Result<List<User>> = withContext(Dispatchers.Default) {
    val raw = apiClient.getUsers()
    raw.map { User(it.id, it.name) }
}
```

不要在 commonMain 寫死 `Dispatchers.Main`；UI 邊界由 Android/iOS 端 collect 時自處理。

## Serialization 選型

| 場景 | 選擇 |
|------|------|
| HTTP JSON | kotlinx-serialization-json |
| 高效率二進位（內部 RPC） | kotlinx-serialization-protobuf |
| 與 Swift `Codable` 互通 | json（兩端都熟） |
| 大量 binary blob（圖片、音檔） | 不用 serialization，用 byte array |

## Compose Multiplatform 決策

| 因素 | 走 CMP | 各平台原生 UI |
|------|--------|----------------|
| 設計強調平台原生感 | ❌ | ✅ |
| iOS HIG 嚴格遵守 | ❌ | ✅ |
| 想最大化共享、設計可調整 | ✅ | |
| 動畫/手勢複雜 | ⚠️（部分 API 仍演進） | ✅ |
| 團隊 iOS 熟悉度低 | ✅ | ❌ |

CMP 1.7+ iOS 已 stable beta，但 `LazyColumn` performance、TextField imm 等仍與原生有差異；建議先在 internal app 試水。

## Cross-Skill References

- `@data_layer_mastery`：Android 端 Room 細節；本 skill 提供 Room↔SQLDelight 決策與遷移。
- `@dependency_injection_mastery`：Android 端用 Hilt；iOS 端常用 Koin（commonMain 可共用 Koin）或手動 wiring。
- `@ui_ux_engineering`：Android Compose 細節；CMP 大量 API 共用但平台差異需注意。
- `@platform_modernization_2026`：Kotlin 2.1 / KSP2 的 Android 升級；本 skill 同步 KMP 端。
- `@tech_stack_migration`：把 Android-only 模組重構成 KMP 模組的步驟。

## Quick Checklist

- [ ] 共享邊界文件化，列出共享/不共享模組與理由
- [ ] Kotlin 2.1 + Gradle 9 + Ktor 3.x + SQLDelight 2.x 全升
- [ ] iOS targets：iosX64 + iosArm64 + iosSimulatorArm64 三個
- [ ] XCFramework + Swift Package 發佈管道
- [ ] commonMain 不寫死 `Dispatchers.Main`
- [ ] Shared ViewModel 用 androidx.lifecycle KMP，iOS 端有 cancellation 處理
- [ ] commonTest 覆蓋核心業務 ≥ 80%
- [ ] iOS Interop helper（KMP-NativeCoroutines 或自家）已就位
- [ ] Room → SQLDelight 遷移已評估或完成
- [ ] CMP vs 原生 UI 決策已記錄
