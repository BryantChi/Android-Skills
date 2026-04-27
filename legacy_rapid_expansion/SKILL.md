---
id: legacy_rapid_expansion
name: Legacy Rapid Expansion
description: 在無法全面重構的舊 Android 專案中，以 Islanding 策略快速建立新功能 — Bridge Pattern、Hybrid Theming（XML↔Material 3）、Wrapper Activities、Remote Config 控制 Feature Toggle、Hybrid 過渡期效能監控
type: skill
---

# Legacy Rapid Expansion

## Instructions

當專案有大量舊 View/Fragment/Dagger/RxJava 但 deadline 不允許全面重構，需要在「孤島」中快速做新功能時載入。本 skill 不取代重構（→ `@tech_stack_migration`），而是讓新功能能用現代 stack 同時不污染 / 不被舊架構污染。

## When to Use

- Scenario B：舊專案加新功能
- Sprint 內要交付新模組，但全面重構無 ROI
- 想試水 Compose / KSP / Hilt 但全 repo 改動風險太高
- 平行兩種架構共存期，需要清楚邊界與下線策略

## When NOT to Use

- 有時間做漸進式正規重構 → `@tech_stack_migration`
- 為現有舊代碼補測試 → `@testing_legacy_strategies`
- 從零開始新專案 → `@project_bootstrapping`
- 純 UI 互通（XML 與 Compose）→ 看 `@tech_stack_migration` 的「View↔Compose」

## Example Prompts

- 「老 App 想加全新會員中心，但其他畫面還是 RxJava+Fragment，怎麼隔出來做 Compose+Hilt？」
- 「Bridge Pattern 怎麼設計才能讓舊代碼喊新模組，又不引入循環依賴？」
- 「新 Compose 畫面想沿用舊 XML Theme，要怎麼做才不違和？」
- 「Feature flag 用 Remote Config 怎麼設？要支援快速回退」
- 「孤島內的新功能想單獨監控 crash 與效能，怎麼分開？」

## Workflow

1. **劃定淨土邊界**：建 `modern/` 套件，明訂「淨土內不准 import 舊 package」。
2. **Bridge 契約**：`bridge/` 提供 `ModernEntryPoint`、`LegacyHostNavigator`，雙向溝通只走介面。
3. **Hybrid Theme**：新 Compose 沿用舊 XML token 一段時間，逐 token 換新。
4. **Wrapper Activity**：舊 `startActivity` → ComposeWrapperActivity → 新 Composable。
5. **Feature Toggle**：所有入口走 flag，預設關閉，灰度開啟。
6. **島內監控**：自家 SDK 標 `feature = "modern_membership"` 維度，獨立看 crash-free 與效能。

## Practical Notes (2026-04)

| 元件 | 推薦 |
|------|------|
| 新 island UI | Compose BOM 2025.04 + Material 3 Expressive |
| 新 island DI | Hilt 2.52 + KSP（與舊 Dagger 共存，見 `@tech_stack_migration`） |
| 新 island stream | Coroutines/Flow（不引 RxJava） |
| Theme bridge | 自寫 `ColorScheme.fromXmlAttrs()` helper（MDC compose-theme-adapter 已棄用） |
| Feature Toggle | Firebase Remote Config + 本地預設值 + DI 介面 |
| 島內 Observability | 共用全域 SDK，但 attribute 標 `feature` 維度 |

## Minimal Template

```
淨土範圍: app/modern/ + bridge/
新 stack: Compose + Hilt(KSP) + Coroutines + StateFlow
Bridge 介面:
  - ModernEntryPoint: 舊→新 啟動入口
  - LegacyHostNavigator: 新→舊 回到舊頁面
  - SharedSession: 雙方共用的 user/token
Theme: XmlBridgeTheme (一段時間) → ModernAppTheme
Toggle: FeatureFlags.useNewMembership (Remote Config)
監控: feature_attr = "modern_membership"，獨立看 crash-free
驗收: Quick Checklist
```

## Islanding Architecture

### 目錄結構

```
app/src/main/kotlin/com/example/
├── legacy/                       # 舊代碼，凍結
│   ├── activities/
│   ├── fragments/
│   └── di/                       # 舊 Dagger 模組
├── modern/                       # 淨土
│   ├── core/
│   │   ├── data/
│   │   ├── domain/
│   │   └── ui/                   # ModernAppTheme
│   ├── feature/
│   │   └── membership/
│   └── di/                       # Hilt 模組（@InstallIn(SingletonComponent::class)）
└── bridge/                       # 雙向橋
    ├── ModernEntryPoint.kt
    ├── LegacyHostNavigator.kt
    ├── SharedSessionBridge.kt
    └── XmlBridgeTheme.kt
```

