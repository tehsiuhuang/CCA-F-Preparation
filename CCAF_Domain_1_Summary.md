# CCAF Domain 1: Agentic Architecture & Orchestration

**佔比 27%(最大 domain)**

整個 domain 的核心思想:**deterministic 的事用 programmatic 方式控制,probabilistic 的事交給 model——絕對不混用。**

---

## Task Statement 1.1 — Agentic Loop 的正確控制流程

### 核心 loop

每次 iteration 做四件事:
1. 送 request 給 Claude
2. 看回應的 `stop_reason`
3. 如果是 `"tool_use"` → 執行 tool → 把 tool result append 回 conversation history → 繼續 loop
4. 如果是 `"end_turn"` → 結束 loop,把最終結果給 user

### 關鍵機制

- **Tool result 必須 append 回 conversation history**,model 才能基於新資訊推理下一步
- **Model-driven decision-making vs pre-configured decision tree**:agentic 系統是前者(LLM 根據 context 自己 reason 要 call 哪個 tool),不是後者(寫死 if-else)

### 三個必考的 anti-patterns(看到選項出現一律刪)

1. **Parsing natural language signals 判斷 loop 結束**(看 model 有沒有講「done」)— 不可靠
2. **Arbitrary iteration cap 當主要停止機制** — 那只能當 safety net
3. **看 assistant 是否有 text content 當 completion indicator** — 有 text 不代表結束

**正確答案永遠是 inspect `stop_reason`。**

### 注意

Claude 一次回應可以同時包含 text content 和 tool_use blocks。所以「有沒有 text」不是判斷依據,看 `stop_reason` 才是 deterministic 的訊號。

---

## Task Statement 1.2 — Coordinator-Subagent Orchestration

### 核心架構:Hub-and-spoke

Coordinator 是中心,所有 subagent 之間的 communication、error handling、information routing 都透過它。Subagent 之間**不直接講話**。

### 四個必背事實

1. **Subagent context 是 isolated 的** — 不會自動繼承 coordinator 的 conversation history
2. **Coordinator 的四個職責**:task decomposition、delegation、result aggregation、deciding which subagents to invoke
3. **不要每次都跑完整 pipeline** — 應該根據 query 動態 select subagent
4. **Overly narrow task decomposition 是經典 bug** — 導致 broad topic coverage 不完整

### Iterative refinement loop 設計

Coordinator 評估 synthesis output 找 gaps → 帶著 targeted query 重新 delegate → 再 invoke synthesis,反覆直到 coverage 夠。

**所有 subagent communication 要 route through coordinator** 來保 observability、consistent error handling、controlled information flow。

### ★ Sample Question 7

題目:研究「impact of AI on creative industries」,subagent 都正常完成,但結果只蓋 visual arts(漏掉 music、writing、film)。Log 顯示 coordinator 拆成 digital art / graphic design / photography。

**正解 = coordinator decomposition 太窄(B)**

### 考試判斷規則

題目給你看 log 顯示「subagent 都正常但有 coverage gap」→ 答案永遠是 coordinator decomposition 太窄,不是下游 agent 的問題。

### 觀念釐清

**Coordinator 是 agent**,本身就是一個 LLM instance + agentic loop + tools。它做的所有判斷(task decomposition、gap evaluation、coverage 判斷)都是 LLM reasoning,不是程式 dispatch。

Subagent 也是 agent(也是 Claude instance)。它們在技術本質上一樣,差別在角色、context、tool 權限。

---

## Task Statement 1.3 — Subagent Spawning & Context Passing

### 四個機制要點

1. **`Task` tool 是 spawn subagent 的機制**;coordinator 的 `allowedTools` 必須包含 `"Task"`,不然不能 spawn
2. **Subagent context 必須 explicitly provided in the prompt** — 不會自動繼承 parent context,也不跨 invocation 共享 memory
3. **`AgentDefinition` 配置**:description(coordinator 用來決定要叫誰)、system prompt、tool restrictions
4. **`fork_session`** 用於 exploring divergent approaches from a shared analysis baseline

### 四個 skill-level 重點

#### 1. 完整 findings 直接放在 subagent 的 prompt 裡

例如 spawn synthesis subagent 時,要把 web_search 跟 document_analysis 的完整輸出寫進 prompt,不能只寫「請整合上面的結果」(它看不到「上面」)。

#### 2. 用 structured data formats 分離 content 和 metadata

傳資料時連 source 一起傳:
```json
{
  "claim": "...",
  "evidence": "...",
  "source_url": "...",
  "publication_date": "..."
}
```

這樣下游(特別是 synthesis)才能保住 attribution。

#### 3. ★ Parallel subagent spawning(高頻考點)

