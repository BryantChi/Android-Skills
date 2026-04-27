---
id: navigation_patterns
name: Navigation Patterns
description: Compose Navigation 3.x（2.8 相容）— Type-safe routes、Deep Links / App Links、跨模組 Navigator API、Predictive Back 動畫整合、NavBackStackEntry ViewModel 清理、Navigation Testing，產出無洩漏可測試的導航
type: skill
---

# Navigation Patterns

## Instructions

當問題涉及路由設計、Deep Link、跨模組導航、Predictive Back、Back Stack 行為、Navigation 測試時載入。本 skill 以 Compose Navigation 3.x 為主、2.8 為相容路徑；Adaptive layout（list-detail / supporting pane）由 `@ui_ux_engineering` 接管，本 skill 提供導航接合。

## When to Use

- Scenario A：新專案導航設計
- Scenario B：舊專案擴充與導航橋接
- Predictive Back 強制執行後 layout 與動畫需要重整
- 多模組環境跨 feature 的導航介面設計
- Navigation Testing 與 NavBackStackEntry ViewModel 洩漏排查

## When NOT to Use

- Adaptive layout 視覺設計（List-Detail Pane） → `@ui_ux_engineering`
- Activity → Compose 整體遷移 → `@tech_stack_migration`
- Edge-to-Edge 與 system bar inset → `@platform_modernization_2026` + `@ui_ux_engineering`

## Example Prompts

- 「Navigation 3 出了之後，我該怎麼從 2.8 升上來？」
- 「Deep Link 從 https://example.com/product/123 進來要直接帶 type-safe arg」
- 「Predictive Back 的進度動畫怎麼接？」
- 「跨模組想呼 feature:checkout 但不能直接 import，怎麼做？」
- 「Bottom Nav 切換時 ViewModel 沒被清掉，記憶體一直漲」

## Workflow

1. **路由型別化**：所有路由用 `@Serializable` data class，禁字串路由。
2. **Deep Link 規劃**：先列 URI matrix（domain × path × query），再對應到 route。
3. **跨模組 API**：`core:navigation` 定義 `Navigator` 介面，feature module 提供導航 token / serializable route。
4. **Back Handling**：Compose 一律用 `PredictiveBackHandler`；ViewModel 不持有 NavController。
5. **Testing**：`TestNavHostController` + `composeTestRule`，驗證 route + arguments + back stack。

## Practical Notes (2026-04)

| 元件 | 版本 | 備註 |
|------|------|------|
| Navigation Compose | 3.x（推薦）／ 2.8.x（相容） | 3.x 用 `androidx.navigation3` artifact |
| Compose | BOM 2025.04.x | `PredictiveBackHandler` 已穩定 |
| `enableOnBackInvokedCallback` | 必開 | targetSdk 35 強制 |
| Type-safe routes | `@Serializable` + Kotlinx Serialization | 不用 String 路徑 |
| Navigation Testing | `androidx.navigation:navigation-testing` | 可注入 fake destinations |

## Minimal Template

```
路由清單: <Home, Detail(productId), Checkout, Login>
Deep Link 矩陣:
  - https://example.com/product/{id}  → DetailRoute(id)
  - app://example/checkout            → CheckoutRoute
跨模組導航介面: core:navigation/Navigator.kt
Back Handling: PredictiveBackHandler in screens with custom back
測試: TestNavHostController in androidTest/
驗收: Quick Checklist
```

## Type-safe Routes（Navigation 2.8+ 與 3.x 通用）

```kotlin
@Serializable data object HomeRoute
@Serializable data class DetailRoute(val productId: String, val highlight: Boolean = false)
@Serializable data class CheckoutRoute(val cartId: String)

NavHost(navController, startDestination = HomeRoute) {
    composable<HomeRoute> {
        HomeScreen(onProductClick = { id -> navController.navigate(DetailRoute(id)) })
    }
    composable<DetailRoute> { entry ->
        val route: DetailRoute = entry.toRoute()
        DetailScreen(route)
    }
}
```

