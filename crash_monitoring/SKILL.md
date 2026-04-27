---
id: crash_monitoring
name: Crash Monitoring
description: Android Crash/ANR/結構化日誌的「實作層」 — Firebase Crashlytics 33.x、Custom Signals/OnFatalException、Android 14 ANR Watchdog（含 foreground threshold）、Memory Warning、Timber、Play Console + Firebase 警報聯動，產出可定位的事件流
type: skill
---

# Crash Monitoring

## Instructions

當問題涉及 Crashlytics 設定、ANR 捕捉、結構化日誌、警報聯動的「實作」時載入。指標策略、SLO、告警門檻設計請載入 `@observability_strategy`，本 skill 只負責「怎麼裝、怎麼接、怎麼讓 dashboard 收到」。

## When to Use

- Scenario D：效能問題排查
- Scenario E：發布前監控準備
- 套用 Crashlytics Custom Signals 與 OnFatalException 新 API
- ANR Watchdog 升級對應 Android 14+ foreground 閾值變更
- Memory Warning / Background Crash 等冷門信號
- Play Console + Firebase 雙向警報聯動

## When NOT to Use

- SLO 設計、告警閾值矩陣、event schema 設計 → `@observability_strategy`
- 效能根因（Macrobenchmark / Perfetto） → `@deep_performance_tuning`
- CI rollout 自動 halt → `@release_automation`
- App 內加密/防竄改信號 → `@release_automation`

## Example Prompts

- 「Crashlytics 33 怎麼裝？BuildConfig 與 mapping.txt 上傳要怎麼設？」
- 「想接 Crashlytics Custom Signals 紀錄 build variant 與 user tier」
- 「Android 14 之後 ANR Watchdog 的 foreground threshold 改了，要怎麼處理？」
- 「Memory Warning 信號怎麼接、要不要當 non-fatal 上報？」
- 「Play Console 看到 ANR 暴增但 Firebase 沒警報，怎麼聯動？」

## Workflow

1. **SDK 安裝**：BOM 33.x、`firebase.crashlytics` plugin、native symbols（NDK）。
2. **Mapping & Symbols**：CI 自動上傳 R8 mapping.txt 與 NDK 符號表。
3. **User & Custom Keys**：登入後 `setUserId(hash)`；常駐 Custom Keys（tier、locale、ab buckets）。
4. **ANR / Memory / Background** 三類冷門信號接 watchdog。
5. **Timber tree** 統一 log 並橋到 Crashlytics non-fatal。
6. **Velocity / Custom alerts** 在 Firebase Console + Play Console 雙設定，互校。

## Practical Notes (2026-04)

| 元件 | 版本 | 備註 |
|------|------|------|
| Firebase BOM | 33.x | Crashlytics 19+, Analytics 22+ |
| Crashlytics Plugin | 3.0+ | 含 `OnFatalException` API |
| Custom Signals | 33.x | 取代部分 Custom Keys 用法 |
| ANR Watchdog（自寫） | — | Android 14+ foreground 5s、background 10s |
| Timber | 5.x | Kotlin extensions |
| Play Console Vitals | 持續更新 | 為 ANR rate 真相之源（24-48h 延遲） |

## Minimal Template

```
SDK: Firebase BOM 33.x + Crashlytics 19.x + NDK symbols
User scope: setUserId(SHA-256(uid)), 不存原值
Custom Signals: build, build_type, locale, user_tier, ab_buckets
ANR Watchdog: 5s foreground / 10s background
Memory: ComponentCallbacks2 + onLowMemory upload
Logger: Timber + CrashlyticsTree (INFO+ → log; WARN+ → non-fatal)
Alerts: Crashlytics velocity 0.5% + Play Console threshold + Slack hook
驗收: Quick Checklist
```

## Crashlytics 33.x 安裝

```kotlin
// build.gradle.kts (root)
plugins {
    id("com.google.gms.google-services") version "4.4.2" apply false
    id("com.google.firebase.crashlytics") version "3.0.2" apply false
}

// app/build.gradle.kts
plugins {
    id("com.google.gms.google-services")
    id("com.google.firebase.crashlytics")
}

android {
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
        }
    }
}

dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.7.0"))
    implementation("com.google.firebase:firebase-crashlytics")
    implementation("com.google.firebase:firebase-crashlytics-ndk")  // 若有 native
    implementation("com.google.firebase:firebase-analytics")
}

// CI 上傳 mapping.txt 自動處理：crashlytics plugin 在 :app:assembleRelease 時上傳
// NDK symbols：./gradlew :app:uploadCrashlyticsSymbolFileRelease
```