**正確做法**:在**同一個 coordinator response** 裡 emit 多個 `Task` tool calls
```
Turn 1:
  → Task(web_search, "music")
  → Task(web_search, "writing")
  → Task(web_search, "film")
  (三個 in same response → parallel execution)
```

**錯誤做法**:每 turn 發一個 Task call,等回來再發下一個(變 sequential)

考試題目問「怎麼平行 spawn 減少 latency」→ 答「same response 裡 emit 多個 Task calls」

#### 4. Coordinator prompt 寫 goals + criteria,不是 step-by-step

**好**:"Research goal: ... Quality criteria: cover at least 5 sub-domains, all claims sourced"

**壞**:"Step 1: call web_search. Step 2: call document_analysis. ..."

寫死 procedural instruction 讓 subagent 失去 adaptability。

### `fork_session` vs spawn subagent 的區別

| | Spawn subagent (`Task`) | `fork_session` |
|---|---|---|
| Context | 全新空白,要塞 prompt | 完整 copy 來源 session |
| 用途 | 派工給專家 | 從 baseline 分支探索 |
| 比喻 | 老闆派工給員工 | Git branch |

**`fork_session` 會繼承 context**(這是它的核心功能,「shared baseline」就是這意思)。

---

## Task Statement 1.4 — Multi-step Workflow Enforcement(★ 最高頻考點)

### 核心觀念

**Programmatic enforcement(hooks、prerequisite gates)是 deterministic;prompt-based guidance 是 probabilistic。當業務規則需要 100% 保證時,只能用前者。**

關鍵句:**prompt instructions alone have a non-zero failure rate**

### 三個 Knowledge of 重點

1. **Programmatic enforcement vs prompt-based guidance** — 機制 vs 請求
2. **Deterministic compliance 在 financial / identity verification 場景必須** — non-zero failure 不能接受
3. **Structured handoff protocols** — mid-process escalation 要附 customer details、root cause、recommended action

### 三個 Skills in 重點

1. **Programmatic prerequisites** 擋住下游 tool calls(例:擋 `process_refund` 直到 `get_customer` 完成)
2. **Multi-concern decomposition** — 多 concern 拆獨立 item、parallel investigation with shared context、synthesize unified resolution
3. **Structured handoff summaries** — 因為 human agents lack access to conversation transcript,要把 context 濃縮成 structured 形式

### Mid-process escalation

「處理到一半才升級給人類」的情境。Agent 累積了 context,但人類接手是空白的,所以 structured summary 必須包含:customer ID、root cause、refund amount、recommended action、investigation so far。

### ★ Sample Question 1

題目:12% case agent 跳過 `get_customer` 直接 call `lookup_order`,造成 incorrect refunds。

**正解 = (A) Programmatic prerequisite 擋住 `lookup_order` 跟 `process_refund` 直到 `get_customer` 回傳 verified ID**

(B)(C)是 prompt-based — probabilistic,在 financial 場景不接受
(D)解錯問題(tool availability 不是 tool ordering)

### ★ 考試判斷規則(必背)

題目出現以下任一關鍵字 → **選 hooks / programmatic enforcement**:

| 關鍵字 | 為什麼 |
|---|---|
| Financial | 錯了會虧錢 |
| Policy | 違反政策有合規後果 |
| Compliance | 需要 audit |
| Critical business logic | 核心邏輯不能錯 |
| Guarantee / 100% / always / must | 字面就在說 deterministic |
| Identity verification | 安全相關 |

反過來,題目線索是「improving accuracy」、「reducing inconsistency」、「better tool selection」這種**品質改善** → 選 prompt-based(few-shot、enhanced description)。

---

## Task Statement 1.5 — Agent SDK Hooks

### 兩種 hook 對照

| | `PostToolUse` | `PreToolUse`(PDF 用 "tool call interception") |
|---|---|---|
| **時機** | Tool 執行**後**,進 model 之前 | Tool 執行**前**,model 發出 call 時 |
| **方向** | 處理進來的 data(tool → model) | 處理出去的 action(model → tool) |
| **典型用途** | Data normalization、格式統一 | 擋 policy violation、redirect、強制順序 |
| **比喻** | 進口檢疫 | 出口管制 |

### `PostToolUse` 經典用例

不同 MCP tool 回傳異質格式(Unix timestamp / ISO 8601 / numeric code / string status)→ 用 `PostToolUse` hook normalize 成統一格式才給 model 看。

### `PreToolUse` 經典用例

擋掉 policy-violating action(例:refund > $500)然後 redirect 到 alternative workflow(例:human escalation)。

不只擋,還要**告訴 model 該怎麼做**(回有用的 error message,讓 model 走 escalation 而不是傻傻重試)。

### `PreToolUse` 的 permissionDecision

可以設定三個值:
- `"allow"` — 放行
- `"deny"` — 擋下
- `"ask"` — 問人類(human-in-the-loop)

