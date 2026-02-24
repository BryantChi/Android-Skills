---
id: supply_chain_security
name: Supply Chain Security
description: 依賴治理、SCA、簽章與密鑰管理
---

# Supply Chain Security (供應鏈安全)

## Instructions
- 先盤點依賴來源與版本策略
- 建立 SCA 掃描與審核流程
- 一次只強化一個供應鏈節點
- 完成後對照 Quick Checklist

## When to Use
- 專案依賴多、更新頻繁
- 發布前需要風險檢查
- 需要建立依賴治理標準

## Example Prompts
- "請設計依賴版本鎖定與更新策略"
- "幫我加上 SCA 掃描與風險門檻"
- "請建立密鑰與簽章管理規範"

## Workflow
1. 建立依賴來源與版本鎖定策略
2. 加入 SCA 掃描與審核流程
3. 設定簽章與密鑰管理規範
4. 將風險門檻納入 CI Gate

## Practical Notes (2026)
- 版本鎖定與審核是最小安全基線
- 依賴更新與安全修補分開處理
- SCA 結果必須有處置規則

## Minimal Template
```
目標: 
依賴來源: 
版本策略: 
SCA 門檻: 
驗收: Quick Checklist
```

---

## Dependency Governance

- 使用 Version Catalog 作為單一來源
- 重大更新需審核與回歸測試
- 禁止未授權的第三方來源

---

## SCA / Vulnerability Policy

- 將依賴掃描納入 CI
- 高風險漏洞阻擋合併
- 低風險需標註與期限修補

---

## Signing & Secrets

- 密鑰僅由環境變數或安全儲存注入
- Release 簽章流程需可追蹤
- 禁止在 Repo 內存放敏感資訊

---

## Quick Checklist

- [ ] 依賴來源與版本鎖定完成
- [ ] SCA 掃描納入 CI Gate
- [ ] 風險處置規則明確
- [ ] 簽章與密鑰管理可追蹤
- [ ] Release 前完成依賴風險檢查
