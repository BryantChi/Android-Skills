---
id: coding_style_conventions
name: Coding Style Conventions
description: Kotlin 程式碼規範 — 命名、Detekt 1.25+ / Ktlint 1.4+ 配置、Compose Stability/Compiler Metrics、SuppressLint 多模組策略、KDoc 與 Code Review Checklist，產出可在 CI 強制的風格基線
type: skill
---

# Coding Style Conventions

## Instructions

當需要建立、檢視、強化 Kotlin 風格與靜態檢查規則時載入。本 skill 只談「規則本身」與「IDE/CI 整合方式」；CI workflow 框架由 `@release_automation` 負責，平台升級由 `@platform_modernization_2026` 負責。

## When to Use

- Scenario A：新專案建立規範
- Scenario C：舊專案現代化基準
- 統一 Compose 命名與 modifier 順序
- 加入 Compose Stability / Compiler Metrics 至 review
- 解決多模組 SuppressLint 失控

## When NOT to Use

- CI workflow 設計與並行策略 → `@release_automation`
- Compose UI 設計（Spacing、Theme）→ `@ui_ux_engineering`
- 解決效能瓶頸 → `@deep_performance_tuning`

## Example Prompts

- 「我想升 Detekt 到 1.25，新規則我該開哪些？」
- 「Compose Stability 的 strong-skipping 怎麼配？」
- 「PR review 想自動化，KDoc 與命名規則的 checklist」
- 「多模組 baseline.xml 怎麼管才不爆增？」

## Workflow

1. `.editorconfig` + Ktlint 1.4+ 規則對齊 IDE。
2. Detekt 1.25+ 套自訂 yml + 可選 baseline。
3. Compose Compiler Metrics 啟用，輸出至 `build/compose-metrics`。
4. SuppressLint 政策：以 `lint-baseline.xml` 收容歷史，新代碼禁止 SuppressLint 而無註解理由。
5. Code Review Checklist 嵌入 PR 模板。

## Practical Notes (2026-04)

| 工具 | 版本 | 備註 |
|------|------|------|
| Detekt | 1.25+ | 含 `detekt-formatting`（Ktlint 規則整合） |
| Ktlint | 1.4+ | trailing comma 預設啟用 |
| Compose Compiler | Kotlin 2.1 內建 plugin | `composeOptions` 已不需要 |
| Strong Skipping | 預設啟用（Compose Compiler 1.5.4+） | 大幅減少不必要重組 |
| Lint | AGP 8.7+ 內建 | 與 Detekt 互補不衝突 |

## Minimal Template

```
規範範圍: <new project | legacy module | single PR>
工具: detekt + ktlint + lint
強制機制: pre-commit hook + CI gate
特例豁免: lint-baseline.xml（凍結） + SuppressLint with TODO comment（限期）
驗收: Quick Checklist
```

## Naming Conventions

| 類型 | 規則 | 範例 |
|------|------|------|
| Class / Interface | PascalCase | `UserRepository` |
| Function | camelCase | `getUserById()` |
| @Composable Function | PascalCase | `LoginScreen()` |
| Variable / Property | camelCase | `userName`, `isLoading` |
| Constant（top-level / object） | SCREAMING_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Package | lowercase | `com.example.feature.auth` |
| Backing Property | `_camelCase` | `private val _uiState` |
| Test Method | backtick + 中英文皆可 | `` `clicking save emits success`() `` |

### Compose 命名與 modifier 順序

```kotlin
// ✅ 簽章順序：必填 → modifier → 可選 → callback → state slots
@Composable
fun UserCard(
    user: User,
    modifier: Modifier = Modifier,
    elevation: Dp = 2.dp,
    onClick: (User) -> Unit = {},
    actions: @Composable RowScope.() -> Unit = {},
) { /* ... */ }

// ❌ Modifier 不在第二個位置、不是 Modifier.Companion
fun UserCard(user: User, onClick: () -> Unit, m: Modifier)

// Modifier 鏈順序：size → shape/background → padding → behavior（clickable/scroll）→ semantics
Modifier
    .size(64.dp)
    .clip(RoundedCornerShape(8.dp))
    .background(MaterialTheme.colorScheme.surface)
    .padding(8.dp)
    .clickable(onClick = onClick)
    .semantics { contentDescription = "user card" }
```

## Detekt 1.25+

```kotlin
// build.gradle.kts (root)
plugins {
    id("io.gitlab.arturbosch.detekt") version "1.25.0"
}

allprojects {
    apply(plugin = "io.gitlab.arturbosch.detekt")
    detekt {
        buildUponDefaultConfig = true
        config.setFrom("$rootDir/config/detekt/detekt.yml")
        baseline = file("$rootDir/config/detekt/baseline.xml")
        autoCorrect = false   // 由 ktlint 處理格式化
    }
    dependencies {
        "detektPlugins"("io.gitlab.arturbosch.detekt:detekt-formatting:1.25.0")
        "detektPlugins"("com.twitter.compose.rules:detekt:0.0.26")
    }
}
```

```yaml
# config/detekt/detekt.yml
complexity:
  LongMethod: { threshold: 30 }
  LongParameterList: { functionThreshold: 6, constructorThreshold: 8 }
  CyclomaticComplexMethod: { threshold: 12 }
  TooManyFunctions: { thresholdInClasses: 15 }

style:
  MaxLineLength: { maxLineLength: 120 }
  WildcardImport: { active: true }
  MagicNumber:
    ignorePropertyDeclaration: true
    ignoreCompanionObjectPropertyDeclaration: true
  ReturnCount: { max: 3 }

naming:
  FunctionNaming:
    ignoreAnnotated: ['Composable']
  TopLevelPropertyNaming:
    constantPattern: '[A-Z][_A-Z0-9]*'

# Twitter Compose Rules
Compose:
  ComposableNaming: { active: true }
  ModifierComposable: { active: true }
  ModifierMissing: { active: true }
  ModifierNaming: { active: true }
  ParameterNaming: { active: true }
  PreviewPublic: { active: true }
  RememberMissing: { active: true }
  UnstableCollections: { active: true }
  ViewModelForwarding: { active: true }
```