### 邊界規則（CI gate）

```bash
# tools/check-island-boundary.sh
violations=$(grep -RnE "import com\.example\.legacy" app/src/main/kotlin/com/example/modern/ || true)
[ -z "$violations" ] || { echo "島內不允許 import legacy:"; echo "$violations"; exit 1; }
```

接到 `@release_automation` 的 CI gate，違反就阻擋 merge。

## Bridge Pattern

### 舊 → 新（ModernEntryPoint）

```kotlin
object ModernEntryPoint {
    fun startMembership(context: Context, userId: String) {
        val intent = ComposeHostActivity.intent(context, "membership", bundleOf("uid" to userId))
        context.startActivity(intent)
    }
}

// 舊 Activity 呼叫
class LegacyHomeActivity : AppCompatActivity() {
    private fun onMembershipClick() {
        ModernEntryPoint.startMembership(this, currentUser.id)
    }
}
```

### 新 → 舊（LegacyHostNavigator）

```kotlin
interface LegacyHostNavigator {
    fun toLegacyOrders(activity: Activity)
    fun toLegacySettings(activity: Activity)
}

class LegacyHostNavigatorImpl @Inject constructor() : LegacyHostNavigator {
    override fun toLegacyOrders(activity: Activity) {
        activity.startActivity(Intent(activity, LegacyOrdersActivity::class.java))
    }
    // ...
}

@Module @InstallIn(SingletonComponent::class)
abstract class BridgeModule {
    @Binds abstract fun bind(impl: LegacyHostNavigatorImpl): LegacyHostNavigator
}
```

新代碼絕不直接 `Intent(this, LegacyXxxActivity::class.java)`，避免日後刪舊頁面時島內也要改。

### Shared Session

```kotlin
@Singleton
class SharedSessionBridge @Inject constructor(
    private val legacyAuth: LegacyAuthManager,   // 舊 Dagger 提供，透過 Hilt LegacyBridgeModule 暴露
) {
    val currentUser: Flow<User?> = legacyAuth.userObservable
        .toFlow()                                // RxJava → Flow，避免島內依 RxJava
        .map { it?.toModern() }
}
```

## Hybrid Theming（不依賴 MDC compose-theme-adapter）

### 從 XML attrs 建 Compose ColorScheme

```kotlin
@Composable
fun XmlBridgeTheme(content: @Composable () -> Unit) {
    val ctx = LocalContext.current
    val colorScheme = remember(ctx) { resolveColorSchemeFromXml(ctx) }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = bridgedTypography(ctx),
        shapes = MaterialTheme.shapes,
        content = content,
    )
}

private fun resolveColorSchemeFromXml(ctx: Context): ColorScheme {
    val a = ctx.obtainStyledAttributes(intArrayOf(
        com.google.android.material.R.attr.colorPrimary,
        com.google.android.material.R.attr.colorOnPrimary,
        com.google.android.material.R.attr.colorSurface,
        com.google.android.material.R.attr.colorOnSurface,
    ))
    val isDark = (ctx.resources.configuration.uiMode and Configuration.UI_MODE_NIGHT_MASK) == Configuration.UI_MODE_NIGHT_YES
    val base = if (isDark) darkColorScheme() else lightColorScheme()
    return base.copy(
        primary = Color(a.getColor(0, base.primary.toArgb())),
        onPrimary = Color(a.getColor(1, base.onPrimary.toArgb())),
        surface = Color(a.getColor(2, base.surface.toArgb())),
        onSurface = Color(a.getColor(3, base.onSurface.toArgb())),
    ).also { a.recycle() }
}
```

### 漸進淘汰

```
Phase 1: 全島 @XmlBridgeTheme
Phase 2: 新元件用 ModernAppTheme，殘留舊元件包 XmlBridgeTheme
Phase 3: 全島 ModernAppTheme，移除 bridge
```

## Wrapper Activity

```kotlin
@AndroidEntryPoint
class ComposeHostActivity : ComponentActivity() {
    override fun onCreate(s: Bundle?) {
        super.onCreate(s)
        enableEdgeToEdge()
        val screen = intent.getStringExtra(EXTRA_SCREEN) ?: "home"
        val args = intent.extras ?: Bundle.EMPTY
        setContent {
            ModernAppTheme {
                when (screen) {
                    "membership" -> MembershipScreen(uid = args.getString("uid").orEmpty())
                    "settings" -> SettingsScreen()
                    else -> ErrorScreen()
                }
            }
        }
    }
    companion object {
        private const val EXTRA_SCREEN = "screen"
        fun intent(ctx: Context, screen: String, args: Bundle = Bundle.EMPTY) =
            Intent(ctx, ComposeHostActivity::class.java).putExtra(EXTRA_SCREEN, screen).putExtras(args)
    }
}
```