## Custom Keys / Custom Signals

```kotlin
class CrashlyticsUserScope @Inject constructor(
    private val crashlytics: FirebaseCrashlytics,
) {
    fun bind(user: User?) {
        if (user == null) {
            crashlytics.setUserId("")
            return
        }
        crashlytics.setUserId(user.id.sha256())   // 不存原值
        crashlytics.setCustomKeys {
            key("user_tier", user.tier.name)
            key("locale", Locale.getDefault().toLanguageTag())
            key("build", BuildConfig.VERSION_NAME)
            key("build_type", BuildConfig.BUILD_TYPE)
        }
    }
}
```

### Custom Signals（Crashlytics 33+）

```kotlin
// 替代部分 setCustomKey；Custom Signals 可在 Crashlytics 控制台快速 filter
crashlytics.setCustomKeys {
    key("ab_checkout_v3", "treatment")
    key("network", connectivityType())
}
```

### 流程上下文 log

```kotlin
crashlytics.log("checkout: payment_started method=$method amount=$amount")
try {
    processPayment()
} catch (e: PaymentException) {
    crashlytics.setCustomKeys {
        key("payment_amount", amount.toLong())
        key("payment_method", method.name)
    }
    crashlytics.recordException(e)
}
```

## OnFatalException（Crashlytics 33 新 API）

```kotlin
class CrashlyticsBootstrap @Inject constructor(
    private val crashlytics: FirebaseCrashlytics,
    private val sessionEnder: SessionEnder,
) {
    fun start() {
        crashlytics.setOnFatalCrashListener { event ->
            // 在進程被終結前同步收尾：flush 重要 buffer、釋放 file lock
            sessionEnder.flushSync()
        }
    }
}
```

注意：listener 內 **只能做同步、不可 IO 阻塞長時間**；JVM 將立刻 die。

## ANR Analysis

### StrictMode（debug only）

```kotlin
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.DEBUG) installStrictMode()
        installAnrWatchdog()
        installMemoryCallback()
    }
}

private fun installStrictMode() {
    StrictMode.setThreadPolicy(
        StrictMode.ThreadPolicy.Builder()
            .detectDiskReads()
            .detectDiskWrites()
            .detectNetwork()
            .penaltyLog()
            .penaltyDeath()
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
```

### ANR Watchdog（Android 14+ foreground 5 秒閾值）

```kotlin
class AnrWatchdog(
    private val foregroundTimeoutMs: Long = 5_000,
    private val backgroundTimeoutMs: Long = 10_000,
    private val onAnr: (StackTraceElement[]) -> Unit,
) : Thread("anr-watchdog") {
    private val mainHandler = Handler(Looper.getMainLooper())
    @Volatile private var tick = 0L
    @Volatile private var inForeground = true

    fun setForeground(value: Boolean) { inForeground = value }

    override fun run() {
        while (!isInterrupted) {
            val current = tick
            mainHandler.post { tick++ }
            val sleep = if (inForeground) foregroundTimeoutMs else backgroundTimeoutMs
            try { sleep(sleep) } catch (_: InterruptedException) { return }
            if (current == tick) {
                val stack = Looper.getMainLooper().thread.stackTrace
                onAnr(stack)
                // 等待恢復後再重置
                while (current == tick && !isInterrupted) try { sleep(500) } catch (_: InterruptedException) { return }
            }
        }
    }
}

class AnrException(stack: Array<StackTraceElement>) : RuntimeException("ANR detected") {
    init { stackTrace = stack }
}
```

接到 Application + ProcessLifecycleOwner：

```kotlin
val watchdog = AnrWatchdog { stack ->
    crashlytics.log("ANR detected")
    crashlytics.recordException(AnrException(stack))
}.apply { isDaemon = true; start() }

ProcessLifecycleOwner.get().lifecycle.addObserver(object : DefaultLifecycleObserver {
    override fun onStart(owner: LifecycleOwner) { watchdog.setForeground(true) }
    override fun onStop(owner: LifecycleOwner) { watchdog.setForeground(false) }
})
```

Android 14+ 系統 ANR 偵測 foreground 閾值由 5 秒改為以 input + broadcast 等為主；自家 watchdog 仍以 5s 為基準對齊使用者感受。

## Memory Warning

