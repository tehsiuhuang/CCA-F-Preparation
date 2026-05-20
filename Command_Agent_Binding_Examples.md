# Command 怎麼指定 Agent - 完整範例

## Command 文件結構

```
.claude/commands/
├── simple-review.md           (普通 command - 不指定 agent)
├── comprehensive-review.md    (指定 coordinator agent)
├── security-audit.md          (指定特定 agent)
└── quick-fix.md              (不指定 - 使用 main agent)
```

---

## 範例 1: 不指定 Agent（預設行為）

### 檔案：.claude/commands/simple-review.md

```markdown
---
description: Simple code review
---

Review the provided code for basic quality issues.

Focus on:
- Code style
- Readability
- Basic best practices
```

**當 user 打這個 command 時**：
```
User: /simple-review auth.py
  ↓
Claude Code 讀取 simple-review.md
  ↓
看 frontmatter，沒有 agent 指定
  ↓
使用預設 main agent
  ↓
Main agent 的 Claude Model 決策
    「我應該自己做，還是產生 subagent？」
```

---

## 範例 2: 指定 Coordinator Agent

### 檔案：.claude/commands/comprehensive-review.md

```markdown
---
description: Comprehensive code review with multiple agents
agent: coordinator
---

Perform a comprehensive code review of the provided file(s).

The coordinator will analyze the code from multiple perspectives:
1. Code quality and style
2. Security vulnerabilities
3. Performance issues
4. Best practices

This command uses the coordinator agent to orchestrate multiple subagents.
```

**當 user 打這個 command 時**：
```
User: /comprehensive-review auth.py
  ↓
Claude Code 讀取 comprehensive-review.md
  ↓
看 frontmatter，發現 agent: coordinator
  ↓
讀取 .claude/agents/coordinator.md
  ↓
啟動 Coordinator Agent Loop
  ↓
Coordinator 決策產生：
  - code-reviewer subagent
  - security-auditor subagent
  - performance-checker subagent
```

**關鍵**：
- `agent: coordinator` 這一行指定使用 coordinator agent
- 不用 coordinator 自己決策是否執行
- 直接啟動 coordinator 作為 main agent

---

## 範例 3: 指定其他特定 Agent

### 檔案：.claude/commands/security-audit.md

```markdown
---
description: Deep security audit
agent: security-auditor
---

Perform a deep security audit of the code.

This focuses specifically on:
- Authentication and authorization
- Data protection and encryption
- Input validation
- SQL injection and XSS prevention
- API security
- Session management

Uses the security-auditor agent which specializes in security analysis.
```

**當 user 打這個 command 時**：
```
User: /security-audit auth.py
  ↓
Claude Code 讀取 security-audit.md
  ↓
看 frontmatter，發現 agent: security-auditor
  ↓
讀取 .claude/agents/security-auditor.md
  ↓
啟動 Security-auditor Agent Loop
  ↓
Security-auditor 用自己的專門知識做安全審計
```

---

## 範例 4: Coordinator 帶不同參數

### 檔案：.claude/commands/quick-review.md

```markdown
---
description: Quick coordinator review (fast mode)
agent: coordinator
coordinator_mode: fast
---

Quick comprehensive review - coordinator will use only essential subagents.

Will check:
- Critical security issues
- Major code quality problems

Skips:
- Performance analysis
- Minor style issues
```

**當 user 打這個 command 時**：
```
User: /quick-review auth.py
  ↓
Claude Code 讀取 quick-review.md
  ↓
看 frontmatter：
  agent: coordinator
  coordinator_mode: fast
  ↓
啟動 Coordinator Agent
  message 包含：
    "This is a QUICK review mode"
    "Skip performance analysis"
    ↓
Coordinator 的 Claude Model 根據 prompt 決策
  「這是 fast 模式，我應該只用必要的 subagents」
```

---

## Coordinator 的不同配置 Command

你可以為 coordinator 創建多個 commands，根據不同場景：

### 快速版本

```markdown
---
description: Quick comprehensive review
agent: coordinator
mode: quick
---

Perform a quick but thorough review.
Use only essential subagents (code-reviewer, security-auditor).
```

### 完整版本

```markdown
---
description: Full comprehensive review
agent: coordinator
mode: full
---

Perform a complete and detailed review.
Use all available subagents (code-reviewer, security-auditor, performance-checker).
Provide detailed analysis on each aspect.
```

### 安全優先版本

```markdown
---
description: Security-focused comprehensive review
agent: coordinator
focus: security
---

Perform a comprehensive review with emphasis on security.
Give priority to security-auditor findings.
Highlight all potential security issues.
```

---

## Agent Definition 對應

### Coordinator Agent 定義

