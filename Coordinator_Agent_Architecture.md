# Coordinator Agent 完整架構說明

## 核心概念

Coordinator Agent 是一個主 agent，負責協調多個 subagents 完成複雜任務。

```
Coordinator Agent
├─ 讀 prompt（決策規則）
├─ 掃描 skills（詳細指導）
├─ 根據 context 決策
│  ├─ 要不要用什麼 skill
│  ├─ 要不要產生什麼 subagent
│  └─ 要跑什麼 tool
└─ 產生 subagents 並整合結果
```

---

## 誰啟動 Coordinator Agent

### 啟動的決策鏈

```
User 是起點
  ↓
User 選擇方式：

方式 A: 打 Command
  User: /comprehensive-review auth.py
    ↓
  Claude Code 讀取 command
    ↓
  Command 指定 agent: coordinator
    ↓
  啟動 Coordinator 作為 Main Agent

方式 B: 打普通 Message
  User: "Review this code"
    ↓
  Main Agent 啟動
    ↓
  Main Agent 的 Claude Model 決策
    「我應該產生 coordinator subagent 嗎？」
    ↓
    如果決策「是」→ spawn coordinator 作為 subagent
    如果決策「否」→ main agent 自己處理
```

### 兩種啟動方式的對比

#### 方式 1: 作為 Main Agent（透過 Command）

```
User: /comprehensive-review auth.py
  ↓
Claude Code 讀取 .claude/commands/comprehensive-review.md
  看到：agent: coordinator
  ↓
Claude Code 讀取 .claude/agents/coordinator.md
  ↓
啟動 Coordinator Agent Loop（作為 main agent）
  └─ 自己的 Claude Model
  └─ 自己的 messages context
  └─ 自己的 loop
  ↓
Coordinator 開始決策、掃描 skills、產生 subagents
  ↓
最後回報結果給用戶
```

**優點**：
- 清晰、明確
- User 明確選擇用 coordinator
- Coordinator 有完整的 main 身份

**使用時機**：
- User 知道需要複雜協調
- 任務適合多 agent 合作

#### 方式 2: 作為 Subagent（被 Main Agent Spawn）

```
User: "Review this code"
  ↓
Main Agent 啟動（default agent）
  ↓
Main Agent 的 Claude Model 看著 prompt
  決策：「這個任務需要複雜的協調」
  ↓
  發出 Task tool_use: spawn coordinator
  ↓
Claude Code 執行 Task tool
  讀取 .claude/agents/coordinator.md
  ↓
啟動 Coordinator Agent Loop（作為 subagent）
  └─ 隔離的 context
  └─ 自己的 loop
  └─ 自己的 Claude Model
  ↓
Coordinator 在自己的 loop 裡工作
  ↓
完成後回報 task result 給 main agent
  ↓
Main agent 看 result，決定下一步或完成
```

**優點**：
- 自動化決策
- Main agent 可以靈活選擇
- 不需要 user 預先知道

**使用時機**：
- User 不確定是否需要複雜協調
- Main agent 自動評估決策

### 誰決定「要開始 Coordinator」

```
直接答案：User 決定

但決策的方式不同：

方式 A（Command）：
  User 直接選擇
  → /comprehensive-review 就是選擇 coordinator
  → 明確決策

方式 B（Main Agent Spawn）：
  User 間接選擇
  → 寫普通 message
  → Main agent 評估後決策 spawn coordinator
  → 間接決策
```

---

## 完整的啟動到 Loop 流程

### 場景 A: 透過 Command 啟動

```
1. User 打 Command
   /comprehensive-review auth.py

2. Claude Code 識別 Command
   讀 .claude/commands/comprehensive-review.md
   
3. 檢查 Agent 綁定
   看 frontmatter: agent: coordinator
   
4. 讀取 Agent Definition
   讀 .claude/agents/coordinator.md
   
5. 初始化 Coordinator Loop
   初始化 messages
   初始化 coordinator 環境

6. 開始 Agent Loop
   Cycle 1: 掃描 skills → 推薦 → 呼叫 model
   Cycle 2: 執行決策
   Cycle 3: 可能 spawn subagents
   ...
   直到 stop_reason: end_turn
```

### 場景 B: 被 Main Agent Spawn

```
1. User 打普通 Message
   "Review this code"

2. Main Agent 啟動
   讀 main agent definition
   
3. Main Agent Loop Cycle 1
   掃描 skills
   呼叫 model

4. Main Agent 的 Model 決策
   「我應該產生 coordinator subagent」
   發出 Task tool_use: spawn coordinator
   input: {
     subagent_type: "coordinator",
     prompt: "幫我協調代碼審查..."
   }

5. Claude Code 執行 Task Tool
   讀 .claude/agents/coordinator.md
   啟動 Coordinator 作為 subagent

6. Coordinator Loop（隔離執行）
   有自己的 messages
   有自己的 loop
   獨立運作

7. Coordinator 完成
   回傳 task result 給 main agent

8. Main Agent Loop 繼續
   看 coordinator 的 result
   決策下一步
```

