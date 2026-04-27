---
id: android_skill_index
name: Android Skill Index
description: Android 17 個技能的導航中樞 — 依場景路由到 2-3 個技能組合，含 Skill Dependency Graph、易混淆 skill 邊界對照表、2026-04 重構說明
type: skill
---

# Android Skill Index

導航中樞：當不確定該載入哪些技能時，從這裡開始。本檔案不含具體實作，只負責**指路**。

## Instructions

- 先簡述目標與現況（新專案 / 舊專案 / 效能 / 發布）。
- 從 §「Scenario Router」找到最接近的場景。
- 一次載入 **2-3 個** 技能，不要全部塞給 AI。
- 任務結束前回到 §「Skill Dependency Graph」確認沒漏。

## When to Use

- 不確定該載入哪些 skill
- 跨多個領域的任務需要組合
- 想依情境快速建立流程

## When NOT to Use

- 已經確定要用某個 skill → 直接載入它，不用先過本 skill
- 純單一明確問題（例如 "Detekt 規則怎麼寫"） → 直接 `@coding_style_conventions`

## Example Prompts

- 「我有舊專案要現代化，請推薦技能組合」
- 「效能很差，要先用哪些 skill 排查？」
- 「準備上 Play Store production，哪些 skill 要過？」

## Workflow

1. 在 Scenario Router 找場景。
2. 依順序載入對應 skill 並執行。
3. 完成後對照 Skill Dependency Graph 與 Boundary Table 確認無漏。

## Minimal Template

```
目標: <例：上線新支付>
現況: <例：targetSdk 33、無 Macrobenchmark>
情境: <Scenario E + Scenario M>
建議技能: platform_modernization_2026, deep_performance_tuning, release_automation
驗收: 各 skill 的 Quick Checklist
```

---

## Quick Reference

17 個 skill（含本 index 共 17；外部視角為 16 + index = 17）：

| Skill | 一句話 | 主要場景 |
|-------|--------|----------|
| `platform_modernization_2026` ⏰ | 年度平台升級（KSP2/Gradle 9/AGP 8.7+/targetSdk 35/Edge-to-Edge） | M |
| `project_bootstrapping` | 新專案/新模組骨架、Convention Plugins | A |
| `coding_style_conventions` | Detekt 1.25+、Ktlint 1.4+、Compose Stability、Code Review | A, C, G |
| `dependency_injection_mastery` | Hilt 2.52+ 進階、Custom Components、KSP2 | A, C, F |
| `data_layer_mastery` | Room 2.7+、Retrofit、Offline-First、Sync Manager | A, D, F |
| `navigation_patterns` | Compose Navigation 3.x、Deep Links、Predictive Back | A, B |
| `ui_ux_engineering` | Material 3 Expressive、Edge-to-Edge、Adaptive、a11y | A, B, D |
| `kotlin_multiplatform` | KMP 2.1、Ktor 3、SQLDelight 2、iOS Interop、CMP | F |
| `tech_stack_migration` | View→Compose、RxJava→Flow、Dagger→Hilt、kapt→KSP2 | C |
| `legacy_rapid_expansion` | 舊架構 Islanding 策略快速建新功能 | B, K |
| `testing_legacy_strategies` | Characterization、Robolectric、Roborazzi、Coverage 量化 | C, F, G |
| `crash_monitoring` | Crashlytics 33.x、ANR Watchdog、Timber、警報聯動（實作層） | D, E, I |
| `observability_strategy` | SLI/SLO 設計、告警閾值矩陣、feedback loop（設計層） | I, O |
| `deep_performance_tuning` | Macrobenchmark、Baseline/Startup Profiles、Perfetto、R8 | D, E, H |
| `release_automation` | CI Gates、Configuration Cache 9、Fastlane、Play Integrity、Cert Pinning | E, G, H |
| `supply_chain_security` | OSV-Scanner、SBOM、SLSA、Sigstore、Signing & Secrets、License | E, J, N |

⏰ = 年度更新 skill；2027-04 前需重新審視。

---

## Scenario Router