**禁止**：`navController.navigate("detail/${id}")` 字串路由，沒有編譯期保證。

## Deep Links / App Links

### Manifest

```xml
<activity android:name=".MainActivity" android:launchMode="singleTop">
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" android:host="example.com" />
        <data android:pathPrefix="/product" />
        <data android:pathPrefix="/checkout" />
    </intent-filter>
</activity>
```

### `assetlinks.json`

```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example.app",
    "sha256_cert_fingerprints": ["AA:BB:..."]
  }
}]
```

驗證：

```bash
adb shell pm get-app-links com.example.app
# Domain verification state: example.com -> verified
```

### 與 type-safe route 結合

```kotlin
composable<DetailRoute>(
    deepLinks = listOf(navDeepLink<DetailRoute>(basePath = "https://example.com/product"))
) { entry ->
    val route: DetailRoute = entry.toRoute()
    DetailScreen(route)
}
```

`navDeepLink<DetailRoute>` 會自動把 `/product/{productId}?highlight={highlight}` 路徑映射到 data class 欄位。

### 進入時驗證

```kotlin
class DetailViewModel @AssistedInject constructor(
    @Assisted private val productId: String,
    private val productRepo: ProductRepository,
) : ViewModel() {
    init {
        require(productId.isNotBlank()) { "Invalid productId from deep link" }
    }
}
```

deep link 進入的參數一律當作 untrusted input 校驗。

## Multi-Module Navigation

### 介面在 `core:navigation`

```kotlin
// core:navigation
@Serializable sealed interface NavigationCommand {
    @Serializable data class ToDetail(val productId: String) : NavigationCommand
    @Serializable data class ToCheckout(val cartId: String) : NavigationCommand
    @Serializable data object Back : NavigationCommand
}

interface Navigator {
    val commands: Flow<NavigationCommand>
    suspend fun navigate(command: NavigationCommand)
}

@Singleton
class NavigatorImpl @Inject constructor() : Navigator {
    private val _commands = MutableSharedFlow<NavigationCommand>(extraBufferCapacity = 8)
    override val commands = _commands.asSharedFlow()
    override suspend fun navigate(command: NavigationCommand) { _commands.emit(command) }
}
```

### feature 端只發 command，**不持有 NavController**

```kotlin
// feature:home
class HomeViewModel @Inject constructor(private val navigator: Navigator) : ViewModel() {
    fun onProductClick(id: String) {
        viewModelScope.launch { navigator.navigate(NavigationCommand.ToDetail(id)) }
    }
}
```

### app 模組單一處 collect

```kotlin
@Composable
fun AppNavHost(navigator: Navigator) {
    val navController = rememberNavController()
    LaunchedEffect(Unit) {
        navigator.commands.collect { cmd ->
            when (cmd) {
                is NavigationCommand.ToDetail -> navController.navigate(DetailRoute(cmd.productId))
                is NavigationCommand.ToCheckout -> navController.navigate(CheckoutRoute(cmd.cartId))
                NavigationCommand.Back -> navController.popBackStack()
            }
        }
    }
    NavHost(navController, startDestination = HomeRoute) { /* ... */ }
}
```

## Predictive Back Animation

`targetSdk 35` 起強制執行。`AndroidManifest.xml`：

```xml
<application android:enableOnBackInvokedCallback="true">
```

### Compose 端

```kotlin
@Composable
fun EditScreen(state: EditState, onBack: () -> Unit) {
    PredictiveBackHandler(enabled = state.hasUnsavedChanges) { progress ->
        try {
            progress.collect { event ->
                // 0.0 → 1.0；可同步動畫進度
                animatableOffset.snapTo(event.progress * screenWidth)
            }
            // 完成
            confirmDiscard()
            onBack()
        } catch (_: CancellationException) {
            // 使用者取消（鬆手返回）
            animatableOffset.animateTo(0f)
        }
    }
}
```

### 系統提供的 shared element back animation

Navigation 3.x 內建 destination 切換的 predictive back transition。客製：