---

## 三層結構

### 第 1 層：Agent Definition (.claude/agents/)

**文件位置**：`.claude/agents/security-auditor.md`

**給誰看**：被 spawn 出來的 security-auditor agent

**內容包括**：
- Frontmatter：name, tools, model
- Body：agent 的角色、工作說明

**Model 看嗎**：❌ 不看

**例子**：
```markdown
---
name: security-auditor
tools: [Read, Grep, Bash]
model: sonnet
---

你是一個安全審計員。
你的工作是找出代碼中的安全漏洞。

檢查項目：
1. Input validation
2. Authentication
3. Authorization
4. Password handling
```

**何時被使用**：
- Coordinator 決策產生 subagent 時
- Claude Code 執行 Task tool 時
- Subagent 啟動並運作

---

### 第 2 層：Skill Definition (.claude/skills/)

**文件位置**：`.claude/skills/security-checklist/SKILL.md`

**給誰看**：Coordinator model（或任何 agent）

**內容包括**：
- Frontmatter：description, allowed-tools, context
- Body：步驟、檢查項目、詳細指導

**Model 看嗎**：✅ 一定要看 body

**例子**：
```markdown
---
name: security-checklist
description: Security audit checklist and steps
allowed-tools: [Read, Grep]
context: inherit
---

代碼安全審查檢查清單

## 第一步：Input Validation
檢查所有用戶輸入是否都被驗證：
- 沒有 SQL injection
- 沒有 XSS
- 沒有 command injection

## 第二步：Authentication
檢查認證機制：
- 密碼是否被正確哈希
- 會話是否安全管理

## 第三步：Data Protection
檢查資料保護：
- 敏感資料是否加密
- 是否使用安全的存儲方式
```

**何時被使用**：
- Claude Code 每個 cycle 掃描 skills
- 根據相似度推薦給 Model
- Model 看 body 的指導決策

---

### 第 3 層：Tool Execution

**來源**：
- Built-in tools：Read, Write, Grep, Bash, Task
- MCP tools：外部 server 提供

**誰決策**：Claude Model

**何時被使用**：
- Model 根據 skill 決策要用什麼 tool
- 發出 tool_use block
- Claude Code 執行

---

## Coordinator 的 Prompt

**文件位置**：`.claude/agents/coordinator.md`

**內容模板**：
```markdown
---
name: coordinator
tools: [Read, Grep, Task]
---

你是代碼審查 coordinator。

任務：對用戶提供的代碼進行全面審查。

可用的 subagents：
1. code-reviewer
   說明：審查代碼品質、風格、最佳實踐
   使用時機：所有代碼都需要

2. security-auditor
   說明：檢查安全漏洞、認證、授權、密碼
   使用時機：代碼涉及認證、密碼、資料庫時

3. performance-checker
   說明：檢查性能問題、優化機會
   使用時機：代碼有迴圈、資料庫查詢、API 呼叫時

決策規則：
- 如果代碼涉及認證或資料保護 → 一定要用 security-auditor
- 如果代碼有性能相關操作 → 要用 performance-checker
- 所有代碼都要用 code-reviewer

執行步驟：
1. 先讀取代碼
2. 評估需要哪些 subagents
3. 同時產生所需的 subagents
4. 等待所有 subagents 完成
5. 整合報告並報告給用戶
```

**重要**：
- Prompt 裡明確寫「什麼時候用什麼 subagent」
- Model 根據這些規則決策
- Subagent 的「功能」隱含在「使用時機」裡

---

## Command 配置方式

### Coordinator Command 定義

```markdown
# .claude/commands/comprehensive-review.md

---
description: Perform comprehensive code review
agent: coordinator
---

Perform a comprehensive code review of the provided file(s).

The coordinator will:
1. Read the code
2. Determine which subagents to use
3. Generate appropriate subagents (code-reviewer, security-auditor, etc.)
4. Integrate and report all findings
```

**重要字段**：
- `agent: coordinator` - 指定這個 command 使用 coordinator agent
- `description` - User 看到的說明

---

## 完整執行流程

### Cycle 1: Coordinator 初始化和決策

```
Coordinator Claude Code:
  1. 讀 coordinator agent definition
  2. 讀 coordinator prompt
  3. 初始化 messages
     messages = [user prompt]

Coordinator Claude Code 掃描 Skills:
  4. 根據當前 messages 推薦相關 skills
     推薦: [code-review-checklist, security-checklist]
  
  5. 把 skill bodies 加入 messages

Coordinator Claude Code 呼叫 Model:
  6. model.create(messages=messages)

Coordinator Claude Model 決策:
  7. 看著 messages (含 coordinator prompt + skill bodies)
  8. 讀懂 coordinator prompt 的「決策規則」
  9. 看著當前 context (user input, 代碼內容)
  10. 決策：「根據規則，我應該先讀代碼」
  11. 發出 tool_use: Read
```

