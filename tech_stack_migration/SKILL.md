---
id: tech_stack_migration
name: Tech Stack Migration
description: Android 技術棧遷移權威指南 — View→Compose（含 Fragment）、RxJava→Flow（含 backpressure / timeout 映射）、LiveData→StateFlow、Dagger→Hilt、kapt→KSP2、Accompanist 棄用對照、WorkManager API 演進、測試遷移（TestScheduler→TestDispatcher）
type: skill
---

# Tech Stack Migration

## Instructions

當需要把舊技術棧漸進改為現代等價物時載入。本 skill 是 **Dagger→Hilt 與 kapt→KSP 的唯一權威來源**（DI mastery 與 platform_modernization 都指向這裡，避免講法不一致）。遷移前必先有測試安全網（→ `@testing_legacy_strategies`）。

## When to Use

- Scenario C：舊專案現代化
- View → Compose（含 Fragment → Composable）
- RxJava 全套 → Coroutines/Flow
- LiveData → StateFlow
- Dagger 2 → Hilt
- kapt → KSP2
- 用 Accompanist 的功能想換到主 Compose API
- WorkManager Listenable→Coroutine、RxJava 版改 Coroutines 版

## When NOT to Use

- Hilt 已就位後的進階使用 → `@dependency_injection_mastery`
- KMP 抽取（要把 Android 模組改 commonMain）→ `@kotlin_multiplatform`
- 平台版本升級（targetSdk、Gradle、Kotlin 主版本）→ `@platform_modernization_2026`
- 為舊代碼補測試 → `@testing_legacy_strategies`

## Example Prompts

- 「Fragment 改 Composable，complex navigation 怎麼處理？」
- 「我們用了 RxJava2 大量 backpressure，改 Flow 怎麼映射？」
- 「LiveData 過半 ViewModel 想統一改 StateFlow，有哪些坑？」
- 「Dagger 2 巨 graph 怎麼漸進改 Hilt 不爆掉？」
- 「Accompanist Pager / Permissions / SystemUiController 都棄用了，要換什麼？」

## Workflow

1. **建測試安全網**：先載 `@testing_legacy_strategies` 加 characterization tests + visual regression。
2. **設定回歸指標**：APK size、cold start、jank、crash-free 等基準。
3. **單一維度遷移**：一次只動一個技術（UI 或 Stream 或 DI）。
4. **可驗證 PR**：每個 PR ≤ 1 螢幕或 1 ViewModel 規模。
5. **指標驗收**：每次 merge 後對比基準，超出閾值停。

## Practical Notes (2026-04)

| 從 | 到 | 推薦時機 | 注意 |
|---|---|----------|------|
| View XML | Compose | 全部 | 用 `ComposeView` 漸進嵌入 |
| Fragment | Composable + Navigation 3 | 大多場景 | 複雜 lifecycle 維持 Fragment 殼 |
| RxJava 2/3 | Coroutines + Flow | 全部 | 注意 backpressure / cancellation |
| LiveData | StateFlow / SharedFlow | 全部 | UI 端用 `collectAsStateWithLifecycle` |
| Dagger 2 | Hilt 2.52+ | 全部 | API 介面層先共存，逐 module 遷移 |
| kapt | KSP 2.x | 全部 | DataBinding 仍綁 kapt → 棄用 |
| Accompanist Pager | `androidx.compose.foundation.pager` | 立即 | 已主線化 |
| Accompanist SystemUiController | `enableEdgeToEdge()` + WindowInsets | 立即 | 配 `@platform_modernization_2026` |
| Accompanist Permissions | `androidx.activity.compose.rememberLauncher...` 或 Accompanist 仍維護版 | 視情況 | |
| WorkManager RxWorker | CoroutineWorker | 全部 | |

## Minimal Template

```
遷移範圍: <例：Fragment X 換 Composable>
測試安全網: characterization 5 cases + screenshot 3 states
回歸指標: cold start ms, jank %, crash-free
PR 拆分: 1 file ≈ 1 PR
驗收: Quick Checklist
```