```kotlin
NavHost(
    navController,
    startDestination = HomeRoute,
    popExitTransition = { slideOutHorizontally { it } + fadeOut() },
    popEnterTransition = { slideInHorizontally { -it } + fadeIn() },
)
```

## NavBackStackEntry ViewModel 生命週期

### 常見洩漏：ViewModel 持有 Context / Activity

```kotlin
// ❌ 洩漏
class DetailViewModel(val activity: Activity) : ViewModel()

// ✅ 用 hiltViewModel + savedStateHandle
class DetailViewModel @Inject constructor(savedStateHandle: SavedStateHandle) : ViewModel() {
    private val productId: String = savedStateHandle.toRoute<DetailRoute>().productId
}
```

### Bottom Nav 切換造成 ViewModel 不被清

```kotlin
NavHost(navController, startDestination = HomeRoute) {
    // ❌ 預設 saveState = false 但 popUpTo 設 saveState = true 時 ViewModel 不釋放
    // 解法：明確界定哪些 tab 需保留 state
}

bottomNavController.navigate(item.route) {
    popUpTo(bottomNavController.graph.findStartDestination().id) {
        saveState = item.preserveState   // 不是所有 tab 都要 saveState
    }
    launchSingleTop = true
    restoreState = item.preserveState
}
```

### 在 nested graph 共享 ViewModel

```kotlin
@Composable
fun CheckoutFlowScreen(navController: NavController) {
    val parentEntry = remember(navController) {
        navController.getBackStackEntry<CheckoutRoute>()
    }
    val sharedVm = hiltViewModel<CheckoutViewModel>(parentEntry)
}
```

避免在子畫面取 root ViewModel；用最近的 graph entry。

## Navigation Testing

```kotlin
@HiltAndroidTest
class NavigationTest {
    @get:Rule(order = 0) val hiltRule = HiltAndroidRule(this)
    @get:Rule(order = 1) val composeRule = createAndroidComposeRule<HiltTestActivity>()

    @Test fun deepLinkToDetail_arrivesWithProductId() {
        val navController = TestNavHostController(ApplicationProvider.getApplicationContext())
            .apply { navigatorProvider.addNavigator(ComposeNavigator()) }

        composeRule.setContent {
            AppTheme { AppNavHost(navController = navController) }
        }

        composeRule.runOnUiThread {
            navController.handleDeepLink(
                NavDeepLinkRequest.Builder
                    .fromUri(Uri.parse("https://example.com/product/p-42"))
                    .build()
            )
        }

        val current: DetailRoute = navController.currentBackStackEntry!!.toRoute()
        assertEquals("p-42", current.productId)
    }
}
```

## Cross-Skill References

- `@ui_ux_engineering`：Adaptive layout（List-Detail Pane）的視覺設計，本 skill 接其導航。
- `@platform_modernization_2026`：`enableOnBackInvokedCallback`、Edge-to-Edge inset 處理。
- `@dependency_injection_mastery`：`Navigator` singleton 與 Hilt 整合。
- `@tech_stack_migration`：Activity/Fragment 改 Compose Navigation 的橋接。
- `@data_layer_mastery`：route 帶 ID 而非整個物件，由 ViewModel 從 repo 載入。

## Quick Checklist

- [ ] 所有路由為 `@Serializable` data class，無字串路由
- [ ] Deep Link 與 type-safe route 對應且驗證 host
- [ ] `assetlinks.json` 部署且 `adb shell pm get-app-links` 顯示 verified
- [ ] `enableOnBackInvokedCallback="true"` 已設
- [ ] 自訂 back 用 `PredictiveBackHandler`，含 cancellation 還原
- [ ] 跨模組透過 `Navigator` 介面，feature 不持 NavController
- [ ] Bottom Nav 的 saveState/restoreState 經設計，無 ViewModel 洩漏
- [ ] Nested graph 用 `getBackStackEntry<Route>()` 取共享 VM
- [ ] Navigation Testing 至少覆蓋 deep link 與 back behavior
