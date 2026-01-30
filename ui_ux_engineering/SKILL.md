---
name: UI/UX Engineering
description: Design System 實作、複雜 UI 模式與 Accessibility
---

# UI/UX Engineering (UI/UX 工程化)

## Instructions
- 確認需求屬於 UI 架構、互動或可近用性
- 依照下方章節順序套用
- 一次只調整一種 UI 模式或設計 token
- 完成後對照 Quick Checklist

## When to Use
- Scenario A：新專案設計系統建立
- Scenario B：舊專案新增 UI 功能
- Scenario D：UI 效能與體驗問題

## Example Prompts
- "請依照 Design System Implementation 章節，建立主題與 spacing"
- "請參考 Complex UI Patterns，實作 Collapsing Toolbar"
- "用 Accessibility 章節檢視主要流程是否符合 a11y"

## Workflow
1. 先建立 Design System 的基礎 tokens
2. 再套用 Complex UI Patterns 與 Adaptive Layouts
3. 最後用 Accessibility 與 Quick Checklist 驗收

## Practical Notes (2026)
- Compose-first，但保留 View/Fragment 互通規範
- a11y 驗收必包含 TalkBack 走查與觸控目標
- UI 效能以重組與列表滾動為優先檢查點

## Minimal Template
```
目標: 
畫面範圍: 
設計規範: 
a11y 要求: 
驗收: Quick Checklist
```

---

## Design System Implementation

### Theme 架構

```kotlin
// ui/theme/Theme.kt
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colorScheme = if (darkTheme) DarkColorScheme else LightColorScheme
    
    CompositionLocalProvider(
        LocalSpacing provides Spacing(),
        LocalTypography provides AppTypography,
    ) {
        MaterialTheme(
            colorScheme = colorScheme,
            typography = Typography,
            content = content
        )
    }
}
```

### Spacing System

```kotlin
data class Spacing(
    val xs: Dp = 4.dp,
    val sm: Dp = 8.dp,
    val md: Dp = 16.dp,
    val lg: Dp = 24.dp,
    val xl: Dp = 32.dp,
)

val LocalSpacing = staticCompositionLocalOf { Spacing() }

// 使用
val spacing = LocalSpacing.current
Spacer(modifier = Modifier.height(spacing.md))
```

### Figma Token Sync

```kotlin
// 從 Figma 導出的 Tokens
object DesignTokens {
    object Colors {
        val Primary = Color(0xFF6200EE)
        val OnPrimary = Color(0xFFFFFFFF)
        val Surface = Color(0xFFFFFBFE)
    }
    
    object Typography {
        val HeadlineLarge = TextStyle(
            fontFamily = FontFamily.Default,
            fontWeight = FontWeight.Bold,
            fontSize = 32.sp,
            lineHeight = 40.sp
        )
    }
}
```

---

## Complex UI Patterns

### Collapsing Toolbar

```kotlin
@Composable
fun CollapsingToolbarScreen() {
    val scrollState = rememberLazyListState()
    val toolbarHeight = 200.dp
    val minToolbarHeight = 56.dp
    
    val toolbarHeightPx = with(LocalDensity.current) { toolbarHeight.toPx() }
    val minToolbarHeightPx = with(LocalDensity.current) { minToolbarHeight.toPx() }
    
    val toolbarOffsetHeightPx = remember { mutableFloatStateOf(0f) }
    
    val nestedScrollConnection = remember {
        object : NestedScrollConnection {
            override fun onPreScroll(available: Offset, source: NestedScrollSource): Offset {
                val delta = available.y
                val newOffset = toolbarOffsetHeightPx.floatValue + delta
                toolbarOffsetHeightPx.floatValue = newOffset.coerceIn(
                    minToolbarHeightPx - toolbarHeightPx, 0f
                )
                return Offset.Zero
            }
        }
    }
    
    Box(
        modifier = Modifier
            .fillMaxSize()
            .nestedScroll(nestedScrollConnection)
    ) {
        LazyColumn(state = scrollState) { /* content */ }
        
        Box(
            modifier = Modifier
                .height(with(LocalDensity.current) {
                    (toolbarHeightPx + toolbarOffsetHeightPx.floatValue).toDp()
                })
                .fillMaxWidth()
                .background(MaterialTheme.colorScheme.primary)
        )
    }
}
```

### Shared Element Transition

```kotlin
// Navigation 3.0+ with SharedTransitionLayout
SharedTransitionLayout {
    AnimatedContent(targetState = screen) { targetScreen ->
        when (targetScreen) {
            is Screen.List -> {
                ListItem(
                    modifier = Modifier.sharedElement(
                        state = rememberSharedContentState(key = "image-${item.id}"),
                        animatedVisibilityScope = this@AnimatedContent
                    )
                )
            }
            is Screen.Detail -> {
                DetailImage(
                    modifier = Modifier.sharedElement(
                        state = rememberSharedContentState(key = "image-${item.id}"),
                        animatedVisibilityScope = this@AnimatedContent
                    )
                )
            }
        }
    }
}
```

---

## Adaptive Layouts

### WindowSizeClass

```kotlin
@Composable
fun AdaptiveLayout() {
    val windowSizeClass = calculateWindowSizeClass(activity)
    
    when (windowSizeClass.widthSizeClass) {
        WindowWidthSizeClass.Compact -> {
            // Phone: Single column, bottom nav
            CompactLayout()
        }
        WindowWidthSizeClass.Medium -> {
            // Tablet portrait: Navigation rail
            MediumLayout()
        }
        WindowWidthSizeClass.Expanded -> {
            // Tablet landscape / Desktop: Permanent drawer
            ExpandedLayout()
        }
    }
}
```

---

## Accessibility (a11y)

### Semantic Properties

```kotlin
@Composable
fun AccessibleButton(
    onClick: () -> Unit,
    label: String
) {
    Button(
        onClick = onClick,
        modifier = Modifier.semantics {
            contentDescription = label
            role = Role.Button
        }
    )
}
```

### TalkBack Testing Checklist

- [ ] 所有可互動元素都有 `contentDescription`
- [ ] 圖片有適當的 `contentDescription` 或標記為 decorative
- [ ] 觸控目標 >= 48dp
- [ ] 顏色對比度符合 WCAG 2.1 標準
- [ ] 使用 TalkBack 完整走過主要流程

### Decorative Images

```kotlin
Image(
    painter = painterResource(R.drawable.decoration),
    contentDescription = null,  // 裝飾性圖片
    modifier = Modifier.semantics { invisibleToUser() }
)
```

---

## Quick Checklist

### Design System
- [ ] Color Scheme 定義 (Light/Dark)
- [ ] Typography Scale 定義
- [ ] Spacing System (Tokens)
- [ ] Common Components 封裝

### Accessibility
- [ ] 所有按鈕有 contentDescription
- [ ] 觸控目標 >= 48dp
- [ ] 顏色對比度檢查
- [ ] TalkBack 測試通過
