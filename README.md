# Android Skills 技能庫（2026-04）

為資深 Android 工程師設計的 Claude / Copilot / Cursor / Antigravity 通用 skill 集合。版本對齊 2026-04 業界主流（Kotlin 2.1、Gradle 9、AGP 8.7+、compileSdk 35、Compose BOM 2025.04、Hilt 2.52、Navigation 3.x）。

## 📖 閱讀順序

1. **首次使用者** → 看 [`android_skill_index/SKILL.md`](./android_skill_index/SKILL.md) 的 Scenario Router，從場景找技能組合。
2. **找 AI 工具用法** → 看 [`ANDROID_SKILLS_GUIDE.md`](./ANDROID_SKILLS_GUIDE.md) 的 18+ 工具教學。
3. **直接載入單一技能** → 在下方表格選擇，引用語法統一為 `@skill_name`。

## 🛠️ 技能列表（17 + 1 導航 = 18 項）

| Skill | 一句話描述 |
|-------|-----------|
| **`android_skill_index`** | 導航中樞，依場景推薦技能組合 |
| **`platform_modernization_2026`** ⏰ | 年度平台升級（KSP2、Gradle 9、AGP 8.7+、targetSdk 35、Edge-to-Edge、Predictive Back） |
| **`project_bootstrapping`** | 新專案/新模組骨架、Convention Plugins、Version Catalog |
| **`coding_style_conventions`** | Detekt 1.25+ / Ktlint 1.4+、Compose Stability、Code Review Checklist |
| **`dependency_injection_mastery`** | Hilt 2.52+、Custom Components、Multi-binding、KSP2 |
| **`data_layer_mastery`** | Room 2.7+、Retrofit、Offline-First、Sync Manager、Proto DataStore |
| **`navigation_patterns`** | Compose Navigation 3.x、Deep Links、Predictive Back、跨模組 Navigator |
| **`ui_ux_engineering`** | Material 3 Expressive、Edge-to-Edge、Adaptive、Compose Stability、a11y |
| **`kotlin_multiplatform`** | KMP 2.1、Ktor 3.x、SQLDelight 2.x、iOS Interop、CMP 決策 |
| **`tech_stack_migration`** | View→Compose、RxJava→Flow、Dagger→Hilt、kapt→KSP2（**唯一權威**） |
| **`legacy_rapid_expansion`** | 舊架構中 Islanding 策略快速建新功能 |
| **`testing_legacy_strategies`** | Characterization、Robolectric、Roborazzi、Compose Preview Screenshot |
| **`crash_monitoring`** | Firebase Crashlytics 33.x、ANR Watchdog、Timber、Play Console 警報聯動 |
| **`observability_strategy`** | SLI/SLO 設計、告警閾值矩陣、event schema、feedback loop（設計層） |
| **`deep_performance_tuning`** | Macrobenchmark、Baseline/Startup Profiles、Perfetto、JankStats、R8 |
| **`release_automation`** | CI Quality Gates、Configuration Cache 9、Fastlane、Play Integrity、Cert Pinning |
| **`supply_chain_security`** | OSV-Scanner、SBOM、SLSA、Sigstore、Signing & Secrets、License Compliance |

⏰ 標示為「年度更新」skill。

## 🆕 2026-04 重構紀錄

| 變更 | 從 | 到 |
|------|----|----|
| 改名 + 重新定位 | `observability_first` | `observability_strategy`（指標設計層） |
| 改名 + 收斂職責 | `devops_and_security` | `release_automation`（移出 Secrets/Signing） |
| 強化 + 接收 Secrets | `supply_chain_security` | `supply_chain_security`（職責擴張至 SBOM/SLSA/Sigstore） |
| 新增 | — | `platform_modernization_2026`（年度平台升級） |

外部引用更新：把舊的 `@observability_first` / `@devops_and_security` 改為新名。

## 🚀 快速開始

### Antigravity / Cursor / Windsurf
```
@android_skill_index 我有一個舊專案要現代化，請推薦技能組合
@project_bootstrapping 幫我建立 Auth 模組
```

### Claude Code（plugin）
透過 Skill tool 呼叫對應名稱即可，runtime 自動載入。

### CLI（Aider 等）
```bash
# 把需要的 skill 加進 context
/add skills/data_layer_mastery/SKILL.md
/add skills/observability_strategy/SKILL.md
```

詳細 18+ AI 工具教學請看 [`ANDROID_SKILLS_GUIDE.md`](./ANDROID_SKILLS_GUIDE.md)。
