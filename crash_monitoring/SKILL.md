---
name: Crash Monitoring
description: Crashlytics 設定、ANR 分析與結構化日誌
---

# Crash Monitoring (當機監控)

**Related Scenarios**: D (效能問題), E (發布準備)

---

## Firebase Crashlytics Setup

### 基本設定

```kotlin
// build.gradle.kts (app)
plugins {
    id("com.google.firebase.crashlytics")
}

dependencies {
    implementation(platform("com.google.firebase:firebase-bom:32.7.0"))
    implementation("com.google.firebase:firebase-crashlytics-ktx")
    implementation("com.google.firebase:firebase-analytics-ktx")
}
```

### Custom Keys

```kotlin
// 記錄使用者狀態
Firebase.crashlytics.apply {
    setUserId("user_123")
    setCustomKey("subscription_tier", "premium")
    setCustomKey("feature_flags", "new_checkout:true")
}
```

### Non-Fatal Logging

```kotlin
// 記錄非致命錯誤
try {
    riskyOperation()
} catch (e: Exception) {
    Firebase.crashlytics.recordException(e)
    // 繼續正常流程
}

// 帶上下文的 Non-Fatal
Firebase.crashlytics.log("Starting payment flow")
try {
    processPayment()
} catch (e: PaymentException) {
    Firebase.crashlytics.apply {
        setCustomKey("payment_amount", amount)
        setCustomKey("payment_method", method)
        recordException(e)
    }
}
```

---

## ANR Analysis

### StrictMode (Debug)

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        
        if (BuildConfig.DEBUG) {
            StrictMode.setThreadPolicy(
                StrictMode.ThreadPolicy.Builder()
                    .detectDiskReads()
                    .detectDiskWrites()
                    .detectNetwork()
                    .penaltyLog()
                    .penaltyDeath()  // 強制 Crash
                    .build()
            )
            
            StrictMode.setVmPolicy(
                StrictMode.VmPolicy.Builder()
                    .detectLeakedClosableObjects()
                    .detectActivityLeaks()
                    .penaltyLog()
                    .build()
            )
        }
    }
}
```

### ANR Watchdog

```kotlin
class ANRWatchDog(private val timeoutMs: Long = 5000) : Thread() {
    
    private val mainHandler = Handler(Looper.getMainLooper())
    private var tick = 0
    private var reported = false
    
    override fun run() {
        while (!isInterrupted) {
            val currentTick = tick
            mainHandler.post { tick++ }
            
            Thread.sleep(timeoutMs)
            
            if (currentTick == tick && !reported) {
                reported = true
                val stackTraces = Looper.getMainLooper().thread.stackTrace
                Firebase.crashlytics.log("ANR Detected")
                Firebase.crashlytics.recordException(ANRException(stackTraces))
            }
        }
    }
}
```

---

## Structured Logging

### Timber Setup

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        
        if (BuildConfig.DEBUG) {
            Timber.plant(Timber.DebugTree())
        } else {
            Timber.plant(CrashlyticsTree())
        }
    }
}

class CrashlyticsTree : Timber.Tree() {
    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        if (priority >= Log.INFO) {
            Firebase.crashlytics.log("[$tag] $message")
        }
        
        t?.let {
            Firebase.crashlytics.recordException(it)
        }
    }
}
```

### Log Levels 使用規範

| Level | 使用場景 | 例子 |
|-------|---------|------|
| VERBOSE | Debug 專用細節 | API Response body |
| DEBUG | 開發除錯 | ViewModel state changes |
| INFO | 重要里程碑 | User login success |
| WARN | 潛在問題 | Retry attempt |
| ERROR | 可恢復錯誤 | Network timeout |

---

## Dashboard & Alerting

### Crashlytics Velocity Alerts

```
Firebase Console > Crashlytics > Settings
- Enable velocity alerts
- Set threshold: 1% of sessions
```

### Custom Metrics

```kotlin
// 追蹤關鍵指標
Firebase.analytics.logEvent("checkout_started") {
    param("cart_value", cartValue)
}

Firebase.analytics.logEvent("checkout_completed") {
    param("payment_method", method)
    param("time_to_complete_ms", duration)
}
```

---

## Quick Checklist

- [ ] Crashlytics 整合完成
- [ ] Custom Keys 設定 (User tier, Feature flags)
- [ ] Non-Fatal 記錄重要異常
- [ ] StrictMode 在 Debug 啟用
- [ ] Timber 統一日誌
- [ ] Velocity Alerts 設定
