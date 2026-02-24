---
id: data_layer_mastery
name: Data Layer Mastery
description: Room 進階用法、Retrofit 整合與 Offline-First 架構
---

# Data Layer Mastery (資料層專精)

## Instructions
- 確認需求屬於資料層（Room、網路、離線策略）
- 依照下方章節順序套用
- 一次只調整一個資料流或責任邊界
- 完成後對照 Quick Checklist

## When to Use
- Scenario A：新專案資料層建立
- Scenario D：效能問題的資料層瓶頸
- Scenario F：KMP 共享資料層設計

## Example Prompts
- "請參考 Room Advanced，幫我設計 Migration 策略"
- "依照 Network Layer 章節，建立統一的錯誤處理"
- "請用 Offline-First 章節檢視目前 Repository 是否符合 SSOT"

## Workflow
1. 先檢查 Room / Network 的基礎設計
2. 再確立 Offline-First 與資料同步策略
3. 最後用 Quick Checklist 驗收

## Practical Notes (2026)
- Offline-first 只在不穩網路或高一致性需求時啟用
- Repository 必須是 SSOT，避免多處來源競爭
- 錯誤處理統一化，避免每層自行判斷

## Minimal Template
```
目標: 
資料來源: 
快取策略: 
錯誤處理: 
驗收: Quick Checklist
```

---

## Room Advanced

### Migration 策略

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("ALTER TABLE users ADD COLUMN avatar_url TEXT")
    }
}

val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(database: SupportSQLiteDatabase) {
        // 複雜遷移：建立新表、複製資料、刪除舊表
        database.execSQL("CREATE TABLE users_new (...)")
        database.execSQL("INSERT INTO users_new SELECT ... FROM users")
        database.execSQL("DROP TABLE users")
        database.execSQL("ALTER TABLE users_new RENAME TO users")
    }
}

Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
    .build()
```

### Paging 3 整合

```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM users ORDER BY name")
    fun pagingSource(): PagingSource<Int, User>
}

// Repository
class UserRepository(private val dao: UserDao) {
    fun getUsers(): Flow<PagingData<User>> = Pager(
        config = PagingConfig(pageSize = 20, prefetchDistance = 5),
        pagingSourceFactory = { dao.pagingSource() }
    ).flow
}

// ViewModel
val users = repository.getUsers().cachedIn(viewModelScope)
```

### Full-Text Search (FTS)

```kotlin
@Fts4(contentEntity = Article::class)
@Entity(tableName = "articles_fts")
data class ArticleFts(
    @ColumnInfo(name = "title") val title: String,
    @ColumnInfo(name = "content") val content: String
)

@Dao
interface ArticleDao {
    @Query("SELECT * FROM articles WHERE rowid IN (SELECT rowid FROM articles_fts WHERE articles_fts MATCH :query)")
    fun search(query: String): Flow<List<Article>>
}
```

---

## Network Layer (Retrofit + OkHttp)

### Error Handling Strategy

```kotlin
sealed class NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>()
    data class Error(val code: Int, val message: String) : NetworkResult<Nothing>()
    data object NetworkError : NetworkResult<Nothing>()
}

suspend fun <T> safeApiCall(apiCall: suspend () -> Response<T>): NetworkResult<T> {
    return try {
        val response = apiCall()
        if (response.isSuccessful) {
            NetworkResult.Success(response.body()!!)
        } else {
            NetworkResult.Error(response.code(), response.message())
        }
    } catch (e: IOException) {
        NetworkResult.NetworkError
    }
}
```

### Interceptors

```kotlin
// Logging
val loggingInterceptor = HttpLoggingInterceptor().apply {
    level = if (BuildConfig.DEBUG) BODY else NONE
}

// Auth Token
class AuthInterceptor(private val tokenProvider: TokenProvider) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request().newBuilder()
            .addHeader("Authorization", "Bearer ${tokenProvider.token}")
            .build()
        return chain.proceed(request)
    }
}

// Retry
class RetryInterceptor(private val maxRetries: Int = 3) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        var attempt = 0
        var response: Response? = null
        while (attempt < maxRetries) {
            try {
                response = chain.proceed(chain.request())
                if (response.isSuccessful) return response
            } catch (e: IOException) {
                attempt++
                if (attempt >= maxRetries) throw e
            }
        }
        return response!!
    }
}
```

---

## Offline-First Architecture

### Repository Pattern (SSOT)

```kotlin
class UserRepository(
    private val remoteDataSource: UserRemoteDataSource,
    private val localDataSource: UserLocalDataSource
) {
    fun getUser(id: String): Flow<User> = flow {
        // 1. 先從 Local 發射
        localDataSource.getUser(id)?.let { emit(it) }
        
        // 2. 從 Remote 取得最新
        val remote = remoteDataSource.fetchUser(id)
        
        // 3. 存入 Local
        localDataSource.saveUser(remote)
        
        // 4. 發射更新後的資料
        emit(remote)
    }
    
    // 或使用 NetworkBoundResource pattern
    fun getUserWithCache(id: String): Flow<Resource<User>> = networkBoundResource(
        query = { localDataSource.getUserFlow(id) },
        fetch = { remoteDataSource.fetchUser(id) },
        saveFetchResult = { localDataSource.saveUser(it) },
        shouldFetch = { it == null || it.isStale() }
    )
}
```

---

## DataStore Migration

### SharedPreferences → Preferences DataStore

```kotlin
val Context.dataStore by preferencesDataStore(
    name = "settings",
    produceMigrations = { context ->
        listOf(SharedPreferencesMigration(context, "old_prefs"))
    }
)

// 使用
val themeKey = booleanPreferencesKey("dark_theme")

suspend fun setDarkTheme(enabled: Boolean) {
    context.dataStore.edit { prefs ->
        prefs[themeKey] = enabled
    }
}

val darkThemeFlow: Flow<Boolean> = context.dataStore.data
    .map { it[themeKey] ?: false }
```

---

## Quick Checklist

- [ ] Room Migration 測試通過
- [ ] Network Error 統一處理
- [ ] Repository 實作 SSOT
- [ ] DataStore 取代 SharedPreferences
- [ ] Paging 用於大量資料列表
