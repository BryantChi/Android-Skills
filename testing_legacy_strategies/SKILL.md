---
id: testing_legacy_strategies
name: Testing Legacy Strategies
description: 為無測試的 Android 舊代碼建立安全網 — Characterization Tests、Robolectric Native Graphics、MockK、Compose Preview Screenshot Test、Roborazzi/Paparazzi 視覺迴歸、Detekt/Lint baseline、Golden Master、覆蓋率量化目標
type: skill
---

# Testing Legacy Strategies

## Instructions

當需要在無測試或測試稀少的舊代碼上建立安全網（通常為了重構或加新功能）時載入。本 skill 不教 TDD（新代碼正規 TDD 不在範圍）；本 skill 是「**先把現狀凍結，才能安全動**」的策略。

## When to Use

- Scenario C：舊專案現代化前的安全網
- Scenario F：KMP commonMain 的測試
- 加入 Compose Preview Screenshot Test 防 UI 退化
- 對複雜輸出（HTML、JSON、PDF）做 Golden Master
- 用 Roborazzi 對 Compose 做視覺迴歸（無需裝置）

## When NOT to Use

- 新代碼的 TDD（不在本 skill 範圍）
- CI workflow 設定 → `@release_automation`
- Hilt 測試替換策略 → `@dependency_injection_mastery`
- 效能 benchmark → `@deep_performance_tuning`

## Example Prompts

- 「這個 calc 沒有測試，重構前要先 lock 現狀，怎麼開始？」
- 「Robolectric 跑 Compose Preview 截圖，怎麼設？」
- 「Roborazzi vs Paparazzi 該選哪個？我們大多 Compose」
- 「Lint baseline 已經 1.2MB，怎麼有秩序地縮減？」
- 「金本位測試（Golden Master）的 diff 工具怎麼設」

## Workflow

1. **盤點高風險檔案**（修改頻率 × 無測試 × 業務關鍵）。
2. **Characterization Tests** 鎖定現狀（不論對錯）。
3. **Robolectric / MockK** 處理 Framework 與依賴。
4. **Visual Regression**（Roborazzi / Paparazzi）覆蓋 UI states。
5. **Compose Preview Screenshot Test** 把 `@Preview` 自動轉測試。
6. **Baseline + 收斂計畫**：detekt baseline 凍結，每季減 N 條。
7. **覆蓋率門檻**：critical path ≥ 80%、整體 ≥ 50%。

## Practical Notes (2026-04)

| 工具 | 版本 | 用途 |
|------|------|------|
| Robolectric | 4.13+ | Native Graphics 模式可 render Compose |
| MockK | 1.13+ | Coroutines 支援、relaxed mock |
| Roborazzi | 1.30+ | JVM 上對 Compose 截圖（推薦） |
| Paparazzi | 1.3+ | Square 老牌、純 JVM 截圖 |
| Compose Preview Screenshot Test | AGP 8.5+ 內建 | `@Preview` 自動轉 screenshot test |
| Truth | 1.4+ | 比 JUnit assertions 易讀 |
| Turbine | 1.1+ | Flow 測試 |

## Minimal Template

```
高風險檔案清單: <Top 10 churn × no test>
Characterization 覆蓋: <按檔案逐個鎖定>
Visual Regression: Roborazzi - <screen list>
Coverage 目標: critical path 80%, overall 50%
Baseline 收斂: detekt -10 / lint -50 per sprint
驗收: Quick Checklist
```

## 高風險檔案盤點

```bash
# Top 20 修改頻繁但無測試的檔案
git log --since='1 year ago' --pretty=format: --name-only \
  | grep '\.kt$' | sort | uniq -c | sort -rn | head -50 \
  | while read count file; do
      test_path=$(echo "$file" | sed 's|/main/|/test/|;s|\.kt$|Test.kt|')
      [ -f "$test_path" ] || echo "$count $file"
    done | head -20
```

優先補測：churn ≥ 5 / 年 + 無對應 test file + 業務關鍵路徑。

## Characterization Tests（鎖現狀）

```kotlin
// 1. 寫一個故意失敗的測試
@Test fun `calculateDiscount(100, VIP) returns ?`() {
    val r = legacyCalc.calculateDiscount(100.0, "VIP")
    assertThat(r).isEqualTo(0.0)   // 故意錯
}
// 2. 跑測試，記下實際值（例如 15.0）
// 3. 改成正確的 expected
@Test fun `calculateDiscount(100, VIP) returns 15`() {
    assertThat(legacyCalc.calculateDiscount(100.0, "VIP")).isEqualTo(15.0)
}
```

