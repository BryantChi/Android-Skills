---
id: dependency_injection_mastery
name: Dependency Injection Mastery
description: Hilt 2.52+ 進階用法 — Assisted Injection、Custom Components/Scopes、Multi-binding、EntryPoint 跨模組安全、KSP2 取代 kapt、@UninstallModules 測試策略，產出可測試的多模組 DI 架構
type: skill
---

# Dependency Injection Mastery

## Instructions

當問題涉及 Hilt 進階模式、scope 設計、跨模組存取、測試替換時載入。Dagger 2 → Hilt 的「遷移步驟」由 `@tech_stack_migration` 為唯一權威來源，本 skill 不重複（去重）；本 skill 處理 Hilt 已就位後的進階使用。

## When to Use

- Scenario A：新專案 DI 架構建立
- Scenario C：舊專案模組拆分，Hilt 已存在
- Scenario F：KMP 共用模組與 Android 端 DI 整合
- 設計 User Session / Tenant Session 等 Custom Scope
- 多模組間以 EntryPoint 安全存取服務
- 用 Multi-binding 做 plugin 架構

## When NOT to Use

- Dagger 2 → Hilt 遷移步驟 → `@tech_stack_migration`
- Hilt plugin 安裝、KSP 設定（Convention Plugin）→ `@project_bootstrapping`
- 平台升級造成的 KSP 版本問題 → `@platform_modernization_2026`

## Example Prompts

- 「ViewModel 要吃導航參數又要用 Hilt，怎麼接 Assisted？」
- 「想做 User Session Scope，登入後的服務都該綁這個 scope」
- 「多種支付方式想用 plugin 架構，Multi-binding 怎麼設計？」
- 「跨 module 想取得 feature 的服務，又不想直接 import，EntryPoint 怎麼用？」
- 「unit test 想 fake repository，`@TestInstallIn` 怎麼配？」

## Workflow

1. **盤生命週期**：列出實際 scope（App / User / Activity / Screen），對應到 Hilt 內建 component 或 Custom Component。
2. **設計 Module 邊界**：依模組（network / database / repository / usecase）分檔，避免一檔過大。
3. **Qualifier**：相同型別不同實例（IO/Default/Main dispatcher、Auth/NoAuth client）用 qualifier 區分。
4. **跨模組存取**：用 `@EntryPoint` 而非全域 singleton；遠離 `applicationContext.applicationContext as App`。
5. **測試**：`@TestInstallIn(replaces = [RealModule::class])` + `HiltAndroidRule`，可重組整個 graph。

## Practical Notes (2026-04)

| 項目 | 推薦做法 |
|------|----------|
| 編譯器 | KSP 2.x（**不用 kapt**） |
| Hilt 版本 | 2.52+ |
| Compose ViewModel | `hiltViewModel<VM, Factory> { factory.create(...) }` |
| Worker | `@HiltWorker` + `HiltWorkerFactory` |
| 第三方無法注入 | `@EntryPoint` from non-Hilt class |
| 測試替換 | `@TestInstallIn(replaces = [...])` |
| Scope 過度設計風險 | **能用 `@Singleton` 解決就不要建 Custom Component** |

## Minimal Template

```
目標: <例：登入後的服務綁 User Session>
Scope 對應:
  - SingletonComponent: Network, Database, Analytics
  - ActivityRetainedComponent: NavigationRouter
  - ViewModelComponent: 表單 state holder
  - UserSessionComponent (custom): UserPreferences, FeatureFlags
Module 結構:
  - core:network/di/NetworkModule
  - core:database/di/DatabaseModule
  - feature:checkout/di/CheckoutBindModule
Qualifier: @IoDispatcher, @AuthenticatedClient
測試替換: @TestInstallIn 在 androidTest/
驗收: Quick Checklist
```

## Assisted Injection

### Compose ViewModel 接導航參數

```kotlin
@HiltViewModel(assistedFactory = DetailViewModel.Factory::class)
class DetailViewModel @AssistedInject constructor(
    @Assisted private val productId: String,
    @Assisted savedStateHandle: SavedStateHandle,
    private val repository: ProductRepository,
) : ViewModel() {
    @AssistedFactory
    interface Factory {
        fun create(productId: String, savedStateHandle: SavedStateHandle): DetailViewModel
    }
}

@Composable
fun DetailScreen(productId: String) {
    val viewModel = hiltViewModel<DetailViewModel, DetailViewModel.Factory> { factory ->
        factory.create(productId, createSavedStateHandle())
    }
}
```

