---
id: supply_chain_security
name: Supply Chain Security
description: Android 供應鏈安全 — 依賴治理、SCA（OSV-Scanner / Dependabot）、SBOM（CycloneDX/SPDX）、SLSA Provenance、Sigstore 簽章、Signing & Secrets 管理、License Compliance，產出可審計的供應鏈防護
type: skill
---

# Supply Chain Security

## Instructions

本 skill 涵蓋從「第三方依賴進入專案」到「APK/AAB 簽章上架」的完整供應鏈防護。Secrets 與 Signing 在 2026-04 重構時由 `release_automation` 移交至此，集中治理。CI/CD pipeline 本身的設定與 build optimization 仍屬 `@release_automation`。

## When to Use

- Scenario N：供應鏈合規（SOC2 / ISO 27001 / 上市公司審計）
- 依賴漏洞被 OSV / NVD 點名要評估與修補
- 建立或重整 SBOM、簽章、provenance pipeline
- Secrets 外洩事件後的密鑰輪轉與 repo 清理
- License audit（OSS 商業使用合規）

## When NOT to Use

- 應用層執行時安全（Cert Pinning、Root Detection、Network Security Config）→ `@release_automation`
- App 內部資料加密（Keystore、EncryptedSharedPreferences）→ `@data_layer_mastery`
- CI workflow 設計與 build cache → `@release_automation`
- Code signing 在 Play Console 的設定步驟（操作層）→ Play Console 文件，但密鑰管理策略在此

## Example Prompts

- 「OSV-Scanner 報出 transitive dep 有 CVE，要怎麼追到 root 並修？」
- 「審計要求我們提供 SBOM，每個 release 都要附，怎麼自動化？」
- 「想用 Sigstore keyless signing 簽 release artifact，Android 怎麼做？」
- 「signing key 誤推到 GitHub 了，怎麼輪轉並清理 history？」
- 「我們用了 GPL library，怎麼知道有沒有違反授權？」

## Workflow

1. **Inventory**：產生 SBOM，列出所有 direct + transitive 依賴。
2. **Govern**：Version Catalog 鎖版本、設定 repository 白名單、禁用 dynamic version。
3. **Scan**：每 PR 跑 OSV-Scanner / Dependabot，設定漏洞嚴重度門檻。
4. **Sign**：Release 用 Play App Signing + 自家 upload key + Sigstore 補充 provenance。
5. **Audit**：保存 SBOM + provenance + scan report 至 immutable storage（≥ 1 年）。

## Practical Notes (2026-04)

| 工具 | 用途 | 推薦理由 |
|------|------|----------|
| OSV-Scanner | 依賴漏洞掃描 | Google 官方、覆蓋 OSV.dev DB、JVM/Maven 支援好 |
| Dependabot | 自動 PR 升級 | 與 GitHub 整合最深 |
| CycloneDX Gradle Plugin | SBOM 產生 | 業界主流、CycloneDX 1.5/SPDX 雙格式 |
| SLSA GitHub Generator | Build provenance | SLSA Level 3，GitHub Actions 原生支援 |
| Sigstore (cosign) | Keyless signing | 短壽命憑證、Rekor 透明日誌 |
| FOSSA / Black Duck / ScanCode | License compliance | 商用 vs OSS 二選一 |
| TruffleHog / gitleaks | Secret scanning | pre-commit + CI 雙層 |

## Minimal Template

```
目標: <例：每月 release 附 SBOM + SLSA L3 provenance>
依賴清單來源: gradle/libs.versions.toml
SCA 工具: OSV-Scanner (CI), Dependabot (PR)
SBOM 格式: CycloneDX 1.5 (生產), SPDX 2.3 (備用)
簽章策略:
  - Upload key: AWS KMS (HSM-backed)
  - Distribution: Play App Signing
  - Provenance: cosign + Rekor
Secret store: GitHub OIDC + cloud KMS（無長壽憑證）
驗收: Quick Checklist
```

## Dependency Governance

### Version Catalog 為唯一來源

```toml
# gradle/libs.versions.toml
[versions]
kotlin = "2.1.0"
hilt = "2.52"
retrofit = "2.11.0"

[libraries]
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
retrofit = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
```