也可以設 `updatedInput` 修改 tool 的 input 參數再放行。

### 核心判斷

**Business rule require guaranteed compliance → 用 hook,不要 prompt**

### 考試判斷規則

| 題目線索 | 選哪種 |
|---|---|
| 不同 tool 格式不一,model 處理混亂 | `PostToolUse` |
| Unix timestamp / ISO 8601 / numeric 並存 | `PostToolUse` |
| 擋掉超過 threshold 的金額操作 | `PreToolUse` |
| Policy violation 必須阻止 | `PreToolUse` |
| 保證某個 tool 順序 | `PreToolUse`(prerequisite gate) |

### 1.4 vs 1.5 的關係

1.4 講「該用什麼方法」(觀念層 → 答 hook)
1.5 講「用哪種 hook」(機制層 → 答 PostToolUse / PreToolUse)

兩者經常一起出題。1.4 的 prerequisite gate 實作上就是 PreToolUse hook。

---

## Task Statement 1.6 — Task Decomposition Strategies

### 兩種策略

| 策略 | 特性 | 適用 |
|---|---|---|
| **Prompt chaining (fixed sequential)** | 步驟寫死按順序跑 | 預測性高的 multi-aspect work |
| **Dynamic adaptive decomposition** | 根據中間發現產生新 subtask | 開放式調查、走向不確定 |

### Prompt chaining 經典用例

Code review 拆成 per-file pass + cross-file integration pass。每步任務固定可預測。

### Adaptive decomposition 經典用例

「為 legacy codebase 加全面測試」這種開放式任務,三步流程:
1. **Map structure** — 摸清 codebase
2. **Identify high-impact areas** — 找加測試最有效益的地方
3. **Create prioritized plan that adapts** — 邊做邊調整

### 大型 code review 的標準拆法

關鍵詞:**attention dilution**(注意力被稀釋)

當一次看 10+ 檔案,model 注意力分散,某些檔案分析深、某些淺,甚至自相矛盾。

正確拆法:
1. **Per-file pass**:每個 file 獨立分析 local issue
2. **Cross-file integration pass**:獨立 pass 看跨檔案 data flow

### ★ Sample Question 12

題目:14 個檔案的 PR,single-pass review 結果不一致、漏 bug、自相矛盾。

**正解 = (A) 拆成 focused passes:per-file local pass + 獨立 cross-file integration pass**

### 為什麼其他選項都錯

- **(B) 要求 developer 拆 PR** → shift burden to user,沒解決系統問題
- **(C) 換更大 context window** → ★ 高頻陷阱:**larger context ≠ better attention**
- **(D) 三次 review 取交集** → 會抑制 intermittent bug 的偵測

### ★ 重要陷阱

**「換更大 model / 更大 context window」這種選項在整個 Domain 1 幾乎都是錯的**。考試哲學是「問題出在架構,不在 capacity」。

### 考試判斷規則

| 題目線索 | 答案 |
|---|---|
| Single pass 結果不一致 / 矛盾 / 漏 bug | 拆成 focused passes |
| 大 PR 處理品質差 | 拆,不是換大 model、不是要人類拆 |
| Open-ended task 不知道怎麼拆 | Adaptive decomposition |
| 每個面向都要看,工作可預測 | Prompt chaining |

### 記憶法

- **Prompt chaining = 火車軌道**(固定路線,事先決定停哪幾站)
- **Dynamic decomposition = 偵探辦案**(順著線索走,下一步看上一步發現)

---

## Task Statement 1.7 — Session State, Resumption, Forking

### 三個機制

| 機制 | Context | 適用 |
|---|---|---|
| **`--resume <name>`** | 完整繼承之前 session | 之前 context 還有效,要無縫接續 |
| **`fork_session`** | 從 baseline 完整繼承再分支 | 從同一分析點探索 divergent approaches |
| **新 session + summary** | 空白起點,注入 structured summary | 舊 tool results 已 stale |

### Resume 的注意事項

如果 resume 後檔案改過了,要主動**告訴 agent 哪些檔案變了**做 targeted re-analysis,不必整個重 explore。

### 何時新開比較好

當之前的 tool results 已大幅 stale,硬 resume 會拖一堆 stale data 誤導 agent。新開 session + structured summary 反而可靠。

### Decision Tree

```
Q1: 之前的 tool results / file 分析還有效嗎?
├─ 大致有效 → 繼續 Q2
└─ 已大幅 stale → 新 session + structured summary

Q2: 是要單純繼續同一條路,還是要分支探索?
├─ 單純繼續 → --resume
└─ 要平行比較多種方案 → fork_session

Q3: 用 --resume 後,有檔案被改過嗎?
├─ 沒有 → 直接繼續
└─ 有 → 主動告訴 agent 哪些檔案變了
```