### 🚀 場景 A：從零建立新專案
```
1. project_bootstrapping        → 骨架 + Convention Plugins
2. coding_style_conventions     → Detekt/Ktlint + Compose Stability
3. dependency_injection_mastery → Hilt module 設計
4. ui_ux_engineering            → Design System (M3 Expressive)
5. data_layer_mastery           → Room + Retrofit + Sync
6. navigation_patterns          → Compose Navigation 3
```

### 🔧 場景 B：舊專案加新功能模組
```
1. legacy_rapid_expansion       → Islanding + Bridge
2. ui_ux_engineering            → XmlBridgeTheme 過渡
3. navigation_patterns          → Compose Navigation 接 Wrapper Activity
4. testing_legacy_strategies    → 邊界 characterization tests
```

### 🔄 場景 C：舊專案全面現代化
```
1. testing_legacy_strategies    → Characterization Tests + Visual Regression
2. coding_style_conventions     → Baseline 規範 + Compose Stability
3. tech_stack_migration         → 技術棧遷移（唯一權威）
4. dependency_injection_mastery → Hilt 共存與收斂
5. deep_performance_tuning      → 遷移後回歸量測
```

### ⚡ 場景 D：效能問題排查
```
1. observability_strategy       → 先確認指標與基準
2. deep_performance_tuning      → Perfetto + Macrobenchmark 定位
3. ui_ux_engineering            → Compose Stability 修法
4. data_layer_mastery           → DAO/網路熱區優化
```

### 📦 場景 E：App 發布準備
```
1. release_automation           → CI Gates + Fastlane + Staged Rollout
2. supply_chain_security        → SBOM + SCA + Signing
3. deep_performance_tuning      → Macrobenchmark gate
4. crash_monitoring             → Crashlytics + Velocity Alert
5. observability_strategy       → SLO 與 rollout halt 條件
```

### 🌐 場景 F：跨平台共享邏輯
```
1. kotlin_multiplatform         → 共享邊界決策 + Setup
2. data_layer_mastery           → Room vs SQLDelight 決策
3. dependency_injection_mastery → 跨平台 DI 接合
4. testing_legacy_strategies    → commonTest 安全網
```

### 🧭 場景 G：CI / Quality Gates
```
1. coding_style_conventions     → Detekt/Ktlint/Lint 規則
2. release_automation           → CI workflow + Quality Gates
3. testing_legacy_strategies    → 覆蓋率 gate
4. supply_chain_security        → OSV-Scanner / SBOM CI
```

### 📈 場景 H：Performance-by-default
```
1. deep_performance_tuning      → Macrobench + Baseline/Startup Profiles
2. project_bootstrapping        → 預設 benchmark 模組
3. release_automation           → CI 退化 > 5% 阻擋
```

### 🔍 場景 I：Observability-first（既有實作層）
```
1. observability_strategy       → SLI/SLO/告警閾值
2. crash_monitoring             → Crashlytics/ANR 實作
3. deep_performance_tuning      → 效能信號接入
```

### 🧱 場景 J：Supply Chain Compliance
```
1. supply_chain_security        → 依賴/SBOM/SLSA/Sigstore
2. release_automation           → CI gate + 簽章流程
3. coding_style_conventions     → SuppressLint 與 baseline 治理
```

### 🧩 場景 K：Compose-first + Legacy Interop
```
1. tech_stack_migration         → ComposeView/AndroidView/Fragment
2. ui_ux_engineering            → Material 3 + a11y
3. legacy_rapid_expansion       → 島內新代碼策略
```

### 🧭 場景 L：多模組擴展與導航治理
```
1. project_bootstrapping        → 模組與 Convention Plugins
2. dependency_injection_mastery → 模組邊界與 EntryPoint
3. navigation_patterns          → 跨模組 Navigator API
```

### ⏰ 場景 M：版本/平台升級（**新**）
```
1. platform_modernization_2026  → KSP2 / Gradle 9 / AGP 8.7+ / targetSdk 35
2. project_bootstrapping        → 重整 Convention Plugin
3. tech_stack_migration         → 過時 API 替換
4. deep_performance_tuning      → 升級後 Baseline Profile 重生
```