批量化：

```kotlin
class DiscountCharacterizationTest {
    @ParameterizedTest
    @CsvSource(
        "100.0, VIP, 15.0",
        "100.0, REGULAR, 5.0",
        "50.0, VIP, 7.5",
        "0.0, VIP, 0.0",
        "-100.0, VIP, 0.0",       // 即使是 bug，先鎖定
    )
    fun characterization(price: Double, tier: String, expected: Double) {
        assertThat(legacyCalc.calculateDiscount(price, tier)).isEqualTo(expected)
    }
}
```

**規則**：先鎖再修。哪怕現狀是 bug，也先 lock，重構後改測試 + 改實作同 PR。

## Robolectric（含 Native Graphics）

```kotlin
// gradle
testImplementation("org.robolectric:robolectric:4.13")

// build.gradle.kts
android {
    testOptions {
        unitTests {
            isIncludeAndroidResources = true
            all {
                it.systemProperty("robolectric.graphicsMode", "NATIVE")  // 啟用 Native Graphics
            }
        }
    }
}
```

```kotlin
@RunWith(RobolectricTestRunner::class)
@Config(sdk = [35], qualifiers = "w400dp-h800dp")
class LegacyActivityTest {
    @Test fun `activity displays correct title`() {
        val activity = Robolectric.buildActivity(LegacyActivity::class.java).setup().get()
        assertThat(activity.title).isEqualTo("Expected")
    }

    @Test fun `toast shown on error`() {
        activity.showError("oops")
        assertThat(ShadowToast.getTextOfLatestToast()).isEqualTo("oops")
    }
}
```

Native Graphics 模式可實際 render Bitmap，是 Roborazzi 與 Compose Preview Screenshot Test 的前置。

## MockK

```kotlin
@Test fun `repository returns cached`() {
    val repo = mockk<UserRepository>()
    every { repo.getUser("123") } returns User("123", "Alice")
    assertThat(repo.getUser("123").name).isEqualTo("Alice")
    verify { repo.getUser("123") }
}

@Test fun `suspend mocking`() = runTest {
    val api = mockk<UserApi>()
    coEvery { api.fetch("123") } returns User("123", "Alice")
    api.fetch("123")
    coVerify(exactly = 1) { api.fetch("123") }
}

// relaxed: 自動為未 stub 方法回傳預設值，適合舊代碼大量無關依賴
val legacy = mockk<LegacyHelper>(relaxed = true)
```

避免在 ViewModel 測試中 mock 純 data class 或 use case 簡單實作；mock 越淺越好。

## Roborazzi（推薦：JVM 上對 Compose 截圖）

```kotlin
// gradle
plugins { id("io.github.takahirom.roborazzi") version "1.30.0" }

testImplementation("io.github.takahirom.roborazzi:roborazzi:1.30.0")
testImplementation("io.github.takahirom.roborazzi:roborazzi-compose:1.30.0")
testImplementation("io.github.takahirom.roborazzi:roborazzi-junit-rule:1.30.0")
```

```kotlin
@RunWith(RobolectricTestRunner::class)
@GraphicsMode(GraphicsMode.Mode.NATIVE)
@Config(sdk = [35])
class MembershipScreenScreenshotTest {
    @get:Rule val composeRule = createComposeRule()
    @get:Rule val roborazziRule = RoborazziRule(composeRule, RoborazziOptions())

    @Test fun loaded() {
        composeRule.setContent {
            ModernAppTheme { MembershipScreen(state = MembershipState.loaded()) }
        }
        composeRule.onRoot().captureRoboImage("MembershipScreen_loaded.png")
    }

    @Test fun loading() {
        composeRule.setContent {
            ModernAppTheme { MembershipScreen(state = MembershipState.loading()) }
        }
        composeRule.onRoot().captureRoboImage("MembershipScreen_loading.png")
    }
}
```

```bash
./gradlew :app:recordRoborazziDebug    # 生成基準
./gradlew :app:verifyRoborazziDebug    # CI 比對
./gradlew :app:compareRoborazziDebug   # 產 diff 報告
```

CI 上 diff > threshold 阻擋 merge；diff 報告 upload 為 artifact。

### Roborazzi vs Paparazzi

