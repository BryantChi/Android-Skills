---
id: deep_performance_tuning
name: Deep Performance Tuning
description: Android 效能定位與優化（數據驅動） — Macrobenchmark / Baseline Profiles / Startup Profiles、Perfetto trace、JankStats 量化門檻、Memory Profiler / Native Profiler / Flame Graph、R8 優化、Play Console App Quality Insights，產出可進 CI gate 的效能基線
type: skill
---

# Deep Performance Tuning

## Instructions

當有量測數據（Macrobenchmark、Perfetto trace、JankStats、Play Console Vitals）或明確使用者抱怨時載入。**沒有數據前不做主觀調整**。本 skill 處理「定位與優化」；指標策略與告警設計由 `@observability_strategy` 負責。

## When to Use

- Scenario D：效能問題排查
- Scenario E：發布前效能驗證
- Cold start 退化 / Jank rate 上升
- Memory leak 與 OOM 排查
- APK 過大 / R8 規則調整
- 從 Play Console App Quality Insights 拿到具體 trace 要分析

## When NOT to Use

- 設計監控指標、告警閾值、event schema → `@observability_strategy`
- Crashlytics SDK 與 ANR Watchdog 安裝 → `@crash_monitoring`
- 平台升級造成的 Configuration Cache 修復 → `@release_automation`
- 直觀「我覺得卡」沒有量測數據 → 先去 `@observability_strategy` 設指標再回來

## Example Prompts

- 「Cold start 從 1.2s 變 1.8s，怎麼定位？」
- 「Macrobenchmark 怎麼測 Baseline Profile 的效益？」
- 「Compose LazyColumn 滑動 jank，怎麼拆 trace？」
- 「APK 從 32MB 漲到 45MB，R8 規則怎麼查？」
- 「Play Console App Quality Insights 看到一條 Slow rendering，怎麼下載 trace 分析？」

## Workflow

1. **Reproduce + Measure**：用 Macrobenchmark 復現並量化基線。
2. **Locate**：Perfetto trace（system + app slices）或 Profiler（Memory / Native）找熱區。
3. **Hypothesize**：列出最可能的 1-3 個原因，每個有 trace 證據。
4. **Fix one**：一次只改一處，重跑 Macrobenchmark 對比。
5. **Lock in**：通過後把該 metric 寫入 CI gate 阻擋退化。

## Practical Notes (2026-04)

| 工具 | 版本 / 推薦 | 用途 |
|------|------------|------|
| Macrobenchmark | androidx.benchmark:macro 1.3+ | 自動化 cold/warm/hot start、frame metrics |
| Baseline Profile Gradle Plugin | 1.3+ | 生成與打包 baseline-prof.txt |
| Startup Profiles | 1.3+ | 啟動專屬 profile，更小更精準 |
| Perfetto | 內建 chrome://tracing 或 ui.perfetto.dev | system + app trace 視覺化 |
| Android Studio Profiler | Ladybug+ | Memory / CPU / Energy |
| Native Profiler（NDK） | studio 內建 | Flame Graph |
| LeakCanary | 2.14+ | debug-only 自動偵測洩漏 |
| JankStats | 1.0+ | 量化 jank frame |
| Play Console App Quality Insights | 持續更新 | 真實裝置 trace 來源 |
| R8 | AGP 8.7+ 內建 | full mode 預設 |

## Minimal Template

```
量測基準: cold start P95, jank %, trace samples
Macrobenchmark targets:
  - StartupTimingMetric (cold/warm/hot)
  - FrameTimingMetric (P50/P95/P99)
  - MemoryUsageMetric
Baseline Profile: 必生成
Startup Profile: 啟動關鍵路徑專屬
CI Gate: 退化 > 5% 阻擋
驗收: Quick Checklist
```

## App Startup Optimization

### Macrobenchmark 設定

```kotlin
// :benchmark 模組（com.android.test）
@LargeTest
@RunWith(AndroidJUnit4::class)
class StartupBenchmark {
    @get:Rule val rule = MacrobenchmarkRule()

    @Test fun startupCold_none() = startup(CompilationMode.None())
    @Test fun startupCold_partial() = startup(CompilationMode.Partial())
    @Test fun startupCold_full() = startup(CompilationMode.Full())

    private fun startup(mode: CompilationMode) {
        rule.measureRepeated(
            packageName = "com.example.app",
            metrics = listOf(StartupTimingMetric()),
            compilationMode = mode,
            iterations = 10,
            startupMode = StartupMode.COLD,
        ) {
            pressHome()
            startActivityAndWait()
        }
    }
}
```

```bash
./gradlew :benchmark:connectedReleaseAndroidTest
# 結果：build/outputs/connected_android_test_additional_output/.../*.json
```

### Baseline Profile

```kotlin
// :baseline-profile 模組
@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {
    @get:Rule val rule = BaselineProfileRule()

    @Test fun generate() = rule.collect(packageName = "com.example.app") {
        pressHome()
        startActivityAndWait()
        device.findObject(By.text("Home")).waitForExists(5_000)
        device.findObject(By.res("home_recycler_view"))?.fling(Direction.DOWN)
        device.findObject(By.text("Detail"))?.click()
        device.wait(Until.hasObject(By.res("detail_image")), 5_000)
    }
}
```

