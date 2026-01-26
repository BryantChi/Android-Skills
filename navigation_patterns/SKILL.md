---
name: Navigation Patterns
description: Deep Links、跨模組導航與複雜 Back Stack 管理
---

# Navigation Patterns (導航模式)

**Related Scenarios**: A (新專案), B (舊專案擴充)

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