### Cycle 2: 執行 Tool 並重新評估

```
Coordinator Claude Code:
  1. 執行 Read tool
  2. 得到代碼內容
  3. 加入 messages
  
  4. 重新掃描 skills
     新 context 包含：user input + 代碼內容
     推薦改變: [security-checklist, code-review-checklist]
  
  5. 把新 skill bodies 加入 messages
  6. 呼叫 Model

Coordinator Claude Model 決策:
  7. 看著 coordinator prompt: "涉及認證 → 用 security-auditor"
  8. 看著 skill body 的詳細步驟
  9. 看著代碼內容：有認證、有密碼比對、有資料庫操作
  10. 決策：「符合『涉及認證』條件，我應該產生 subagents」
  11. 決策：「產生 code-reviewer 和 security-auditor」
  12. 發出 Task tool_use (2 個)
```

### Cycle 3: Spawn Subagents (並行)

```
Coordinator Claude Code:
  1. 執行 Task tool_use (code-reviewer)
     讀 .claude/agents/code-reviewer.md
     啟動 code-reviewer agent
     
  2. 執行 Task tool_use (security-auditor)
     讀 .claude/agents/security-auditor.md
     啟動 security-auditor agent
  
  (BLOCKING - 等兩個都完成)

Code-reviewer Agent (獨立 loop):
  獨立運作
  完成 → 回傳 task result

Security-auditor Agent (獨立 loop):
  獨立運作
  完成 → 回傳 task result

Coordinator Claude Code:
  3. 收集 2 個 task results
  4. 加入 messages
```

### Cycle 4: 整合結果並決定下一步

```
Coordinator Claude Code:
  1. 重新掃描 skills
     新 context 包含：task results from both subagents
     推薦改變: [performance-checklist, ...]
  
  2. 把新 skill bodies 加入 messages
  3. 呼叫 Model

Coordinator Claude Model 決策:
  4. 看著兩個 subagent 的結果
  5. 看著新推薦的 skills
  6. 看著 coordinator prompt: "有迴圈 → 用 performance-checker"
  7. 評估結果是否還需要性能檢查
  8. 決策：
     a. 產生 performance-checker？
     b. 整合結果並報告？
     c. 其他？
  
  9. 發出 Task tool_use 或 stop_reason: "end_turn"
```

---

## 關鍵決策點

### Model 決策「要不要用 Subagent」

```
決策依據：
  1. Coordinator Prompt
     「什麼時候用什麼 subagent」
  
  2. Skill Bodies
     「這是什麼情況」的詳細指導
  
  3. 當前 Context
     「現在是什麼情況」

決策過程：
  看 coordinator prompt: "涉及認證 → 用 security-auditor"
    ↓
  看 security-checklist skill: "檢查項目包括認證相關..."
    ↓
  看代碼內容: "有認證相關代碼"
    ↓
  匹配規則: "涉及認證" ✓
    ↓
  決策: "我應該產生 security-auditor"
    ↓
  發出 Task tool_use
```

### Model 決策「要跑什麼 Tool」

```
決策依據：
  1. Skill Body 的步驟
  2. 當前 Context
  3. Skill 的 allowed-tools

決策過程：
  看 skill body: "第一步：檢查 input validation"
    ↓
  看代碼: "有用戶輸入"
    ↓
  決策: "我需要搜尋 input validation 相關代碼"
    ↓
  看 skill allowed-tools: [Read, Grep]
    ↓
  決策: "我應該用 Grep 搜尋"
    ↓
  發出 tool_use: Grep
```

---

## Agent.md vs Skill.md 對比

| 層級 | 位置 | 給誰看 | 何時看 | 內容 |
|------|------|--------|--------|------|
| **Agent.md** | `.claude/agents/xxx.md` | Subagent 自己 | Subagent 被 spawn 時 | 角色、工具、工作 |
| **Skill.md** | `.claude/skills/xxx/SKILL.md` | Coordinator Model | 每個 cycle 掃描 | 步驟、指導、檢查項目 |
| **Tool** | Built-in 或 MCP | Claude Code 執行 | Model 決策後 | 執行動作 |

---

## 執行流程圖

