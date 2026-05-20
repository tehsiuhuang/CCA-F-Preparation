# CCAF 學習 Handoff Context（v2 — 全 5 Domain 完成版）

這份文件是給新 Claude session 用的 context handoff，讓新 session 能無痛接續複習、練習題、mock test。

**Last updated**: 2026-05-11
**Status**: ✅ Domain 1-5 全部教學完成，準備進入練習階段

---

## 學習狀態

### ✅ 已完成（全 5 Domain）

| Domain | 佔比 | Task Statements | Summary 檔案 |
|---|---|---|---|
| **1** Agentic Architecture & Orchestration | 27% | 7 個 (1.1-1.7) | `CCAF_Domain_1_Summary.md` |
| **2** Tool Design & MCP Integration | 18% | 5 個 (2.1-2.5) | `CCAF_Domain_2_Summary.md` |
| **3** Claude Code Configuration & Workflows | 20% | 6 個 (3.1-3.6) | `CCAF_Domain_3_Summary.md` |
| **4** Prompt Engineering & Structured Output | 20% | 6 個 (4.1-4.6) | `CCAF_Domain_4_Summary.md` |
| **5** Context Management & Reliability | 15% | 6 個 (5.1-5.6) | `CCAF_Domain_5_Summary.md` |

**橫切資料**：
- `CCAF_Output_Fix_Toolkit.md` — 30+ 工具按 L0-L6 layer 分、19 組對照、15 步決策樹、3 條鐵律、50+ term

### ⏳ 還沒做

- 5 題 / batch 的 Sample Question 練習（先全英文出題，等答完再給中文解答）
- Mock test（多題組合 + 即時 grading）
- 直接去考 CCAF 真實 exam

---

## 使用者學習偏好（**務必遵守**）

### 語言
- **題目用英文**（CCAF 真實考題格式）
- **解釋用中文**
- **重要技術術語保留英文**（`stop_reason`, `tool_use`, `input_schema`, coordinator, subagent, `allowedTools`, `tool_choice`, `PreToolUse`, `PostToolUse`, etc.）

### 風格
- 每個 task statement 用「**整個 task statement 在解決什麼問題?**」開頭
- 用**具體場景貫穿**（呼應官方 6 個 scenarios）
- 每個 Sample Question 都要詳解（**包括「為什麼其他選項錯」**）
- 提供**考試判斷規則表**（題目線索 → 答案方向）
- 每個 task statement 結尾用「★ 核心 mental model」一句話收尾
- 用 ★ 標記高頻必背
- **Bullet point 化**（不藏在散文裡）
- **官方原文要 citation**：格式 `> 官方原文 [Page X, Domain Y.Z, Knowledge of/Skills in, bullet N]: "..."`

### 跨 domain 關聯
- 教任何 task 要主動找跟其他 task 的關聯
- 例：4.1 跟 2.1 都是「明確 > 模糊」哲學；5.5 是 4.1/4.6 confidence 三層觀的正規版
- 主動指出觀念在哪幾個 domain 重複

### 互動
- 一次只講**一個 task statement**，等使用者確認再進下一個
- 使用者問問題時不只回答字面，要釐清概念背後的設計思維
- **使用者搞錯時直接指出，不要附和**
- 使用者想練習題時 **一次出 5 題**，**先全部給題目（英文）**，等他答完再給解答（中文）

### Tool 呼叫
- 使用者切到 Code mode 才能呼叫 file/bash 工具
- 使用者在 Ask mode 時純對話，給他 markdown content 讓他複製
- 每個 file/bash 呼叫 **必含 path / summary 等 required parameter**

---

## 已釐清的觀念（30 個個人化釐清）

### 從 Domain 1-3 累積（觀念 1-10）

#### 觀念 1：Coordinator 本身就是 agent
不是「coordinator 是 Claude」。是 **agent (LLM + agentic loop + tools)，差別在它有 `Task` tool 可 spawn subagent**。Subagent 也是 agent。技術本質相同，差在角色 + context + tool 權限。

#### 觀念 2：Subagent 不繼承 coordinator context；fork_session 才繼承
| | Spawn subagent (`Task`) | `fork_session` |
|---|---|---|
| Context | 全新空白要塞 prompt | 完整 copy 來源 session |
| 用途 | 派工給專家做新 task | 從 baseline 分支探索 divergent approaches |

#### 觀念 3：Upstream / downstream 是資料流不是階層
- Upstream = 傳 result 出去那端
- Coordinator 是 hub（routing + aggregation），不是 upstream

