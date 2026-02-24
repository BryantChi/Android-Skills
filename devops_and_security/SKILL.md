---
id: devops_and_security
name: DevOps and Security
description: CI/CD 自動化、Gradle 優化與應用程式安全加固
---

# DevOps and Security (DevOps 與資安)

## Instructions
- 僅在發布準備或流程自動化需求時使用
- 依照下方章節順序套用
- 一次只處理一個 pipeline 或安全措施
- 完成後對照 Quick Checklist

## When to Use
- Scenario E：App 發布準備

## Example Prompts
- "請依照 Build Speed Optimization，調整 Gradle 設定"
- "用 CI Quality Gates 章節建立 GitHub Actions"
- "請參考 Security Hardening，檢查 secrets 與網路安全"

## Workflow
1. 先建立 Build Speed 與 CI Quality Gates
2. 再導入 Fastlane 與 Security Hardening
3. 最後用 Quick Checklist 驗收

## Practical Notes (2026)
- 依賴來源與版本必有審核與鎖定策略
- Secrets 僅能透過環境變數或安全儲存
- CI Gate 必含 Lint/Detekt/Test/Build

## Minimal Template
```
目標: 
CI Gate: 
安全措施: 
發版流程: 
驗收: Quick Checklist
```

---

## Build Speed Optimization

### Configuration Cache

```kotlin
// gradle.properties
org.gradle.configuration-cache=true
org.gradle.configuration-cache.problems=warn
```

### Build Cache

```kotlin
// settings.gradle.kts
buildCache {
    local {
        directory = File(rootDir, "build-cache")
        removeUnusedEntriesAfterDays = 7
    }
    
    // 企業級：Remote cache
    remote<HttpBuildCache> {
        url = uri("https://cache.example.com/")
        isPush = System.getenv("CI") != null
    }
}
```

### Parallel Execution

```kotlin
// gradle.properties
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.jvmargs=-Xmx4g -XX:+HeapDumpOnOutOfMemoryError
```

---

## CI Quality Gates

### GitHub Actions 範例

```yaml
name: Android CI

on:
  pull_request:
    branches: [main, develop]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup JDK
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Lint
        run: ./gradlew lintDebug
      
      - name: Detekt
        run: ./gradlew detekt
      
      - name: Unit Tests
        run: ./gradlew testDebugUnitTest
      
      - name: Build
        run: ./gradlew assembleDebug
```

### Danger for PR Review

```ruby
# Dangerfile
# APK Size Check
apk_size = File.size("app/build/outputs/apk/debug/app-debug.apk") / 1024.0 / 1024.0
warn "APK size is #{apk_size.round(2)}MB" if apk_size > 50

# Kotlin Files changed
kotlin_files = git.modified_files.select { |f| f.end_with?(".kt") }
warn "Large PR with #{kotlin_files.count} Kotlin files" if kotlin_files.count > 20
```

---

## Fastlane Automation

### Fastfile

```ruby
default_platform(:android)

platform :android do
  
  desc "Deploy to Play Store Internal Track"
  lane :internal do
    gradle(task: "bundleRelease")
    upload_to_play_store(
      track: "internal",
      aab: "app/build/outputs/bundle/release/app-release.aab"
    )
  end
  
  desc "Promote Internal to Production"
  lane :promote do
    upload_to_play_store(
      track: "internal",
      track_promote_to: "production",
      skip_upload_apk: true,
      skip_upload_aab: true
    )
  end
end
```

---

## Security Hardening

### Secrets Management

```kotlin
// build.gradle.kts
android {
    buildFeatures {
        buildConfig = true
    }
    
    buildTypes {
        release {
            buildConfigField("String", "API_KEY", "\"${System.getenv("API_KEY")}\"")
        }
    }
}

// 使用 secrets-gradle-plugin
plugins {
    id("com.google.android.libraries.mapsplatform.secrets-gradle-plugin")
}

// local.properties (不 commit)
API_KEY=your_secret_key
```

### Certificate Pinning

```kotlin
// OkHttp
val certificatePinner = CertificatePinner.Builder()
    .add("example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
    .build()

val client = OkHttpClient.Builder()
    .certificatePinner(certificatePinner)
    .build()
```

### Root Detection

```kotlin
class RootDetection {
    fun isDeviceRooted(): Boolean {
        return checkRootBinaries() || checkSuExists() || checkRootCloaking()
    }
    
    private fun checkRootBinaries(): Boolean {
        val paths = arrayOf("/system/bin/su", "/system/xbin/su", "/sbin/su")
        return paths.any { File(it).exists() }
    }
}
```

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
        <domain includeSubdomains="true">example.com</domain>
        <pin-set>
            <pin digest="SHA-256">...</pin>
        </pin-set>
    </domain-config>
</network-security-config>
```

---

## Quick Checklist

- [ ] Build Cache 啟用 (Local + Remote)
- [ ] CI 包含 Lint, Detekt, Unit Test
- [ ] Fastlane 自動化部署
- [ ] Secrets 不進版控
- [ ] Certificate Pinning 啟用
- [ ] Network Security Config 禁止 Cleartext
