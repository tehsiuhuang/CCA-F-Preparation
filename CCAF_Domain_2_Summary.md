# CCAF Domain 2: Tool Design & MCP Integration

**佔比 18%**

整個 domain 的核心思想:**Tool 是 LLM 的接口。LLM 透過 description 認識它、透過 structured error 知道該怎麼反應、透過合理數量保持 selection 可靠。**

---

## Task Statement 2.1 — Design Effective Tool Interfaces with Clear Descriptions

### 核心觀念

**Tool description 是 LLM 選工具的主要訊號**。Description 寫不好,model 沒辦法在相似 tool 之間做出可靠選擇。

### 四個 Knowledge of 重點

1. **Tool description 是 LLM 選 tool 的 primary mechanism** — minimal description 導致 unreliable selection
2. **Description 該包含**:input formats、example queries、edge cases、boundary explanations
3. **Ambiguous / overlapping description 造成 misrouting**(經典例子:`analyze_content` vs `analyze_document`)
4. **System prompt 的 keyword 會干擾 tool selection** — keyword-sensitive instructions 可能 override well-written descriptions

### 四個 Skills in 重點

1. **寫清楚的 differentiation**:purpose、expected inputs、outputs、**when to use it vs similar alternatives**
2. **Renaming + 改 description 消除 overlap**(例:`analyze_content` → `extract_web_results`)
3. **Generic tool 拆成 purpose-specific tools**(例:`analyze_document` → `extract_data_points` + `summarize_content` + `verify_claim_against_source`)
4. **檢查 system prompt 是否干擾 tool selection**

### ★ Sample Question 2

題目:Agent 常常在 user 問 order 問題時 call `get_customer` 而不是 `lookup_order`。兩個 tool 描述都很短(「Retrieves customer information」/「Retrieves order details」)。最有效的**第一步**?

**正解 = (B) Expand 每個 tool 的 description**(input format、example query、edge case、boundary)

### 為什麼其他選項錯

- **(A) Few-shot examples** — 不解決根本問題,只是疊 token 補償
- **(C) Routing layer** — over-engineered,bypass LLM 的 natural language understanding
- **(D) 合併成單一 tool** — 算合理架構選擇但**過度作為「first step」**

### 考試判斷規則

| 題目線索 | 答案方向 |
|---|---|
| Tool selection 不可靠 / 選錯 tool | 修 **tool description** |
| Two tools 描述幾乎一樣 | 修 description + 可能 rename |
| 「First step」「low-effort」「proportionate」 | 選**最小改動**的選項 |
| Routing layer / pre-select tool by keyword | ❌ 通常是錯的 |
| Consolidate tools into one generic | ❌ 通常是錯的 |

### Tool 跟 source code 的關係(觀念釐清)

- Tool 沒有「自己的 md file」標準格式
- Tool 是 **code**(function),description 是程式碼裡的字串(docstring 或 decorator 參數)
- Description 跟 implementation **寫在一起**,透過 SDK 的 registration 機制綁定
- 修 description = 改程式碼裡那個字串參數,low-effort high-leverage

---

## Task Statement 2.2 — Structured Error Responses for MCP Tools

### 核心觀念

**Generic error 沒用,要 structured error 才能讓 agent 做出正確的 recovery 決策**。

### 四個 Knowledge of 重點

1. **MCP `isError` flag** — boolean 告訴 agent 成功/失敗
2. **四種 error category**:
   - **Transient**(timeout、service unavailable)→ retry
   - **Validation**(invalid input)→ 修正 input 再試
   - **Business**(policy violation)→ 不能 retry,解釋或 escalate
   - **Permission**(沒權限)→ 不能 retry,可能 escalate
3. **Generic error 阻止 agent 做正確決策**(「Operation failed」沒資訊量)
4. **Retryable vs non-retryable** — structured metadata 防止 wasted retry

### 四個 Skills in 重點

1. **回傳 structured error metadata**:
   - `errorCategory`(transient/validation/permission)
   - `isRetryable` boolean
   - human-readable description

