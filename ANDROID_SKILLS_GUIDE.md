# Android Skills 完整使用教學

這份指南詳細說明如何在各種 AI 工具中使用這 14 個 Android 技能。

---

## 📚 目錄

1. [Skills 總覽](#skills-總覽)
2. [通用使用原則](#通用使用原則)
3. [各 AI 工具詳細教學](#各-ai-工具詳細教學)
   - [Antigravity (Gemini CLI/VS Code)](#a-antigravity)
   - [Cursor](#b-cursor)
   - [Windsurf](#c-windsurf)
   - [Roo Code (VS Code Extension)](#d-roo-code)
   - [Cline (VS Code Extension)](#e-cline)
   - [Aider (CLI)](#f-aider-cli)
   - [Claude Code (CLI)](#g-claude-code-cli)
   - [GitHub Copilot](#h-github-copilot)
   - [Amazon Q Developer](#i-amazon-q-developer)
   - [JetBrains AI Assistant](#j-jetbrains-ai-assistant)
   - [Claude Web / Claude Projects](#k-claude-web)
   - [ChatGPT / Custom GPT](#l-chatgpt)
   - [Google AI Studio / Gemini](#m-google-ai-studio)
   - [LLM CLI](#n-llm-cli)
   - [Codex CLI (OpenAI)](#o-codex-cli-openai)
   - [Gemini CLI (Google)](#p-gemini-cli-google)
   - [OpenInterpreter](#q-openinterpreter)
   - [Fabric](#r-fabric)
   - [Ollama + Open WebUI](#s-ollama--open-webui)
4. [場景導向完整範例](#場景導向完整範例)
5. [進階整合技巧](#進階整合技巧)
6. [常見問題](#常見問題)

---

## Skills 總覽

| # | Skill 名稱 | 用途簡述 | 檔案路徑 |
|---|-----------|---------|---------|
| 1 | `skill_index` | 技能導航中心 | `skill_index/SKILL.md` |
| 2 | `coding_style_conventions` | 代碼規範、Detekt/Ktlint | `coding_style_conventions/SKILL.md` |
| 3 | `project_bootstrapping` | 專案快速建置、Convention Plugins | `project_bootstrapping/SKILL.md` |
| 4 | `ui_ux_engineering` | Design System、Accessibility | `ui_ux_engineering/SKILL.md` |
| 5 | `dependency_injection_mastery` | Hilt 進階、Multibinding | `dependency_injection_mastery/SKILL.md` |
| 6 | `data_layer_mastery` | Room、Retrofit、Offline-First | `data_layer_mastery/SKILL.md` |
| 7 | `navigation_patterns` | Deep Links、跨模組導航 | `navigation_patterns/SKILL.md` |
| 8 | `legacy_rapid_expansion` | 舊專案快速擴充、Islanding | `legacy_rapid_expansion/SKILL.md` |
| 9 | `tech_stack_migration` | Rx→Flow、View→Compose | `tech_stack_migration/SKILL.md` |
| 10 | `testing_legacy_strategies` | 遺留代碼測試策略 | `testing_legacy_strategies/SKILL.md` |
| 11 | `deep_performance_tuning` | 效能深度優化 | `deep_performance_tuning/SKILL.md` |
| 12 | `devops_and_security` | CI/CD、安全加固 | `devops_and_security/SKILL.md` |
| 13 | `crash_monitoring` | Crashlytics、ANR 分析 | `crash_monitoring/SKILL.md` |
| 14 | `kotlin_multiplatform` | KMP 跨平台架構 | `kotlin_multiplatform/SKILL.md` |

---

## 通用使用原則

### 原則 1：Context 管理 (不浪費 Token)

```
❌ 錯誤做法：把 14 個檔案全部丟給 AI
   → Token 浪費、AI 注意力分散、回應品質下降

✅ 正確做法：根據任務只載入 2-3 個相關技能
   → Token 節省、AI 專注、回應精準
```

### 原則 2：使用場景路由 (Scenario Router)

先參考 `skill_index/SKILL.md` 選擇技能組合：

| 場景 | 描述 | 建議載入的技能 |
|------|------|--------------|
| A | 從零建立新專案 | `project_bootstrapping` + `coding_style_conventions` + `ui_ux_engineering` |
| B | 舊專案加新功能 | `legacy_rapid_expansion` + `navigation_patterns` |
| C | 舊專案全面現代化 | `testing_legacy_strategies` + `tech_stack_migration` |
| D | 效能問題排查 | `deep_performance_tuning` + `crash_monitoring` |
| E | App 發布準備 | `devops_and_security` + `deep_performance_tuning` |
| F | 跨平台共享邏輯 | `kotlin_multiplatform` + `data_layer_mastery` |

### 原則 3：明確引用章節

```
❌ 模糊指令：
   「幫我優化這段代碼」

✅ 明確指令：
   「請參考 @ui_ux_engineering 中的【Accessibility】章節，
    檢查這個按鈕的 contentDescription 和觸控目標大小」
```

### 原則 4：強制 Checklist 驗收

```
「任務完成後，請逐一對照 @coding_style_conventions 中的 Quick Checklist，
 確認每一項都符合規範」
```

---

## 各 AI 工具詳細教學

---

### A. Antigravity (Gemini CLI / VS Code)

**契合度：⭐⭐⭐⭐⭐ (完美 - 您正在使用的工具)**

Antigravity 是 Google DeepMind 開發的 Agentic AI 編程助手，支援 CLI 和 VS Code 兩種模式。這些技能就是專為 Antigravity 設計的。

#### 安裝方式

```bash
# CLI 安裝
npm install -g @anthropic-ai/antigravity

# VS Code Extension
# 在 Extensions 中搜尋 "Antigravity" 並安裝
```

#### 技能存放位置

```
~/.gemini/antigravity/skills/
├── skill_index/SKILL.md
├── coding_style_conventions/SKILL.md
├── project_bootstrapping/SKILL.md
└── ... (共 14 個)
```

#### 使用方式 1：自動識別

Antigravity 會自動掃描 `~/.gemini/antigravity/skills/` 目錄下的技能。
在對話中直接提到技能名稱即可：

```
請根據 coding_style_conventions 技能，檢查這個檔案的命名規範
```

#### 使用方式 2：使用 `@` 符號引用

```
@coding_style_conventions 請幫我檢查目前開啟的檔案

# 組合多個技能
請同時參考：
@coding_style_conventions
@dependency_injection_mastery

幫我建立一個 UserRepository
```

#### 使用方式 3：引用技能與程式碼

```
請對照 @data_layer_mastery 的 Offline-First 架構，
檢視目前開啟的 UserRepository.kt 是否符合規範
```

#### 使用方式 4：場景導向

```
# 先諮詢 skill_index
@skill_index 我要進行舊專案現代化，請推薦適合的技能組合

# AI 會推薦：
# - testing_legacy_strategies
# - tech_stack_migration
# - coding_style_conventions

# 然後引用推薦的技能
@testing_legacy_strategies @tech_stack_migration 
請按照這兩份技能的指南，幫我重構 PaymentManager
```

#### 進階：Planning Mode

Antigravity 支援 Planning Mode，適合複雜任務：

```
# 開啟 Planning Mode (在 VS Code 中)
# 使用 task.md 追蹤進度

請根據 @project_bootstrapping 建立一個完整的專案架構，
使用 Planning Mode 先規劃再執行
```

#### 進階：使用 Workflows

可以將常用流程存為 Workflow：

```markdown
<!-- .agent/workflows/android-review.md -->
---
description: Android Code Review 流程
---

1. 讀取 @coding_style_conventions 技能
2. 檢查命名規範
3. 檢查 Compose 相關規範
4. 對照 Quick Checklist
5. 生成報告
```

使用：
```
/android-review
```

#### 最佳實踐

1. **技能已內建**：這些技能放在 `~/.gemini/antigravity/skills/`，Antigravity 會自動識別
2. **使用場景路由**：先 `@skill_index` 獲取建議
3. **組合使用**：同時引用 2-3 個相關技能
4. **Checklist 驗收**：任務結束前要求使用 Quick Checklist

---

### B. Cursor

**契合度：⭐⭐⭐⭐⭐ (完美)**

Cursor 是目前最適合使用這些技能的工具之一。

#### 環境設定

1. **開啟 Cursor Settings** (`Ctrl/Cmd + ,`)
2. **進入 Features > Rules**
3. **新增 Project Rules**（可選）

#### 方法 1：使用 `@` 符號直接引用

```
# 在 Chat 中輸入
@coding_style_conventions

# 然後輸入指令
請依照這份規範，檢查目前開啟的檔案
```

#### 方法 2：引用多個技能

```
請同時參考以下規範：
@coding_style_conventions
@dependency_injection_mastery

幫我建立一個符合規範的 UserRepository，使用 Hilt 注入
```

#### 方法 3：同時引用技能與程式碼

```
請對照 @data_layer_mastery 的 Offline-First 架構，
檢視 @src/main/kotlin/data/UserRepository.kt 是否符合規範，
並列出需要修改的地方
```

#### 方法 4：使用 Cursor Rules (自動套用)

在專案根目錄建立 `.cursor/rules/android.mdc`：

```markdown
---
description: Android 開發規範
globs: ["**/*.kt", "**/*.kts"]
---

這是 Android 專案，請遵循以下規範：

1. 命名規則：參考 @coding_style_conventions
2. 架構設計：參考 @dependency_injection_mastery
3. UI 開發：參考 @ui_ux_engineering
```

#### Cursor Composer 模式

```
# 適合大規模重構
/composer

請根據 @project_bootstrapping 的架構，
幫我將這個 monolithic app 拆分成以下模組：
- :core:common
- :core:data
- :core:ui
- :feature:auth
- :feature:home
```

---

### C. Windsurf

**契合度：⭐⭐⭐⭐⭐ (完美)**

Windsurf 的 Cascade 功能非常適合多步驟任務。

#### 基本使用

```
# 開啟 Cascade (Cmd/Ctrl + L)

@skill_index 我要進行舊專案現代化，請告訴我步驟

# Cascade 會自動規劃多步驟任務
```

#### 引用技能進行任務

```
請根據 @testing_legacy_strategies：

1. 分析 @src/main/kotlin/legacy/PaymentManager.kt
2. 列出所有公開方法
3. 為每個方法生成 Characterization Test
4. 將測試檔案放在對應的 test 目錄
```

#### 使用 Windsurf Rules

在 `.windsurf/rules.md` 中：

```markdown
# Android Project Rules

當處理 Kotlin 檔案時：
- 遵循 coding_style_conventions 中的命名規則
- Compose 函數使用 PascalCase
- ViewModel 使用 Hilt 注入

當建立新模組時：
- 參考 project_bootstrapping 的 Package Structure
- 使用 Convention Plugins
```

---

### D. Roo Code

**契合度：⭐⭐⭐⭐⭐ (完美)**

VS Code Extension，支援多種 AI Provider。

#### 安裝

1. VS Code Extensions 搜尋 "Roo Code"
2. 安裝並設定 API Key

#### 使用方式

```
# 開啟 Roo Code Panel (Ctrl + Shift + P > Roo Code)

# 引用技能
請參考 @~/.gemini/antigravity/skills/coding_style_conventions/SKILL.md

幫我重構這個檔案
```

#### 使用 Custom Instructions

在 Roo Code 設定中加入：

```
當處理 Android/Kotlin 代碼時，請參考以下技能檔案：
- ~/.gemini/antigravity/skills/coding_style_conventions/SKILL.md
- ~/.gemini/antigravity/skills/ui_ux_engineering/SKILL.md
```

---

### E. Cline

**契合度：⭐⭐⭐⭐⭐ (完美)**

VS Code Extension，支援自主執行任務。

#### 安裝

1. VS Code Extensions 搜尋 "Cline"
2. 設定 API Key (支援 Claude, OpenAI, Gemini 等)

#### 使用方式

```
# 開啟 Cline (Ctrl + Shift + P > Cline: Open)

請讀取 ~/.gemini/antigravity/skills/project_bootstrapping/SKILL.md
然後根據內容，在當前目錄建立一個新的 Android 模組骨架
```

#### 使用 .clinerules

在專案根目錄建立 `.clinerules`：

```
# Android Development Rules

When working with Kotlin files:
1. Read and follow: ~/.gemini/antigravity/skills/coding_style_conventions/SKILL.md
2. For UI: ~/.gemini/antigravity/skills/ui_ux_engineering/SKILL.md
3. For DI: ~/.gemini/antigravity/skills/dependency_injection_mastery/SKILL.md

Always run Quick Checklist before completing a task.
```

---

### F. Aider (CLI)

**契合度：⭐⭐⭐⭐⭐ (高效)**

Terminal 愛好者的首選，功能強大。

#### 安裝

```bash
# 使用 pip
pip install aider-chat

# 或使用 pipx (推薦)
pipx install aider-chat
```

#### 設定環境變數

```bash
# ~/.bashrc 或 ~/.zshrc

# Claude (推薦)
export ANTHROPIC_API_KEY="your-key"

# OpenAI
export OPENAI_API_KEY="your-key"

# Gemini
export GEMINI_API_KEY="your-key"
```

#### 啟動方式

```bash
# 使用 Claude Sonnet (推薦，平衡性能與成本)
aider --model claude-3-5-sonnet-20241022

# 使用 Claude Opus (最強，成本高)
aider --model claude-3-opus-20240229

# 使用 GPT-4 Turbo
aider --model gpt-4-turbo

# 使用 GPT-4o
aider --model gpt-4o

# 使用 Gemini Pro
aider --model gemini/gemini-1.5-pro-latest

# 使用 DeepSeek (成本低)
aider --model deepseek/deepseek-chat
```

#### 基本操作流程

```bash
# Step 1: 啟動 aider
aider --model claude-3-5-sonnet-20241022

# Step 2: 加入技能檔案到 Context (唯讀)
/read ~/.gemini/antigravity/skills/coding_style_conventions/SKILL.md

# Step 3: 加入要修改的程式碼 (可寫)
/add app/src/main/kotlin/data/UserRepository.kt

# Step 4: 下達指令
請依照 coding_style_conventions 的規範重構 UserRepository，
特別注意命名規則和 Compose 相關的命名
```

#### 重要指令大全

```bash
# 檔案管理
/add <file>          # 加入可編輯的檔案
/read <file>         # 加入唯讀檔案 (適合技能)
/drop <file>         # 移除檔案
/ls                  # 列出目前載入的檔案

# 模式切換
/architect           # 切換 Architect 模式 (先規劃再執行)
/code                # 切換回 Code 模式

# 執行控制
/undo                # 撤銷上一次修改
/diff                # 顯示差異
/commit              # 提交變更
/clear               # 清除對話

# 設定
/model <name>        # 切換模型
/tokens              # 顯示 Token 使用量
```

#### 進階：場景 C 完整流程

```bash
# Step 1: 啟動並載入技能
aider --model claude-3-5-sonnet-20241022 \
  --read ~/.gemini/antigravity/skills/testing_legacy_strategies/SKILL.md \
  --read ~/.gemini/antigravity/skills/tech_stack_migration/SKILL.md

# Step 2: 加入目標檔案
/add app/src/main/kotlin/legacy/PaymentManager.kt
/add app/src/test/kotlin/legacy/PaymentManagerTest.kt

# Step 3: 建立 Characterization Tests
依照 testing_legacy_strategies 的指南，
為 PaymentManager 的所有公開方法建立 Characterization Tests

# Step 4: 執行測試確認
/run ./gradlew test --tests "PaymentManagerTest"

# Step 5: 進行遷移
測試全部通過了。
現在依照 tech_stack_migration 的 RxJava → Flow 對照表，
將 PaymentManager 中的 Observable 改為 Flow

# Step 6: 再次測試
/run ./gradlew test --tests "PaymentManagerTest"

# Step 7: 使用 Checklist 驗收
請對照 coding_style_conventions 的 Quick Checklist，
確認修改後的代碼符合所有規範
```

#### Aider 設定檔

建立 `~/.aider.conf.yml`：

```yaml
# 預設模型
model: claude-3-5-sonnet-20241022

# 自動載入規範
read:
  - ~/.gemini/antigravity/skills/coding_style_conventions/SKILL.md

# Git 設定
auto-commits: true
dirty-commits: true

# 編輯器設定
editor: code --wait
```

#### 建立 Shell Alias

```bash
# ~/.bashrc 或 ~/.zshrc

# 快速啟動 + 載入 Android 技能
alias aider-android='aider --model claude-3-5-sonnet-20241022 \
  --read ~/.gemini/antigravity/skills/coding_style_conventions/SKILL.md \
  --read ~/.gemini/antigravity/skills/dependency_injection_mastery/SKILL.md'

# 專門用於 Code Review
alias aider-review='aider --model claude-3-5-sonnet-20241022 \
  --read ~/.gemini/antigravity/skills/coding_style_conventions/SKILL.md \
  --read ~/.gemini/antigravity/skills/deep_performance_tuning/SKILL.md'

# 專門用於舊專案
alias aider-legacy='aider --model claude-3-5-sonnet-20241022 \
  --read ~/.gemini/antigravity/skills/legacy_rapid_expansion/SKILL.md \
  --read ~/.gemini/antigravity/skills/tech_stack_migration/SKILL.md'
```

---

### G. Claude Code (CLI)

**契合度：⭐⭐⭐⭐⭐ (完美)**

Anthropic 官方的 Agentic CLI 工具，支援 Skills 系統。

#### 安裝

```bash
npm install -g @anthropic-ai/claude-code
```

#### Skills 位置

```
~/.claude/skills/
├── coding_style_conventions/
│   └── SKILL.md
├── data_layer_mastery/
│   └── SKILL.md
└── ... (共 14 個 Android skills)
```

#### 使用方式 1：Slash Command（直接呼叫）

Skill 的 `name` 會自動成為 slash command：

```bash
# 啟動
claude

# 使用 slash command 直接呼叫 skill
> /coding_style_conventions
> 請檢查這段 Kotlin 代碼的命名規範

# 多個 skill 組合
> /coding_style_conventions
> /dependency_injection_mastery
> 幫我建立一個 UserRepository
```

#### 使用方式 2：自動識別

Claude 會根據 skill 的 `description` 自動判斷是否載入：

```bash
> 請檢查這段 Kotlin 代碼的命名規範
# Claude 自動識別並載入 coding_style_conventions skill
```

#### 使用方式 3：動態 Context（進階）

使用 `! command` 語法注入動態資料：

```bash
# 在 SKILL.md 中可以使用
! gh pr diff        # 注入 GitHub PR 差異
! cat package.json  # 注入檔案內容
```

#### 進階：停用 Skills

```bash
# 啟動時停用所有 skills
claude --disable-slash-commands
```

---

### H. GitHub Copilot

**契合度：⭐⭐⭐⭐⭐ (完美 - 2026 支援 Agent Skills)**

2026 年的 GitHub Copilot 支援 Agent Skills 和多層級 Instructions。

#### Skills 位置（2026 新功能）

Copilot 現在支援類似 Claude Code 的 Agent Skills：

```
.github/
├── copilot-instructions.md      # Workspace 全域指令
├── instructions/                # 檔案特定指令
│   ├── kotlin.instructions.md   # Kotlin 檔案適用
│   └── compose.instructions.md  # Compose 檔案適用
└── skills/                      # Agent Skills (2026)
    ├── coding_style_conventions/
    │   └── SKILL.md
    └── ...
```

#### 方法 1：Workspace Instructions（自動套用）

建立 `.github/copilot-instructions.md`：

```markdown
# Android 專案規範

## Kotlin Coding Style
- Class/Interface: PascalCase
- Function/Variable: camelCase
- Constants: SCREAMING_SNAKE_CASE
- @Composable: PascalCase

## Architecture
- 使用 Clean Architecture
- ViewModel 使用 Hilt @HiltViewModel
- Repository 實作 SSOT 模式

## Compose
- Modifier 作為第一個可選參數
- 使用 collectAsStateWithLifecycle
```

#### 方法 2：File-Specific Instructions（2026）

建立 `.github/instructions/kotlin.instructions.md`：

```markdown
---
applyTo: "**/*.kt"
---
處理 Kotlin 檔案時，請遵循 coding_style_conventions 技能的規範。
```

#### 方法 3：使用 @workspace 引用

```
@workspace 請根據 .github/skills/coding_style_conventions/SKILL.md，
檢查目前檔案的命名規範
```

#### 方法 4：Open Tab Context

1. **開啟 SKILL.md 作為 Tab**
2. **使用 Copilot Chat**：
```
參考我開啟的技能文件，檢查這段代碼
```

#### 方法 5：Copilot Edits

```
# 開啟 Copilot Edits (Ctrl + Shift + I)
請根據 #file:SKILL.md 中的規範，
重構 #file:UserRepository.kt
```

---

### I. Amazon Q Developer

**契合度：⭐⭐⭐⭐ (良好)**

AWS 官方的 AI Coding Assistant。

#### 使用方式

```
# 在 Chat 中使用 @file 引用
請參考 @file:.gemini/antigravity/skills/coding_style_conventions/SKILL.md
幫我檢查這段代碼
```

---

### J. JetBrains AI Assistant

**契合度：⭐⭐⭐⭐ (良好)**

Android Studio / IntelliJ IDEA 的原生 AI。

#### 方法 1：透過 Chat

1. **開啟 AI Assistant (Alt + Enter > AI)**
2. **在 Chat 中貼入技能內容**

```
請依照以下規範進行 Code Review：

[貼入 SKILL.md 內容]

現在請檢查這段代碼...
```

#### 方法 2：Custom Prompts

在 Settings > AI Assistant > Custom Prompts：

```
Name: Android Style Check
Prompt: 
依照以下 Kotlin 命名規範檢查選中的代碼：
- Class: PascalCase
- Function: camelCase
- Constants: SCREAMING_SNAKE_CASE
- Composable: PascalCase
列出所有違規項目。
```

---

### K. Claude Web

**契合度：⭐⭐⭐⭐ (良好)**

使用 Claude Projects 可大幅提升體驗。

#### 方法 1：Claude Projects (推薦)

1. **建立 Project** (claude.ai > Projects > New)
2. **命名為 "Android Development"**
3. **上傳所有 SKILL.md 到 Project Knowledge**
4. **設定 Project Instructions**：

```
你是一位資深 Android 工程師。
在回答問題時，請優先參考 Project Knowledge 中的技能文件。
每次程式碼產出後，請對照相關技能的 Quick Checklist 驗收。
```

5. **在對話中使用**：

```
請參考 coding_style_conventions 技能，
幫我審查以下 Kotlin 代碼：

```kotlin
[貼上代碼]
```
```

#### 方法 2：一般對話

```
# System Prompt 區塊
你是一位資深 Android 工程師。請依照以下規範進行開發：

---
[貼入 SKILL.md 完整內容]
---

# User Prompt
現在請幫我...
```

---

### L. ChatGPT

**契合度：⭐⭐⭐ (手動)**

#### 方法 1：Custom GPT (推薦)

1. **ChatGPT Plus > Explore GPTs > Create**
2. **配置**：
   - Name: Android Senior Engineer
   - Instructions: 貼入 `skill_index/SKILL.md` 內容
   - Knowledge: 上傳所有 SKILL.md 檔案

3. **使用**：
```
請參考 coding_style_conventions，檢查以下代碼...
```

#### 方法 2：一般對話

```
# 在對話開頭設定 Context
請扮演資深 Android 工程師，依照以下規範：

[貼入 SKILL.md 內容]

---

現在請幫我...
```

---

### M. Google AI Studio / Gemini

**契合度：⭐⭐⭐⭐ (良好)**

#### 方法 1：System Instructions

1. **開啟 AI Studio**
2. **設定 System Instructions**：

```
你是資深 Android 工程師。
請遵循以下開發規範：

[貼入技能內容]
```

#### 方法 2：上傳檔案

1. **點擊 📎 圖示**
2. **上傳 SKILL.md 檔案**
3. **下達指令**：

```
請依照上傳的規範文件，檢查以下代碼...
```

---

### N. LLM CLI

**契合度：⭐⭐⭐⭐⭐ (高效)**

Simon Willison 開發的強大 CLI 工具。

#### 安裝

```bash
pip install llm

# 安裝 Claude plugin
llm install llm-claude-3

# 設定 API Key
llm keys set anthropic
```

#### 基本使用

```bash
# 單一技能
cat ~/.gemini/antigravity/skills/coding_style_conventions/SKILL.md | \
llm "根據這份規範，給我一個標準的 Kotlin ViewModel 範例"

# 多個技能組合
cat ~/.gemini/antigravity/skills/project_bootstrapping/SKILL.md \
    ~/.gemini/antigravity/skills/dependency_injection_mastery/SKILL.md | \
llm "根據這兩份技能，幫我設計一個 Feature Module 的架構"
```

#### 搭配程式碼

```bash
# 讀取規範 + 程式碼，進行 Review
cat ~/.gemini/antigravity/skills/coding_style_conventions/SKILL.md \
    ./app/src/main/kotlin/UserRepository.kt | \
llm "請依照第一份文件的規範，Review 第二份的程式碼"
```

#### 建立 Alias

```bash
# ~/.bashrc
alias android-check='function _ac() { 
  cat ~/.gemini/antigravity/skills/coding_style_conventions/SKILL.md "$1" | \
  llm "依照規範檢查程式碼，列出問題"
}; _ac'

# 使用
android-check ./UserRepository.kt
```

---

### O. Codex CLI (OpenAI)

**契合度：⭐⭐⭐⭐⭐ (完美)**

OpenAI 官方的 Agentic Coding CLI，支援 Skills 系統。

#### 安裝

```bash
npm install -g @openai/codex-cli
```

#### Skills 位置

```
~/.codex/skills/
├── coding_style_conventions/
│   └── SKILL.md
├── data_layer_mastery/
│   └── SKILL.md
└── ... (共 14 個 Android skills)
```

#### 啟用 Skills 功能

```bash
# 啟動時啟用 skills
codex --enable skills

# 或在 ~/.codex/config.toml 中設定
# [features]
# skills = true
```

#### 使用方式 1：$ 符號引用 Skill

```bash
# 啟動
codex --enable skills

# 使用 $ 符號引用 skill
> $coding_style_conventions 請檢查這段 Kotlin 代碼

# 多個 skill
> $coding_style_conventions $dependency_injection_mastery 
> 幫我建立一個 UserRepository
```

#### 使用方式 2：/skills 指令

```bash
# 列出可用 skills
> /skills

# 查看特定 skill 詳情
> /skills coding_style_conventions
```

#### 使用方式 3：自動識別

Codex 會根據 skill 的 `description` 自動判斷：

```bash
> 請檢查這段代碼的命名規範
# Codex 自動識別並載入相關 skill
```

---

### P. Gemini CLI (Google)

**契合度：⭐⭐⭐⭐⭐ (完美)**

Google 官方的 Agentic CLI，支援 Agent Skills 系統。

#### 安裝

```bash
npm install -g @google/gemini-cli

# 或使用 preview 版本（包含最新 skills 功能）
npm install -g @google/gemini-cli@preview
```

#### Skills 位置

```
~/.gemini/skills/
├── coding_style_conventions/
│   └── SKILL.md
├── data_layer_mastery/
│   └── SKILL.md
└── ... (共 14 個 Android skills)
```

#### 啟用 Agent Skills（首次需要）

```bash
# 啟動 Gemini CLI
gemini

# 開啟設定
> /settings

# 搜尋 "Skills" → 開啟 "Agent Skills" → 按 Esc 儲存
```

#### 使用方式 1：/skills 管理指令

```bash
# 列出所有 skills
> /skills list

# 啟用特定 skill
> /skills enable coding_style_conventions

# 停用特定 skill
> /skills disable coding_style_conventions

# 重新載入 skills（新增 skill 後使用）
> /skills reload
```

#### 使用方式 2：自動識別（需確認）

Gemini 會根據 skill 的 `description` 自動判斷，但會詢問確認：

```bash
> 請檢查這段 Kotlin 代碼的命名規範

# Gemini 提示：
# 「偵測到相關 skill: coding_style_conventions
#   是否要啟用此 skill？(Y/n)」

> Y
# skill 載入後繼續執行
```

#### 使用方式 3：明確引用

```bash
> 使用 coding_style_conventions skill 來檢查代碼
```

---

### Q. OpenInterpreter

**契合度：⭐⭐⭐⭐ (良好)**

可以自主執行任務的 AI。

#### 安裝

```bash
pip install open-interpreter
```

#### 使用

```bash
# 啟動
interpreter

# 載入技能
>>> 請讀取並記住這個檔案的規範：~/.gemini/antigravity/skills/coding_style_conventions/SKILL.md

# 下達任務
>>> 現在請掃描 ./app/src/main/kotlin 下所有 .kt 檔案，
    找出不符合命名規範的地方，生成報告
```

---

### R. Fabric

**契合度：⭐⭐⭐⭐ (良好)**

Daniel Miessler 開發的 AI Pattern 系統。

#### 安裝

```bash
go install github.com/danielmiessler/fabric@latest
fabric --setup
```

#### 建立 Android Pattern

```bash
# 建立 pattern 目錄
mkdir -p ~/.config/fabric/patterns/android_review

# 建立 system.md
cat > ~/.config/fabric/patterns/android_review/system.md << 'EOF'
# IDENTITY and PURPOSE

你是資深 Android 工程師，專門進行 Code Review。

# STEPS

1. 讀取輸入的 Kotlin 代碼
2. 依照以下規範檢查
3. 列出所有違規項目
4. 提供修正建議

# RULES

## Naming
- Class/Interface: PascalCase
- Function/Variable: camelCase  
- Constants: SCREAMING_SNAKE_CASE
- @Composable: PascalCase

## Compose
- Modifier 為第一個可選參數
- 使用 remember 管理狀態

# OUTPUT

以 Markdown 格式輸出報告
EOF
```

#### 使用

```bash
# Review 單一檔案
cat ./UserRepository.kt | fabric --pattern android_review

# Review 多個檔案
find ./app/src -name "*.kt" -exec cat {} \; | fabric --pattern android_review
```

---

### S. Ollama + Open WebUI

**契合度：⭐⭐⭐⭐ (本地運行)**

完全本地運行，適合敏感專案。

#### 安裝 Ollama

```bash
# macOS/Linux
curl -fsSL https://ollama.com/install.sh | sh

# 下載模型
ollama pull llama3.1:70b
ollama pull codellama:34b
```

#### 安裝 Open WebUI

```bash
docker run -d -p 3000:8080 \
  -v open-webui:/app/backend/data \
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  ghcr.io/open-webui/open-webui:main
```

#### 使用

1. **開啟 http://localhost:3000**
2. **上傳 SKILL.md 到 Documents**
3. **在對話中使用**：

```
請參考已上傳的 coding_style_conventions 文件，
檢查以下代碼...
```

---

## 場景導向完整範例

### 場景 A：從零建立新專案 (完整流程)

#### 使用 Cursor

```
Step 1: 規劃
@skill_index 我要建立一個新的電商 App，
需要 Auth, Product, Cart, Checkout 四個功能，
請告訴我應該使用哪些技能和步驟

Step 2: 專案骨架
@project_bootstrapping 請建立完整的專案結構，包含：
- build-logic/ 目錄
- Convention Plugins (android-library, compose-feature)
- Version Catalog
- 模組結構

Step 3: 規範設定
@coding_style_conventions 請設定：
- Detekt 配置
- Ktlint 配置
- .editorconfig

Step 4: Design System
@ui_ux_engineering 請建立基礎 Design System：
- Color Scheme (Light/Dark)
- Typography
- Spacing Tokens
- 基礎 Components

Step 5: 驗收
請對照所有使用到的技能的 Quick Checklist，
確認專案設定完整
```

#### 使用 Aider

```bash
# 啟動
aider --model claude-3-5-sonnet-20241022 \
  --read ~/.gemini/antigravity/skills/skill_index/SKILL.md \
  --read ~/.gemini/antigravity/skills/project_bootstrapping/SKILL.md \
  --read ~/.gemini/antigravity/skills/coding_style_conventions/SKILL.md \
  --read ~/.gemini/antigravity/skills/ui_ux_engineering/SKILL.md

# 執行
我要建立一個電商 App 專案，請依照載入的技能：
1. 先依照 project_bootstrapping 建立專案骨架
2. 再依照 coding_style_conventions 設定 linter
3. 最後依照 ui_ux_engineering 建立 Design System

請建立所有必要的檔案
```

---

### 場景 C：舊專案現代化 (完整流程)

#### 使用 Aider

```bash
# Step 1: 啟動
aider --model claude-3-5-sonnet-20241022 \
  --read ~/.gemini/antigravity/skills/testing_legacy_strategies/SKILL.md \
  --read ~/.gemini/antigravity/skills/tech_stack_migration/SKILL.md \
  --read ~/.gemini/antigravity/skills/coding_style_conventions/SKILL.md

# Step 2: 分析目標
/add app/src/main/kotlin/legacy/PaymentManager.kt

請分析這個類別：
1. 列出所有公開方法
2. 識別使用的 RxJava 操作
3. 找出潛在的問題點

# Step 3: 建立測試
/add app/src/test/kotlin/legacy/PaymentManagerTest.kt

依照 testing_legacy_strategies 的 Characterization Tests 指南，
為 PaymentManager 的每個公開方法建立測試

# Step 4: 執行測試
/run ./gradlew test --tests "*PaymentManagerTest*"

# Step 5: 遷移
測試通過了。現在依照 tech_stack_migration：
1. 將 Observable 改為 Flow
2. 將 subscribe 改為 collect
3. 處理錯誤使用 catch

# Step 6: 再次測試
/run ./gradlew test --tests "*PaymentManagerTest*"

# Step 7: 程式碼風格
依照 coding_style_conventions 的 Quick Checklist 檢查最終代碼
```

---

## 進階整合技巧

### 1. Git Pre-commit Hook

```bash
#!/bin/bash
# .git/hooks/pre-commit

# 取得 staged 的 Kotlin 檔案
STAGED=$(git diff --cached --name-only --diff-filter=ACM | grep "\.kt$")

if [ -n "$STAGED" ]; then
  echo "Running AI Code Review..."
  
  for file in $STAGED; do
    cat ~/.gemini/antigravity/skills/coding_style_conventions/SKILL.md "$file" | \
    llm "依照規範快速檢查，只回報嚴重問題" || exit 1
  done
fi
```

### 2. VS Code Task

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Android: Code Review",
      "type": "shell",
      "command": "cat ${workspaceFolder}/.gemini/antigravity/skills/coding_style_conventions/SKILL.md ${file} | llm '依照規範 Review 代碼'",
      "problemMatcher": []
    },
    {
      "label": "Android: Generate Tests",
      "type": "shell", 
      "command": "aider --model claude-3-5-sonnet-20241022 --read ${workspaceFolder}/.gemini/antigravity/skills/testing_legacy_strategies/SKILL.md ${file} --message '為這個類別生成 Characterization Tests'",
      "problemMatcher": []
    }
  ]
}
```

### 3. PowerShell 函數 (Windows)

```powershell
# $PROFILE

function Invoke-AndroidReview {
    param([string]$FilePath)
    
    $skill = Get-Content "$env:USERPROFILE\.gemini\antigravity\skills\coding_style_conventions\SKILL.md" -Raw
    $code = Get-Content $FilePath -Raw
    
    "$skill`n---`n$code" | llm "依照規範 Review 代碼"
}

Set-Alias -Name android-review -Value Invoke-AndroidReview
```

---

## 常見問題

### Q1: Token 不夠怎麼辦？
**A:** 
- 只載入當下需要的 2-3 個技能
- 使用 `skill_index` 的場景路由選擇組合
- 對於 CLI 工具，使用 `-read` 而非 `/add`

### Q2: AI 沒有遵循規範怎麼辦？
**A:** 
- 在指令最後加上：「請逐一對照 Quick Checklist」
- 分步驟執行，每步驟驗證
- 明確指定要參考的章節

### Q3: 如何更新技能內容？
**A:** 
- 直接編輯對應的 SKILL.md
- 所有工具會自動讀取最新版本
- 建議使用 Git 追蹤變更

### Q4: 哪個工具最推薦？
**A:** 
- **Gemini 使用者**: Antigravity / Gemini CLI (技能已內建)
- **Claude 使用者**: Claude Code CLI (使用 /skill_name)
- **OpenAI 使用者**: Codex CLI (使用 $skill_name)
- **最佳體驗**: Cursor / Windsurf
- **CLI 愛好者**: Aider
- **完全本地**: Ollama + Open WebUI
- **團隊共享**: Claude Projects

---

## 快速參考卡

```
┌───────────────────────────────────────────────────────────────────────┐
│                    Android Skills 快速使用 (2026)                      │
├───────────────────────────────────────────────────────────────────────┤
│                        🔥 CLI 工具 (Native Skills)                     │
├───────────────────────────────────────────────────────────────────────┤
│ CLAUDE CODE    │ /skill_name (Slash Command)                         │
│ CODEX CLI      │ $skill_name (需 --enable skills)                    │
│ GEMINI CLI     │ /skills list, /skills enable (需開啟 Agent Skills)  │
├───────────────────────────────────────────────────────────────────────┤
│                        🖥️ Agentic IDE                                  │
├───────────────────────────────────────────────────────────────────────┤
│ ANTIGRAVITY    │ @skill_name (技能已內建，自動識別)                    │
│ CURSOR         │ @skill_name                                         │
│ WINDSURF       │ @skill_name 在 Cascade 中                            │
│ ROO CODE       │ @~/.gemini/skills/xxx/SKILL.md                      │
│ CLINE          │ 請讀取 ~/.claude/skills/xxx/SKILL.md                 │
├───────────────────────────────────────────────────────────────────────┤
│                        🔧 其他 CLI 工具                                │
├───────────────────────────────────────────────────────────────────────┤
│ AIDER          │ /read ~/.claude/skills/xxx/SKILL.md                 │
│ LLM CLI        │ cat SKILL.md | llm "指令"                           │
│ FABRIC         │ cat code.kt | fabric --pattern android_review       │
├───────────────────────────────────────────────────────────────────────┤
│                        🌐 Web / IDE                                    │
├───────────────────────────────────────────────────────────────────────┤
│ COPILOT        │ Open Tab + @workspace                               │
│ CLAUDE WEB     │ Projects > Knowledge 上傳                            │
│ CHATGPT        │ Custom GPT > Knowledge 上傳                          │
│ OLLAMA         │ Open WebUI > Documents 上傳                          │
└───────────────────────────────────────────────────────────────────────┘
```

### Skills 安裝位置對照

```
┌─────────────────────────────────────────────────────────────┐
│                    2026 CLI Skills 標準位置                  │
├─────────────────────────────────────────────────────────────┤
│ Claude Code  │ ~/.claude/skills/<skill-name>/SKILL.md      │
│ Codex CLI    │ ~/.codex/skills/<skill-name>/SKILL.md       │
│ Gemini CLI   │ ~/.gemini/skills/<skill-name>/SKILL.md      │
└─────────────────────────────────────────────────────────────┘
```


