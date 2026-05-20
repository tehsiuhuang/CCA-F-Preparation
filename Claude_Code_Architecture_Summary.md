# Claude Code 完整架構最終總結

## 核心分工

### Claude Code（執行層 + 管理層）
- 讀取所有 MD 文件
- 掃描 skills 並篩選推薦（用相似度，不用 LLM）
- 初始化和管理 messages
- 呼叫 Claude Model
- 執行 tools（Built-in 或 MCP）
- 自動重新篩選（每個 cycle）

### Claude Model（決策層）
- 看著 messages context（包含 skill bodies）
- 決策用哪個 skill
- 根據 skill body 的指導決策
- 決策跑什麼 tool
- 發出 tool_use block

---

## 五大 Components 層級

| Component | 說明 | 檔案位置 |
|-----------|------|---------|
| Command | 用戶入口 (/xxx) | `.claude/commands/*.md` |
| Agent | 執行單位（有 loop） | `.claude/agents/*.md` |
| Skill | 指引手冊（步驟） | `.claude/skills/*/SKILL.md` |
| Tool | 執行動作（讀檔等） | Built-in + MCP |
| Rules | 系統規則（自動載入） | `.claude/CLAUDE.md` + `.claude/rules/` |

---

## SKILL.md 的雙重角色

### Frontmatter (YAML)
- `name`, `description`, `allowed-tools`, `context`
- **用途**：Claude Code 用來篩選推薦

### Body (Markdown)
- 步驟、指引、檢查清單
- **用途**：Claude Model 用來決策做什麼

---

## 每個 Cycle 的完整 Flow

**Step 1: User Input (Command)**

**Step 2: Claude Code 初始化**
- 讀文件 → 讀 rules → 初始 messages

**Step 3: Claude Code 掃描 Skills (自動)**
- 根據當前 messages 內容
- 提取關鍵字 → 計算相似度 → 篩選推薦
- 把 skill bodies 加入 messages

**Step 4: Claude Code 呼叫 Model**

**Step 5: Claude Model 決策**
- 看著 messages (含 skills + rules)
- 決策: 用哪個 skill → 跑什麼 tool
- 發出 tool_use block

**Step 6: Claude Code 執行 Tool**
- Built-in → 直接執行
- MCP → 透過 protocol 給 server

**Step 7: 結果加入 Messages**
- 進入下一個 Cycle (回到 Step 3)

**重複直到** Model 發出 `stop_reason: "end_turn"`

---

## 關鍵認識

### Skills 的推薦
- Claude Code 自動篩選（每個 cycle）
- 根據「當前 messages 內容」計算相似度
- 不用 LLM，純算法（快速 + 免費）
- Messages 改變 → Skills 推薦可能改變

### Skills 的使用
- Claude Model 看著推薦的 skill bodies
- Model 自己決策用哪個
- Model 根據 skill body 的指導決策跑什麼 tool

### Messages 的變化
- **初始**：`[user prompt]`
- **Cycle 1**：`+ skill bodies + tool result`
- **Cycle 2**：`+ 新的 skill bodies + 新的 tool result`
- **Cycle N**：`+ 越來越多累積的資訊`

---

## 簡單記憶

### Claude Code = 推薦引擎 + 執行引擎
- 自動掃描 skills
- 根據相似度推薦

### Claude Model = 決策引擎
- 看著推薦的 skills
- 決策用哪個 + 跑什麼 tool

### 每個 cycle：
```
Messages → 重新掃描 → 新推薦 → 新決策 → 執行
```

---

## 核心結論

**整個系統就是：Claude Code 負責「推薦和執行」，Claude Model 負責「決策」。每個 cycle 自動重新推薦，直到完成。**

---

## 決策分佈總表

### Claude Code 決策（無需 LLM）
- ✅ 推薦哪些 skills (相似度)
- ✅ 執行哪個 tool
- ✅ 何時重新篩選 (自動)
- ⏱ 無需 LLM，純算法

### Claude Model 決策（需要 LLM）
- ✅ 用哪個 skill
- ✅ 根據 skill 做什麼
- ✅ 跑什麼 tool
- ⏱ 用 LLM，消耗 token

---

## 完整 Messages 變化示例

### Cycle 1 的 Messages：
```
[
  {"role": "user", "content": "Review auth.py for security"},
  {"role": "user", "content": "code-review-checklist body: 1. 檢查..."}
]
```

### Cycle 2 的 Messages：
```
[
  {"role": "user", "content": "Review auth.py for security"},
  {"role": "user", "content": "code-review-checklist body: ..."},
  {"role": "assistant", "content": model_response_1},
  {"role": "user", "content": "Read tool result: def login..."},
  {"role": "user", "content": "encryption-guide body: ..."}  ← 新的 skill
]
```

### Cycle 3 的 Messages：
```
[
  前面所有內容,
  {"role": "assistant", "content": model_response_2},
  {"role": "user", "content": "Grep tool result: Found 5 instances..."},
  {"role": "user", "content": "password-hashing body: ..."}  ← 又新的 skill
]
```

---

## 實際例子：Security Audit

### 初始推薦（Cycle 1）
```
Context: "Review auth.py for security"
推薦: [code-review-checklist, security-audit]
```

### 發現密碼問題後（Cycle 2）
```
Context: "Review auth.py + 密碼沒有哈希的發現 + code-review body"
新關鍵字: ["review", "security", "password", "encryption"]
推薦改變: [encryption-guide, password-hashing, security-audit]
```

### 需要修復時（Cycle 3）
```
Context: "所有前面的內容 + grep 結果"
推薦改變: [password-hashing, bcrypt-implementation, testing-guide]
```

---

生成於：2025-05-08

