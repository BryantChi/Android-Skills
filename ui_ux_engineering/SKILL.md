---
id: ui_ux_engineering
name: UI/UX Engineering
description: Compose UI 工程化 — Material 3 Expressive、Edge-to-Edge (API 35)、Adaptive (List-Detail / Supporting Pane)、Spacing/Typography token、Predictive Back 動畫、Compose Stability/Compiler Metrics、Haptic Feedback、Espresso Accessibility 自動化，產出可長期維護的 Design System
type: skill
---

# UI/UX Engineering

## Instructions

當問題涉及 Compose UI 架構、Design System、複雜互動模式、Edge-to-Edge、Adaptive、a11y 時載入。本 skill 不處理導航（→ `@navigation_patterns`）、不處理效能 profiling（→ `@deep_performance_tuning`）。Material 3 Expressive 的 dynamic color 與 motion token 為 2026 預設。

## When to Use

- Scenario A：新專案 Design System 建立
- Scenario B：舊專案新增 UI 功能
- Scenario D：UI 重組過多 / 卡頓的視覺面排查（profiling 端在 `@deep_performance_tuning`）
- Edge-to-Edge 強制執行後 layout 重整
- 把 Figma Token 帶進來、版本化
- a11y 自動化掃描

## When NOT to Use

- 導航與 Predictive Back 路由邏輯 → `@navigation_patterns`
- Compose 命名與 modifier 順序規則 → `@coding_style_conventions`
- Macrobenchmark / Baseline Profile → `@deep_performance_tuning`
- 平台升級造成的行為變更 → `@platform_modernization_2026`

## Example Prompts

- 「Material 3 Expressive 的 dynamic color 怎麼接？需要 fallback palette 嗎？」
- 「升 targetSdk 35 後底部被導覽列蓋住，Edge-to-Edge 該怎麼處理？」
- 「平板要做 List-Detail Pane，Compose Adaptive 怎麼用？」
- 「Compose 重組過多，stability config 怎麼配？」
- 「TalkBack 自動化測試，Espresso Accessibility 怎麼接 CI？」

## Workflow

1. **Token 層**：Color / Typography / Spacing / Shape / Motion 五類 token 定義。
2. **Theme**：`MaterialTheme` + Expressive 擴充 + `CompositionLocalProvider` 提供自家 token。
3. **Component 層**：基於 token 封裝 Button/Card/TextField 等基礎元件。
4. **Pattern 層**：Adaptive、Collapsing、Shared Element、List-Detail。
5. **a11y + Edge-to-Edge** 收尾，Espresso Accessibility CI gate。

## Practical Notes (2026-04)

| 元件 | 推薦 |
|------|------|
| Material 3 | 1.4.x（含 Expressive） |
| `androidx.compose.material3.adaptive` | 1.0+ |
| Dynamic Color | Android 12+ 啟用、低版本 fallback |
| Edge-to-Edge | targetSdk 35 強制 |
| Predictive Back | 視覺由本 skill；路由由 navigation skill |
| Haptic | `HapticFeedback.performHapticFeedback(HapticFeedbackType.Confirm)` |
| Accessibility 測試 | Espresso Accessibility + Compose semantics test |

## Minimal Template

```
Token: colors / typography / spacing / shape / motion
Theme: AppTheme(dynamicColor = true) with fallback
Components: Button, Card, TextField, EmptyState, LoadingState, ErrorState
Patterns: AdaptivePane, CollapsingTopBar, SharedElement, ListDetailLayout
a11y: contentDescription audit, 48dp touch target, contrast >= 4.5:1
驗收: Quick Checklist
```

## Design System：Token 與 Theme

### Color（Material 3 Expressive + Dynamic Color）

```kotlin
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit,
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val ctx = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(ctx) else dynamicLightColorScheme(ctx)
        }
        darkTheme -> ExpressiveDarkColorScheme  // brand fallback
        else -> ExpressiveLightColorScheme
    }

    CompositionLocalProvider(
        LocalSpacing provides Spacing(),
        LocalShape provides AppShapes(),
        LocalMotion provides MotionTokens(),
    ) {
        MaterialTheme(
            colorScheme = colorScheme,
            typography = ExpressiveTypography,
            shapes = MaterialTheme.shapes,
            content = content,
        )
    }
}
```

Expressive palette 含 `surfaceContainerHighest`、`surfaceContainerLow`、`onSurfaceVariant`，比舊 M3 顏色更層次。

### Spacing / Shape / Motion Token