## View → Compose

### Compose 嵌入 XML（過渡期）

```kotlin
class LegacyActivity : AppCompatActivity() {
    override fun onCreate(s: Bundle?) {
        super.onCreate(s)
        setContentView(R.layout.activity_legacy)
        findViewById<ComposeView>(R.id.compose_view).apply {
            setViewCompositionStrategy(ViewCompositionStrategy.DisposeOnLifecycleDestroyed(lifecycle))
            setContent { AppTheme { NewFeatureCard(viewModel = ...) } }
        }
    }
}
```

`setViewCompositionStrategy` 必設，否則 Composition 不會在 Activity 銷毀時清理。

### XML 嵌入 Compose

```kotlin
@Composable
fun MapView(modifier: Modifier = Modifier) {
    AndroidView(
        modifier = modifier,
        factory = { ctx -> MapView(ctx).apply { onCreate(null) } },
        update = { mv -> mv.getMapAsync { /* config */ } },
        onRelease = { mv -> mv.onDestroy() },
    )
}
```

`onRelease` **必填**，否則 leak。

### Fragment → Composable 完整步驟

| 步驟 | 動作 |
|------|------|
| 1 | Fragment 內 ViewModel 改用 `viewModel()` 並暴露 `StateFlow` |
| 2 | 抽出 UI 為 `@Composable` 接 state + callbacks |
| 3 | Fragment 改用 `ComposeView` 容器只渲染上述 Composable |
| 4 | 確認所有 Fragment lifecycle 行為（`onResume` 計時、`onPause` 暫停動畫）已搬到 Composable 的 `LaunchedEffect` / `DisposableEffect` |
| 5 | 上層 NavGraph 改 Compose Navigation，把 Fragment 殼移除 |
| 6 | 驗證 back stack、Activity result、結果回傳 |

`Activity Result API`：

```kotlin
val launcher = rememberLauncherForActivityResult(ActivityResultContracts.PickVisualMedia()) { uri ->
    viewModel.onMediaPicked(uri)
}
```

替代 `Fragment.startActivityForResult`。

## RxJava → Flow

### Operator 完整對照

| RxJava | Coroutines / Flow | 備註 |
|--------|-------------------|------|
| `Observable<T>` | `Flow<T>` | cold |
| `Single<T>` | `suspend fun (): T` | 單值 |
| `Maybe<T>` | `suspend fun (): T?` | 可空 |
| `Completable` | `suspend fun (): Unit` | 無回傳 |
| `Flowable<T>` + Backpressure | `Flow<T>` + `buffer / conflate / collectLatest` | 見下 |
| `flatMap` | `flatMapMerge` | 並行 |
| `concatMap` | `flatMapConcat` | 依序 |
| `switchMap` | `flatMapLatest` | 取消舊 |
| `combineLatest` | `combine` | |
| `zip` | `zip` | |
| `merge` | `merge` | |
| `debounce` | `debounce` | |
| `throttleFirst` | `sample` 或自寫 | |
| `subscribeOn(io)` | `flowOn(IO)` | 影響上游 |
| `observeOn(main)` | `.flowOn(Main)` 之後或 `collect` 在 main scope | 位置不同 |
| `retry(3)` | `retry(3) { it is IOException }` | |
| `timeout(5s)` | `withTimeout(5_000)` 或 `timeout(5.seconds)`（Flow 1.7+） | |
| `onErrorReturn` | `catch { emit(default) }` | |
| `Disposable.dispose()` | `Job.cancel()` / scope 結束自清 | |

### Backpressure 映射

```kotlin
// RxJava: BackpressureStrategy.LATEST
upstream.toFlowable(BackpressureStrategy.LATEST)
    .observeOn(...).subscribe { ... }

// Flow 等價：collectLatest 或 conflate
upstream.conflate().collect { ... }       // 跳過中間值，只處理最新
upstream.buffer(64).collect { ... }       // 有限 buffer
upstream.collectLatest { value ->         // 上游新值來，取消當前 collect
    longRunning(value)
}
```

