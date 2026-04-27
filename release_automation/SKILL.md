---
id: release_automation
name: Release Automation
description: Android CI/CD 自動化 — Build Speed（Configuration Cache 9 / Build Cache）、CI Quality Gates、Fastlane 發版、Play Console staged rollout、Play Integrity API gate、應用層執行時安全（Cert Pinning、NSC、Root Detection），產出可重現的 release pipeline
type: skill
---

# Release Automation

## Instructions

本 skill 處理「**從 commit 到 Play Store**」的整個 pipeline 自動化與應用層執行時安全。Secrets / Signing / SBOM / SCA 自 2026-04 起由 `@supply_chain_security` 接管；本 skill 只負責 pipeline 框架與執行時防護。

## When to Use

- Scenario E：發布準備與流程自動化
- Scenario G：CI 速度退化排查
- 需要 staged rollout、自動 promote、自動 hotfix
- 加上 Play Integrity 作為登入/支付前的裝置完整性檢查
- 強化 App 執行時的 network / TLS / root 防護

## When NOT to Use

- 依賴漏洞、SBOM、簽章密鑰管理 → `@supply_chain_security`
- 程式碼品質規範（Detekt/Ktlint）的「規則本身」→ `@coding_style_conventions`（本 skill 只負責把它接到 CI）
- 效能 benchmark 在 CI 的設定 → `@deep_performance_tuning`（本 skill 提供 gate 機制）
- App 內部資料加密 → `@data_layer_mastery`

## Example Prompts

- 「幫我建立 GitHub Actions 含 Lint/Detekt/Test/Macrobench 四道 gate」
- 「設定 Fastlane 從 internal 自動 promote 到 production，含 staged rollout」
- 「Configuration Cache 在 Gradle 9 開了之後 task 一直失敗，怎麼修？」
- 「登入前要做 Play Integrity 檢查，後端怎麼驗證 token？」
- 「Cert Pinning 過期那天會炸，怎麼設熱更新？」

## Workflow

1. **Build Foundation**：Configuration Cache + Build Cache + Parallel + Remote Cache。
2. **CI Gates**：Lint → Detekt → Unit Test → Instrumentation → Macrobench → Build。
3. **Release Pipeline**：Fastlane lane（internal → alpha → beta → production），各段 gate。
4. **Runtime Hardening**：Cert Pinning、NSC、Root/Tamper Detection、Play Integrity。
5. **Observability Hook**：rollout 後監看 `@observability_strategy` 的 SLO，超標自動停推。

## Practical Notes (2026-04)

| 元件 | 版本 | 備註 |
|------|------|------|
| GitHub Actions runner | ubuntu-24.04 | JDK 21 預裝 |
| `actions/setup-java` | v4 | v3 已過時 |
| `actions/checkout` | v4 | |
| Gradle | 9.0+ | Configuration Cache 預設開 |
| AGP | 8.7+ | Variant API 穩定 |
| Fastlane | 2.222+ | 支援 Play Console v3 API |
| Play Integrity API | 1.4+ | 取代已棄用 SafetyNet |

## Minimal Template

```
目標: <例：每次 PR < 12 分鐘 CI、release 全自動 staged rollout>
CI Gates: lint, detekt, unit, instrumentation, macrobench, build
Pipeline: feature branch → PR → main → internal → alpha → production
Rollout: 1% → 5% → 20% → 100%（每段觀察 24h SLO）
Runtime hardening: Cert Pinning + NSC + Play Integrity
驗收: Quick Checklist
```

## Build Speed Optimization

### Configuration Cache（Gradle 9 預設）

```properties
# gradle.properties
org.gradle.configuration-cache=true
org.gradle.configuration-cache.problems=fail
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.jvmargs=-Xmx4g -XX:+HeapDumpOnOutOfMemoryError -XX:+UseG1GC
```

常見破壞 Configuration Cache 的寫法：

```kotlin
// ❌ task action 內取 project
tasks.register("bad") {
    doLast { println(project.version) }
}

// ✅ 用 Provider
tasks.register("good") {
    val v = providers.provider { project.version.toString() }
    doLast { println(v.get()) }
}
```

### Build Cache（Local + Remote）

```kotlin
// settings.gradle.kts
buildCache {
    local { removeUnusedEntriesAfterDays = 7 }
    remote<HttpBuildCache> {
        url = uri(providers.gradleProperty("build.cache.url").get())
        isPush = providers.environmentVariable("CI").isPresent
        credentials {
            username = providers.environmentVariable("BUILD_CACHE_USER").get()
            password = providers.environmentVariable("BUILD_CACHE_PASS").get()
        }
    }
}
```