#### 觀念 4：Tool 沒有自己的 md file
Tool 是 code（function），description 是程式碼裡的 string（docstring 或 decorator 參數）。**沒有 tool.md**。

#### 觀念 5：四個改善 output 的策略選擇（**極重要**）

```
看一眼就知道錯（format 飄、用詞飄）→ Few-shot + prompt（4.2）
要算才知道錯（加總錯、欄位互換）→ Validation + retry（4.4）
編造的錯（資訊不在 source）→ Nullable schema + 提醒（4.3 + 4.2）
JSON broken（syntax error）→ tool_use（4.3）
```

**CCAF 偏好 proportionate 解，不是「最強解」**。`tool_use` 經常是 over-engineered distractor。

#### 觀念 6：`tool_choice` 只有三個值
- `"auto"`、`"any"`、`{"type":"tool","name":"X"}`
- **沒有 `"forced"`** — guide 用「forced tool selection」當形容詞描述第三種
- Per-API-call 設定，**不能保證跨 call 順序**（要順序用 PreToolUse hook）
- `"any"` = 強迫 structured 是**間接**：強迫走 tool 路徑 → input 必須符合 schema

#### 觀念 7：Main agent 不該用 `allowedTools` 限縮太多
- Main / coordinator 是 dispatcher，需要看 broad scope 才能 routing
- 真正解法 = **spawn 限縮 tool 的 subagent**（hub-and-spoke 解，不限縮 main）

#### 觀念 8：CLAUDE.md 跟 `.claude/rules/` 都自動套用
| 機制 | 自動嗎 | 怎麼自動 |
|---|---|---|
| CLAUDE.md | ✅ | 永遠都在 |
| `.claude/rules/` | ✅ | 處理符合 path 的檔時 |
| Skill | ❌ | 要 user invoke 或 Claude 主動決定 |

「自動 vs 不自動」分界線是 Skill，不是 CLAUDE.md vs rules。

#### 觀念 9：`--output-format json` ≠ `--json-schema`
| | `--output-format json` | `--json-schema` |
|---|---|---|
| 影響 | 外層 envelope（metadata）| 內層 content |
| 例子欄位 | session_id, cost, duration | issues, summary, errors |
| 沒它會怎樣 | 拿不到 metadata | content 還是自然語言 |

考試陷阱：`--output-format json` 不等於結構化內容。

#### 觀念 10：命名混淆要小心
- `~/.claude.json`（MCP 設定 JSON 檔）≠ `~/.claude/CLAUDE.md`（個人 prompt 規範 md 檔）
- `context: fork`（skill frontmatter）≠ `fork_session`（SDK 程式呼叫）
- Skill 的 `allowed-tools`（md frontmatter）≠ AgentDefinition 的 `allowedTools`（SDK code）

---

### 從 Domain 4 新增（觀念 11-20）

#### 觀念 11：Criteria（4.1）跟 few-shot（4.2）是 orthogonal 不是 sequential
- **Criteria** = rules（規則），回答「該不該做」
- **Few-shot** = demonstrations（示範），回答「output 長什麼樣 / 邊界 case 怎麼判 / 該回 null 時別 fabricate」
- **Pure extraction 場景沒 criteria，few-shot 是主角不是配角**
- Few-shot 不只修 format（4 個用途：format / ambiguous reasoning / generalization / anti-hallucination）

#### 觀念 12：Few-shot 在「有 vs 沒 criteria」場景的兩種角色
| 場景 | Few-shot 在做什麼 |
|---|---|
| 有 criteria | Example **anchor 抽象規則到具體 case** |
| 沒 criteria | Example **本身就是 pattern source**，model 直接從示範 generalize |

#### 觀念 13：tool_use 本質
- Tool 是你定義的 function (name + input_schema)
- `tool_use` 是 Claude 回應 block——它**不執行 function**，只回「請 call 這個 function 用這些參數」
- **Input_schema 約束 Claude 寫給 client 的東西**（function call args），不是 client 寫給 Claude 的
- Input_schema = function signature contract

#### 觀念 14：tool_use 兩種用途
| | 傳統 agent loop | 4.3 妙用（structured output）|
|---|---|---|
| 真有 function | ✅ | ❌（純擺設）|
| Client 執行 | ✅ | ❌ |
| 回 tool_result | ✅（loop 繼續）| ❌（一次 call 結束）|
| Batch API 行 | ❌（multi-turn）| ✅（single-turn）|