```kotlin
@Immutable data class Spacing(
    val xxs: Dp = 2.dp,
    val xs: Dp = 4.dp,
    val sm: Dp = 8.dp,
    val md: Dp = 16.dp,
    val lg: Dp = 24.dp,
    val xl: Dp = 32.dp,
    val xxl: Dp = 48.dp,
)
val LocalSpacing = staticCompositionLocalOf { Spacing() }

@Immutable data class MotionTokens(
    val short: AnimationSpec<Float> = tween(150, easing = FastOutSlowInEasing),
    val medium: AnimationSpec<Float> = tween(300, easing = FastOutSlowInEasing),
    val emphasized: AnimationSpec<Float> = tween(500, easing = EmphasizedEasing),
)
val LocalMotion = staticCompositionLocalOf { MotionTokens() }
```

### Figma Token Sync（版本化）

```
config/design-tokens/
├── tokens.v3.json        # Figma 匯出（Style Dictionary 格式）
└── README.md             # 紀錄 Figma 檔案 URL + commit hash
```

```kotlin
// build-logic plugin: 把 tokens.json 編譯為 Kotlin object
// gradle task: ./gradlew generateDesignTokens
```

每次 Figma 改動 → 提 PR 改 `tokens.v3.json` + 更新 commit hash + 重新 codegen。

## Edge-to-Edge（Android 15 強制）

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    enableEdgeToEdge()    // androidx.activity:activity-compose
    setContent { AppTheme { App() } }
}
```

```kotlin
@Composable
fun App() {
    Scaffold(
        contentWindowInsets = WindowInsets.safeDrawing,
    ) { inner ->
        NavHost(
            navController = rememberNavController(),
            startDestination = HomeRoute,
            modifier = Modifier.padding(inner),
        ) { /* ... */ }
    }
}
```

底部 BottomBar / FAB 用 `Modifier.windowInsetsPadding(WindowInsets.navigationBars)`：

```kotlin
BottomAppBar(
    modifier = Modifier.windowInsetsPadding(WindowInsets.navigationBars),
) { /* actions */ }
```

System bar 顏色：

```kotlin
WindowCompat.getInsetsController(window, window.decorView).apply {
    isAppearanceLightStatusBars = !darkTheme
    isAppearanceLightNavigationBars = !darkTheme
}
```

## Adaptive（List-Detail / Supporting Pane）

```kotlin
@OptIn(ExperimentalMaterial3AdaptiveApi::class)
@Composable
fun ProductListDetail() {
    val navigator = rememberListDetailPaneScaffoldNavigator<String>()

    BackHandler(navigator.canNavigateBack()) { navigator.navigateBack() }

    ListDetailPaneScaffold(
        directive = navigator.scaffoldDirective,
        value = navigator.scaffoldValue,
        listPane = {
            AnimatedPane {
                ProductList(onClick = { id -> navigator.navigateTo(ListDetailPaneScaffoldRole.Detail, id) })
            }
        },
        detailPane = {
            AnimatedPane {
                navigator.currentDestination?.content?.let { id -> ProductDetail(id) }
                    ?: EmptyDetailPane()
            }
        },
    )
}
```

寬度 < 600dp（手機）自動 single-pane；600-840dp 並排顯示；> 840dp 含 supporting pane。

## Complex UI Patterns

### Collapsing Top App Bar（Material 3 內建）

```kotlin
val scrollBehavior = TopAppBarDefaults.exitUntilCollapsedScrollBehavior()

Scaffold(
    modifier = Modifier.nestedScroll(scrollBehavior.nestedScrollConnection),
    topBar = {
        LargeTopAppBar(
            title = { Text("Products") },
            scrollBehavior = scrollBehavior,
        )
    },
) { padding ->
    LazyColumn(contentPadding = padding) { /* items */ }
}
```

不需要再手寫 nested scroll 計算。

### Shared Element Transition

```kotlin
SharedTransitionLayout {
    AnimatedContent(targetState = screen, label = "shared") { target ->
        when (target) {
            is Screen.List -> ProductList(
                onClick = { /* ... */ },
                imageModifier = { id ->
                    Modifier.sharedElement(
                        rememberSharedContentState("image-$id"),
                        animatedVisibilityScope = this@AnimatedContent,
                    )
                },
            )
            is Screen.Detail -> ProductDetail(
                imageModifier = Modifier.sharedElement(
                    rememberSharedContentState("image-${target.id}"),
                    animatedVisibilityScope = this@AnimatedContent,
                ),
            )
        }
    }
}
```

### Predictive Back 視覺

導航與 cancellation 路徑見 `@navigation_patterns`；視覺端使用 `PredictiveBackHandler` 接 progress 0..1 動畫場景：

```kotlin
val offset = remember { Animatable(0f) }
PredictiveBackHandler { progress ->
    progress.collect { offset.snapTo(it.progress * 200f) }
}
Box(Modifier.offset { IntOffset(offset.value.toInt(), 0) }) { content() }
```

## Compose Stability（避免不必要重組）

啟用 Compiler Metrics（在 `@coding_style_conventions` 已介紹），本 skill 教怎麼讀報告與調整。

```
# build/compose-reports/MyComposable-composables.txt
restartable scheme("[androidx.compose.ui.UiComposable]") fun MyCard(
  stable user: User
  unstable items: List<Item>     ← 問題
  stable onClick: Function0<Unit>
)
```

修法：

1. **改用 immutable collection**：`androidx.compose.runtime:runtime` 的 `kotlinx.collections.immutable.ImmutableList`。
2. **資料類加 `@Immutable` 或 `@Stable`**（保證契約）。
3. **stability config 標第三方型別**為 stable。

```kotlin
@Immutable data class UiItem(val id: String, val title: String)