```kotlin
class App : Application(), ComponentCallbacks2 {
    override fun onTrimMemory(level: Int) {
        super.onTrimMemory(level)
        if (level >= ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL) {
            crashlytics.log("memory_critical level=$level rss=${getRssKb()}KB")
            // 不上報為 non-fatal（量太大），改成事件
            analytics.logEvent("memory_warning") {
                param("level", level.toLong())
                param("rss_kb", getRssKb())
            }
        }
    }
}

private fun getRssKb(): Long =
    runCatching { File("/proc/self/status").readLines().firstOrNull { it.startsWith("VmRSS:") }
        ?.replace(Regex("\\D+"), "")?.toLongOrNull() }.getOrNull() ?: 0L
```

## Background Crash 加旗標

背景啟動時 crash 與 foreground crash 的優先級不同；前景 crash 對體驗更敏感。

```kotlin
crashlytics.setCustomKeys {
    key("app_state", if (ProcessLifecycleOwner.get().lifecycle.currentState.isAtLeast(Lifecycle.State.STARTED)) "foreground" else "background")
}
```

## Structured Logging（Timber + CrashlyticsTree）

```kotlin
class App : Application() {
    @Inject lateinit var crashlyticsTree: CrashlyticsTree
    override fun onCreate() {
        super.onCreate()
        Timber.plant(if (BuildConfig.DEBUG) Timber.DebugTree() else crashlyticsTree)
    }
}

class CrashlyticsTree @Inject constructor(
    private val crashlytics: FirebaseCrashlytics,
) : Timber.Tree() {
    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        if (priority < Log.INFO) return
        crashlytics.log("[$tag] $message")
        if (priority >= Log.WARN && t != null) {
            crashlytics.recordException(t)
        }
    }
}
```

| Level | 使用場景 |
|-------|----------|
| VERBOSE | API body / state machine trace（debug only） |
| DEBUG | 開發除錯 |
| INFO | 重要里程碑（login success / payment started） |
| WARN | 潛在問題（retry / cache miss） |
| ERROR | 可恢復錯誤（network timeout） |

避免在 `Timber.e()` 帶整個物件 `toString()`，太肥；只帶必要 key。

## Dashboard & Alerting

### Crashlytics Velocity Alert

Firebase Console → Crashlytics → Settings → Velocity alerts：

- Alert when crash rate > 0.5% over 1 hour（P0）
- Alert when ANR rate > 0.5% over 6 hours（P1）

### Crashlytics Issue Alert

新 issue 出現 + impact > N users → Slack via webhook。

### Play Console Vitals 雙保險

```yaml
# .github/workflows/play-vitals-watch.yml
on: { schedule: [{ cron: '0 */2 * * *' }] }
jobs:
  check:
    runs-on: ubuntu-24.04
    steps:
      - uses: google-github-actions/auth@v2
      - run: |
          rate=$(gcloud play androidpublisher v3 ... metrics anrRate)
          if (( $(echo "$rate > 0.47" | bc -l) )); then
            curl -X POST $SLACK_WEBHOOK -d '{"text":"Play Console ANR rate '$rate'% above Vitals良好門檻"}'
          fi
```

Play Console 有 24-48h 延遲，但分母真實；Firebase 即時但採樣可能不全。兩者比對發現偏差 → 檢查 SDK 初始化失敗或事件去重問題（→ `@observability_strategy`）。

## Cross-Skill References

- `@observability_strategy`：SLO 設計、告警閾值矩陣、event schema、feedback loop。本 skill 是其「實作層」。
- `@deep_performance_tuning`：根因定位（Macrobenchmark / Perfetto）。
- `@release_automation`：rollout 自動 halt、Play Console 操作；本 skill 提供觸發信號。
- `@supply_chain_security`：mapping.txt 上傳鏈、簽章 token；本 skill 只用結果。
- `@platform_modernization_2026`：Android 14/15 行為變更（ANR foreground threshold、ForegroundServiceTypeException）。

## Quick Checklist

- [ ] Firebase BOM 33.x，Crashlytics + NDK symbols 安裝
- [ ] R8 mapping.txt 與 NDK symbols 在 CI 自動上傳
- [ ] `setUserId` 用 hash，不存原 PII
- [ ] Custom Keys / Signals：build / tier / locale / ab_buckets 常駐
- [ ] ANR Watchdog 含 foreground 5s / background 10s 閾值
- [ ] StrictMode 僅 debug 開啟
- [ ] Memory Warning 接 ComponentCallbacks2，rss 寫入事件
- [ ] Background vs Foreground crash 區分（custom key `app_state`）
- [ ] Timber + CrashlyticsTree：INFO 進 log，WARN+ 進 non-fatal
- [ ] Velocity Alert 0.5% 設定 + Play Console Vitals 警報雙設定 + Slack hook
- [ ] Play Console vs Crashlytics 數據定期對校（差異 > 10% 警示）