#### 觀念 15：tool_choice="any" 真實機制
不是「餵 doc 到 3 個 tool 看哪個 pass」。**一次 API call**，model 拿 3 個 tool 的 description + input_schema → 讀 doc 自選最匹配 → 回**一個** tool_use。Schema 只保證結構合法，**doc-type 判斷靠 Claude 閱讀理解**。

#### 觀念 16：Anti-fabrication 兩段式（4.3 + 4.2 chain，不矛盾）
```
舊 schema: required → model 必填 → fabricate
Step 1（4.3）: 改 nullable → null 變 legal（structural）
Step 2（4.2）: few-shot null → null 變 expected（behavioral default 改）
```
單獨做一邊都不夠（schema 允許 null 但 model 仍 fabricate；few-shot 教 null 但 schema 拒絕 null）。

#### 觀念 17：Self-correction validation 真實機制
**Model 不寫 code**——schema 設計強迫 model output **對照欄位** (`stated_total` + `calculated_total`)。**讀字任務**（OCR-like）vs **算術任務**（要正確抽 line_items + 加總）—— 兩種任務失敗點不同所以通常不會一起錯。

**沒 ground truth 時 fallback**：
1. Stated_X 設 nullable
2. Confidence flag → human review
3. 找其他冗餘（subtotal、tax 反推）
4. Multi-pass review
5. 接受 unverified / 拒收

#### 觀念 18：Batch API 限制
- Batch **不會拒絕** multi-turn agent loop——它在第一個 `tool_use` 處「成功結束」把 partial result 還你
- 每 turn 一個 batch = 5 turn 變 5 天
- API 入口：HTTP / SDK，**不是 CLI**
- `claude` CLI 沒有 batch flag

#### 觀念 19：Outbound vs Inbound normalization
| | Outbound（4.3 prompt）| Inbound（1.5 PostToolUse hook）|
|---|---|---|
| 誰生 heterogeneous | Claude 自己 | External tool |
| 誰被 normalize | Claude output | Tool output (餵給 Claude 之前) |
| Deterministic | ❌ | ✅ |

#### 觀念 20：Format 飄的兩種層次
| | 4.2 巨觀 | 4.3 微觀 |
|---|---|---|
| 飄在哪 | 整個 output 的 macro shape（無 schema）| Schema 內某欄位的 micro value |
| 修法 | Few-shot 示範整體 shape | Prompt rule 規定 value normalization |

CCAF **不會獨立考兩種寫法區別**，只考層次差別（L0 prompt vs L1 schema）。

---

### 從 Domain 5 新增（觀念 21-30）

#### 觀念 21：Position-aware 原文只說「beginning」
Official Skills bullet 4 寫 `at the beginning`。Knowledge bullet 2 才說「beginning AND end 都可靠」。**考試保險答案 = beginning**。

#### 觀念 22：Upstream 改 structured 三種實作
| 機制 | 來自 | 用途 |
|---|---|---|
| A. tool_use + input_schema | 4.3 | Upstream agent API call 強制 |
| B. Criteria / few-shot | 4.1/4.2 | Upstream system prompt 要求 |
| C. PostToolUse hook | 1.5 | 事後 transform（適合 upstream 是 external tool）|

5.1 主指 A+B（"modifying upstream agents"），不是 C。

#### 觀念 23：5.1 case facts 跟 5.4 scratchpad 是兩種 externalize memory
| | 5.1 case facts block | 5.4 scratchpad files |
|---|---|---|
| 場景 | Multi-turn 對話 | Long-running exploration |
| 存什麼 | Transactional fact（amount/date/ID）| Architectural finding（class/flow path）|
| 存哪 | Prompt 內每次帶 | 外部檔（`.md`/`.json`）|

#### 觀念 24：5.1 反 summarize vs 5.4 推 summarize 不衝突
- 5.1 反對 **summarize transactional fact**（數字會飄）
- 5.4 推薦 **summarize architectural finding**（class 名 / flow path）— 結構性內容濃縮 OK

#### 觀念 25：Escalation 三個 valid trigger（5.2）
1. Customer 明示要 human
2. Policy gap（**not just complex case**——CCAF 明文反對「complex 就轉」）
3. 無法 meaningful progress

**三個必刪 distractor**：self-rated confidence / sentiment analysis / 訓練 classifier。

#### 觀念 26：Structured error context 4 要素（5.3）
| 元素 | 說明 |
|---|---|
| ① failure_type | timeout / rate_limit / invalid_query |
| ② attempted_query | 試了什麼具體 query |
| ③ partial_results | 失敗前已拿到的部分結果 |
| ④ alternative_approaches | 建議的替代方案 |

