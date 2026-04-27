---
id: data_layer_mastery
name: Data Layer Mastery
description: Android 資料層 — Room 2.7+ Migration / Paging 3 / FTS、Retrofit + OkHttp 統一錯誤處理、Offline-First SSOT、Sync Manager + WorkManager、Protobuf DataStore、Room vs SQLDelight 決策矩陣，產出可離線可同步的資料層
type: skill
---

# Data Layer Mastery

## Instructions

當問題涉及本地儲存、網路、快取一致性、Offline-First 架構時載入。本 skill 預設 Android-only 專案使用 Room；KMP 專案的 SQLDelight 路線由 `@kotlin_multiplatform` 接管，本 skill 提供決策矩陣指向它。

## When to Use

- Scenario A：新專案資料層建立
- Scenario D：資料層效能/一致性瓶頸
- Scenario F：KMP 共享資料層的「Android 端」
- 設計 Sync Manager 與 WorkManager 整合
- SharedPreferences → DataStore 遷移（Preferences 或 Protobuf）

## When NOT to Use

- KMP 共用資料層的 commonMain 設計 → `@kotlin_multiplatform`
- 加密儲存（EncryptedSharedPreferences、Keystore） → `@release_automation`
- 純效能 profiling（Macrobenchmark）→ `@deep_performance_tuning`

## Example Prompts

- 「Room 加欄位的 Migration 怎麼寫，怎麼測？」
- 「我們網路錯誤處理散落各 Repository，幫我統一」
- 「Offline-first 但要保證同步順序，WorkManager 怎麼配？」
- 「該用 Preferences DataStore 還是 Protobuf DataStore？」
- 「KMP 專案要不要從 Room 換 SQLDelight？」

## Workflow

1. **選庫**：對照 Room vs SQLDelight 決策矩陣（見 §「決策矩陣」）。
2. **Schema + Migration**：每次改 schema 必寫 Migration 並建 `MigrationTestHelper` 測試。
3. **Network**：統一 `NetworkResult` + interceptor stack（auth / retry / logging）。
4. **Repository (SSOT)**：local 為單一來源，remote fetch → save local → flow emit。
5. **Sync Manager**：WorkManager 編排背景同步，含衝突策略與失敗退避。

## Practical Notes (2026-04)

| 元件 | 版本 / 推薦 | 備註 |
|------|------------|------|
| Room | 2.7.x（KMP-ready） | KSP2 編譯 |
| Retrofit | 2.11+ | 配 Kotlinx Serialization Converter |
| OkHttp | 4.12+ | call timeout、event listener |
| Ktor Client | 3.x | KMP 場景 |
| DataStore Preferences | 1.1+ | 簡單 key-value |
| DataStore Proto | 1.1+ + protobuf-kotlin | 結構化、版本化、推薦預設 |
| Paging | 3.3+ | Compose 整合用 `LazyPagingItems` |
| WorkManager | 2.10+ | KSP 編譯、含 expedited |

## Minimal Template

```
目標: <例：商品列表離線可看，背景每 30 分鐘同步>
本地: Room (entity + dao) 或 Proto DataStore (settings)
遠端: Retrofit + Kotlinx Serialization
快取策略: read-through (local-first emit, fetch update)
同步: WorkManager periodic + OneTime on connectivity
錯誤模型: NetworkResult<T> + DomainError sealed
驗收: Quick Checklist
```

## Room vs SQLDelight 決策矩陣

| 因素 | Room | SQLDelight |
|------|------|------------|
| Android-only 專案 | ✅ 預設 | 可用但無優勢 |
| KMP（Android + iOS）共享 schema | ⚠️ Room 2.7+ 已支援 KMP，但 iOS 仍實驗 | ✅ 成熟 |
| 寫 SQL（vs 註解） | 註解為主 | 純 SQL |
| 編譯期檢查 | ✅（query 驗證） | ✅ |
| 與 AndroidX 生態（Paging、WorkManager） | ✅ 原生 | 需 wrapper |
| Reactive（Flow） | ✅ 內建 | ✅ Coroutines extension |
| Migration 工具 | ✅ MigrationTestHelper | 手寫 .sqm |
| **建議** | **Android-only / Compose-first** | **真 KMP 跨 iOS / 純 SQL 偏好** |

選擇 SQLDelight 後請載入 `@kotlin_multiplatform` 看 commonMain 設計。

## Room 進階

### Migration 與測試

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE users ADD COLUMN avatar_url TEXT")
    }
}

val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("CREATE TABLE users_new (id TEXT PRIMARY KEY NOT NULL, name TEXT NOT NULL, email TEXT NOT NULL)")
        db.execSQL("INSERT INTO users_new (id, name, email) SELECT id, name, COALESCE(email,'') FROM users")
        db.execSQL("DROP TABLE users")
        db.execSQL("ALTER TABLE users_new RENAME TO users")
    }
}
```

```kotlin
// androidTest/.../MigrationsTest.kt
@get:Rule val helper = MigrationTestHelper(
    InstrumentationRegistry.getInstrumentation(),
    AppDatabase::class.java,
)