## Feature Toggle（Remote Config）

```kotlin
interface FeatureFlags {
    val useNewMembership: Boolean
    val newMembershipPercent: Long       // 0-100，灰度
}

class RemoteFeatureFlags @Inject constructor(
    private val remote: FirebaseRemoteConfig,
    private val userIdProvider: UserIdProvider,
) : FeatureFlags {
    init {
        remote.setDefaultsAsync(mapOf(
            "use_new_membership" to false,
            "new_membership_percent" to 0L,
        ))
        remote.setConfigSettingsAsync(remoteConfigSettings {
            minimumFetchIntervalInSeconds = 600
        })
    }

    override val useNewMembership: Boolean
        get() {
            if (remote.getBoolean("use_new_membership")) return true
            val pct = remote.getLong("new_membership_percent").coerceIn(0, 100)
            val bucket = (userIdProvider.userIdHash() % 100)
            return bucket < pct
        }

    override val newMembershipPercent: Long get() = remote.getLong("new_membership_percent")
}
```

```kotlin
class MembershipNavigator @Inject constructor(
    private val flags: FeatureFlags,
    private val legacy: LegacyHostNavigator,
) {
    fun open(activity: Activity, userId: String) {
        if (flags.useNewMembership) {
            ModernEntryPoint.startMembership(activity, userId)
        } else {
            legacy.toLegacyMembership(activity, userId)
        }
    }
}
```

灰度策略：default 0% → 1% → 5% → 25% → 100%；每段觀察島內 crash-free 與關鍵指標。

## Hybrid 過渡期效能監控

新島應有獨立可觀測性，避免被舊代碼噪音淹沒：

```kotlin
class IslandAttributesPlugin @Inject constructor(
    private val crashlytics: FirebaseCrashlytics,
    private val analytics: FirebaseAnalytics,
) {
    fun mark(feature: String) {
        crashlytics.setCustomKey("feature_island", feature)
        analytics.setUserProperty("feature_island", feature)
    }
}

@Composable
fun MembershipScreen(uid: String) {
    val plugin = hiltViewModel<MembershipViewModel>().islandPlugin
    DisposableEffect(Unit) {
        plugin.mark("modern_membership")
        onDispose { plugin.mark("legacy") }
    }
    // ...
}
```

對應的 SLO/告警設計：請載入 `@observability_strategy`；事件實作層：`@crash_monitoring`。

## 收島時機

當下列條件全達成，可移除 bridge / 把舊代碼下線：

- [ ] Feature Toggle 100% 灰度且 ≥ 1 個月無回退
- [ ] 島內 crash-free 與舊路徑相當或更佳
- [ ] 所有 entry point 改用新路徑（grep 確認 `LegacyMembershipActivity` 無引用）
- [ ] Bridge 介面有明確刪除日期且寫入 README

收島步驟交給 `@tech_stack_migration`。

## Cross-Skill References

- `@tech_stack_migration`：完整重構與技術替換的權威。本 skill 是「先用」的策略。
- `@testing_legacy_strategies`：舊代碼測試安全網。
- `@dependency_injection_mastery`：Hilt 與 Dagger 共存的 module 設計。
- `@navigation_patterns`：島內 Compose Navigation 設定。
- `@ui_ux_engineering`：ModernAppTheme 設計（Material 3 Expressive）。
- `@observability_strategy` + `@crash_monitoring`：島內獨立指標。

## Quick Checklist

- [ ] `modern/` 與 `legacy/` 邊界清晰，CI 阻擋 import 違規
- [ ] Bridge 雙向介面（ModernEntryPoint / LegacyHostNavigator / SharedSession）就位
- [ ] 新 Compose 畫面以 `XmlBridgeTheme` 沿用舊 token，計畫淘汰時間表
- [ ] ComposeHostActivity 一處集中接 deep link / wrapper
- [ ] 所有新功能入口走 FeatureFlags，預設關閉
- [ ] Remote Config 含百分比灰度與 user bucket
- [ ] 島內 attribute 已掛 `feature_island`，crash-free 獨立可看
- [ ] 新代碼禁用 RxJava / Dagger 直接 import；只透過 bridge
- [ ] 收島條件清單寫入 README，bridge 有明確刪除日期
