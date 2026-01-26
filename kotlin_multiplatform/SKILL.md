---
name: Kotlin Multiplatform
description: KMP 跨平台架構、共享邏輯與平台整合
---

# Kotlin Multiplatform (KMP 跨平台)

**Related Scenarios**: F (跨平台共享邏輯)

---

## Architecture Decision

### 共享邊界定義

```
┌─────────────────────────────────────────────────────────┐
│                      Shared (KMP)                       │
├─────────────────────────────────────────────────────────┤
│  Domain Layer    │ Use Cases, Entities, Repository API │
│  Data Layer      │ Repository Impl, API Client, DTOs   │
│  Utilities       │ Date/Time, Validation, Extensions   │
└─────────────────────────────────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Android App   │  │     iOS App     │  │    Web/Desktop  │
│   (Compose UI)  │  │   (SwiftUI)     │  │                 │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

### 不共享的內容

- UI Layer (Compose / SwiftUI)
- Platform-specific APIs (Notifications, Sensors)
- Platform-specific Libraries (WorkManager, HealthKit)

---

## Project Setup

### 目錄結構

```
my-kmp-project/
├── androidApp/
│   ├── build.gradle.kts
│   └── src/main/kotlin/
├── iosApp/
│   └── iosApp.xcodeproj
├── shared/
│   ├── build.gradle.kts
│   └── src/
│       ├── commonMain/kotlin/
│       ├── commonTest/kotlin/
│       ├── androidMain/kotlin/
│       └── iosMain/kotlin/
└── build.gradle.kts
```

### shared/build.gradle.kts

```kotlin
plugins {
    kotlin("multiplatform")
    kotlin("plugin.serialization")
    id("com.android.library")
}

kotlin {
    androidTarget()
    
    listOf(
        iosX64(),
        iosArm64(),
        iosSimulatorArm64()
    ).forEach {
        it.binaries.framework {
            baseName = "shared"
            isStatic = true
        }
    }
    
    sourceSets {
        commonMain.dependencies {
            implementation("io.ktor:ktor-client-core:2.3.7")
            implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
            implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.2")
        }
        
        androidMain.dependencies {
            implementation("io.ktor:ktor-client-okhttp:2.3.7")
        }
        
        iosMain.dependencies {
            implementation("io.ktor:ktor-client-darwin:2.3.7")
        }
    }
}
```

---

## Ktor Client (跨平台 Network)

### Common API

```kotlin
// commonMain
class ApiClient(private val httpClient: HttpClient) {
    
    suspend fun getUser(id: String): User {
        return httpClient.get("https://api.example.com/users/$id").body()
    }
}

// expect/actual for HttpClient
expect fun createHttpClient(): HttpClient

// androidMain
actual fun createHttpClient(): HttpClient = HttpClient(OkHttp) {
    install(ContentNegotiation) {
        json()
    }
}

// iosMain
actual fun createHttpClient(): HttpClient = HttpClient(Darwin) {
    install(ContentNegotiation) {
        json()
    }
}
```

---

## SQLDelight (跨平台 Database)

### 設定

```kotlin
// build.gradle.kts
plugins {
    id("app.cash.sqldelight") version "2.0.1"
}

sqldelight {
    databases {
        create("AppDatabase") {
            packageName.set("com.example.db")
        }
    }
}
```

### Schema

```sql
-- src/commonMain/sqldelight/com/example/db/User.sq
CREATE TABLE user (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT NOT NULL
);

selectAll:
SELECT * FROM user;

insert:
INSERT INTO user(id, name, email) VALUES (?, ?, ?);
```

### expect/actual for Driver

```kotlin
// commonMain
expect class DriverFactory {
    fun createDriver(): SqlDriver
}

// androidMain
actual class DriverFactory(private val context: Context) {
    actual fun createDriver(): SqlDriver {
        return AndroidSqliteDriver(AppDatabase.Schema, context, "app.db")
    }
}

// iosMain
actual class DriverFactory {
    actual fun createDriver(): SqlDriver {
        return NativeSqliteDriver(AppDatabase.Schema, "app.db")
    }
}
```

---

## Testing Shared Code

```kotlin
// commonTest
class UserRepositoryTest {
    
    private val testDriver = JdbcSqliteDriver(JdbcSqliteDriver.IN_MEMORY)
    private val database = AppDatabase(testDriver)
    private val repository = UserRepository(database)
    
    @Test
    fun `insert and retrieve user`() = runTest {
        repository.insertUser(User("1", "Test", "test@example.com"))
        
        val users = repository.getAllUsers()
        
        assertEquals(1, users.size)
        assertEquals("Test", users.first().name)
    }
}
```

---

## Quick Checklist

- [ ] 共享邊界明確 (Domain + Data)
- [ ] UI 保持平台原生
- [ ] Ktor Client 配置 expect/actual
- [ ] SQLDelight 取代 Room (跨平台)
- [ ] commonTest 測試共享邏輯