2. **Business rule violation 要 customer-friendly explanation**:
   - `retriable: false`
   - 直接給 user 看的 `customer_message`
   - 給 agent reasoning 用的 `description`

3. **Local error recovery,只 propagate 解決不了的**:
   - Subagent 自己處理 transient failures
   - 解決不了才 propagate,附 partial results + what was attempted

4. **區分 access failure vs valid empty result**:
   - Access failure(timeout、認證失敗):`isError: true`,要 retry/escalate
   - Valid empty result(成功 query 但無 match):`isError: false`,告訴 user「沒找到」

### 對照範例

```python
# 糟糕版本
return {"isError": True, "content": "Failed"}

# 好版本
return {
  "isError": True,
  "errorCategory": "transient",
  "isRetryable": True,
  "description": "Database timeout after 5s"
}

# Empty result 是 SUCCESS,不是 error
return {
  "isError": False,
  "content": [...],
  "matches": []
}
```

### 考試判斷規則

| 題目線索 | 答案方向 |
|---|---|
| Agent 在錯誤後不知道該怎麼辦 | 加 **structured error metadata** |
| Agent 對所有錯誤都 retry | 加 `isRetryable` flag |
| Tool 把「查無資料」當 error | 區分 access failure vs **valid empty result** |
| Agent 跟 user 解釋錯誤亂講 | 加 **customer-friendly description** |
| Subagent failure 都炸到 coordinator | Local recovery,只 propagate 解決不了的 |

---

## Task Statement 2.3 — Tool Distribution & `tool_choice`

### 核心觀念

**Principle of least privilege**:每個 agent 只給最少必要的 tool,避免 misuse 跟 selection complexity。

### 四個 Knowledge of 重點

1. **Tool 太多降低 selection reliability** — 4-5 個合理,18 個會出問題
2. **Agent 拿到專業以外的 tool 容易 misuse**(例:synthesis agent 拿到 web search → 自己跑去搜)
3. **Scoped tool access**:default 限角色內,exception 是 high-frequency 跨角色需求
4. **`tool_choice` 三種設定**:
   - `"auto"` — 自由決定要不要 call、call 哪個
   - `"any"` — **必須** call 一個 tool(可選哪個)
   - `{"type": "tool", "name": "X"}` — 強制 call 指定 tool

### 五個 Skills in 重點

1. **限制 subagent 的 tool set** — 透過 `AgentDefinition.allowedTools`
2. **Generic tool 換 constrained alternative**(例:`fetch_url` → `load_document` 會 validate)
3. **High-frequency cross-role tool**(例:synthesis agent 給 scoped `verify_fact`,複雜的還是走 coordinator)
4. **`tool_choice: forced` 強制特定 tool 先 call**(例:強制 `extract_metadata` 在 enrichment 之前)
5. **`tool_choice: "any"` 強迫走 tool 而非 text** — 用於 extraction service 想要 structured output

### ★ Sample Question 9

題目:Synthesis agent 常需要 verify claim,目前每次都走 coordinator → web search → 回 synthesis,latency +40%。85% 是簡單 fact-check,15% 需要深入。

**正解 = (A) 給 synthesis agent scoped `verify_fact` tool 處理 common case,複雜的繼續走 coordinator**

### 為什麼其他選項錯

- **(B) Batching** — 創造 blocking dependency(synthesis 後續可能依賴前面的 verified fact)
- **(C) 給完整 web search** — over-provision,違反 separation of concerns
- **(D) Speculative caching** — 猜測 synthesis 要 verify 什麼,不可靠

### `tool_choice` 三種模式對照

| 模式 | API 寫法 | 行為 |
|---|---|---|
| Auto | `"auto"` | 可 call tool 也可回 text |
| Any | `"any"` | 必須 call tool,自選哪個 |
| Forced | `{"type":"tool","name":"X"}` | 必須 call X |
| (None) | `{"type":"none"}` | 禁止 call tool,強迫回 text |

### `tool_choice` 重要觀念