@Composable
fun MyCard(user: User, items: ImmutableList<UiItem>, onClick: () -> Unit) { /* ... */ }
```

Strong Skipping（Compose Compiler 1.5.4+ 預設啟用）會在參數 instance 不變時 skip 即使 unstable。

## Haptic Feedback（Material 3 Expressive 整合）

```kotlin
val haptic = LocalHapticFeedback.current

Button(onClick = {
    haptic.performHapticFeedback(HapticFeedbackType.Confirm)
    onConfirm()
}) { Text("Submit") }
```

`HapticFeedbackType` 含 `Confirm`、`Reject`、`GestureEnd`、`SegmentTick`。系統會依裝置強度調整。

## Accessibility 自動化

### Compose Semantics 測試

```kotlin
composeTestRule.setContent { AppTheme { ProductCard(product = sample, onClick = {}) } }
composeTestRule.onNodeWithContentDescription("商品 ${sample.name}").assertHasClickAction()
composeTestRule.onAllNodes(isFocusable()).assertCountEquals(3)
```

### Espresso Accessibility（整體掃描）

```kotlin
@Before fun enableA11y() {
    AccessibilityChecks.enable()
        .setRunChecksFromRootView(true)
        .setSuppressingResultMatcher(
            allOf(
                matchesViews(withId(R.id.decorative_image)),
                matchesCheckNames(`is`("DuplicateClickableBoundsCheck")),
            )
        )
}

@Test fun productScreen_passesAccessibility() {
    onView(withId(R.id.compose_view)).check(matches(isDisplayed()))
    // AccessibilityChecks 會在每次 ViewAction 時驗證
}
```

CI 跑 `./gradlew :app:connectedDebugAndroidTest -PaccessibilityChecks=true` 為 a11y gate。

### a11y Checklist

- 所有可互動元素 `contentDescription` 或 `Modifier.semantics { contentDescription = ... }`
- 觸控目標 ≥ 48dp（用 `Modifier.minimumInteractiveComponentSize()` 或 `Modifier.size(48.dp)`）
- 顏色對比 ≥ 4.5:1（normal text）/ 3:1（large text）
- 動態字體：`Text(fontSize = MaterialTheme.typography.bodyLarge.fontSize)`，避免 hard-code sp
- 圖片裝飾性 `contentDescription = null`

## Cross-Skill References

- `@coding_style_conventions`：Compose 命名/順序與 Stability config 規則。本 skill 教 token 與 component。
- `@navigation_patterns`：路由與 Predictive Back 邏輯；本 skill 教視覺。
- `@platform_modernization_2026`：Edge-to-Edge / Predictive Back 在 targetSdk 35 強制執行的行為變更。
- `@deep_performance_tuning`：UI 卡頓的 Macrobenchmark / Perfetto 定位。
- `@dependency_injection_mastery`：Theme 配置 / Feature flag 透過 DI 注入。

## Quick Checklist

- [ ] Theme 採 Material 3 Expressive + dynamic color，含 fallback palette
- [ ] Spacing / Shape / Motion token 透過 CompositionLocal 提供
- [ ] Figma token 版本化、commit hash 記錄、自動 codegen
- [ ] `enableEdgeToEdge()` + `WindowInsets.safeDrawing` 全畫面套用
- [ ] Adaptive 用 `ListDetailPaneScaffold` / `SupportingPaneScaffold`
- [ ] Compose Compiler Metrics 監看，unstable composable < 設定門檻
- [ ] Strong Skipping 已啟用、stability.conf 收容第三方型別
- [ ] Predictive Back 視覺 + cancellation 處理
- [ ] Haptic Feedback 套於 confirm/reject 等關鍵互動
- [ ] Espresso Accessibility 在 CI gate；contrast / 48dp / contentDescription 全覆蓋
