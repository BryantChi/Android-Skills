---
name: Legacy Rapid Expansion
description: 在舊架構中快速建立新功能的 Islanding 策略
---

# Legacy Rapid Expansion (舊專案快速擴充)

## Instructions
- 僅在舊專案加入新功能時使用
- 依照下方章節順序套用
- 一次只建立一個隔離區與橋接點
- 完成後對照 Quick Checklist

## When to Use
- Scenario B：舊專案加新功能

## Example Prompts
- "請依照 Islanding Architecture 建立 modern/ 與 bridge/"
- "用 Hybrid Theming 章節讓新 Compose UI 沿用舊 Theme"
- "請用 Feature Toggle 章節設計新功能開關"

## Workflow
1. 先建立 Islanding Architecture 與 Bridge Pattern
2. 再處理 Hybrid Theming 與 Wrapper Activities
3. 最後用 Feature Toggle 與 Quick Checklist 驗收

## Practical Notes (2026)
- 新功能必須獨立於 legacy 內部實作
- Bridge API 需穩定，避免頻繁變更造成擴散
- Feature Toggle 作為上線與回退的唯一入口

## Minimal Template
```
目標: 
隔離範圍: 
Bridge 契約: 
Toggle 策略: 
驗收: Quick Checklist
```

---

## Islanding Architecture (孤島策略)

在舊架構中切出一塊「淨土」，新功能完全使用現代架構開發。

### 目錄結構

```
app/
├── legacy/                # 舊代碼 (不動)
│   ├── activities/
│   └── fragments/
├── modern/                # 新代碼 (淨土)
│   ├── core/
│   │   ├── data/
│   │   ├── domain/
│   │   └── ui/
│   └── feature/
│       └── newfeature/
└── bridge/                # 橋接層
    ├── LegacyNavigator.kt
    └── ModernEntryPoint.kt
```

### Bridge Pattern

```kotlin
// bridge/ModernEntryPoint.kt
object ModernEntryPoint {
    
    fun startNewFeatureActivity(context: Context, params: Bundle) {
        val intent = Intent(context, NewFeatureActivity::class.java).apply {
            putExtras(params)
        }
        context.startActivity(intent)
    }
    
    @Composable
    fun NewFeatureScreen(params: Map<String, Any>) {
        // 現代 Compose UI
    }
}

// 舊代碼呼叫
class LegacyActivity : AppCompatActivity() {
    fun onButtonClick() {
        ModernEntryPoint.startNewFeatureActivity(this, bundleOf("id" to productId))
    }
}
```

---

## Hybrid Theming

讓 Compose UI 沿用舊有的 XML Theme。

### MDC-Android Compose Theme Adapter

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.google.android.material:compose-theme-adapter:1.2.1")
}

// 使用
@Composable
fun NewFeatureScreen() {
    MdcTheme {  // 自動橋接 XML Theme
        Surface {
            // Compose UI 會使用 XML 定義的 colors/typography
        }
    }
}
```

### 漸進式遷移

```kotlin
// 1. 初期：完全沿用 XML Theme
MdcTheme { content() }

// 2. 中期：覆寫部分 Token
MdcTheme(
    setTextColors = true,
    setDefaultFontFamily = true
) { content() }

// 3. 後期：完全使用 Compose Theme
AppTheme { content() }
```

---

## Wrapper Activities

快速將 Compose Screen 包裝供舊有 `startActivity` 呼叫。

### 通用 Wrapper

```kotlin
@AndroidEntryPoint
class ComposeWrapperActivity : ComponentActivity() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        val screenType = intent.getStringExtra(EXTRA_SCREEN_TYPE)
        val params = intent.extras ?: Bundle.EMPTY
        
        setContent {
            AppTheme {
                when (screenType) {
                    "new_feature" -> NewFeatureScreen(params.toMap())
                    "settings" -> SettingsScreen()
                    else -> ErrorScreen()
                }
            }
        }
    }
    
    companion object {
        private const val EXTRA_SCREEN_TYPE = "screen_type"
        
        fun newIntent(context: Context, screenType: String, params: Bundle = Bundle.EMPTY): Intent {
            return Intent(context, ComposeWrapperActivity::class.java).apply {
                putExtra(EXTRA_SCREEN_TYPE, screenType)
                putExtras(params)
            }
        }
    }
}
```

---

## Feature Toggle

安全地在 Production 環境開關新功能。

```kotlin
interface FeatureFlags {
    val useNewCheckout: Boolean
    val enableComposeProfile: Boolean
}

class RemoteFeatureFlags @Inject constructor(
    private val remoteConfig: FirebaseRemoteConfig
) : FeatureFlags {
    override val useNewCheckout: Boolean
        get() = remoteConfig.getBoolean("use_new_checkout")
}

// 使用
class CheckoutNavigator @Inject constructor(
    private val featureFlags: FeatureFlags
) {
    fun navigateToCheckout() {
        if (featureFlags.useNewCheckout) {
            ModernEntryPoint.startCheckoutCompose()
        } else {
            startActivity(LegacyCheckoutActivity::class)
        }
    }
}
```

---

## Quick Checklist

- [ ] 新功能放在 `modern/` 目錄
- [ ] 橋接層清晰定義 (bridge/)
- [ ] Hybrid Theming 確保視覺一致
- [ ] Feature Toggle 控制上線
- [ ] 避免新代碼依賴舊代碼的內部實作
