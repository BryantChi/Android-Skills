---
id: observability_first
name: Observability First
description: Crash/ANR/Logs/Performance 指標與回饋閉環
---

# Observability First (可觀測性優先)

## Instructions
- 先定義要觀測的關鍵流程與指標
- 依序建立 Crash/ANR、結構化日誌、效能指標
- 一次只補強一類訊號，避免噪音擴散
- 完成後對照 Quick Checklist

## When to Use
- 發布前需要穩定監控與回饋閉環
- 事故頻繁但缺乏可定位資訊
- 需要把效能與穩定性納入日常決策

## Example Prompts
- "請建立支付流程的可觀測性指標與事件"
- "請設計 Crash/ANR 的告警門檻"
- "幫我建立結構化日誌的欄位規格"

## Workflow
1. 定義關鍵流程與 SLO
2. 建立 Crash/ANR 與結構化事件
3. 加入效能指標與告警門檻
4. 建立回饋迴路（分析 -> 修復 -> 驗證）

## Practical Notes (2026)
- 先有指標再談優化，避免主觀調整
- 事件欄位需一致，便於查詢與彙整
- 告警門檻要可執行且可回溯

## Minimal Template
```
目標: 
關鍵流程: 
指標/事件: 
告警門檻: 
驗收: Quick Checklist
```

---

## Signals & SLOs

### 關鍵流程清單
- 啟動、登入、主要交易流程
- 網路請求成功率、平均延遲
- ANR/Crash 率與回復時間

### 指標分級
- P0: Crash/ANR 率、關鍵流程失敗率
- P1: 首次渲染時間、列表滾動流暢度
- P2: 特定功能的轉換率或完成率

---

## Structured Events

### 事件欄位規格
- event_name, flow_id, user_tier, build_version
- latency_ms, result, error_code

### 事件紀錄原則
- 只記錄高價值事件，避免淹沒
- 同一流程使用一致欄位

---

## Crash / ANR Strategy

- Crash/ANR 需標註關鍵上下文欄位
- Non-Fatal 只記錄高價值錯誤
- 針對高頻問題建立告警

---

## Performance Signals

- Startup、列表滾動、關鍵頁面渲染
- 量測結果進 CI Gate

---

## Quick Checklist

- [ ] 關鍵流程與 SLO 定義完成
- [ ] Crash/ANR 欄位一致且可追蹤
- [ ] 事件欄位可查詢且可彙整
- [ ] 效能指標有量測與門檻
- [ ] 告警與回饋迴路已建立
