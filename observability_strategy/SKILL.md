---
id: observability_strategy
name: Observability Strategy
description: Android 可觀測性的「設計層」— SLI/SLO 建模、告警門檻矩陣、結構化事件 schema、feedback loop 與 Play Console App Quality Insights 整合，產出指標策略而非工具實作
type: skill
---

# Observability Strategy

## Instructions

本 skill 處理「**該觀測什麼、定多嚴、警報接給誰、回饋怎麼閉環**」的策略決策；不處理「Crashlytics SDK 怎麼裝」「Timber 怎麼接 Logcat」等實作（那是 `@crash_monitoring`）。當使用者問題涉及指標選擇、SLO 建模、告警閾值、事件 schema、儀表板規劃時載入。

## When to Use

- Scenario O：可觀測性建模（事故頻繁、無法量化體驗、要把品質納入決策）
- 設計新功能的 SLI/SLO 與告警門檻
- 整理現有事件 schema、消除噪音事件
- 跨團隊談「品質基線」與「上線條件」
- 規劃 Play Console + Firebase + 自家 BI 的訊號分工

## When NOT to Use

- 工具安裝與 SDK 設定 → `@crash_monitoring`
- 效能瓶頸定位、Macrobenchmark、Perfetto trace → `@deep_performance_tuning`
- 告警觸發後的 CI Gate / Release Stop → `@release_automation`
- 安全/供應鏈相關事件 → `@supply_chain_security`

## Example Prompts

- 「設計支付流程的 SLI/SLO 與告警門檻」
- 「我們事件太多 dashboard 看不出問題，幫我重構 event schema」
- 「Crash 率 0.1% 算高嗎？告警該怎麼設？」
- 「Play Console 的 ANR rate 跟 Firebase 不一致，要以哪個為準？」
- 「上線前要看哪些指標才能決定能不能放量？」

## Workflow

1. **Map Critical User Journeys**：列出 3-5 條關鍵流程（Cold Start、Login、Search、Checkout、Playback）。
2. **Define SLI**（量化指標）→ **Define SLO**（目標）→ **Define Error Budget**（容忍度）。
3. **Design Event Schema**：每個 journey 對應一組結構化事件，欄位統一。
4. **Set Alert Thresholds**：分 P0/P1/P2 三級，含 burn rate alert。
5. **Wire Feedback Loop**：事故 → 警報 → on-call → 修復 → 驗證 → 回填指標。

## Practical Notes (2026-04)

| 訊號來源 | 角色定位 | 即時性 |
|----------|----------|--------|
| Firebase Crashlytics | Crash/ANR/Custom Signals 的事件流 | 秒-分鐘 |
| Play Console (App Quality Insights, Vitals) | 真實用戶分母、ANR/Crash 行業基準 | 1-3 天延遲 |
| Firebase Performance / Macrobenchmark | 啟動、Trace、自訂 metric | 分鐘 |
| 自家事件平台（GA4/BigQuery/Datadog/NewRelic） | 商業 KPI 與技術指標融合 | 分鐘-小時 |
| JankStats / FrameMetrics | UI 卡頓細節 | 即時 |

**原則**：Play Console 是「真相之源」（unbiased denominator），Firebase 是「即時告警源」，兩者互校。

## Minimal Template

```
Journey: <名稱，例：Cold Start to First Screen>
SLI: <量化定義，例：P95 time-to-first-frame, ms>
SLO: <目標，例：P95 < 1500 ms, 30-day rolling>
Error Budget: <容忍度，例：5% of users above SLO>
Event Schema:
  - event_name: <snake_case>
  - required_attrs: [build, locale, network, ...]
Alerts:
  - P0: <條件>, <通知對象>, <SLA>
  - P1: ...
  - P2: ...
Owner: <團隊/人員>
Review cadence: <weekly/monthly>
```

## SLI / SLO 建模

### SLI 選擇原則

- **以使用者為單位**，不是以請求/事件為單位（避免重度用戶被稀釋）
- **二元化**：每個事件可標記 good/bad；good 比例就是 SLI
- **避免重複量測**：同一現象只挑一個 SLI 主指標 + 至多兩個 supporting

### 範例：Checkout 流程

| 構面 | SLI | SLO | Error Budget（30 天） |
|------|-----|-----|------------------------|
| 可用性 | `checkout_success / checkout_attempt` | ≥ 99.5% | 0.5% × MAU 次失敗 |
| 延遲 | `P95(checkout_complete_ms)` | ≤ 3000 ms | P95 超標時數 ≤ 5% |
| 穩定性 | `crash_free_users on Checkout screen` | ≥ 99.9% | 0.1% × DAU |

### Error Budget Policy

- 預算消耗 0-50%：正常開發節奏
- 50-90%：凍結非關鍵 PR、加強監控
- > 90%：停止 release、所有人優先 burn-down

## 告警門檻矩陣

採用 **multi-window, multi-burn-rate** 警報，避免過敏與遲鈍：

