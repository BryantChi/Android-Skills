---
name: Tech Stack Migration
description: View→Compose, RxJava→Flow 等技術遷移指南
---

# Tech Stack Migration (技術遷移)

## Instructions
- 僅在需要漸進式遷移時使用
- 依照下方章節順序套用
- 一次只替換一個技術棧
- 完成後對照 Quick Checklist

## When to Use
- Scenario C：舊專案現代化

## Example Prompts
- "請依照 View → Compose 章節，幫我嵌入 ComposeView"
- "用 RxJava → Flow 對照表，改寫這段 stream"
- "請用 LiveData → StateFlow 的步驟規劃遷移"

## Workflow
1. 先確認要遷移的技術棧與範圍
2. 依序套用對應章節的範例與對照表
3. 最後用 Quick Checklist 驗收

## Practical Notes (2026)
- 遷移必先有可驗證的測試安全網
- 一次只遷移一個維度（UI 或 Data 或 DI）
- 指標回歸後才能推進下一步

## Minimal Template
```
目標: 
遷移範圍: 
測試安全網: 
回歸指標: 
驗收: Quick Checklist
```

---

## View → Compose Interoperability

### Compose in XML

```xml
<!-- layout/activity_main.xml -->
<androidx.compose.ui.platform.ComposeView
    android:id="@+id/compose_view"
    android:layout_width="match_parent"
    android:layout_height="wrap_content" />
```

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        findViewById<ComposeView>(R.id.compose_view).setContent {
            AppTheme {
                NewFeatureCard()
            }
        }
    }
}
```

### XML in Compose

```kotlin
@Composable
fun LegacyMapView(modifier: Modifier = Modifier) {
    AndroidView(
        modifier = modifier,
        factory = { context ->
            MapView(context).apply {
                onCreate(null)
            }
        },
        update = { mapView ->
            mapView.getMapAsync { /* configure */ }
        },
        onRelease = { mapView ->
            mapView.onDestroy()
        }
    )
}
```

### Fragment in Compose

```kotlin
@Composable
fun LegacyFragmentContainer() {
    AndroidViewBinding(LegacyFragmentContainerBinding::inflate) {
        val fragment = LegacyFragment()
        fragmentContainerView.getFragment<Fragment>() ?: run {
            (LocalContext.current as FragmentActivity)
                .supportFragmentManager
                .beginTransaction()
                .replace(fragmentContainerView.id, fragment)
                .commit()
        }
    }
}
```

---

## RxJava → Coroutines/Flow

### Operator Mapping

| RxJava | Coroutines/Flow | 備註 |
|--------|-----------------|------|
| `Observable` | `Flow` | Cold stream |
| `Single` | `suspend fun` | 單值 |
| `Completable` | `suspend fun` | 無回傳 |
| `flatMap` | `flatMapLatest` | 取消前一個 |
| `flatMap` | `flatMapConcat` | 依序執行 |
| `switchMap` | `flatMapLatest` | 等同 |
| `debounce` | `debounce` | 相同 |
| `combineLatest` | `combine` | 相同 |
| `zip` | `zip` | 相同 |
| `observeOn(main)` | `flowOn(Dispatchers.Main)` | 注意位置不同 |
| `subscribeOn(io)` | `flowOn(Dispatchers.IO)` | 影響上游 |

### 範例：Search with Debounce

```kotlin
// Before (RxJava)
searchEditText.textChanges()
    .debounce(300, TimeUnit.MILLISECONDS)
    .switchMap { query -> api.search(query) }
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe { results -> updateUI(results) }

// After (Flow)
searchFlow
    .debounce(300)
    .flatMapLatest { query -> api.search(query) }
    .flowOn(Dispatchers.IO)
    .collect { results -> updateUI(results) }
```

### Error Handling 差異

```kotlin
// RxJava: onError 終止 stream
observable
    .onErrorReturn { defaultValue }
    .subscribe()

// Flow: catch 不終止 stream
flow
    .catch { emit(defaultValue) }
    .collect()

// Flow: 重試
flow
    .retry(3) { e -> e is IOException }
    .collect()
```

---

## LiveData → StateFlow

### 漸進式替換

```kotlin
// Step 1: ViewModel 內部用 StateFlow
private val _uiState = MutableStateFlow(UiState())
val uiState: StateFlow<UiState> = _uiState.asStateFlow()

// Step 2: 暴露 LiveData 給舊 UI (過渡期)
val uiStateLiveData: LiveData<UiState> = uiState.asLiveData()

// Step 3: 新 UI 直接 collect StateFlow
@Composable
fun Screen(viewModel: MyViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
}
```

---

## Dagger → Hilt

### 漸進式遷移

```kotlin
// 1. 保留 Dagger Component，加入 Hilt
@HiltAndroidApp
class MyApp : Application() {
    // 保留舊的 Dagger component (過渡期)
    val legacyComponent by lazy { DaggerLegacyComponent.create() }
}

// 2. 新模組用 Hilt
@Module
@InstallIn(SingletonComponent::class)
object NewModule { }

// 3. 橋接舊模組
@Module
@InstallIn(SingletonComponent::class)
object LegacyBridgeModule {
    @Provides
    fun provideLegacyService(): LegacyService {
        return (application as MyApp).legacyComponent.legacyService()
    }
}
```

---

## Quick Checklist

- [ ] Compose 與 View 的生命週期對齊
- [ ] AndroidView 正確處理 onRelease
- [ ] Flow operator 對照 RxJava 正確
- [ ] StateFlow 使用 collectAsStateWithLifecycle
- [ ] Hilt 遷移維持向後相容