- **Per-API-call setting**,只影響這次 call
- 不能保證整個 workflow 順序(那要用 PreToolUse hook)
- 不支援指定 multiple tools — 想限子集要動態調 `tools` 列表

### `tool_choice: "any"` 為何等於強迫 structured output

不是直接保證,是**間接保證**:
- `"any"` 強迫走 tool 路徑(不能回 text)
- Tool 的 input 必須符合 JSON schema(API enforce)
- → 拿到的就是符合 schema 的 structured data

### 考試判斷規則

| 題目線索 | 答案方向 |
|---|---|
| Round trip 太多、latency 高 | Cross-role scoped tool |
| 85%/90% 都是簡單需求 | 給 limited tool 處理 common case |
| Tool 太多、selection 不可靠 | 限 tool set 到 4-5 |
| Subagent 在做不該做的事 | 限制 tool access |
| 想保證 model 一定走 tool | `tool_choice: "any"` |
| 想保證某個 tool 先 call | `tool_choice: {"type":"tool","name":"X"}` |
| 想保證整個 workflow 順序 | ❌ 不是 `tool_choice`,是 **PreToolUse hook** |

### `tool_choice` vs `allowedTools` 區別

| | `tool_choice` | `allowedTools` |
|---|---|---|
| 目的 | 控制這次該怎麼選 tool | 限制 agent 能存取哪些 tool |
| 生效範圍 | 單次 API call | 整個 agent 生命週期 |
| 設在哪 | API 參數 / SDK 程式碼 | `AgentDefinition` 或 SKILL.md |

---

## Task Statement 2.4 — MCP Server Integration

### 四個 Knowledge of 重點

1. **MCP server 兩種 scope**:
   - **`.mcp.json`**(專案根目錄)— project-level,進 git,團隊共用
   - **`~/.claude.json`**(home dir)— user-level,不進 git,個人/實驗用
   
2. **Environment variable expansion**(`${GITHUB_TOKEN}`)— 進 git 不洩漏 secret

3. **All tools discovered at connection time, simultaneously available** — 設了多少 server 就看到多少 tool

4. **MCP resources** — content catalog,減少 exploratory tool call(issue summaries、doc hierarchies、DB schemas)

### 五個 Skills in 重點

1. Project-scoped server in `.mcp.json` + env var expansion
2. Personal/experimental server in `~/.claude.json`
3. **加強 MCP tool description**,避免 agent 偏好 built-in tool(像 Grep)
4. **優先用 community MCP server**,reserved custom server 給 team-specific
5. **暴露 content catalog 為 MCP resource**(不要用 tool)

### MCP Transport 機制(觀念補充,CCAF 不深入考)

Claude Code 用三種 transport 連 MCP server,JSON-RPC 2.0 over the chosen transport:
- **stdio** — 啟動 local 程序,stdin/stdout 通訊
- **http** — 連 remote URL(現在標準)
- **sse** — 舊的 remote 協定(deprecated)

### Tool vs Resource 對照

| | MCP Tool | MCP Resource |
|---|---|---|
| 性質 | 主動 call 的 function(動作) | 被動讀的 catalog(目錄) |
| 用途 | 執行操作 | 給 agent 看「有什麼可用」 |
| 範例 | `lookup_order(id)`、`process_refund` | Issue summary 列表、doc hierarchy |
| 解決 | 「我要做某件事」 | 「我先看看有什麼選項」 |
| Metadata | Connection 時 push description | Connection 時 push list |
| 內容拿到的方式 | 必須 call 才有結果 | URI read 一次就有 |

### Tool vs Resource 例子(列出 issues)

**用 Tool 設計(差)**:
```
1. Agent 想列 issues
2. Agent reason 該用什麼 query / filter / limit
3. 試 list_issues(status="open", limit=20) 
4. 結果不對 → 改參數再試
5. 反覆 5 次 → exploratory tool calls 浪費
```

**用 Resource 設計(好)**:
```
1. Connection 時 agent 已經看到 issues://summary 這個 resource
2. Read 一次拿完整 catalog(by_status / by_label / recent_issues)
3. 直接知道全貌,不需試錯
```