### WorkManager

```kotlin
@HiltWorker
class SyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val repository: Repository,
) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result = runCatching { repository.sync() }
        .map { Result.success() }
        .getOrElse { Result.retry() }
}
```

## Custom Hilt Components — 決策矩陣

| 情境 | 建議 |
|------|------|
| 同 App 全程都需要的服務 | `@Singleton` 即可 |
| 跨多個 Activity，但 Activity 重建不重生 | `@ActivityRetainedScoped` |
| 單一 Activity / Fragment / ViewModel 範圍 | 內建對應 scope |
| 登入後才存在、登出時應釋放的服務 | **Custom Component（推薦）** |
| 多租戶 app 切換 tenant 時釋放 | **Custom Component** |
| 「我覺得這樣比較乾淨」但無生命週期理由 | ❌ 不要做 |

### User Session 範例

```kotlin
@Scope
@Retention(AnnotationRetention.RUNTIME)
annotation class UserSessionScope

@DefineComponent(parent = SingletonComponent::class)
@UserSessionScope
interface UserSessionComponent

@DefineComponent.Builder
interface UserSessionComponentBuilder {
    fun setUser(@BindsInstance user: User): UserSessionComponentBuilder
    fun build(): UserSessionComponent
}

@EntryPoint
@InstallIn(UserSessionComponent::class)
interface UserSessionEntryPoint {
    fun userPreferences(): UserPreferences
    fun featureFlagSource(): FeatureFlagSource
}

@Singleton
class UserSessionManager @Inject constructor(
    private val componentBuilder: Provider<UserSessionComponentBuilder>,
) {
    private var component: UserSessionComponent? = null

    fun login(user: User) {
        component = componentBuilder.get().setUser(user).build()
    }

    fun logout() {
        component = null
    }

    inline fun <reified T> entryPoint(extract: UserSessionEntryPoint.() -> T): T {
        val c = checkNotNull(component) { "Not logged in" }
        return EntryPoints.get(c, UserSessionEntryPoint::class.java).extract()
    }
}
```

## Multi-binding（Plugin 架構）

### Set

```kotlin
interface PaymentProcessor { fun supports(type: String): Boolean; fun process(amount: Long): Result<Unit> }

@Module
@InstallIn(SingletonComponent::class)
abstract class PaymentBindingModule {
    @Binds @IntoSet abstract fun creditCard(impl: CreditCardProcessor): PaymentProcessor
    @Binds @IntoSet abstract fun paypal(impl: PayPalProcessor): PaymentProcessor
}

class PaymentService @Inject constructor(
    private val processors: Set<@JvmSuppressWildcards PaymentProcessor>,
) {
    fun process(type: String, amount: Long): Result<Unit> =
        processors.first { it.supports(type) }.process(amount)
}
```

### Map（按 key 取）

```kotlin
@MapKey annotation class PaymentTypeKey(val value: PaymentType)

@Module
@InstallIn(SingletonComponent::class)
abstract class PaymentMapModule {
    @Binds @IntoMap @PaymentTypeKey(PaymentType.CREDIT_CARD)
    abstract fun bindCreditCard(impl: CreditCardProcessor): PaymentProcessor
}

class PaymentService @Inject constructor(
    private val processors: Map<PaymentType, @JvmSuppressWildcards Provider<PaymentProcessor>>,
) {
    // Provider 延遲建構，未使用的 processor 不會被生成
    fun process(type: PaymentType, amount: Long) = processors.getValue(type).get().process(amount)
}
```

## EntryPoint：跨模組安全存取

當 `feature:a` 想用 `feature:b` 的服務，**不要直接 import**。改放在 `core:domain` 介面 + `feature:b` 實作 + `feature:a` 透過 EntryPoint 取。

```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface AnalyticsEntryPoint {
    fun analytics(): Analytics
}

// 非 Hilt class（如某 Util object 或第三方 callback）需要服務時
class LegacyCallback(private val context: Context) {
    private val analytics by lazy {
        EntryPointAccessors
            .fromApplication(context, AnalyticsEntryPoint::class.java)
            .analytics()
    }
}
```