```bash
./gradlew :app:generateReleaseBaselineProfile
# 寫入 app/src/release/generated/baselineProfiles/baseline-prof.txt
```

效益量測：跑兩次 Macrobenchmark（含/不含 Baseline Profile）對比 P50/P95。

### Startup Profile（2024+ 新功能）

```kotlin
// 同 BaselineProfileRule，但加 includeInStartupProfile = true
rule.collect(
    packageName = "com.example.app",
    includeInStartupProfile = true,    // 啟動專屬 profile
) {
    pressHome()
    startActivityAndWait()
    // 只蒐集到首屏 idle，不要走深
}
```

Startup Profile 比 Baseline Profile 更小、更精準針對 cold start，可獨立打包進 dex。

### `Application.onCreate` 優化

```kotlin
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        // ❌ 同步、阻塞 cold start
        // FirebaseApp.initializeApp(this); WorkManager.initialize(this, config)

        // ✅ 立即必要的最小集合
        AndroidThreeTen.init(this)

        // ✅ 用 App Startup（androidx.startup） + 延遲
        AppInitializer.getInstance(this).initializeComponent(FirebaseInitializer::class.java)

        // ✅ 後景化的非關鍵
        ProcessLifecycleOwner.get().lifecycle.addObserver(object : DefaultLifecycleObserver {
            override fun onStart(owner: LifecycleOwner) {
                ProcessLifecycleOwner.get().lifecycleScope.launch(Dispatchers.Default) {
                    initAnalytics()
                    initNonCriticalFeatureFlags()
                }
            }
        })
    }
}
```

ContentProvider 是 cold start 殺手；用 androidx.startup 的 `tools:node="merge"` 抹除多餘 ContentProvider。

## Perfetto Trace 流程（取代 systrace）

### 抓 trace

```bash
# 命令列 trace 30 秒
adb shell perfetto -o /data/misc/perfetto-traces/trace.pftrace -t 30s \
  -c - --txt <<'EOF'
buffers: { size_kb: 65536 fill_policy: DISCARD }
data_sources: { config { name: "linux.ftrace" ftrace_config {
  ftrace_events: "sched/sched_switch"
  ftrace_events: "power/cpu_frequency"
  atrace_categories: "view"
  atrace_categories: "wm"
  atrace_apps: "com.example.app"
} } }
data_sources: { config { name: "android.surfaceflinger.frametimeline" } }
duration_ms: 30000
EOF

adb pull /data/misc/perfetto-traces/trace.pftrace
# 上傳到 https://ui.perfetto.dev
```

或在 App 內：

```kotlin
trace("CheckoutFlow") {
    // 區塊內所有運算會出現在 Perfetto 的 app slice
}
```

`androidx.tracing:tracing-perfetto` 可在 release 開啟細粒度 trace。

### 讀 trace 重點

- **Critical path**：focal point 是 main thread；旁邊的執行緒只是輔助。
- **Frame deadline**：Surface Flinger frame timeline 顯示每幀 deadline；超過即 jank。
- **Binder calls**：跨進程呼叫常被忽略，trace 上明顯。
- **GC pauses**：黃色長條；超過 16ms 即影響流暢度。

## JankStats 量化門檻

```kotlin
class MainActivity : ComponentActivity() {
    private lateinit var jankStats: JankStats

    override fun onResume() {
        super.onResume()
        jankStats = JankStats.createAndTrack(window) { frame ->
            if (frame.isJank) {
                analytics.logEvent("jank_frame") {
                    param("duration_ms", (frame.frameDurationUiNanos / 1_000_000).toLong())
                    param("state", currentScreenName)
                }
            }
        }
    }
    override fun onPause() { jankStats.isTrackingEnabled = false; super.onPause() }
}
```

### 量化目標

| 指標 | 目標 |
|------|------|
| Jank frame ratio | < 5%（90Hz 以下）/ < 1%（120Hz） |
| P95 frame duration | < frame deadline（16.7ms / 11.1ms / 8.3ms） |
| Janky scroll session 比例 | < 10% |

不達標的畫面 → Perfetto trace 找根因（recomposition、layout 過深、LazyColumn item 重）。

## Memory Analysis

### LeakCanary

```kotlin
// debug only：app/build.gradle.kts
debugImplementation("com.squareup.leakcanary:leakcanary-android:2.14")
```

CI 端啟動 instrumentation tests 時 fail-on-leak：

```kotlin
LeakCanary.config = LeakCanary.config.copy(
    onHeapAnalyzedListener = { result ->
        if (result.analysisDurationMillis > 0 && result.heapAnalysis is HeapAnalysisSuccess) {
            val leaks = (result.heapAnalysis as HeapAnalysisSuccess).applicationLeaks
            check(leaks.isEmpty()) { "Memory leak detected: $leaks" }
        }
    }
)
```

### Heap Dump

```bash
adb shell am dumpheap com.example.app /data/local/tmp/heap.hprof
adb pull /data/local/tmp/heap.hprof
# Android Studio Profiler → Open .hprof
```