### 考試判斷規則

| 題目線索 | 答案方向 |
|---|---|
| Team 要共用同一組 MCP tool | `.mcp.json`(project) |
| 我個人在試新 MCP server | `~/.claude.json`(user) |
| `.mcp.json` 怎麼放 token 不洩漏 | Env var expansion `${TOKEN}` |
| Agent 偏好 Grep 而不是強的 MCP tool | 加強 MCP tool description |
| 想整合 Jira | Community MCP server,不要自己寫 |
| 想讓 agent 看到有哪些 issue 不用一直探 | MCP **resource**(content catalog) |

---

## Task Statement 2.5 — Built-in Tools

### 六個 Built-in Tool 角色

| Tool | 用途 | 例子 |
|---|---|---|
| **Grep** | 搜檔案內容(content search) | 找 function callers、error messages、imports |
| **Glob** | 檔名 / 路徑 pattern matching | `**/*.test.tsx`、`src/api/*.ts` |
| **Read** | 讀整個檔案 | 載入完整內容到 context |
| **Write** | 創建新檔 / 完整覆蓋 | 寫新檔或全部重寫 |
| **Edit** | 局部修改(unique text matching) | `old_str` → `new_str`,要求 `old_str` 在檔中唯一 |
| **Bash** | 執行 shell 指令 | build、test、git |

### ★ Read+Write 是 Edit 的 fallback

當 Edit 失敗(`old_str` 不 unique)→ fallback 到 **Read 整檔 → context 裡修改 → Write 寫回**。

### 五個 Skills in 重點

1. **Grep 用於 code search** — 找 callers、error messages
2. **Glob 用於檔名 pattern**
3. **Edit 失敗 → Read + Write fallback**
4. **★ Codebase 探索要 incremental**:Grep 找 entry point → Read 跟 imports → 一層層 trace,**不要 Read all upfront**
5. **★ 跨 wrapper module 追 function**:先找所有 exported names → 對每個 name Grep

### Grep vs Glob 區別(常混淆)

| | Grep | Glob |
|---|---|---|
| 找的對象 | **檔案內容** | **檔名 / 路徑** |
| 輸入 | 文字 pattern(regex) | 檔名 pattern(glob 語法) |
| 輸出 | 哪個檔的哪一行 match | 哪些檔名 match |
| 比喻 | 「全文搜索」 | 「按名字找檔」 |

### Codebase 探索:錯誤 vs 正確流程

**錯誤**:
```
1. Read 所有 *.py → context 爆掉、attention 稀釋
2. 一次理解所有東西
```

**正確**:
```
1. Glob "**/auth*.py"          → 找相關檔
2. Grep "def login"             → 找 entry point
3. Read auth/login.py           → 看 entry function
4. Grep "from auth.session import" → 順著 import
5. Read auth/session.py         → 看下一層
6. 重複 trace,逐步建立理解
```

### 考試對照表

| 情境 | 用什麼 |
|---|---|
| 找 function callers | Grep |
| 找 error message 出處 | Grep |
| 找所有 import 某 module 的地方 | Grep |
| 找所有 `*.test.*` 檔 | Glob |
| 找特定 naming pattern 的檔 | Glob |
| 讀完整檔案 | Read |
| 創新檔 / 完整覆蓋 | Write |
| 局部修改(unique text) | Edit |
| Edit 失敗(non-unique) | Read + Write |
| 跑指令 / build / test | Bash |
| Codebase 第一次探索 | Grep entry,**不是 Read all** |
| Function 被 wrapper 包過 | 先找 exported names → Grep each |

---

# Domain 2 反模式速查表

