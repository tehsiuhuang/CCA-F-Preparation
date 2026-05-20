# Pre/Post Tool Use Hooks 完整指南

## 核心概念

```
Hook = 控制 tool 使用前後的行為

PreToolUse  = tool 執行「前」的攔截點
PostToolUse = tool 執行「後」的攔截點
```

任何地方只要能「呼叫 tool」，就可以「定義 hook」來控制 tool 執行的前後行為。

---

## Tool 執行的完整流程

```
Model 決策發出 tool_use
  ↓
Claude Code 攔截到 tool_use block
  ↓
所有匹配的「Pre-Tool-Use Hook」執行
  驗證 input、安全檢查、修改參數
  ↓
實際執行 Tool
  Read、Grep、Bash 等
  ↓
所有匹配的「Post-Tool-Use Hook」執行
  驗證 output、處理結果、快取
  ↓
結果加入 messages
```

---

## Pre/Post Tool Use Hooks 可以定義在 5 個地方

根據官方文檔，hooks 可以定義在以下 5 個層級（每個都會觸發）：

### 1. Settings 檔案

```
位置：
  ~/.claude/settings.json (User-level，所有 projects)
  .claude/settings.json (Project-level，shared)
  .claude/settings.local.json (Project-level，個人)
  Managed policy settings (Org-level，admin 控制)

範圍：全局，所有 tool calls 都會觸發
角色："全局守門員"
```

### 2. Plugin

```
位置：
  plugin's hooks/hooks.json

範圍：只在 plugin 啟用時運作
角色："Plugin 守門員"
```

### 3. Skill (frontmatter)

```
位置：
  .claude/skills/<name>/SKILL.md

範圍：只在 skill 啟用時運作
       一旦這個 skill 不再使用，hook 也停止
       可以用 once: true 限制只運作一次
角色："Skill 守門員"
```

### 4. Agent (frontmatter)

```
位置：
  .claude/agents/<name>.md

範圍：只在 agent 運作期間生效
       Subagent 結束時，hook 也結束
角色："Agent 守門員"
```

### 5. Command (frontmatter)

```
位置：
  .claude/commands/<name>.md

範圍：只在 command 執行期間運作
角色："Command 守門員"
```

---

## 為什麼會這樣設計

```
不同層級需要不同的控制：

Settings 層級:
  例：全局禁止 rm -rf
  原因：保護整個系統

Agent 層級:
  例：security-auditor 對 Read 有更嚴格的檢查
  原因：這個 agent 處理敏感資訊

Skill 層級:
  例：encryption-skill 在 Bash 前驗證密碼學工具
  原因：這個 skill 需要特殊保護

Command 層級:
  例：/deploy command 在所有 Edit 後執行 linter
  原因：deploy 流程需要品質檢查

Plugin 層級:
  例：plugin 提供的 tool 自帶安全檢查
  原因：plugin 自己控制自己的 tools
```

---

## 實際的執行順序

```
當 tool 被呼叫時，所有匹配的 hooks 都會運作，疊加生效：

[Settings 的 PreToolUse hooks]   ← 全局
       ↓
[Plugin 的 PreToolUse hooks]      ← 如果 plugin 啟用
       ↓
[Agent 的 PreToolUse hooks]       ← 如果在 agent 內
       ↓
[Skill 的 PreToolUse hooks]       ← 如果 skill 啟用
       ↓
[Command 的 PreToolUse hooks]     ← 如果在 command 內
       ↓
   執行 Tool
       ↓
[Command 的 PostToolUse hooks]
       ↓
[Skill 的 PostToolUse hooks]
       ↓
[Agent 的 PostToolUse hooks]
       ↓
[Plugin 的 PostToolUse hooks]
       ↓
[Settings 的 PostToolUse hooks]
       ↓
   結果加入 messages
```

---

## 各個層級的範例

### Settings 中的 Hook

```json
// .claude/settings.json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/check.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/lint.sh"
          }
        ]
      }
    ]
  }
}
```

### Skill 中的 Hook

```markdown
---
name: secure-operations
description: Perform operations with security checks
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/check.sh"
          once: true  # 每個 session 只運作一次
  PostToolUse:
    - matcher: "Read"
      hooks:
        - type: command
          command: "./scripts/scan-secrets.sh"
---

You are performing secure operations...
```