```kotlin
// settings.gradle.kts — repository 白名單
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        // 禁用 jcenter()、jitpack 等未審核來源
    }
}
```

### 禁用 dynamic version

```kotlin
// build-logic/.../GuardConventions.kt
configurations.all {
    resolutionStrategy {
        failOnDynamicVersions()      // 禁 1.2.+
        failOnChangingVersions()     // 禁 SNAPSHOT
        failOnNonReproducibleResolution()
    }
}
```

### Dependency Locking

```bash
./gradlew dependencies --write-locks
git add gradle.lockfile
```

每次升級必須重新生成 lock file 並通過 review。

## SCA：漏洞掃描與處置

### OSV-Scanner CI 整合

```yaml
# .github/workflows/sca.yml
name: SCA
on: [pull_request]
jobs:
  osv-scan:
    uses: google/osv-scanner-action/.github/workflows/osv-scanner-reusable.yml@main
    with:
      scan-args: |-
        --recursive
        --skip-git
        ./
      fail-on-vuln: true
```

### 漏洞處置 SLA

| 嚴重度 | 處置時限 | CI 行為 |
|--------|----------|---------|
| Critical (CVSS 9.0+) | 24 小時 | 阻擋 merge、阻擋 release |
| High (7.0-8.9) | 7 天 | 阻擋 merge |
| Medium (4.0-6.9) | 30 天 | 警告但可 merge |
| Low (< 4.0) | 90 天 | 記錄即可 |

### Transitive 漏洞修補策略

```kotlin
// 強制升級 transitive dep（不改 direct dep 版本）
configurations.all {
    resolutionStrategy {
        force("com.fasterxml.jackson.core:jackson-databind:2.17.2")
        eachDependency {
            if (requested.group == "io.netty" && requested.version!!.startsWith("4.1.")) {
                useVersion("4.1.115.Final")
                because("CVE-2024-XXXXX")
            }
        }
    }
}
```

每個 force 必須有 `because` 註記，並追蹤上游發版。

## SBOM 生成（CycloneDX）

```kotlin
// build.gradle.kts
plugins {
    id("org.cyclonedx.bom") version "1.10.0"
}

tasks.cyclonedxBom {
    setIncludeConfigs(listOf("releaseRuntimeClasspath"))
    setSkipConfigs(listOf("debugRuntimeClasspath"))
    setOutputFormat("json")
    setOutputName("sbom-${project.version}")
    setSchemaVersion("1.5")
    setProjectType("application")
}
```

CI 步驟：

```yaml
- name: Generate SBOM
  run: ./gradlew :app:cyclonedxBom

- name: Upload SBOM artifact
  uses: actions/upload-artifact@v4
  with:
    name: sbom
    path: app/build/reports/sbom-*.json
    retention-days: 365
```

審計要求 SPDX 時用 `spdx-gradle-plugin` 並存兩份。

## SLSA Provenance（Build Integrity）

目標：Level 3 — provenance 由可信 builder 簽發、不可竄改。

```yaml
# .github/workflows/release.yml
jobs:
  build:
    outputs:
      hashes: ${{ steps.hash.outputs.hashes }}
    steps:
      - uses: actions/checkout@v4
      - run: ./gradlew :app:bundleRelease
      - id: hash
        run: |
          echo "hashes=$(sha256sum app/build/outputs/bundle/release/*.aab | base64 -w0)" >> "$GITHUB_OUTPUT"

  provenance:
    needs: build
    permissions:
      id-token: write
      contents: write
      actions: read
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v2.0.0
    with:
      base64-subjects: ${{ needs.build.outputs.hashes }}
      provenance-name: app-release.intoto.jsonl
```

驗證：

```bash
slsa-verifier verify-artifact app-release.aab \
  --provenance-path app-release.intoto.jsonl \
  --source-uri github.com/org/repo
```

## Sigstore Keyless Signing

避免長壽 signing key 外洩風險。Android AAB 不能直接用 cosign（需 jarsigner-compatible），但可對「附隨 artifact」（SBOM、provenance、mapping.txt）做 cosign。