```
P0 (Page on-call immediately):
  - 1h burn rate > 14.4x   AND   5m burn rate > 14.4x
  → 1h 燒完 30 天預算的 2%

P1 (Slack within working hours):
  - 6h burn rate > 6x      AND   30m burn rate > 6x
  → 6h 燒完 5%

P2 (Daily digest):
  - 3d burn rate > 1x
  → 持續性退化
```

具體門檻（Crashlytics 範例）：

| 等級 | Crash-Free Users | ANR Rate | Velocity Alert |
|------|------------------|----------|----------------|
| 健康 | ≥ 99.5% | ≤ 0.47% (Vitals 良好門檻) | < 0.1% |
| 警告 (P1) | 99.0-99.5% | 0.47-1.0% | 0.1-0.5% |
| 緊急 (P0) | < 99.0% | > 1.0% | > 0.5%（單版本） |

## Event Schema 設計

### 通用必備欄位（所有事件）

```jsonc
{
  "event_name": "checkout_complete",     // snake_case
  "event_id": "uuid",                    // 去重用
  "session_id": "uuid",
  "user_id_hash": "sha256(...)",         // PII 去識別
  "timestamp_ms": 1714200000000,
  "build": "1.42.0 (3201)",
  "build_type": "release",
  "device": { "model": "Pixel 8", "os": 35, "locale": "zh-TW" },
  "network": "wifi|cellular|none",
  "ab_buckets": { "checkout_v3": "treatment" }
}
```

### Journey 專屬欄位

```jsonc
{
  "journey": "checkout",
  "step": "payment_confirmed",
  "result": "success|user_cancel|system_error",
  "error_code": "PAYMENT_DECLINED",      // 失敗時必填
  "duration_ms": 1850,
  "trace_id": "..."                      // 串接後端
}
```

### Schema 治理

- 每個事件版本化：`event_name + schema_version`
- 新增欄位走 additive only；刪除欄位需先加 deprecated flag 兩個 release
- 入庫前 schema validation（Protobuf / JSON Schema）阻擋不合規事件

## Feedback Loop 系統圖

```
   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
   │ Production   │───►│ Crashlytics  │───►│ PagerDuty/   │
   │ Devices      │    │ Firebase Perf│    │ Slack        │
   │ (App)        │    │ Play Console │    │ (Alert)      │
   └──────────────┘    └──────┬───────┘    └──────┬───────┘
          ▲                   │                   │
          │                   ▼                   ▼
          │            ┌──────────────┐    ┌──────────────┐
          │            │ BigQuery /   │◄───│ On-call      │
          │            │ Datadog      │    │ Triage       │
          │            └──────┬───────┘    └──────┬───────┘
          │                   │                   │
          │                   ▼                   ▼
          │            ┌──────────────┐    ┌──────────────┐
          │            │ SLO/Burndown │    │ Hotfix /     │
          │            │ Dashboard    │    │ Rollback     │
          │            └──────┬───────┘    └──────┬───────┘
          │                   │                   │
          └───────────────────┴───────────────────┘
                       Verify & Close Loop
```

每個事故必有事後檢討（postmortem）回填：是否新增 SLI、是否調整告警、是否補測試。

## Play Console App Quality Insights 整合

- **作為基準分母**：Play Vitals 是裝置端真實採樣，不被 SDK 初始化失敗污染。
- **每週對齊**：Crashlytics crash count vs Play Console crash rate × DAU 應在 ±10% 內；超過代表 SDK 漏報或事件重複。
- **強指標**：Play Vitals 的 user-perceived ANR rate 是 Play Store 排名因子，作為 P0 告警的 ground truth。
- 新版本上架後 24 小時內看 staged rollout 視窗的 Vitals，異常即停推。

## Dashboard 規劃

三層儀表板：

1. **Executive**（單頁）：3 條關鍵 SLO 的 30 天 burn 圖，看就懂。
2. **On-call**：所有 P0/P1 alert 的即時狀態 + 對應 runbook 連結。
3. **Engineering**：事件流、版本對比、user cohort drill-down。

每個 dashboard 都有 **Owner** 與 **Last reviewed** 標籤；超過 90 天沒人看的儀表板砍掉。

## Cross-Skill References

- `@crash_monitoring`：Crashlytics SDK、ANR Watchdog、結構化日誌的「實作層」。
- `@deep_performance_tuning`：根因分析（Macrobenchmark、Perfetto、Memory Profiler）的「定位層」。
- `@release_automation`：把 SLO 違反掛成 CI Gate 與 Play Console staged rollout 自動停推。
- `@supply_chain_security`：依賴/簽章相關事件（如 Sigstore 驗證失敗）的觀測。
- `@legacy_rapid_expansion`：Hybrid 過渡期的 island-level SLO 設計。

## Quick Checklist

- [ ] 3-5 條 critical journey 已盤點且有 owner
- [ ] 每條 journey 有 SLI / SLO / Error Budget
- [ ] Event schema 統一，含 schema_version 與必備欄位
- [ ] 告警採 multi-burn-rate，分 P0/P1/P2
- [ ] Play Console vs Firebase 數據定期對校
- [ ] On-call runbook 與 dashboard owner 明確
- [ ] Postmortem 機制可回填 SLI/告警/測試
- [ ] 90 天無人看的儀表板已下架