| | Roborazzi | Paparazzi |
|---|-----------|-----------|
| 引擎 | Robolectric Native Graphics | Square Layoutlib |
| Compose 支援 | 原生 | 需 wrapper |
| 與 androidTest 一致性 | 高 | 中 |
| Layoutlib 升級延遲 | 低 | 略高 |
| 推薦 | **大多 Compose 專案** | 純 View 老專案 |

## Compose Preview Screenshot Test（AGP 內建）

```kotlin
// gradle
android {
    experimentalProperties["android.experimental.enableScreenshotTest"] = true
}

dependencies {
    screenshotTestImplementation(libs.compose.ui.tooling)
}
```

```kotlin
// src/screenshotTest/kotlin/.../MembershipPreviews.kt
@Preview(showBackground = true)
@Preview(showBackground = true, uiMode = UI_MODE_NIGHT_YES)
@Composable
private fun MembershipScreenLoadedPreview() {
    ModernAppTheme { MembershipScreen(state = MembershipState.loaded()) }
}
```

```bash
./gradlew :app:updateDebugScreenshotTest   # 基準
./gradlew :app:validateDebugScreenshotTest # 驗證
```

每個 `@Preview` 自動變一條測試。適合 Design System component library。

## Detekt / Lint Baseline 收斂

```bash
# 一次性凍結
./gradlew detektBaseline
./gradlew lintDebug -Dlint.baselines.continue=true
```

```kotlin
// build.gradle.kts
android.lint { baseline = file("lint-baseline.xml") }
detekt { baseline = file("$rootDir/config/detekt/baseline.xml") }
```

### 收斂計畫

```bash
# scripts/baseline-shrink.sh — 跑在 PR 之外的 cron
detekt_count=$(grep -c '<ID>' config/detekt/baseline.xml || echo 0)
target=$((detekt_count - 10))
echo "Target this sprint: $detekt_count → $target"
```

每 sprint 減 N 條；CI 阻擋 baseline 條目 **增加**。

## Golden Master Testing

```kotlin
@Test fun `report generator html stable`() {
    val output = legacyReportGenerator.generate(testData)
    val golden = File("src/test/resources/golden/report.html").readText()
    assertThat(output).isEqualTo(golden)
}
```

更新時：

```bash
UPDATE_GOLDEN=1 ./gradlew :reporting:test
```

```kotlin
if (System.getenv("UPDATE_GOLDEN") == "1") goldenFile.writeText(output)
else assertThat(output).isEqualTo(goldenFile.readText())
```

JSON / PDF 適用同樣模式（PDF 取 normalize 後文字 + Roborazzi 視覺截圖雙保險）。

## 覆蓋率量化目標

```kotlin
// koverage 報告（推薦 Kover 2.x）
plugins { id("org.jetbrains.kotlinx.kover") version "0.8.3" }

kover {
    reports {
        verify {
            rule {
                bound { minValue = 50 }                                        // overall
            }
            rule("Critical paths") {
                filters { includes { classes("com.example.checkout.**", "com.example.auth.**") } }
                bound { minValue = 80 }
            }
        }
    }
}
```

```bash
./gradlew koverVerify   # 失敗則 CI 阻擋
```

## Cross-Skill References

- `@tech_stack_migration`：遷移前的安全網來源（兩個 skill 互補）。
- `@legacy_rapid_expansion`：島內新代碼用正規 TDD，不在本 skill 範圍。
- `@dependency_injection_mastery`：Hilt `@TestInstallIn` / `@UninstallModules` 替換策略。
- `@release_automation`：把 detekt baseline、koverage gate 接到 CI。
- `@deep_performance_tuning`：Macrobenchmark 的測試設定（不是本 skill 範圍）。

## Quick Checklist

- [ ] 高風險檔案盤點完成（churn × no test × business critical）
- [ ] 重構前先 Characterization Tests 鎖定現狀
- [ ] Robolectric 4.13+ 含 Native Graphics 啟用
- [ ] MockK relaxed 用於舊代碼，避免 stub 爆量
- [ ] Roborazzi（Compose）或 Paparazzi（View）做視覺迴歸
- [ ] Compose Preview Screenshot Test 已啟用，`@Preview` 自動轉測試
- [ ] Detekt + Lint baseline 凍結，每 sprint 減 N 條
- [ ] Golden Master 用於 HTML/JSON/PDF
- [ ] Kover gate：critical path ≥ 80% / overall ≥ 50%
- [ ] PR 阻擋 baseline 條目數**增加**