## Qualifier 與 Dispatcher 注入

```kotlin
@Qualifier @Retention(AnnotationRetention.BINARY) annotation class IoDispatcher
@Qualifier @Retention(AnnotationRetention.BINARY) annotation class DefaultDispatcher
@Qualifier @Retention(AnnotationRetention.BINARY) annotation class MainDispatcher

@Module
@InstallIn(SingletonComponent::class)
object DispatcherModule {
    @Provides @IoDispatcher fun io(): CoroutineDispatcher = Dispatchers.IO
    @Provides @DefaultDispatcher fun default(): CoroutineDispatcher = Dispatchers.Default
    @Provides @MainDispatcher fun main(): CoroutineDispatcher = Dispatchers.Main.immediate
}
```

注入：

```kotlin
class Repository @Inject constructor(
    @IoDispatcher private val io: CoroutineDispatcher,
    private val api: Api,
) {
    suspend fun load() = withContext(io) { api.fetch() }
}
```

## 測試替換策略（@TestInstallIn / @UninstallModules）

```kotlin
// androidTest/.../FakeRepositoryModule.kt
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [RepositoryModule::class],
)
abstract class FakeRepositoryModule {
    @Binds abstract fun fake(impl: FakeUserRepository): UserRepository
}

@HiltAndroidTest
class LoginScreenTest {
    @get:Rule val hiltRule = HiltAndroidRule(this)
    @Inject lateinit var repository: UserRepository

    @Before fun setup() = hiltRule.inject()
}
```

單一測試需更精細替換時用 `@UninstallModules(RealModule::class)` + `@BindValue`：

```kotlin
@UninstallModules(RepositoryModule::class)
@HiltAndroidTest
class CheckoutFlowTest {
    @BindValue val userRepository: UserRepository = mockk(relaxed = true)
}
```

## Module Organization（多模組）

```
core/network/di/NetworkModule.kt           # OkHttp, Retrofit base
core/network/di/AuthInterceptorModule.kt
core/database/di/DatabaseModule.kt          # Room
core/data/di/RepositoryModule.kt            # Binds Repository impls
core/domain/di/UseCaseModule.kt             # 通常不需要，UseCase 用 constructor inject
feature/checkout/di/CheckoutModule.kt       # feature-local providers
app/di/AppModule.kt                         # Application-level
```

**API/Impl 分離**：

```
core:network:api    -> 介面與 DTO
core:network:impl   -> Retrofit 實作 + Hilt module（@InstallIn(SingletonComponent::class)）
feature:* 只 implementation core:network:api
app implementation core:network:impl
```

## Cross-Skill References

- `@tech_stack_migration`：**Dagger 2 → Hilt 遷移的唯一權威**。本 skill 不重複，遇到遷移問題請載入它。
- `@project_bootstrapping`：Convention Plugin 套 Hilt + KSP；本 skill 是 Hilt 已就位後的進階。
- `@platform_modernization_2026`：KSP 版本與 Kotlin 版本的對齊。
- `@kotlin_multiplatform`：KMP 端 commonMain 的 expect/actual + 平台 DI 接合。
- `@testing_legacy_strategies`：Hilt test rule 在舊代碼覆寫的策略。

## Quick Checklist

- [ ] kapt 完全移除，全用 KSP 2.x
- [ ] ViewModel 動態參數一律用 `@AssistedInject` + `hiltViewModel<VM, Factory>`
- [ ] Custom Component 僅在「真有獨立生命週期」時建立，並有對應的 lifecycle owner
- [ ] 跨模組存取走 `@EntryPoint` 或 `core` 介面，禁止直接 import 對方 impl
- [ ] Qualifier 區分同型別不同實例
- [ ] Dispatcher 一律用 `@IoDispatcher` 等 qualifier 注入，不直接寫 `Dispatchers.IO`
- [ ] 測試用 `@TestInstallIn` 替換整個 module；單一 case 用 `@UninstallModules` + `@BindValue`
- [ ] 無 circular dependency；用 `Lazy<T>` 或 `Provider<T>` 打破
- [ ] Multi-binding 使用 `Provider<T>` 避免不必要的初始化