```
User Input
  ↓
Coordinator Claude Code 初始化
  ├─ 讀 coordinator prompt
  └─ 初始化 messages
  ↓
每個 Cycle:
  ├─ Coordinator Claude Code 掃描 skills
  ├─ 把 skill bodies 加入 messages
  ├─ 呼叫 Coordinator Model
  │  └─ Model 根據「prompt + skills + context」決策
  │     ├─ 決策用什麼 skill
  │     ├─ 決策要不要產生 subagent
  │     └─ 決策要跑什麼 tool
  ├─ Coordinator Claude Code 執行
  │  ├─ 如果 tool_use → 執行 tool
  │  ├─ 如果 Task → spawn subagent (可能並行多個)
  │  └─ 結果加入 messages
  └─ 回到掃描 skills（進入下一個 cycle）

直到 Model 發出 stop_reason: "end_turn"
  ↓
報告最終結果
```

---

## 簡單記

### 啟動 Coordinator
- **起點**：User（透過 command 或 message）
- **兩種方式**：
  1. 作為 Main Agent（command 指定）
  2. 作為 Subagent（main agent 決策 spawn）

### Agent.md
- **位置**：`.claude/agents/xxx.md`
- **內容**：「你的角色是什麼，你的工作是什麼」
- **給誰看**：被 spawn 出來的 agent
- **Model 看嗎**：❌ 不看

### Skill.md
- **位置**：`.claude/skills/xxx/SKILL.md`
- **內容**：「遇到這種情況，你應該這樣做」
- **給誰看**：Coordinator model
- **Model 看嗎**：✅ 一定要看 body

### Coordinator Prompt
- **位置**：`.claude/agents/coordinator.md`
- **內容**：「什麼時候用什麼 subagent」（決策規則）
- **給誰看**：Coordinator model
- **Model 看嗎**：✅ 一定要看

### Command（可選）
- **位置**：`.claude/commands/xxx.md`
- **內容**：User 看到的說明 + agent 指定
- **給誰看**：User 和 Claude Code
- **作用**：綁定特定 agent 到 command

### 決策流程
```
Model 看 Coordinator Prompt（規則）
  + 看 Skill Bodies（指導）
  + 看 Current Context（現況）
    ↓
  決策：要不要用 subagent
  決策：要跑什麼 tool
    ↓
  發出 Task tool_use 或 tool_use
    ↓
  Claude Code 執行
```

### 啟動流程
```
User 選擇方式
  ↓
方式 A: 打 Command
  /comprehensive-review auth.py
    ↓
  Coordinator 作為 Main Agent
  
方式 B: 打普通 Message
  "Review this code"
    ↓
  Main Agent 決策 spawn Coordinator
    ↓
  Coordinator 作為 Subagent
```

---

## 實際例子：Code Review Coordinator

### Coordinator Agent Definition
```markdown
---
name: coordinator
tools: [Read, Grep, Task]
---

你是代碼審查 coordinator。

可用的 subagents：
- code-reviewer: 審查代碼品質和風格
- security-auditor: 檢查安全漏洞
- performance-checker: 檢查性能問題

決策規則：
- 代碼涉及認證、密碼、資料庫 → 用 security-auditor
- 代碼有迴圈、查詢、API 呼叫 → 用 performance-checker
- 所有代碼 → 用 code-reviewer

步驟：
1. 讀代碼
2. 評估需要哪些 subagents
3. 同時產生所需 subagents
4. 整合結果
```

### Security Checklist Skill
```markdown
---
name: security-checklist
description: Security audit checklist
allowed-tools: [Read, Grep]
---

代碼安全檢查清單

## 檢查項目
1. Input Validation
   - SQL injection
   - XSS
   - Command injection

2. Authentication
   - 密碼哈希
   - 會話管理
   - Token 安全

3. Data Protection
   - 敏感資料加密
   - 安全存儲
```

### 執行流程
```
Cycle 1:
  讀代碼 → 發現有認證相關代碼

Cycle 2:
  根據規則：「涉及認證 → 用 security-auditor」
  根據 skill body：「應該檢查認證、密碼、會話」
  決策：產生 security-auditor + code-reviewer

Cycle 3:
  等 subagents 完成

Cycle 4:
  檢查結果：有迴圈、有資料庫查詢
  根據規則：「有迴圈 → 用 performance-checker」
  決策：產生 performance-checker

Cycle 5:
  整合所有結果
  報告給用戶
```

---

## 核心結論

1. **Agent.md** 定義「被產生的 agent 要做什麼」
   - Coordinator Model 不看

2. **Skill.md (Body)** 定義「現有 agent 應該怎麼做」
   - Coordinator Model 一定看
   - Model 根據 skill 決策

3. **Coordinator Prompt** 定義「什麼時候用什麼 subagent」
   - Model 根據 prompt 的規則決策
   - Subagent 功能隱含在「使用時機」裡

4. **Model 的決策順序**
   - 讀 prompt（決策規則）
   - 讀 skill body（詳細指導）
   - 看 context（現在的情況）
   - 匹配規則 → 決策動作

---

生成於：2025-05-08