@Test fun migrate1To3() {
    helper.createDatabase(TEST_DB, 1).apply {
        execSQL("INSERT INTO users (id, name, email) VALUES ('u1', 'Alice', NULL)")
        close()
    }
    val db = helper.runMigrationsAndValidate(TEST_DB, 3, true, MIGRATION_1_2, MIGRATION_2_3)
    val cursor = db.query("SELECT email FROM users WHERE id = 'u1'")
    cursor.moveToFirst()
    assertEquals("", cursor.getString(0))
}
```

### Paging 3 + Compose

```kotlin
@Dao interface UserDao {
    @Query("SELECT * FROM users ORDER BY name") fun pagingSource(): PagingSource<Int, UserEntity>
}

class UserRepository @Inject constructor(private val dao: UserDao) {
    fun pagedUsers(): Flow<PagingData<User>> = Pager(
        config = PagingConfig(pageSize = 30, prefetchDistance = 10, enablePlaceholders = false),
        pagingSourceFactory = { dao.pagingSource() },
    ).flow.map { it.map(UserEntity::toDomain) }
}

@Composable
fun UserListScreen(viewModel: UserListViewModel = hiltViewModel()) {
    val items = viewModel.users.collectAsLazyPagingItems()
    LazyColumn {
        items(items.itemCount, key = items.itemKey { it.id }) { i ->
            items[i]?.let { UserRow(it) }
        }
    }
}
```

### Full-Text Search

```kotlin
@Fts4(contentEntity = ArticleEntity::class)
@Entity(tableName = "articles_fts")
data class ArticleFtsEntity(@ColumnInfo("title") val title: String, @ColumnInfo("content") val content: String)

@Dao interface ArticleDao {
    @Query("SELECT * FROM articles WHERE rowid IN (SELECT rowid FROM articles_fts WHERE articles_fts MATCH :query)")
    fun search(query: String): Flow<List<ArticleEntity>>
}
```

## Network Layer

### NetworkResult + safeApiCall

```kotlin
sealed interface NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>
    data class HttpError(val code: Int, val body: String?) : NetworkResult<Nothing>
    data object NetworkUnavailable : NetworkResult<Nothing>
    data class Unknown(val cause: Throwable) : NetworkResult<Nothing>
}

suspend fun <T> safeApiCall(block: suspend () -> T): NetworkResult<T> = try {
    NetworkResult.Success(block())
} catch (e: HttpException) {
    NetworkResult.HttpError(e.code(), e.response()?.errorBody()?.string())
} catch (e: IOException) {
    NetworkResult.NetworkUnavailable
} catch (e: CancellationException) {
    throw e
} catch (e: Throwable) {
    NetworkResult.Unknown(e)
}
```

### Interceptor Stack

```kotlin
@Provides @Singleton
fun okHttp(
    @AuthenticatedToken tokenProvider: TokenProvider,
): OkHttpClient = OkHttpClient.Builder()
    .callTimeout(30, TimeUnit.SECONDS)
    .connectTimeout(10, TimeUnit.SECONDS)
    .addInterceptor(AuthInterceptor(tokenProvider))
    .addInterceptor(HttpLoggingInterceptor().apply {
        level = if (BuildConfig.DEBUG) BODY else NONE
    })
    .addNetworkInterceptor(EtagCacheInterceptor())
    .eventListenerFactory(OkHttpEventListenerFactory(metrics))
    .build()

class AuthInterceptor(private val tokens: TokenProvider) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val req = chain.request().newBuilder()
            .header("Authorization", "Bearer ${tokens.access()}")
            .build()
        val res = chain.proceed(req)
        if (res.code == 401) {
            tokens.refresh()
            res.close()
            return chain.proceed(req.newBuilder().header("Authorization", "Bearer ${tokens.access()}").build())
        }
        return res
    }
}
```

不要用第三方 RetryInterceptor 包退避策略；交給 OkHttp `callTimeout` + WorkManager backoff，更可控。

## Offline-First Repository (SSOT)

```kotlin
class UserRepository @Inject constructor(
    private val remote: UserRemoteDataSource,
    private val local: UserLocalDataSource,
    @IoDispatcher private val io: CoroutineDispatcher,
) {
    fun observeUser(id: String): Flow<User?> = local.observeUser(id)

    suspend fun refresh(id: String): Result<Unit> = withContext(io) {
        when (val r = safeApiCall { remote.fetchUser(id) }) {
            is NetworkResult.Success -> {
                local.upsert(r.data.toEntity())
                Result.success(Unit)
            }
            is NetworkResult.HttpError -> Result.failure(HttpException(r.code))
            is NetworkResult.NetworkUnavailable -> Result.success(Unit)  // 容忍離線
            is NetworkResult.Unknown -> Result.failure(r.cause)
        }
    }
}
```

UI 永遠 collect 自 local；refresh 失敗不影響顯示，只回報為 snackbar/state。

## Sync Manager（WorkManager）

```kotlin
class SyncWorker @AssistedInject constructor(
    @Assisted ctx: Context,
    @Assisted params: WorkerParameters,
    private val userRepo: UserRepository,
    private val orderRepo: OrderRepository,
) : CoroutineWorker(ctx, params) {
    override suspend fun doWork(): Result {
        val results = listOf(
            userRepo.refresh(currentUserId),
            orderRepo.syncPending(),
        )
        return if (results.all { it.isSuccess }) Result.success() else Result.retry()
    }
}