### Agent 中的 Hook

```markdown
---
name: security-auditor
tools: [Read, Grep]
hooks:
  PreToolUse:
    - matcher: "Read"
      hooks:
        - type: command
          command: "./scripts/validate-file.sh"
  PostToolUse:
    - matcher: "Read"
      hooks:
        - type: command
          command: "./scripts/scan-vulnerabilities.sh"
---

You are a security auditor...
```

### Command 中的 Hook

```markdown
---
description: Comprehensive code review
agent: coordinator
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/check-deploy-safety.sh"
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "./scripts/lint.sh"
---

Review the code...
```

### Plugin 中的 Hook

```json
// plugin's hooks/hooks.json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Plugin_specific_tool",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/validate.sh"
          }
        ]
      }
    ]
  }
}
```

---

## Hooks 都疊加生效

**重點：所有層級的 hooks 都會同時運作，不是擇一**

```
範例情況：

Settings 設定全局：
  PostToolUse: linter (所有 Edit/Write 後執行)

Coordinator agent 設定：
  PreToolUse: security check (所有 Bash 前)

Code-review-checklist skill 設定：
  PostToolUse: formatter (所有 Edit/Write 後)

當 model 發出 tool_use: Edit 時：

Step 1: PreToolUse 階段
  - Settings 的 hooks (沒有匹配 Edit)
  - Coordinator 的 hooks (沒有匹配 Edit)
  - Skill 的 hooks (沒有匹配 Edit)
  → 沒有 hook 觸發

Step 2: 執行 Edit tool

Step 3: PostToolUse 階段
  - Skill 的 formatter ✓ 執行
  - Coordinator 的 hooks (沒有 PostToolUse)
  - Settings 的 linter ✓ 執行

結果：兩個 PostToolUse hooks 都執行了
```

---

## Pre vs Post Hook 的作用對比

### Pre-Tool-Use Hook 的常見作用

```
目的：在執行 tool 之前做防禦性檢查

常見用途：
1. 輸入驗證 (Input Validation)
   - 檢查 file path 有效性
   - 檢查 regex pattern 安全
   - 驗證參數長度

2. 安全檢查 (Security Checks)
   - 檢查文件是否包含密鑰
   - 檢查路徑是否有目錄遍歷
   - 驗證使用者權限

3. 資源限制 (Resource Limits)
   - 限制檔案大小
   - 限制 pattern 複雜度

4. 修改參數 (Parameter Modification)
   - 正規化路徑
   - 補充預設值

5. 阻止 (Block)
   - 阻止危險操作 (例如 rm -rf)
   - 退出 code 2 來阻止
```

### Post-Tool-Use Hook 的常見作用

```
目的：在執行 tool 之後做結果處理

常見用途：
1. 輸出驗證 (Output Validation)
   - 檢查結果格式正確
   - 驗證編碼有效

2. 結果處理 (Result Processing)
   - 移除敏感資訊
   - 格式化輸出
   - 排序或去重

3. 快取 (Caching)
   - 快取常見查詢結果

4. 記錄 (Logging)
   - 記錄 tool 執行
   - 追蹤使用量

5. 自動化動作
   - 執行 linter
   - 執行 formatter
   - 觸發 notifications
```

---

## 簡單記

```
Hook 是什麼？
  → 控制 tool 使用前後的行為

可以定義在哪？
  → 任何能「呼叫 tool」的地方
  → 5 個層級：Settings、Plugin、Skill、Agent、Command

執行順序？
  → 所有匹配的 hooks 都會疊加運作

PreToolUse vs PostToolUse？
  → Pre 在 tool 執行前
  → Post 在 tool 執行後

範圍 (scope) 不同？
  → Settings: 全局
  → Plugin/Skill/Agent/Command: 啟用期間
```

---

## 流程圖簡化版

```
Coordinator Loop:

Model → tool_use
  ↓
[Settings PreHook] [Plugin PreHook] [Agent PreHook] [Skill PreHook] [Command PreHook]
  ↓
Execute Tool
  ↓
[Command PostHook] [Skill PostHook] [Agent PostHook] [Plugin PostHook] [Settings PostHook]
  ↓
Add to messages
  ↓
Loop
```

---

生成於：2025-05-08
更新：加入完整的 5 個 Hook 定義位置和執行順序