### 範例：Search debounce

```kotlin
// Before（RxJava）
RxTextView.textChanges(searchEt)
    .debounce(300, TimeUnit.MILLISECONDS)
    .switchMap { q -> api.search(q.toString()).toObservable() }
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe { results -> updateUi(results) }

// After（Flow）
queryFlow
    .debounce(300.milliseconds)
    .flatMapLatest { q -> flow { emit(api.search(q)) } }
    .flowOn(Dispatchers.IO)
    .collect { results -> updateUi(results) }
```

### Test 遷移：TestScheduler → TestDispatcher

```kotlin
// Before
val scheduler = TestScheduler()
upstream.subscribeOn(scheduler).test()
scheduler.advanceTimeBy(300, TimeUnit.MILLISECONDS)

// After
@OptIn(ExperimentalCoroutinesApi::class)
@Test fun search() = runTest {
    val emissions = mutableListOf<List<Result>>()
    val job = launch { searchFlow.toList(emissions) }
    advanceTimeBy(300)
    job.cancel()
    assertEquals(expected, emissions.last())
}
```

`runTest` 自動使用 `TestDispatcher`，`advanceTimeBy` / `runCurrent` 控時。

## LiveData → StateFlow

### 步驟

```kotlin
// Step 1: ViewModel 內部加 StateFlow，舊 LiveData 改成投影
class MyViewModel : ViewModel() {
    private val _state = MutableStateFlow(UiState())
    val state: StateFlow<UiState> = _state.asStateFlow()

    @Deprecated("use state", ReplaceWith("state"))
    val stateLive: LiveData<UiState> = state.asLiveData(viewModelScope.coroutineContext)
}

// Step 2: 新 UI（Compose）直接 collect
@Composable
fun Screen(vm: MyViewModel = hiltViewModel()) {
    val s by vm.state.collectAsStateWithLifecycle()
}

// Step 3: 舊 Fragment 觀察 LiveData → 改觀察 StateFlow
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) { vm.state.collect { render(it) } }
}

// Step 4: 移除 LiveData 投影
```

**坑**：`MediatorLiveData` / `Transformations.switchMap` 改 `combine` / `flatMapLatest`；`LiveData.observeForever` 在 Flow 沒有等價物，請改成 explicit `MutableSharedFlow(replay = 1)`。

## Dagger → Hilt（唯一權威）

### 共存策略

```kotlin
// 1. App class 加 @HiltAndroidApp，保留舊 Dagger Component 為 lazy
@HiltAndroidApp
class MyApp : Application() {
    val legacy: LegacyComponent by lazy { DaggerLegacyComponent.create() }
}

// 2. 新模組純 Hilt
@Module @InstallIn(SingletonComponent::class)
object NewModule {
    @Provides @Singleton fun provideX(): X = X()
}

// 3. 橋接：用 @Provides 把 Dagger 提供的服務曝給 Hilt
@Module @InstallIn(SingletonComponent::class)
object LegacyBridgeModule {
    @Provides fun provideLegacyService(@ApplicationContext ctx: Context): LegacyService =
        (ctx.applicationContext as MyApp).legacy.legacyService()
}
```

### 漸進步驟

| 階段 | 動作 |
|------|------|
| 1 | App 加 `@HiltAndroidApp`、Activity 加 `@AndroidEntryPoint` |
| 2 | 全部 build 用 KSP（`@platform_modernization_2026` 已說明 kapt→KSP） |
| 3 | 共用基礎服務（Network、DB、Analytics）先以 LegacyBridge 暴給 Hilt |
| 4 | 一次處理一個 feature module：把該 module 的 Dagger Subcomponent 改 Hilt Module |
| 5 | 所有 feature 遷完後，移除 LegacyBridge 與 Dagger Component |

