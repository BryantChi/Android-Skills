---
name: Testing Legacy Strategies
description: 為無測試的舊代碼建立安全網的策略
---

# Testing Legacy Strategies (遺留代碼測試)

**Related Scenarios**: C (舊專案現代化), F (KMP)

---

## Characterization Tests (現狀測試)

為沒有測試的舊代碼撰寫「現狀測試」，不管對錯，先鎖定行為。

### 策略

```kotlin
// 1. 先寫一個會失敗的測試
@Test
fun `calculateDiscount returns unknown value`() {
    val result = legacyCalculator.calculateDiscount(100.0, "VIP")
    assertEquals(0.0, result)  // 故意用錯誤的預期值
}

// 2. 執行測試，記錄實際回傳值
// AssertionError: expected 0.0 but was 15.0

// 3. 更新測試為實際值
@Test
fun `calculateDiscount returns 15 percent for VIP`() {
    val result = legacyCalculator.calculateDiscount(100.0, "VIP")
    assertEquals(15.0, result)  // 鎖定現有行為
}
```

### 批量生成

```kotlin
@ParameterizedTest
@CsvSource(
    "100.0, VIP, 15.0",
    "100.0, REGULAR, 5.0",
    "50.0, VIP, 7.5"
)
fun `calculateDiscount characterization`(
    price: Double,
    tier: String,
    expected: Double
) {
    assertEquals(expected, legacyCalculator.calculateDiscount(price, tier))
}
```

---

## Robolectric (Android Framework 測試)

處理高度依賴 Android Framework 的舊單元測試。

### 設定

```kotlin
// build.gradle.kts
testImplementation("org.robolectric:robolectric:4.11.1")

// 測試類別
@RunWith(RobolectricTestRunner::class)
@Config(sdk = [33])
class LegacyActivityTest {
    
    @Test
    fun `activity displays correct title`() {
        val activity = Robolectric.buildActivity(LegacyActivity::class.java)
            .create()
            .resume()
            .get()
        
        assertEquals("Expected Title", activity.title)
    }
}
```

### Shadow 使用

```kotlin
@Test
fun `shows toast on error`() {
    activity.showError("Network failed")
    
    val toast = ShadowToast.getLatestToast()
    assertEquals("Network failed", ShadowToast.getTextOfLatestToast())
}
```

---

## MockK for Kotlin

### 基本用法

```kotlin
@Test
fun `repository returns cached data`() {
    val repository = mockk<UserRepository>()
    every { repository.getUser("123") } returns User(id = "123", name = "Test")
    
    val result = repository.getUser("123")
    
    assertEquals("Test", result.name)
    verify { repository.getUser("123") }
}
```

### Coroutines Support

```kotlin
@Test
fun `suspend function mocking`() = runTest {
    val api = mockk<UserApi>()
    coEvery { api.fetchUser("123") } returns User(id = "123")
    
    val result = api.fetchUser("123")
    
    coVerify { api.fetchUser("123") }
}
```

### Relaxed Mocks

```kotlin
// 自動回傳預設值，適合舊代碼測試
val service = mockk<LegacyService>(relaxed = true)
```

---

## Detekt/Lint Baseline

漸進式收緊舊專案的品質標準。

### 生成 Baseline

```bash
# Detekt
./gradlew detektBaseline
# 產生 config/detekt/baseline.xml

# Lint
./gradlew lintDebug -Dlint.baselines.continue=true
# 產生 lint-baseline.xml
```

### 只檢查新代碼

```kotlin
// build.gradle.kts
android {
    lint {
        baseline = file("lint-baseline.xml")
    }
}

detekt {
    baseline = file("config/detekt/baseline.xml")
}
```

### 漸進式修復

```bash
# 每個 Sprint 減少 Baseline 中的項目
# 1. 修復一批問題
# 2. 重新生成 Baseline
./gradlew detektBaseline
```

---

## Golden Master Testing

適合複雜輸出 (HTML, JSON) 的舊代碼。

```kotlin
@Test
fun `report generator produces expected output`() {
    val output = legacyReportGenerator.generate(testData)
    
    // 首次執行：儲存為 golden file
    // File("src/test/resources/golden/report.html").writeText(output)
    
    // 後續執行：比對
    val golden = File("src/test/resources/golden/report.html").readText()
    assertEquals(golden, output)
}
```

---

## Quick Checklist

- [ ] 重構前先撰寫 Characterization Tests
- [ ] 使用 Robolectric 處理 Android 依賴
- [ ] MockK 的 relaxed mock 加速舊代碼測試
- [ ] Detekt/Lint Baseline 控制技術債
- [ ] 每個 Sprint 減少 Baseline 項目
