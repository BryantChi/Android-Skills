---
name: Coding Style Conventions
description: Kotlin 代碼規範、Linter 配置與 Code Review 檢核標準
---

# Coding Style Conventions (代碼規範)

**Related Scenarios**: A (新專案), C (舊專案現代化)

---

## Naming Conventions (命名規則)

| 類型 | 規則 | 範例 |
|------|------|------|
| Class / Interface | `PascalCase` | `UserRepository`, `Drawable` |
| Function / Method | `camelCase` | `getUserById()`, `onClick()` |
| Variable / Property | `camelCase` | `userName`, `isLoading` |
| Constant (top-level/object) | `SCREAMING_SNAKE_CASE` | `MAX_RETRY_COUNT` |
| Package | `lowercase` (no underscores) | `com.example.feature.auth` |
| @Composable Function | `PascalCase` | `LoginScreen()`, `UserCard()` |
| Backing Property | `_camelCase` | `private val _uiState` |

### Compose Specific

```kotlin
// ✅ Composable 函數用 PascalCase (像 Class)
@Composable
fun UserProfileCard(user: User, modifier: Modifier = Modifier) { }

// ✅ State holder 用 remember + camelCase
val scrollState = rememberScrollState()

// ✅ Event callback 用 on 前綴
onUserClick: (User) -> Unit
```

---

## Detekt Configuration

### 安裝與設定

```kotlin
// build.gradle.kts (project-level)
plugins {
    id("io.gitlab.arturbosch.detekt") version "1.23.4"
}

// build.gradle.kts (app-level)
detekt {
    buildUponDefaultConfig = true
    config.setFrom("$rootDir/config/detekt/detekt.yml")
    baseline = file("$rootDir/config/detekt/baseline.xml")
}
```

### 建議規則集 (detekt.yml)

```yaml
complexity:
  LongMethod:
    threshold: 30
  LongParameterList:
    functionThreshold: 6
    constructorThreshold: 8
  
naming:
  FunctionNaming:
    excludes: ['**/composables/**']  # Compose 用 PascalCase
  
style:
  MaxLineLength:
    maxLineLength: 120
  WildcardImport:
    active: true
  MagicNumber:
    ignorePropertyDeclaration: true
    ignoreCompanionObjectPropertyDeclaration: true
```

### Baseline 機制 (舊專案適用)

```bash
# 生成 baseline，忽略現有問題
./gradlew detektBaseline

# 之後只檢查新代碼的違規
./gradlew detekt
```

---

## Ktlint Configuration

### 安裝

```kotlin
// build.gradle.kts
plugins {
    id("org.jlleitschuh.gradle.ktlint") version "12.0.3"
}

ktlint {
    android.set(true)
    outputColorName.set("RED")
}
```

### .editorconfig (與 IDE 同步)

```ini
[*.{kt,kts}]
max_line_length = 120
indent_size = 4
insert_final_newline = true

# Ktlint specific
ktlint_standard_no-wildcard-imports = enabled
ktlint_standard_trailing-comma-on-call-site = enabled
ktlint_standard_trailing-comma-on-declaration-site = enabled
```

---

## Documentation Standards (KDoc)

### 何時該寫

- ✅ Public API (SDK, Library)
- ✅ 複雜的業務邏輯
- ✅ 非直觀的參數或回傳值
- ✅ 重要的設計決策

### 何時不該寫

- ❌ 自解釋的代碼 (e.g., `fun getUserName(): String`)
- ❌ 覆寫的方法 (繼承父類文檔)
- ❌ 簡單的 CRUD 操作

### 範例

```kotlin
/**
 * 根據優先級排序並過濾過期的任務。
 *
 * @param tasks 待處理的任務列表
 * @param now 用於判斷過期的時間點，預設為當前時間
 * @return 依優先級排序的有效任務，過期任務會被過濾
 * @throws IllegalArgumentException 如果 tasks 包含 null 元素
 */
fun filterAndSort(tasks: List<Task>, now: Instant = Instant.now()): List<Task>
```

---

## Code Review Checklist

### Naming & Style
- [ ] 命名是否遵循上述規則？
- [ ] Compose 函數是否用 PascalCase？
- [ ] 是否有 Magic Number？

### Structure
- [ ] 函數是否過長 (> 30 行)？
- [ ] 參數是否過多 (> 6 個)？
- [ ] 是否有 God Class 傾向？

### Safety
- [ ] Nullable 處理是否安全？
- [ ] 是否有潛在的 NPE？
- [ ] 異常處理是否完善？

### Compose Specific
- [ ] Modifier 是否為第一個可選參數？
- [ ] State 是否正確 hoist？
- [ ] 是否有 unstable 的參數導致不必要重組？