### 🛡️ 場景 N：供應鏈合規（**新**）
```
1. supply_chain_security        → SBOM / SLSA / Sigstore / OSV
2. release_automation           → CI 串接 + 簽章 OIDC
3. observability_strategy       → 簽章/掃描事件納入告警
```

### 🔭 場景 O：可觀測性建模（**新**）
```
1. observability_strategy       → SLI/SLO/event schema/feedback loop
2. crash_monitoring             → 實作層接入
3. deep_performance_tuning      → 效能 metric 上報
4. release_automation           → SLO 違反自動 halt rollout
```

---

## Skill Dependency Graph

```
                    ┌─────────────────────────────────────┐
                    │       coding_style_conventions      │  ← 所有技能的基礎
                    └──────────────────┬──────────────────┘
                                       │
        ┌───────────────────┬──────────┼──────────┬───────────────────┐
        ▼                   ▼          ▼          ▼                   ▼
┌──────────────┐  ┌──────────────┐  ┌──────┐  ┌──────────────┐  ┌──────────────┐
│ project_     │  │ legacy_      │  │ tech_│  │ deep_perfor- │  │ platform_    │
│ bootstrapping│  │ rapid_       │  │ stack│  │ mance_tuning │  │ modernizati- │
│              │  │ expansion    │  │ _mig │  │              │  │ on_2026 ⏰   │
└──────┬───────┘  └──────┬───────┘  └──┬───┘  └──────┬───────┘  └──────┬───────┘
       │                 │             │             │                 │
   ┌───┴────────┬────────┴───┐    ┌────┴───┐    ┌────┴───┐         ┌──┴─┐
   ▼            ▼            ▼    ▼        ▼    ▼        ▼         ▼    ▼
┌──────┐  ┌──────────┐  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────────┐
│ ui_ux│  │dependency│  │data_ │ │testi-│ │crash_│ │relea-│ │navigation│
│ _eng │  │_injectio-│  │layer_│ │ng_   │ │monit-│ │se_   │ │_patterns │
│      │  │n_mastery │  │maste-│ │legacy│ │oring │ │autom-│ │          │
│      │  │          │  │ry    │ │      │ │      │ │ation │ │          │
└──┬───┘  └──────────┘  └──┬───┘ └──────┘ └──┬───┘ └──┬───┘ └──────────┘
   │                       │                 │        │
   │                       ▼                 ▼        ▼
   │                  ┌──────────┐    ┌──────────────┐
   │                  │ kotlin_  │    │observability_│
   │                  │ multipla-│    │ strategy     │  ← 設計層
   │                  │ tform    │    └──────┬───────┘
   │                  └──────────┘           │
   │                                         ▼
   └──────────────────────────────────► 指向 crash_monitoring
                                              + deep_performance_tuning
                                              （實作層）

                                ┌──────────────────────┐
                                │ supply_chain_security │  ← 含 Signing/Secrets
                                └──────────────────────┘
                                              │
                                              ▼
                                     接到 release_automation 與 CI
```

---

## Skill 邊界對照表（易混淆組合）

當你不確定某個任務該載哪個 skill，先看這張表：