### Baseline 政策

- 第一次接舊專案：`./gradlew detektBaseline` 凍結。
- baseline.xml **每季審視一次**，刪除已修復項目；不允許無限增長。
- 新模組禁止 baseline；違規必須當下處理。

## Ktlint 1.4+

```kotlin
plugins {
    id("org.jlleitschuh.gradle.ktlint") version "12.1.2"
}
ktlint {
    version.set("1.4.1")
    android.set(true)
    ignoreFailures.set(false)
    reporters {
        reporter(org.jlleitschuh.gradle.ktlint.reporter.ReporterType.SARIF)
    }
}
```

```ini
# .editorconfig
root = true

[*]
end_of_line = lf
insert_final_newline = true
charset = utf-8

[*.{kt,kts}]
indent_size = 4
max_line_length = 120
ij_kotlin_allow_trailing_comma = true
ij_kotlin_allow_trailing_comma_on_call_site = true

ktlint_standard_no-wildcard-imports = enabled
ktlint_standard_trailing-comma-on-call-site = enabled
ktlint_standard_trailing-comma-on-declaration-site = enabled
ktlint_standard_function-signature = enabled
ktlint_function_signature_body_expression_wrapping = always
ktlint_standard_property-naming = enabled
```

## Compose Stability / Compiler Metrics

```kotlin
// build-logic 中的 AndroidComposeConventionPlugin
extensions.configure<ComposeCompilerGradlePluginExtension> {
    metricsDestination.set(layout.buildDirectory.dir("compose-metrics"))
    reportsDestination.set(layout.buildDirectory.dir("compose-reports"))
    stabilityConfigurationFile.set(rootProject.file("config/compose/stability.conf"))
}
```

```
# config/compose/stability.conf — 標記第三方/不可變類型為 stable
java.time.Instant
java.util.UUID
com.example.network.dto.*Response
kotlinx.collections.immutable.*
```

Strong Skipping：Compose Compiler 1.5.4+ 預設啟用；非 stable 但相同 instance 的參數仍會 skip。

CI 檢查：

```bash
# 偵測 unstable composable 數
unstable=$(grep -c "unstable params" app/build/compose-reports/*.txt || true)
[ "$unstable" -lt "20" ] || { echo "Unstable composable 數 $unstable 超標"; exit 1; }
```

## SuppressLint 多模組策略

- **Lint baseline 路徑**：每模組 `lint-baseline.xml`，避免 root 一個巨型檔。
- **新增 SuppressLint 規則**：必須附原因 + 期限 TODO，例如：
  ```kotlin
  @SuppressLint("MissingPermission") // FIXME(2026-Q3): 由 PermissionManager 統一處理
  ```
- **CI 阻擋無理由 SuppressLint**：寫個 detekt rule 或 grep 檢查。

## KDoc Standards

寫：public API、複雜業務邏輯、非直觀參數、重要設計決策。
不寫：自解釋程式碼、覆寫方法、簡單 getter。

```kotlin
/**
 * 依優先級排序並過濾過期任務。
 *
 * @param tasks 待處理任務列表
 * @param now 判斷過期的時間點，預設為當前時間
 * @return 依優先級排序的有效任務
 * @throws IllegalArgumentException 若 tasks 含 null
 */
fun filterAndSort(tasks: List<Task>, now: Instant = Instant.now()): List<Task>
```

## Code Review Checklist

### Naming & Style
- [ ] 遵循命名規則
- [ ] Composable 為 PascalCase 且第一個可選參數為 `modifier: Modifier = Modifier`
- [ ] 無 magic number（或加註解理由）

### Structure
- [ ] 函數 ≤ 30 行
- [ ] 參數 ≤ 6 個（建構式 ≤ 8）
- [ ] 無 God Class（單檔 ≤ 500 行為原則）

### Safety
- [ ] Nullable 處理採 safe call / requireNotNull
- [ ] 不吞 exception；至少 log 並上報
- [ ] 無 `runBlocking` 在主執行緒

### Compose
- [ ] modifier 在第二個位置且名為 `modifier`
- [ ] State 正確 hoist（screen 持有，children 無狀態）
- [ ] Unstable 參數已標 stable 或改用 immutable collection
- [ ] `remember { ... }` key 正確
- [ ] Preview 為 `@Preview` 私有 composable

## Cross-Skill References

- `@project_bootstrapping`：Convention Plugin 接 Detekt/Ktlint。
- `@release_automation`：將 detekt/ktlint 加入 CI gate；本 skill 提供規則檔。
- `@ui_ux_engineering`：Compose Theme/Spacing 設計與本 skill 的命名/順序一致。
- `@platform_modernization_2026`：升 Kotlin 2.1 後 ktlint/detekt 版本同步升級。

## Quick Checklist

- [ ] `.editorconfig` 與 ktlint 1.4+ 規則對齊 IDE
- [ ] Detekt 1.25+ + detekt-formatting + Twitter Compose Rules 啟用
- [ ] Compose Compiler Metrics 啟用，CI 監看 unstable composable 數
- [ ] Stability config 收容第三方不變類型
- [ ] `lint-baseline.xml` 每季審視，禁止無限增長
- [ ] SuppressLint 必有 TODO + 期限
- [ ] PR template 嵌入 Code Review Checklist
- [ ] 規範變更與功能變更分開 PR