### Git 類比

- **`--resume`** ≈ `git checkout` 同一個 branch 繼續 commit
- **`fork_session`** ≈ `git branch new-branch` 從當前 commit 開新分支
- **新 session + summary** ≈ `git init` 新 repo 但 cherry-pick 重要 commits

### 容易混淆:`fork_session` ≠ spawn subagent

| | `fork_session` | Spawn subagent |
|---|---|---|
| Context | 完整繼承歷史 | 空白 |
| 用途 | 探索 divergent approaches | 派專家做新 task |

題目線索:
- 「explore divergent approaches from shared baseline」→ `fork_session`
- 「派一個專家做特定 task」→ subagent

### Resume vs 新開的核心:staleness

不是看「時間久不久」,是看「tool results 還準不準」。

---

# Domain 1 反模式速查表(必背)

| 反模式 | 為什麼錯 | 屬於 |
|---|---|---|
| Parsing natural language 判斷 loop 結束 | 不可靠,要看 `stop_reason` | 1.1 |
| Iteration cap 當主要停止機制 | 那是 safety net | 1.1 |
| 看 text content 是否存在當 completion indicator | 有 text 不代表結束 | 1.1 |
| 用 prompt 強制 financial 順序 | Probabilistic,要 hook | 1.4 |
| 假設 subagent 自動繼承 coordinator context | Context 是 isolated | 1.3 |
| Coordinator 永遠跑完整 pipeline | 要動態 select | 1.2 |
| 大 PR 用 single-pass review | Attention dilution,要拆 | 1.6 |
| 換更大 context window 解決品質問題 | Larger context ≠ better attention | 1.6 |
| 把 fork_session 當 spawn subagent 用 | 兩個機制不同 | 1.3 / 1.7 |
| Resume stale session 不告知變動 | 用 stale data 誤導 agent | 1.7 |
| Shift burden to developer(要 user 拆 PR) | 系統該自己解決 | 1.6 |
| LLM self-reported confidence 當決策依據 | 校準很差,錯題反而更自信 | 1.4 / 1.5 |
| Sentiment-based escalation | 情緒 ≠ 複雜度 | (Domain 5 也會) |

---

# 高頻必背 Sample Questions

| Sample Q | 考點 | 正解核心 |
|---|---|---|
| **Q1** | 1.4 — financial 用 hook | Programmatic prerequisite |
| **Q7** | 1.2 — coordinator decomposition 太窄 | Coordinator 是 root cause |
| **Q12** | 1.6 — attention dilution | Per-file + cross-file integration pass |

---

# Domain 1 七個 task statement 串起來看

```
1.1  Agentic loop 怎麼跑(stop_reason 控制)
 ↓
1.2  多 agent 怎麼編排(coordinator-subagent hub-and-spoke)
 ↓
1.3  Subagent 怎麼 spawn、context 怎麼傳(Task tool, explicit prompt)
 ↓
1.4  Workflow 順序怎麼保證(programmatic > prompt for compliance)
 ↓
1.5  保證機制具體怎麼做(PreToolUse / PostToolUse hooks)
 ↓
1.6  複雜 task 怎麼拆(prompt chaining vs adaptive decomposition)
 ↓
1.7  Session 狀態怎麼管(resume / fork / fresh)
```

---

# 終極 Mental Model

> **「LLM 自己 reason 的 model-driven 決策(`stop_reason`、coordinator 動態選擇 subagent、adaptive decomposition)是 agent 的核心價值。但涉及 compliance、財務、安全、順序保證的 deterministic 需求,絕不能交給 LLM——用 hook、用 programmatic enforcement。」**

兩個層次要分清楚:該交給 LLM 判斷的就放手,不該交的就強制。考試題目大多在測你能不能分清楚這個界線。

---

# 重要 Term 速記表

| Term | 意思 |
|---|---|
| `stop_reason` | API 回應的訊號,`"tool_use"` / `"end_turn"` |
| `Task` tool | Spawn subagent 的機制 |
| `allowedTools` | Coordinator 配置,要包含 `"Task"` 才能 spawn subagent |
| `AgentDefinition` | 定義 subagent 型別(description / system prompt / tool restrictions) |
| `fork_session` | 從 baseline session 分支 |
| `--resume <name>` | 用名字繼續特定 session |
| `PreToolUse` hook | Tool 執行前攔截 |
| `PostToolUse` hook | Tool 執行後攔截 |
| `permissionDecision` | PreToolUse 的決策值:`allow` / `deny` / `ask` |
| Hub-and-spoke | Coordinator 為中心的架構 |
| Attention dilution | 一次塞太多檔案,attention 品質下降 |
| Mid-process escalation | 處理到一半升級給人類 |
| Structured handoff | 給人類接手用的濃縮摘要 |