object SyncScheduler {
    fun schedulePeriodic(ctx: Context) {
        val req = PeriodicWorkRequestBuilder<SyncWorker>(30, TimeUnit.MINUTES)
            .setConstraints(Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .setRequiresBatteryNotLow(true)
                .build())
            .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 5, TimeUnit.MINUTES)
            .build()
        WorkManager.getInstance(ctx)
            .enqueueUniquePeriodicWork("sync", ExistingPeriodicWorkPolicy.UPDATE, req)
    }

    fun syncOnConnectivity(ctx: Context) {
        val req = OneTimeWorkRequestBuilder<SyncWorker>()
            .setConstraints(Constraints.Builder().setRequiredNetworkType(NetworkType.CONNECTED).build())
            .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
            .build()
        WorkManager.getInstance(ctx).enqueueUniqueWork("sync_now", ExistingWorkPolicy.KEEP, req)
    }
}
```

**衝突策略**：local 帶 `lastUpdatedAt` 時間戳；同步時 server-wins 或 client-wins 或 merge，依業務寫入 SyncPolicy。

## DataStore：Preferences vs Proto

### Preferences DataStore（簡單 key-value）

```kotlin
val Context.settings by preferencesDataStore(
    name = "settings",
    produceMigrations = { ctx -> listOf(SharedPreferencesMigration(ctx, "old_prefs")) },
)

private val DARK_THEME = booleanPreferencesKey("dark_theme")

class SettingsRepository @Inject constructor(@ApplicationContext private val ctx: Context) {
    val darkTheme: Flow<Boolean> = ctx.settings.data.map { it[DARK_THEME] ?: false }
    suspend fun setDarkTheme(v: Boolean) { ctx.settings.edit { it[DARK_THEME] = v } }
}
```

### Protobuf DataStore（推薦預設）

```proto
// app/src/main/proto/user_settings.proto
syntax = "proto3";
option java_package = "com.example.proto";
option java_multiple_files = true;
message UserSettings {
  bool dark_theme = 1;
  string locale = 2;
  int32 sync_interval_min = 3;
}
```

```kotlin
object UserSettingsSerializer : Serializer<UserSettings> {
    override val defaultValue: UserSettings = UserSettings.getDefaultInstance()
    override suspend fun readFrom(input: InputStream): UserSettings = UserSettings.parseFrom(input)
    override suspend fun writeTo(t: UserSettings, output: OutputStream) = t.writeTo(output)
}

val Context.userSettings by dataStore("user_settings.pb", UserSettingsSerializer)
```

選用：是結構化資料、需版本化、要型別安全 → Proto；單純開關/字串 → Preferences。

## Cross-Skill References

- `@kotlin_multiplatform`：跨 iOS 共享 → SQLDelight + Ktor，本 skill 提供 Android-side 接合。
- `@dependency_injection_mastery`：Repository / DataSource / OkHttpClient 的 Hilt module 設計。
- `@navigation_patterns`：以 type-safe routes 帶資料 ID 而非整個物件。
- `@deep_performance_tuning`：DAO query 慢時用 Macrobenchmark + Perfetto 分析。
- `@release_automation`：API key 與 base URL 注入由它管。

## Quick Checklist

- [ ] Room schema 變更必有 Migration + MigrationTestHelper 測試
- [ ] 所有 API 呼叫包在 `safeApiCall`，回 `NetworkResult<T>`
- [ ] Auth Interceptor 處理 401 refresh，無無限迴圈
- [ ] Repository 為 SSOT；UI 永遠 collect 自 local
- [ ] WorkManager 同步含 backoff + connectivity constraint
- [ ] SharedPreferences 已遷移到 DataStore（Proto 為預設）
- [ ] 大量資料列表用 Paging 3 + LazyPagingItems
- [ ] 跨 KMP 場景已選 Room 2.7+ KMP 或 SQLDelight，並記錄理由