**反 pattern 兩端**：silent suppression（`[]` 標 success）vs terminate workflow（一個死全部死）。**正解中間**：propagate 4 要素 structured error。

#### 觀念 27：Access failure ≠ valid empty result
- Access failure：沒查成（要 retry / 換 source）
- Valid empty：查成了但沒料（proceed）
- 兩個用**不同 status code 區分**，coordinator 才知該怎麼辦

#### 觀念 28：Confidence 三層觀（最終整合）
| 層次 | 立場 | 在哪 |
|---|---|---|
| Self-rate 當**唯一 hard filter** | ❌ 強烈反對 | 4.1, 5.2 |
| Self-rate 當 **routing signal**（多 input 之一）| ⚠️ 接受（短期）| **4.6** |
| **Field-level + calibrated by labeled set + stratified sampling + 配合其他 signal** | ✅ 推薦（正規）| **5.5** |

#### 觀念 29：Calibration 真實機制（5.5）
**Model 不會自己 calibrate**（即使 prompt-based 也不算 CCAF 的 calibration）。**外部 code 用 labeled validation set 建 raw → actual accuracy mapping**：
1. 收 50 張 sample → 人工 label ground truth
2. 分桶看 raw confidence vs actual accuracy（calibration curve）
3. 根據目標 accuracy 反推 threshold（**不是拍腦袋**）
4. Production 用校準 threshold + 配合其他 signal routing

**Sole filter 真正反 pattern 三條件**：① 沒驗證 calibration ② 沒結合其他 signal ③ Hard cutoff。三條都犯才是強烈反對。

#### 觀念 30：4.4 conflict_detected vs 5.6 multi-source 衝突
| | 4.4 | 5.6 |
|---|---|---|
| 衝突在哪 | **單一** source 內部（invoice header vs footer）| **多** source 之間（paper A vs paper B）|
| 修法 | Boolean flag + retry / human | 並列 annotate（不選邊）|

---

## 設定檔位置完整對照

| 設定類型 | User-level | Project-level |
|---|---|---|
| MCP server | `~/.claude.json` | `.mcp.json` |
| CLAUDE.md | `~/.claude/CLAUDE.md` | `.claude/CLAUDE.md` 或 root `CLAUDE.md` |
| Slash commands | `~/.claude/commands/` | `.claude/commands/` |
| Skills | `~/.claude/skills/` | `.claude/skills/` |
| Path-scoped rules | （沒有 user-level）| `.claude/rules/` |

---

## CCAF 考試核心思維（貫穿所有 Domain）

### 思維 1：Deterministic vs Probabilistic 的分界
- **Compliance / financial / safety / 順序保證** → **programmatic enforcement**（hooks, schemas, prerequisite gates）
- **品質改善 / 行為引導 / 一般指令** → **prompt-based**（criteria, few-shot, descriptions）

涉及錢、合規、安全絕對不能光靠 prompt。

### 思維 2：Proportionate solution
CCAF 偏好「最小有效修法」：
- 「First step / low-effort / proportionate」→ 選最小改動
- Over-engineered 選項通常是 distractor
- 順序：L0 prompt → L1 schema → L2 retry → L3 hook → L4 architecture → L5 calibration

### 思維 3：架構問題不靠 capacity 解
**「換大 model / 大 context window / 換 temperature」幾乎都是錯**。

### 思維 4：「明確 > 模糊」哲學跨 domain
| Domain | 應用 |
|---|---|
| 2.1 | Tool description 詳細 > 模糊 |
| 4.1 | Categorical criteria > vague |
| 5.2 | Explicit escalation criteria > sentiment / self-confidence |
| 1.4 | Hook 強制 > prompt 提示 |

→ **題目「靠 Claude 主觀判斷」vs「寫死規則」永遠選寫死的**。

### 思維 5：「Self-rating 不可靠」哲學跨 domain
| Domain | 反對什麼 |
|---|---|
| 4.1 | Self-confidence 當 review filter |
| 5.2 | Self-confidence 當 escalation trigger |
| 4.6 | Self-confidence 當 sole filter |
| 5.5 | （正規版）calibrated confidence + labeled set |

