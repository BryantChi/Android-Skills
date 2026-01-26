---
name: Dependency Injection Mastery
description: Hilt 進階用法、Custom Components 與 Multi-binding 模式
---

# Dependency Injection Mastery (依賴注入專精)

**Related Scenarios**: A (新專案), C (現代化), F (KMP)

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