CI cache hit rate 目標 > 80%；可在 Gradle Enterprise / Develocity dashboard 監看。

### Module Graph 最小化

- `buildFeatures { viewBinding = false; dataBinding = false }`（除非真用到）
- `kotlin.incremental = true`、`android.useAndroidX = true`
- 避免單一模組 > 200 檔；定期重切分

## CI Quality Gates

### GitHub Actions（reusable workflow）

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
  push:
    branches: [main]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  validate:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'

      - uses: gradle/actions/setup-gradle@v4
        with:
          cache-read-only: ${{ github.ref != 'refs/heads/main' }}

      - run: ./gradlew lintDebug detekt ktlintCheck testDebugUnitTest --scan

      - name: Upload reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: reports
          path: '**/build/reports/**'

  instrumentation:
    runs-on: ubuntu-24.04-large
    needs: validate
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: '21' }
      - name: Run on Firebase Test Lab
        uses: google-github-actions/auth@v2
        with: { workload_identity_provider: ${{ secrets.WIP }}, service_account: ${{ secrets.SA }} }
      - run: |
          ./gradlew :app:bundleDebug :app:bundleDebugAndroidTest
          gcloud firebase test android run \
            --app app/build/outputs/bundle/debug/app-debug.aab \
            --test app/build/outputs/bundle/debugAndroidTest/app-debug-androidTest.aab \
            --device model=MediumPhone.arm,version=33

  macrobench:
    runs-on: ubuntu-24.04
    needs: validate
    if: github.ref == 'refs/heads/main'
    steps:
      - run: ./gradlew :benchmark:connectedReleaseAndroidTest
```

### Gate 矩陣

| Gate | Trigger | 失敗影響 |
|------|---------|----------|
| Lint / Detekt / Ktlint | 每 PR | 阻擋 merge |
| Unit Test + Coverage | 每 PR | 阻擋 merge（cov < 70% 警告） |
| Instrumentation (FTL) | 每 PR | 阻擋 merge |
| Macrobenchmark | merge to main | 退化 > 5% 阻擋 release |
| OSV-Scanner / SBOM | 每 PR | 見 `@supply_chain_security` |

### PR 大小提示（Danger）

```ruby
# Dangerfile
warn("PR 變更檔案 > 30，考慮拆分") if git.modified_files.count > 30
fail("缺少 PR description") if github.pr_body.length < 20
```

## Fastlane 發版

```ruby
# fastlane/Fastfile
default_platform(:android)

platform :android do
  desc "Build & upload to Internal track"
  lane :internal do
    gradle(task: "bundleRelease")
    upload_to_play_store(
      track: "internal",
      aab: "app/build/outputs/bundle/release/app-release.aab",
      skip_upload_metadata: true,
      skip_upload_changelogs: false,
      release_status: "completed"
    )
  end

  desc "Promote to Production with staged rollout"
  lane :promote do |options|
    upload_to_play_store(
      track: "internal",
      track_promote_to: "production",
      rollout: options[:rollout] || "0.05",   # 5% staged
      skip_upload_apk: true,
      skip_upload_aab: true,
      skip_upload_metadata: true,
      skip_upload_changelogs: true
    )
  end

  desc "Halt rollout (emergency)"
  lane :halt do
    upload_to_play_store(
      track: "production",
      rollout: "0",
      skip_upload_apk: true,
      skip_upload_aab: true
    )
  end
end
```

Staged rollout 配套 SLO 自動停推：

```yaml
# .github/workflows/rollout-watch.yml
on:
  schedule: [{ cron: '*/30 * * * *' }]
jobs:
  watch:
    runs-on: ubuntu-24.04
    steps:
      - name: Check Crashlytics SLO
        run: |
          rate=$(curl -s -H "Authorization: Bearer $TOKEN" \
            "https://firebase.googleapis.com/.../crash_free_users")
          if (( $(echo "$rate < 99.0" | bc -l) )); then
            bundle exec fastlane halt
            curl -X POST $SLACK_WEBHOOK -d '{"text":"🚨 Rollout halted: crash-free users < 99%"}'
          fi
```

## Play Integrity API Gate

替代已棄用的 SafetyNet。客戶端要 token，後端驗證。

```kotlin
// 客戶端
val integrityManager = IntegrityManagerFactory.create(context)
val nonce = generateNonce()  // 從後端取
val request = IntegrityTokenRequest.builder()
    .setNonce(nonce)
    .setCloudProjectNumber(BuildConfig.GCP_PROJECT_NUMBER)
    .build()