### 常見錯誤

- **`@AssistedInject` 與 Dagger 並存**：必須兩端都用 Hilt 版的 AssistedInject。
- **`@Subcomponent` 直接搬**：Hilt 用 `@DefineComponent` + `@DefineComponent.Builder`，不是 1:1 對應。
- **Custom Scope 衝突**：Hilt 內建 scope 已涵蓋 99% 場景；若舊 `@PerActivity` 想保留，定義 Custom Component（見 `@dependency_injection_mastery`）。

## kapt → KSP2

完整步驟在 `@platform_modernization_2026` 的 §「kapt → KSP2 遷移」。本 skill 補充：

- **DataBinding** 仍依 kapt（已棄用）→ 改 ViewBinding 或 Compose。
- **Glide kapt-codegen** → 改 `glide-ksp` 或改用 Coil 3（KMP-ready）。
- **Moshi @JsonClass codegen** → moshi-ksp。

## Accompanist 棄用對照表

| Accompanist | 官方替代 |
|-------------|----------|
| `accompanist-pager` / `pager-indicators` | `androidx.compose.foundation.pager.HorizontalPager / VerticalPager` |
| `accompanist-systemuicontroller` | `enableEdgeToEdge()` + `WindowCompat.getInsetsController` |
| `accompanist-flowlayout` | `androidx.compose.foundation.layout.FlowRow / FlowColumn` |
| `accompanist-swiperefresh` | `material3.pulltorefresh.PullToRefreshBox` |
| `accompanist-placeholder` | 自寫或 `Modifier.placeholder()`（社群） |
| `accompanist-navigation-animation` | Navigation Compose 內建 transitions |
| `accompanist-permissions` | 仍維護，可繼續用；或 `rememberLauncherForActivityResult` |
| `accompanist-insets` | `WindowInsets` 與 `windowInsetsPadding` |

## WorkManager API 演進

```kotlin
// 舊 RxWorker
class MyRxWorker(ctx: Context, params: WorkerParameters) : RxWorker(ctx, params) {
    override fun createWork(): Single<Result> = api.sync()
        .map { Result.success() }
        .onErrorReturn { Result.retry() }
}

// 新 CoroutineWorker
@HiltWorker
class MyWorker @AssistedInject constructor(
    @Assisted ctx: Context, @Assisted params: WorkerParameters,
    private val repo: Repo,
) : CoroutineWorker(ctx, params) {
    override suspend fun doWork(): Result = runCatching { repo.sync() }
        .map { Result.success() }
        .getOrElse { Result.retry() }
}
```

`ListenableWorker` callback 風格也可改用 `await()` extension。

## Cross-Skill References

- `@testing_legacy_strategies`：遷移前的安全網。
- `@dependency_injection_mastery`：Hilt 已就位後的進階用法。
- `@platform_modernization_2026`：Kotlin/Gradle/AGP/KSP 主版本升級。
- `@kotlin_multiplatform`：把 Android-only 模組改 commonMain 的後續。
- `@deep_performance_tuning`：遷移後的回歸量測（cold start、jank）。

## Quick Checklist

- [ ] 遷移前已加 characterization test + screenshot test
- [ ] 設定回歸指標基線（cold start、APK size、jank、crash-free）
- [ ] 一次只動一個維度（UI / Stream / DI）
- [ ] `ComposeView` 設 `setViewCompositionStrategy`
- [ ] `AndroidView` 含 `onRelease` 釋放資源
- [ ] Flow 取代 RxJava 後 backpressure 行為驗證（conflate / collectLatest）
- [ ] StateFlow 用 `collectAsStateWithLifecycle` 觀察
- [ ] Dagger 與 Hilt 共存階段有明確收斂時間表
- [ ] kapt 全數移除，改 KSP 2.x（DataBinding 已棄用或改 Compose）
- [ ] Accompanist 棄用元件已換主線 API
- [ ] 遷移後跑 Macrobenchmark，無 > 5% 退化