```markdown
# .claude/agents/coordinator.md

---
name: coordinator
tools: [Read, Grep, Task]
model: sonnet
---

你是代碼審查 coordinator。

# 可用的 Subagents

- code-reviewer: 審查代碼品質、風格、最佳實踐
- security-auditor: 檢查安全漏洞
- performance-checker: 檢查性能問題

# 決策規則

根據 command 的 mode 和 focus：

如果 mode: quick
  只用 code-reviewer 和 security-auditor
  
如果 mode: full
  用所有 subagents
  
如果 focus: security
  優先 security-auditor 的結果

# 執行步驟

1. 根據 mode 評估需要哪些 subagents
2. 產生所需的 subagents
3. 整合結果並報告
```

---

## 實際的 Frontmatter 字段

### 標準字段

```markdown
---
description: [必須] User 看到的說明
agent: [可選] 指定使用哪個 agent
---
```

### 擴展字段（可自訂）

```markdown
---
description: Comprehensive code review
agent: coordinator
mode: full                    # 可自訂字段
focus: security              # 可自訂字段
priority_level: high         # 可自訂字段
include_performance: true    # 可自訂字段
---
```

**這些擴展字段會進入 messages，coordinator 可以看到：**

```
Frontmatter 中的所有自訂字段
  ↓
會被 Claude Code 提取
  ↓
加入 coordinator 的 messages
  ↓
coordinator 的 Claude Model 可以看到
  ↓
根據這些字段調整行為
```

---

## 三個 Command 的完整例子

### 命令 1：簡單審查（不指定 agent）

```markdown
# .claude/commands/review.md

---
description: Basic code review
---

Review the code for quality and style issues.
```

執行結果：
- Main agent 啟動
- Main agent 決策是否需要複雜協調
- 如果簡單，自己做；如果複雜，spawn coordinator

---

### 命令 2：全面審查（指定 coordinator）

```markdown
# .claude/commands/full-review.md

---
description: Comprehensive multi-agent review
agent: coordinator
mode: full
---

Perform comprehensive review using all available subagents.
```

執行結果：
- Coordinator 直接作為 main agent 啟動
- 根據 mode: full 決策用所有 subagents
- Code-reviewer + security-auditor + performance-checker

---

### 命令 3：安全優先（指定 coordinator）

```markdown
# .claude/commands/security-review.md

---
description: Security-focused comprehensive review
agent: coordinator
mode: full
focus: security
---

Comprehensive review with security as primary focus.
Detailed analysis of all security aspects.
```

執行結果：
- Coordinator 作為 main agent 啟動
- 看到 focus: security
- 優先執行 security-auditor
- 強調安全相關發現

---

## Command 怎麼指定 Agent 的原理

### 步驟 1: 讀取 Command

```
user: /comprehensive-review auth.py
  ↓
Claude Code 讀取 .claude/commands/comprehensive-review.md
```

### 步驟 2: 解析 Frontmatter

```
Frontmatter YAML:
  description: Comprehensive code review with multiple agents
  agent: coordinator
  mode: full
  ↓
解析成：
  {
    "description": "Comprehensive code review with multiple agents",
    "agent": "coordinator",
    "mode": "full"
  }
```

### 步驟 3: 檢查 agent 欄位

```
如果 agent 欄位存在：
  agent_name = coordinator
  
  讀取 .claude/agents/coordinator.md
  啟動 coordinator agent
  
如果 agent 欄位不存在：
  使用 default main agent
```

### 步驟 4: 傳入 Messages

```
Command 的 frontmatter 和 body
  ↓
都被加入 messages
  ↓
Agent 的 Claude Model 可以看到
  ↓
例如：mode: full 被 model 看到
  ↓
Model 根據 mode 決策
```

---

## Coordinator 怎麼看到 Command 的信息

```
.claude/commands/comprehensive-review.md:

---
description: Comprehensive code review
agent: coordinator
mode: full
focus: security
---

Body content here...

↓

Claude Code 讀取此文件
↓
初始化 coordinator messages：

[
  {
    "role": "user",
    "content": "user input: /comprehensive-review auth.py"
  },
  {
    "role": "system",
    "content": """
    Command: comprehensive-review
    Description: Comprehensive code review
    Mode: full
    Focus: security
    
    Body:
    Comprehensive code review...
    """
  }
]

↓

Coordinator Claude Model 看到：
- description
- mode: full
- focus: security
- Body content

↓

Model 根據這些信息決策
```

---

## 簡單記

### 怎麼指定 Agent

在 command 的 frontmatter 裡加一行：
```markdown
agent: coordinator
```

### 怎麼傳參數給 Agent

在 frontmatter 裡加自訂字段：
```markdown
agent: coordinator
mode: full
focus: security
```

### Agent 怎麼看到這些

- Claude Code 解析 frontmatter
- 把所有字段加入 messages
- Agent 的 Model 可以看到
- Model 根據這些信息決策

### 三種情況

1. **不指定 agent**
   - 使用預設 main agent
   - Main agent 自己決策

2. **指定 agent: coordinator**
   - Coordinator 作為 main agent
   - 啟動 coordinator loop

3. **指定 agent: security-auditor**
   - Security-auditor 作為 main agent
   - 啟動 security-auditor 專門審查

---

生成於：2025-05-08