### Native Profiler / Flame Graph

```bash
# 啟動 Native Profiler（需 NDK）
# Android Studio → Profile → CPU → Native Memory Allocations
# 或 simpleperf
adb shell simpleperf record -p $(adb shell pidof com.example.app) -g --duration 10
adb shell simpleperf report-html -i /data/local/tmp/perf.data
```

### Bitmap 記憶體

```kotlin
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data(url)
        .size(Size(360, 180))                  // 明確尺寸
        .scale(Scale.FILL)
        .memoryCachePolicy(CachePolicy.ENABLED)
        .build(),
    contentDescription = null,
)
```

不要 `Size.ORIGINAL` 載入巨圖。Coil 3 內建 hardware bitmap 與多進程 cache。

## R8 / ProGuard

### Full Mode（AGP 8.x 預設）

```properties
# gradle.properties
android.enableR8.fullMode=true
```

### 規則範例

```pro
# proguard-rules.pro
-keepattributes SourceFile,LineNumberTable
-renamesourcefileattribute SourceFile

# Kotlinx Serialization
-keepattributes *Annotation*, InnerClasses
-dontnote kotlinx.serialization.AnnotationsKt
-keep,includedescriptorclasses class com.example.**$$serializer { *; }
-keepclassmembers class com.example.** { *** Companion; }
-keepclasseswithmembers class com.example.** { kotlinx.serialization.KSerializer serializer(...); }

# Retrofit
-keepattributes Signature, Exceptions
-keep,allowobfuscation,allowshrinking interface retrofit2.Call
-keep,allowobfuscation,allowshrinking class retrofit2.Response

# Compose runtime（Stability annotations）
-keep class * { @androidx.compose.runtime.Stable *; @androidx.compose.runtime.Immutable *; }
```

### APK Size 分析

```bash
bundletool build-apks --bundle=app.aab --output=app.apks
bundletool get-size total --apks=app.apks
# Android Studio: Build > Analyze APK / 對比兩個 APK 看 diff
```

## Compose Recomposition 分析

啟用 Composer Metrics（細節在 `@coding_style_conventions`）：

```bash
ls app/build/compose-reports/
# *-composables.txt   每個 composable 的 stability
# *-composables.csv   方便 grep
# *-classes.txt       資料類 stability
```

熱區排查：

```bash
# 找重組最多的 composable
grep -E "restartable.*scheme.*UiComposable" app/build/compose-reports/*-composables.txt | head -20

# 用 Layout Inspector 即時看 Recomposition counts
# Android Studio → Tools → Layout Inspector
```

修法：標 stable / immutable / 換 ImmutableList / Strong Skipping（`@coding_style_conventions` + `@ui_ux_engineering`）。

## Play Console App Quality Insights 整合

Play Console → Android Vitals → Performance → 可下載「真實裝置 Perfetto trace」。

工作流：

1. 從 Play Console 下載 `.perfetto-trace`。
2. 上傳到 `ui.perfetto.dev`。
3. 對照本地 Macrobenchmark trace，看 main thread 是否同樣熱區。
4. 真實裝置上的瓶頸寫入 Macrobenchmark 用例，避免 regression。

## CI Gate

```yaml
# benchmark.yml（簡化）
jobs:
  bench:
    runs-on: ubuntu-24.04-large
    steps:
      - run: ./gradlew :benchmark:connectedReleaseAndroidTest
      - name: Compare with baseline
        run: |
          python tools/compare_bench.py \
            --baseline benchmark/baseline.json \
            --current build/.../bench.json \
            --threshold 5
```

`benchmark/baseline.json` 提交到 repo；超 5% 退化阻擋 merge。每季更新一次基線。

## Cross-Skill References

- `@observability_strategy`：SLO / 告警閾值 / event schema 設計；本 skill 提供量化基準餵給它。
- `@crash_monitoring`：ANR / Memory Warning 信號上報。
- `@coding_style_conventions`：Compose Compiler Metrics 與 stability 規則。
- `@ui_ux_engineering`：unstable composable 修法（ImmutableList、@Stable）。
- `@release_automation`：Macrobenchmark 接 CI gate；本 skill 設計 metric。
- `@platform_modernization_2026`：升級後 Baseline Profile 須重生。

## Quick Checklist

- [ ] Macrobenchmark 模組存在，覆蓋 cold/warm startup + 關鍵 frame timing
- [ ] Baseline Profile 已生成並打包進 release APK
- [ ] Startup Profile 啟用，cold start P95 改善 ≥ 20%
- [ ] Perfetto trace 為主要定位工具（取代 systrace）
- [ ] JankStats 已接，jank frame ratio < 5%
- [ ] LeakCanary 在 debug + instrumentation 開啟，CI fail-on-leak
- [ ] R8 full mode 啟用，proguard rules 含序列化/反射保護
- [ ] APK size diff 有 baseline 監控
- [ ] Compose Compiler Metrics 監看 unstable composable 數
- [ ] Play Console Vitals 真實 trace 流程已建立
- [ ] CI 退化 > 5% 阻擋 merge