integrityManager.requestIntegrityToken(request)
    .addOnSuccessListener { response ->
        val token = response.token()
        api.verifyIntegrity(token)   // 送後端驗證
    }
```

```kotlin
// 後端（Kotlin server）— 使用 Google Play Integrity API
val verdict = playIntegrityApi.decodeIntegrityToken(token, nonce)
require(verdict.deviceIntegrity.deviceRecognitionVerdict.contains("MEETS_DEVICE_INTEGRITY"))
require(verdict.appIntegrity.appRecognitionVerdict == "PLAY_RECOGNIZED")
require(verdict.accountDetails.appLicensingVerdict == "LICENSED")
```

CI Gate：每次 release build 跑一次 token 取得 + 後端驗證，確認專案號與簽章對應正確。

## 應用層執行時安全

### Certificate Pinning（含熱更新）

```kotlin
// 用 Trust Kit 或自訂可熱更新的 pinner
class DynamicPinner(private val configRepo: PinConfigRepo) : CertificatePinner {
    override fun check(hostname: String, peerCertificates: List<Certificate>) {
        val pins = configRepo.pinsFor(hostname)  // 從 Remote Config 取
        // 比對 pins，至少一個吻合；皆不吻合則 throw SSLPeerUnverifiedException
    }
}

val client = OkHttpClient.Builder()
    .certificatePinner(DynamicPinner(configRepo))
    .build()
```

避免硬編碼 pin 過期 → 失聯。Remote Config 同時保留兩組 pin（current + next），輪換時先推 next 再切 current。

### Network Security Config

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
  <base-config cleartextTrafficPermitted="false">
    <trust-anchors>
      <certificates src="system" />
    </trust-anchors>
  </base-config>
  <domain-config>
    <domain includeSubdomains="true">api.example.com</domain>
    <pin-set expiration="2027-01-01">
      <pin digest="SHA-256">PRIMARY_PIN==</pin>
      <pin digest="SHA-256">BACKUP_PIN==</pin>
    </pin-set>
  </domain-config>
  <debug-overrides>
    <trust-anchors>
      <certificates src="user" />
    </trust-anchors>
  </debug-overrides>
</network-security-config>
```

### Root / Tamper Detection（信號型）

不要把 root detection 當權限決策；當作風險信號上報後端。

```kotlin
class IntegritySignals {
    fun collect(): List<String> = buildList {
        if (File("/system/xbin/su").exists()) add("su_binary")
        if (File("/system/app/Magisk.apk").exists()) add("magisk")
        if (Build.TAGS?.contains("test-keys") == true) add("test_keys")
        if (Debug.isDebuggerConnected()) add("debugger_attached")
    }
}

// 真正的決策：呼叫 Play Integrity API（信任較高）
```

### R8 / ProGuard 強化

```pro
# proguard-rules.pro
-keepattributes SourceFile,LineNumberTable
-renamesourcefileattribute SourceFile

# 防止 reflection attack 暴露 model
-keep,allowobfuscation class com.example.api.dto.** { *; }
-keep public class * extends androidx.compose.runtime.Composer
```

## Cross-Skill References

- `@supply_chain_security`：依賴掃描、SBOM、Signing 與 Secrets 管理（CI 用 OIDC 取簽章權限）。
- `@platform_modernization_2026`：Gradle 9 / AGP 8.7+ 升級時的 Configuration Cache 修復。
- `@coding_style_conventions`：CI Gate 中 Detekt / Ktlint 規則本身的維護。
- `@deep_performance_tuning`：Macrobenchmark 規則設計與 baseline；本 skill 把它接到 gate。
- `@observability_strategy`：rollout 自動停推的 SLO 來源與閾值定義。
- `@crash_monitoring`：rollout 期間實時 crash-free users 的拉取來源。

## Quick Checklist

- [ ] Configuration Cache + Build Cache 雙開，CI cache hit > 80%
- [ ] CI 含 Lint / Detekt / Ktlint / Unit / Instrumentation / Macrobench 六道 gate
- [ ] Fastlane lane：internal / promote / halt 全部可用
- [ ] Staged rollout 從 1% 起，配 SLO 自動 halt
- [ ] Play Integrity API 取代 SafetyNet，後端 token 驗證生效
- [ ] Cert Pinning 從 Remote Config 熱更新，含 backup pin
- [ ] NSC 禁 cleartext，pin 含 expiration 與雙 pin
- [ ] R8 / ProGuard 開且 mapping.txt 上傳 Crashlytics
- [ ] Secrets 全走 OIDC + KMS（見 `@supply_chain_security`）
- [ ] Rollout watch workflow 與 Slack/PagerDuty 串接
