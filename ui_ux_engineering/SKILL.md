---
name: UI/UX Engineering
description: Design System 實作、複雜 UI 模式與 Accessibility
---

# UI/UX Engineering (UI/UX 工程化)

**Related Scenarios**: A (新專案), B (舊專案加功能), D (效能問題)

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