| 易混淆 | 領域分工 |
|--------|----------|
| `observability_strategy` ↔ `crash_monitoring` | **strategy 設計** SLI/SLO/告警閾值/event schema；**monitoring 實作** Crashlytics SDK / ANR Watchdog / Timber。談「該量什麼、警報多嚴」→ strategy；談「怎麼裝、怎麼接、怎麼讓 dashboard 收到」→ monitoring。|
| `release_automation` ↔ `supply_chain_security` | **release_automation** = CI workflow / Build Speed / Fastlane / Play Console / 應用層執行時安全（Cert Pinning / NSC / Root）。**supply_chain_security** = 依賴掃描 / SBOM / SLSA / Sigstore / Signing & Secrets / License。Secrets/Signing 在 supply_chain。|
| `testing_legacy_strategies` ↔ `legacy_rapid_expansion` | **testing** 為「給舊代碼補測試」；**rapid_expansion** 為「在舊代碼旁建新功能孤島」。先 testing 再 rapid_expansion。|
| `data_layer_mastery` ↔ `kotlin_multiplatform` | Android-only 用 Room → data_layer；KMP 跨 iOS 用 SQLDelight → kmp（data_layer 提供決策矩陣指向 kmp）。|
| `dependency_injection_mastery` ↔ `tech_stack_migration` | Hilt **遷移步驟**（從 Dagger 2）→ tech_stack_migration（唯一權威）；Hilt 已就位後的進階用法 → di_mastery。|
| `project_bootstrapping` ↔ `platform_modernization_2026` | bootstrapping 處理「新專案/新模組」骨架；platform_modernization 處理「升級既有專案到新平台版本」。新專案會直接套後者的版本基準。|
| `coding_style_conventions` ↔ `release_automation` | conventions 提供 **規則檔**（detekt.yml / .editorconfig / lint baseline）；release_automation 把它**接進 CI gate**。|
| `coding_style_conventions` ↔ `ui_ux_engineering` | conventions 處理 Compose **命名/順序/Stability config**；ui_ux 處理 Compose **Theme/Token/Component/Pattern**。|
| `navigation_patterns` ↔ `ui_ux_engineering` | navigation 處理路由邏輯與 Predictive Back **行為**；ui_ux 處理 Predictive Back **動畫視覺** 與 Adaptive layout 視覺。|

---

## 2026-04 重構紀錄

| 變更 | 從 | 到 | 影響 |
|------|----|----|------|
| 改名 + 重新定位 | `observability_first` | `observability_strategy` | 舊引用 `@observability_first` 需改 |
| 改名 + 收斂職責 | `devops_and_security` | `release_automation` | 舊引用 `@devops_and_security` 需改 |
| 強化（接收 Secrets） | `supply_chain_security` | `supply_chain_security`（同名擴張） | 內容由 78 行擴至 ~250 行 |
| 新增 | — | `platform_modernization_2026` ⏰ | 年度更新；2027-04 前需重審 |

整體版本基準對齊：Kotlin 2.1.x / Gradle 9.0+ / AGP 8.7+ / compileSdk 35 / Compose BOM 2025.04 / Hilt 2.52 / Navigation 3.x。

---

## How to Use

1. **確認場景**：從上方 Scenario Router 選最接近的。
2. **載入 2-3 個 skill**：避免一次塞太多。
3. **走 skill 的 Workflow**：每個 skill 都有固定 6 段（Instructions → When to Use → When NOT to Use → Example Prompts → Workflow → Practical Notes → Minimal Template → 領域章節 → Cross-Skill References → Quick Checklist）。
4. **以 Quick Checklist 收尾**：當作 self-review 與 PR description 模板。

---

## AI Tool Compatibility

所有 SKILL.md 採標準 Markdown，含 frontmatter、章節階層、程式碼塊、checklist；任何支援 context 的 AI 工具都能讀。

| 工具 | 引用方式 |
|------|----------|
| Claude Code（plugin） | Skill tool / `@skill_name` |
| Antigravity / Cursor / Windsurf | `@skill_name` |
| Aider / 其他 CLI | `/add skills/<name>/SKILL.md` |
| Copilot | 開啟對應 SKILL.md tab 作為 context |
| Web Chat | 複製貼上單一 skill 內容 |

詳細 18+ 工具教學請見 `ANDROID_SKILLS_GUIDE.md`。

### Best Practices

1. **Context Management**：依 Scenario Router 載 2-3 個，不要全塞。
2. **Focusing**：明確引用章節，例「@deep_performance_tuning 的 Macrobenchmark 章節」。
3. **Enforce Checklists**：任務結束前要求 AI 檢查對應 skill 的 Quick Checklist。

---

## Quick Checklist（載入前自我檢查）

- [ ] 我已用一句話描述「目標 + 現況」
- [ ] 我從 Scenario Router 找到對應場景（A-O）
- [ ] 我只載入該場景列出的 2-3 個 skill
- [ ] 任務結束時對照各 skill 的 Quick Checklist 收尾
- [ ] 跨 skill 衝突時先看 §「Skill 邊界對照表」