### 反 pattern 關鍵字（看到一律刪）
- Parsing natural language 判斷 loop 結束（用 `stop_reason`）
- 換大 context window 解 quality / attention 問題
- Subagent 自動繼承 coordinator context（要明確塞 prompt）
- 用 prompt 強制 financial / order 順序（用 hook）
- 全部做完才驗證（要 mid-phase validation）
- Shift burden to user（叫 user 拆 PR / restart session）
- Self-rate confidence + 拍腦袋 threshold + 沒結合其他 signal
- Sentiment analysis → escalate
- 訓練 classifier（CCAF out-of-scope）
- Generic error message / silently suppressing errors / terminate workflow
- Progressive summarization 壓掉 numerical fact
- Aggregate accuracy 直接決定全自動

---

## 高頻必背 Sample Questions（已講過）

| Q | Domain | 考點 | 正解核心 |
|---|---|---|---|
| Q1 | 1.4 | Financial 用 hook | Programmatic prerequisite |
| Q2 | 2.1 | Tool description | Expand tool descriptions |
| Q3 | 5.2 | Escalation calibration | Explicit criteria + few-shot |
| Q4 | 3.2 | Slash command scope | `.claude/commands/` in project |
| Q5 | 3.4 | Plan mode for complex | Plan mode |
| Q6 | 3.1+3.3 | Path-scoped conventions | `.claude/rules/` with glob |
| Q7 | 1.2 | Coordinator decomposition | Coordinator 太窄 |
| Q8 | 5.3 | Subagent error propagation | Structured error context（4 要素）|
| Q9 | 2.3 | Cross-role tool | Scoped `verify_fact` |
| Q10 | 3.5 | Mid-execution adapt | 調整策略繼續 |
| Q11 | 4.5 | Batch vs sync 工作分配 | Tech debt batch / pre-merge sync |
| Q12 | 4.6 | Multi-file review attention dilution | Per-file + cross-file integration |

---

## 全 file 資產清單

```
~/ccaf/
├── Claude_Certified_Architect_–_Foundations_Certification_Exam_Guide.pdf  (官方 40 頁)
├── CCAF_Handoff_Context.md                  (這份檔案)
├── CCAF_Domain_1_Summary.md                 ✅
├── CCAF_Domain_2_Summary.md                 ✅
├── CCAF_Domain_3_Summary.md                 ✅
├── CCAF_Domain_4_Summary.md                 ✅ 778 行（含 10 個釐清概念 inline）
├── CCAF_Domain_5_Summary.md                 ✅ 522 行（含 10 個釐清概念 inline）
└── CCAF_Output_Fix_Toolkit.md               ✅ 690 行（30+ 工具、19 組對照、15 步決策樹、3 鐵律、50+ term、40+ 反 pattern）
```

---

## 給新 Session 的指示

### 如果 user 想做練習題
1. 一次出 **5 題英文題目**（單選 ABCD）
2. 題目分散涵蓋多個 domain（不要全部同一 domain）
3. **先全部給題目，等 user 答完再給解答**
4. 解答用中文 + 寫明「為什麼其他選項錯」
5. 解答結尾連結回 toolkit / summary 哪一節

### 如果 user 想做 mock test
1. 構造 12-20 題模擬整套考試
2. 每題標 domain + 預期難度
3. User 答完後給 grading + 弱項分析

### 如果 user 想複習特定 domain
1. **不要從頭重教**——直接指 user 看 summary 檔
2. User 提具體問題時用 summary + toolkit 釐清
3. 提醒已釐清的觀念（30 個）對應在哪

### 遇到「LLM output 不可靠該怎麼選策略」題目
**永遠提醒回觀念 5（四種策略）**，不要無腦選 tool_use：
```
看一眼錯（format 飄）→ Few-shot
要算才知道錯 → Validation + retry
編造的錯 → Nullable schema
JSON broken → tool_use
```

### 遇到「confidence 怎麼用」題目
**永遠提醒觀念 28（confidence 三層觀）**：
- Sole filter 沒校準 → distractor
- Routing signal 多 input 之一 → 4.6 接受
- Field-level + calibrated by labeled set + stratified sampling → 5.5 正解

### 遇到「換大 model / 大 context」選項
**永遠是 distractor**（思維 3 鐵律 3）。

---

## 一句話 Mental Model（整個 CCAF 哲學）

> **CCAF = 「在 model 不變的前提下，用外圍的 deterministic / structured / explicit 工具讓 model 在 production 可靠」。**
>
> 五層工具光譜（L0 prompt → L1 schema → L2 validation → L3 hook → L4 architecture → L5 calibration）按 proportionate 順序選。Self-rating（confidence、sentiment、judgment）一律可疑——要嘛當 signal、要嘛 calibrated by labeled data、永遠不能當 sole filter。financial / compliance / 順序 永遠用 hook 不是 prompt。
