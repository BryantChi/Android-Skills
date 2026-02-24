---
id: deep_performance_tuning
name: Deep Performance Tuning
description: Systrace, Memory Analysis, R8 優化與 App Startup 調校
---

# Deep Performance Tuning (深度效能優化)

## Instructions
- 僅在有量測數據或明確瓶頸時使用
- 依照下方章節順序套用
- 一次只施作一種優化並驗證效果
- 完成後對照 Quick Checklist

## When to Use
- Scenario D：效能問題排查
- Scenario E：發布前效能驗證

## Example Prompts
- "請參考 App Startup Optimization，建立 Macrobenchmark 測量"
- "用 Memory Analysis 章節，規劃記憶體分析流程"
- "請依照 R8/Proguard Optimization，檢查規則是否完整"

## Workflow
1. 先建立量測基準（Startup/Memory/UI）
2. 再逐一套用對應優化手段
3. 最後用 Quick Checklist 驗收

## Practical Notes (2026)
- Baseline Profile + Macrobenchmark 作為預設流程
- 每次只做一項優化並量測差異
- 效能門檻要進 CI Gate

## Minimal Template
```
目標: 
量測基準: 
優化範圍: 
回歸指標: 
驗收: Quick Checklist
```

---

## App Startup Optimization

### Macrobenchmark 測量

```kotlin
@LargeTest
@RunWith(AndroidJUnit4::class)
class StartupBenchmark {
    
    @get:Rule
    val benchmarkRule = MacrobenchmarkRule()
    
    @Test
    fun startupCompilationNone() = startup(CompilationMode.None())
    
    @Test
    fun startupCompilationPartial() = startup(CompilationMode.Partial())
    
    private fun startup(compilationMode: CompilationMode) {
        benchmarkRule.measureRepeated(
            packageName = "com.example.app",
            metrics = listOf(StartupTimingMetric()),
            compilationMode = compilationMode,
            iterations = 5,
            startupMode = StartupMode.COLD
        ) {
            pressHome()
            startActivityAndWait()
        }
    }
}
```

### Baseline Profiles

```kotlin
// 生成 Baseline Profile
@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {
    
    @get:Rule
    val rule = BaselineProfileRule()
    
    @Test
    fun generate() {
        rule.collect("com.example.app") {
            pressHome()
            startActivityAndWait()
            
            // 關鍵路徑
            device.findObject(By.text("Login")).click()
            device.wait(Until.hasObject(By.text("Home")), 5000)
        }
    }
}
```

### Application onCreate 優化

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        
        // ❌ 同步初始化 (Block UI)
        // Firebase.initialize(this)
        // Timber.plant(DebugTree())
        
        // ✅ 延遲初始化
        AppInitializer.getInstance(this)
            .initializeComponent(FirebaseInitializer::class.java)
        
        // ✅ 背景初始化
        ProcessLifecycleOwner.get().lifecycle.addObserver(
            object : DefaultLifecycleObserver {
                override fun onCreate(owner: LifecycleOwner) {
                    lifecycleScope.launch(Dispatchers.Default) {
                        // 非關鍵初始化
                    }
                }
            }
        )
    }
}
```

---

## Memory Analysis

### Heap Dump 分析

```bash
# 取得 Heap Dump
adb shell am dumpheap com.example.app /data/local/tmp/heap.hprof
adb pull /data/local/tmp/heap.hprof

# 使用 Android Studio Profiler 分析
# 或使用 MAT (Memory Analyzer Tool)
```

### LeakCanary 設定

```kotlin
// build.gradle.kts
debugImplementation("com.squareup.leakcanary:leakcanary-android:2.12")

// 自定義報告
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        LeakCanary.config = LeakCanary.config.copy(
            retainedVisibleThreshold = 3,
            objectInspectors = AndroidObjectInspectors.appDefaults
        )
    }
}
```

### Bitmap 記憶體管理

```kotlin
// 使用 Coil 自動管理
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data(imageUrl)
        .size(Size.ORIGINAL)  // 根據需求調整
        .memoryCachePolicy(CachePolicy.ENABLED)
        .build(),
    contentDescription = null
)
```

---

## R8/Proguard Optimization

### 自定義規則

```proguard
# 保留 Serializable
-keepclassmembers class * implements java.io.Serializable {
    static final long serialVersionUID;
    private static final java.io.ObjectStreamField[] serialPersistentFields;
    !static !transient <fields>;
    private void writeObject(java.io.ObjectOutputStream);
    private void readObject(java.io.ObjectInputStream);
}

# 保留 Retrofit Interface
-keep,allowobfuscation interface * {
    @retrofit2.http.* <methods>;
}

# 保留 Compose stability
-keep class * {
    @androidx.compose.runtime.Stable *;
    @androidx.compose.runtime.Immutable *;
}
```

### APK Size 分析

```bash
# 使用 bundletool
bundletool build-apks --bundle=app.aab --output=app.apks
bundletool get-size total --apks=app.apks

# Android Studio: Build > Analyze APK
```

---

## UI Performance

### JankStats

```kotlin
class MainActivity : ComponentActivity() {
    private lateinit var jankStats: JankStats
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        jankStats = JankStats.createAndTrack(window) { frameData ->
            if (frameData.isJank) {
                Log.w("Jank", "Frame took ${frameData.frameDurationUiNanos / 1_000_000}ms")
            }
        }
    }
}
```

### Compose Recomposition Tracking

```kotlin
// 開啟 Composition Tracking
@Composable
fun DebugComposable() {
    SideEffect {
        Log.d("Recomposition", "DebugComposable recomposed")
    }
}

// 使用 Layout Inspector 檢查 recomposition count
```

---

## Quick Checklist

- [ ] Startup: Baseline Profiles 生成
- [ ] Startup: Application onCreate 延遲初始化
- [ ] Memory: LeakCanary 無洩漏
- [ ] APK: R8 規則完善
- [ ] UI: JankStats 無嚴重掉幀