| 反模式 | 為什麼錯 | 屬於 |
|---|---|---|
| Tool description 寫太短 | Selection 不可靠 | 2.1 |
| 相似 tool 沒區分 boundary | Misrouting | 2.1 |
| Generic error response(「Operation failed」) | Agent 無法做 recovery 決策 | 2.2 |
| 把「查無資料」當 access failure | 兩種完全不同 | 2.2 |
| 給 agent 18 個 tool | Selection complexity 高 | 2.3 |
| 給 synthesis agent 完整 web search | Misuse、破壞 hub-and-spoke | 2.3 |
| 用 `tool_choice` 保證整個 workflow 順序 | 它只影響單次 call,要用 hook | 2.3 |
| `.mcp.json` 直接寫 secret | 進 git 會洩漏,要 env var | 2.4 |
| MCP tool description 比 built-in 短 | Agent 偏好 built-in | 2.4 |
| 重複造輪子寫 Jira MCP server | Community 已有 | 2.4 |
| 用 tool 暴露 catalog 而不是 resource | Exploratory call 浪費 | 2.4 |
| 一上來 Read 所有檔案 | Context 爆 + attention 稀釋 | 2.5 |
| Edit 失敗硬用 Edit 不 fallback | 改不了 | 2.5 |

---

# 高頻必背 Sample Questions

| Sample Q | 考點 | 正解核心 |
|---|---|---|
| **Q2** | 2.1 — Tool description 是 selection 主訊號 | Expand tool descriptions |
| **Q9** | 2.3 — High-frequency cross-role tool | Scoped `verify_fact` for synthesis |

---

# 關鍵 Term 速記

| Term | 意思 |
|---|---|
| `isError` | MCP tool response 的 error flag |
| `errorCategory` | transient / validation / business / permission |
| `isRetryable` | 該不該 retry 的 boolean |
| `tool_choice` | 控制這次 API call 怎麼選 tool |
| `"auto"` / `"any"` / `{"type":"tool","name":"X"}` | tool_choice 三種模式 |
| `allowedTools` | Agent 能用的 tool 清單(包括 `"Task"` for coordinator) |
| `.mcp.json` | Project-scoped MCP server 設定 |
| `~/.claude.json` | User-scoped MCP server 設定 |
| MCP Tool | 主動 call 的 function |
| MCP Resource | 被動讀的 content catalog |
| Grep | 搜檔案內容 |
| Glob | 搜檔名 pattern |
| Edit | unique text matching 修改 |
| Read + Write | Edit fallback |

---

# 五個 Task Statement 串起來看

```
2.1  Tool description 是 selection 主訊號(寫清楚、區別於相似 tool)
 ↓
2.2  Tool 失敗回 structured error(category / isRetryable / friendly description)
 ↓
2.3  Tool 太多影響 selection(scoped access,跨角色用 limited cross-role tool)
       `tool_choice` 三種模式(auto / any / forced)— per-call 控制
 ↓
2.4  MCP server 設定(.mcp.json shared / ~/.claude.json personal)
       Env var 保護 secret
       Resource 暴露 catalog 減少 exploratory call
 ↓
2.5  Built-in tools 選擇(Grep 內容 / Glob 檔名 / Edit unique → Read+Write fallback)
       Codebase 要 incremental 探索
```

---

# 終極 Mental Model

> **Tool 是 LLM 的接口。LLM 透過 description 認識它(2.1)、透過 structured error 知道何時該 retry(2.2)、透過合理數量保持 selection 可靠(2.3)。MCP 是 Anthropic 的外掛系統(2.4),搭配 built-in tool 形成完整工具箱(2.5)。每個層級都要寫清楚、用對、限縮恰當的 scope。**

# 關鍵觀念區辨

## Main agent 不能用 `allowedTools` 限制 tool 太多嗎?

**技術上可以,但 CCAF 想考的是另一個觀念**:

- 情境 A:agent 任務範圍狹窄 → 直接 `allowedTools` 限縮 ✅
- 情境 B:**main agent 是 general-purpose / coordinator** → 它必須看到 broad scope 才能 routing,**不能限縮**
  - 解法是 **spawn 限縮 tool 的 subagent** 來解決,不是限縮 main agent

CCAF 把 main agent 想成 dispatcher / tech lead,看不到全局就沒法派工。Worker(subagent)那層才用 `allowedTools` 限縮。
