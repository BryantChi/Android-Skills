---
name: Navigation Patterns
description: Deep Links、跨模組導航與複雜 Back Stack 管理
---

# Navigation Patterns (導航模式)

## Instructions
- 確認需求屬於導覽與 Back Stack 管理
- 依照下方章節順序套用
- 一次只處理一個導航面向（Deep Link、跨模組、Back Stack）
- 完成後對照 Quick Checklist

## When to Use
- Scenario A：新專案導航設計
- Scenario B：舊專案擴充與導航橋接

## Example Prompts
- "請參考 Type-Safe Args，更新我的 NavHost 寫法"
- "依照 Deep Links 章節，幫我加入 App Links"
- "請用 Multi-Module Navigation 設計跨模組導航介面"

## Workflow
1. 先建立 Compose Navigation 基礎路由
2. 再加入 Deep Links 與跨模組導航
3. 最後用 Back Stack 管理與 Quick Checklist 驗收

## Practical Notes (2026)
- 預設採用 type-safe args，避免字串路由散落
- Deep Link 必須有驗證與回歸測試流程
- Back Stack 規則統一化，避免各模組自訂邏輯

## Minimal Template
```
目標: 
路由範圍: 
Deep Link: 
Back Stack 規則: 
驗收: Quick Checklist
```

---

## Compose Navigation Basics

### Type-Safe Args (Navigation 2.8+)

```kotlin
// 定義路由
@Serializable
data class DetailRoute(val productId: String)

@Serializable
object HomeRoute

// NavHost
NavHost(navController, startDestination = HomeRoute) {
    composable<HomeRoute> { 
        HomeScreen(onProductClick = { id ->
            navController.navigate(DetailRoute(id))
        })
    }
    
    composable<DetailRoute> { backStackEntry ->
        val route = backStackEntry.toRoute<DetailRoute>()
        DetailScreen(productId = route.productId)
    }
}
```

### Nested Graphs

```kotlin
NavHost(navController, startDestination = "main") {
    navigation(startDestination = "home", route = "main") {
        composable("home") { HomeScreen() }
        composable("profile") { ProfileScreen() }
    }
    
    navigation(startDestination = "login", route = "auth") {
        composable("login") { LoginScreen() }
        composable("register") { RegisterScreen() }
    }
}
```

---

## Deep Links

### App Links 設定

```xml
<!-- AndroidManifest.xml -->
<activity android:name=".MainActivity">
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:scheme="https"
            android:host="example.com"
            android:pathPrefix="/product" />
    </intent-filter>
</activity>
```

### assetlinks.json (Host 驗證)

```json
// https://example.com/.well-known/assetlinks.json
[{
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
        "namespace": "android_app",
        "package_name": "com.example.app",
        "sha256_cert_fingerprints": ["..."]
    }
}]
```

### Navigation 整合

```kotlin
composable(
    route = "product/{id}",
    deepLinks = listOf(
        navDeepLink { uriPattern = "https://example.com/product/{id}" }
    )
) { backStackEntry ->
    val id = backStackEntry.arguments?.getString("id")
    ProductScreen(id)
}
```

---

## Multi-Module Navigation

### API Module Pattern

```kotlin
// :feature:product:api
interface ProductNavigator {
    fun navigateToProduct(productId: String)
    fun navigateToProductList()
}

// :feature:product:impl
class ProductNavigatorImpl @Inject constructor(
    private val navController: NavController
) : ProductNavigator {
    override fun navigateToProduct(productId: String) {
        navController.navigate("product/$productId")
    }
}

// 其他模組使用
class HomeViewModel @Inject constructor(
    private val productNavigator: ProductNavigator
) {
    fun onProductClick(id: String) {
        productNavigator.navigateToProduct(id)
    }
}
```

### Navigation Events (Single Event)

```kotlin
// ViewModel
sealed class NavigationEvent {
    data class ToDetail(val id: String) : NavigationEvent()
    object Back : NavigationEvent()
}

private val _navigationEvent = Channel<NavigationEvent>()
val navigationEvent = _navigationEvent.receiveAsFlow()

// Composable
LaunchedEffect(Unit) {
    viewModel.navigationEvent.collect { event ->
        when (event) {
            is NavigationEvent.ToDetail -> navController.navigate("detail/${event.id}")
            NavigationEvent.Back -> navController.popBackStack()
        }
    }
}
```

---

## Complex Back Stack Management

### Auth Flow (Clear Stack)

```kotlin
fun navigateToHome() {
    navController.navigate("home") {
        popUpTo("auth") { inclusive = true }  // 清除 auth 流程
        launchSingleTop = true
    }
}
```

### Bottom Nav with Separate Stacks

```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()
    
    Scaffold(
        bottomBar = {
            NavigationBar {
                items.forEach { item ->
                    NavigationBarItem(
                        selected = currentRoute == item.route,
                        onClick = {
                            navController.navigate(item.route) {
                                popUpTo(navController.graph.findStartDestination().id) {
                                    saveState = true
                                }
                                launchSingleTop = true
                                restoreState = true
                            }
                        }
                    )
                }
            }
        }
    ) { /* NavHost */ }
}
```

---

## Quick Checklist

- [ ] 使用 Type-Safe Args (Navigation 2.8+)
- [ ] Deep Links 配置 assetlinks.json
- [ ] 跨模組使用 Navigator interface
- [ ] Navigation Events 作為 Single Event 處理
- [ ] Bottom Nav 正確保存/恢復 State
