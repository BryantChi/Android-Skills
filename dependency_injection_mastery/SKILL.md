---
id: dependency_injection_mastery
name: Dependency Injection Mastery
description: Hilt 進階用法、Custom Components 與 Multi-binding 模式
---

# Dependency Injection Mastery (依賴注入專精)

## Instructions
- 確認問題屬於 DI 架構或 Scope 設計
- 依照下方章節順序套用
- 一次只調整一種注入模式或 module
- 完成後對照 Quick Checklist

## When to Use
- Scenario A：新專案 DI 架構建立
- Scenario C：舊專案現代化與模組拆分
- Scenario F：KMP 共用模組的 DI 調整

## Example Prompts
- "請參考 Assisted Injection，替 ViewModel 加入動態參數"
- "依照 Custom Hilt Components 建立 User Session Scope"
- "請用 Multi-binding 章節設計插件式支付模組"

## Workflow
1. 先確認 Assisted Injection 與 Scope 需求
2. 再整理 Module Organization 與 Qualifier
3. 最後用 Quick Checklist 驗收

## Practical Notes (2026)
- 多模組採 API/impl 分離，避免跨模組直接依賴實作
- 跨模組導航與 Service 以 interface + EntryPoint 統一
- Scope 設計先畫出生命週期，再落地到 Module

## Minimal Template
```
目標: 
Scope: 
Module 結構: 
注入模式: 
驗收: Quick Checklist
```

---

## Assisted Injection

當 ViewModel 或 Worker 需要同時接收 DI 的依賴與 Runtime 參數時使用。

### ViewModel with SavedStateHandle + Custom Args

```kotlin
@HiltViewModel(assistedFactory = DetailViewModel.Factory::class)
class DetailViewModel @AssistedInject constructor(
    @Assisted private val productId: String,
    @Assisted savedStateHandle: SavedStateHandle,
    private val repository: ProductRepository
) : ViewModel() {
    
    @AssistedFactory
    interface Factory {
        fun create(productId: String, savedStateHandle: SavedStateHandle): DetailViewModel
    }
}

// 在 Compose 中使用
@Composable
fun DetailScreen(productId: String) {
    val viewModel = hiltViewModel<DetailViewModel, DetailViewModel.Factory> { factory ->
        factory.create(productId, createSavedStateHandle())
    }
}
```

### WorkManager with Assisted Injection

```kotlin
@HiltWorker
class SyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val repository: Repository
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        repository.sync()
        return Result.success()
    }
}
```

---

## Custom Hilt Components (Scopes)

建立自定義的生命週期範圍，例如 User Session。

### 定義 Custom Scope

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
```

### 管理 Component 生命週期

```kotlin
@Singleton
class UserSessionManager @Inject constructor(
    private val componentBuilder: Provider<UserSessionComponentBuilder>
) {
    private var component: UserSessionComponent? = null
    
    fun login(user: User) {
        component = componentBuilder.get().setUser(user).build()
    }
    
    fun logout() {
        component = null
    }
    
    fun <T> getService(clazz: Class<T>): T {
        return EntryPoints.get(component!!, UserSessionEntryPoint::class.java)
            .let { /* get service */ }
    }
}
```

---

## Multi-binding (Set & Map)

實作 Plugin 架構，例如多種支付方式。

### Set Multibinding

```kotlin
interface PaymentProcessor {
    fun process(amount: Double): Result
}

@Module
@InstallIn(SingletonComponent::class)
abstract class PaymentModule {
    
    @Binds
    @IntoSet
    abstract fun bindCreditCard(impl: CreditCardProcessor): PaymentProcessor
    
    @Binds
    @IntoSet
    abstract fun bindPayPal(impl: PayPalProcessor): PaymentProcessor
}

// 注入所有實作
class PaymentService @Inject constructor(
    private val processors: Set<@JvmSuppressWildcards PaymentProcessor>
) {
    fun processAll(amount: Double) {
        processors.forEach { it.process(amount) }
    }
}
```

### Map Multibinding (with Key)

```kotlin
enum class PaymentType { CREDIT_CARD, PAYPAL, GOOGLE_PAY }

@MapKey
annotation class PaymentTypeKey(val value: PaymentType)

@Module
@InstallIn(SingletonComponent::class)
abstract class PaymentModule {
    
    @Binds
    @IntoMap
    @PaymentTypeKey(PaymentType.CREDIT_CARD)
    abstract fun bindCreditCard(impl: CreditCardProcessor): PaymentProcessor
}

// 按 Key 取得特定實作
class PaymentService @Inject constructor(
    private val processors: Map<PaymentType, @JvmSuppressWildcards PaymentProcessor>
) {
    fun process(type: PaymentType, amount: Double) {
        processors[type]?.process(amount)
    }
}
```

---

## Module Organization

### 分層架構

```
di/
├── AppModule.kt           # Application-level
├── NetworkModule.kt       # Retrofit, OkHttp
├── DatabaseModule.kt      # Room
├── RepositoryModule.kt    # Repository bindings
└── UseCaseModule.kt       # UseCase bindings
```

### Qualifier 使用

```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class IoDispatcher

@Module
@InstallIn(SingletonComponent::class)
object DispatcherModule {
    @Provides
    @IoDispatcher
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO
}
```

---

## Quick Checklist

- [ ] ViewModel 動態參數使用 Assisted Injection
- [ ] 避免在 Module 中使用 `@Provides` 建立複雜邏輯
- [ ] Qualifier 用於區分相同類型的不同實例
- [ ] 避免 Circular Dependencies
- [ ] 測試時使用 `@UninstallModules` + `@TestInstallIn`