```yaml
- uses: sigstore/cosign-installer@v3
- name: Sign SBOM
  env:
    COSIGN_EXPERIMENTAL: "true"
  run: |
    cosign sign-blob \
      --bundle sbom.cosign.bundle \
      app/build/reports/sbom-${VERSION}.json
```

驗證者只需 cosign + repo 名，不需密鑰。

## Signing & Secrets（從 release_automation 移入）

### Upload Key 託管

- **生產**：私鑰存於 cloud KMS（GCP KMS / AWS KMS / HashiCorp Vault），HSM-backed。CI 透過短壽 OIDC 換取簽章權限，**不下載私鑰**。
- **本地開發**：使用 debug keystore（不可上 release）。
- **App Signing**：開啟 Play App Signing，由 Google 持有 distribution key；upload key 僅用於上傳。

### GitHub Actions OIDC + GCP KMS 範例

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  release:
    steps:
      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/X/locations/global/workloadIdentityPools/gh/providers/gh
          service_account: ci-signer@project.iam.gserviceaccount.com

      - name: Sign AAB via KMS
        run: |
          gcloud kms ... # 產生簽章請求
          jarsigner -keystore NONE -storetype PKCS11 \
            -providerClass sun.security.pkcs11.SunPKCS11 \
            -providerArg pkcs11.cfg \
            app-release.aab key-alias
```

### Secret Scanning（防外洩）

```yaml
# .github/workflows/secrets.yml
- uses: trufflesecurity/trufflehog@main
  with:
    extra_args: --only-verified --fail
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.0
    hooks:
      - id: gitleaks
```

### Secret 外洩應變

1. **立即輪轉**該密鑰（KMS rotate / 重簽 token）。
2. `git filter-repo` 清除歷史並 force push（評估影響範圍後）。
3. 告知所有 fork / clone 重新 clone。
4. 寫 postmortem 並指向 `@observability_strategy` 補相對應的告警。

## License Compliance

```kotlin
plugins {
    id("com.github.jk1.dependency-license-report") version "2.9"
}

licenseReport {
    allowedLicensesFile = file("$rootDir/config/allowed-licenses.json")
    filters = arrayOf(
        com.github.jk1.license.filter.LicenseBundleNormalizer()
    )
}
```

```json
// config/allowed-licenses.json
{
  "allowedLicenses": [
    { "moduleLicense": "Apache.*" },
    { "moduleLicense": "MIT" },
    { "moduleLicense": "BSD.*" }
  ]
}
```

CI 步驟：

```bash
./gradlew checkLicense || exit 1
```

GPL/AGPL/SSPL 預設禁用；商業專案使用 GPL library 需法務簽核並記錄豁免。

## Cross-Skill References

- `@release_automation`：CI workflow 框架、Build Speed、應用層安全（Cert Pinning / NSC）；本 skill 提供 SCA / SBOM / Signing 區塊讓它組裝。
- `@platform_modernization_2026`：升級 Gradle/AGP 後需重生 SBOM、重跑 OSV-Scanner。
- `@observability_strategy`：把 SCA 失敗、簽章驗證失敗事件納入告警。
- `@project_bootstrapping`：新專案直接套用本 skill 的 Version Catalog + 鎖版本範例。

## Quick Checklist

- [ ] Version Catalog + repository 白名單 + dependency locking 生效
- [ ] OSV-Scanner / Dependabot 在 PR 與 main 雙觸發
- [ ] 漏洞處置 SLA（Critical 24h / High 7d）寫入 README
- [ ] CycloneDX SBOM 每 release 自動產生並保留 ≥ 365 天
- [ ] SLSA Level 3 provenance 由 GitHub OIDC builder 產生
- [ ] Sigstore 對 SBOM / mapping.txt 等附隨 artifact 簽章
- [ ] Upload key 在 KMS、CI 用 OIDC 取權限，不存 base64 secret
- [ ] TruffleHog / gitleaks 在 pre-commit + CI 雙層執行
- [ ] License audit 通過、GPL/AGPL 已標註與審核
- [ ] Secret 外洩應變 runbook 文件化
