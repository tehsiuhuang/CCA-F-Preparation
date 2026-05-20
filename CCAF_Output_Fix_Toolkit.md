# CCAF 修 Model Output / Agent 行為的完整工具光譜

整個 CCAF exam 散落在 5 個 domain 的「修 model output / 修 agent 行為」工具，按**便宜→貴、probabilistic→deterministic** 整理。

考試最容易卡的就是「該用哪個工具修？」這份是針對**選擇困難**設計的速查表。

---

## 核心 framing：兩個軸

```
                    Probabilistic ←────────────→ Deterministic
                    (靠 LLM 推理)                  (寫死規則)
                          ↑
                    便宜（改 prompt）
                          │
                    中等（改 schema）
                          │
                    貴（加 validation/retry）
                          │
                    最貴（加 hook / 改架構）
                          ↓
```

**CCAF 偏好原則**：**proportionate solution**——便宜的能解就別用貴的，但 **deterministic 的場景（financial / compliance / 順序保證）絕對不能省**。

---

## 全部工具一覽（按便宜→貴排）

| Layer | 工具 | Domain | 解什麼問題 | Deterministic? |
|---|---|---|---|---|
| **L0 Prompt 文字** | Explicit criteria | 4.1 | Decision boundary 不清楚、FP 高 | ❌ Probabilistic |
| | Few-shot examples | 4.2 | Format 飄、ambiguous reasoning、normalize、null behavior | ❌ Probabilistic |
| | Tool descriptions | 2.1 | Tool selection 錯、tool 之間混淆 | ❌ Probabilistic |
| | CLAUDE.md / `.claude/rules/` | 3.1, 3.3 | 沒套用 project conventions | ❌ Probabilistic |
| **L1 Schema 結構** | tool_use + JSON schema | 4.3 | JSON syntax 錯、guarantee structured output | ✅ Syntax-deterministic |
| | tool_choice (`any` / forced) | 2.3, 4.3 | Model 該 call tool 時回 text、特定 tool 必須先跑 | ✅ Per-call deterministic |
| | Nullable schema | 4.3 | Required field 強迫 fabricate | ✅ Allows null |
| | Restrict `allowedTools` | 2.3 | Agent 用了不該用的 tool（cross-role misuse）| ✅ 該 tool 直接看不到 |
| **L2 Validation 後驗** | Validate + retry with error feedback | 4.4 | Semantic error（加總錯、欄位互換、format mismatch）| ❌ Probabilistic（但有 retry）|
| | Self-correction validation flow | 4.4 | calculated_total vs stated_total 不一致 | ❌ Probabilistic |
| **L3 Hook 攔截** | PreToolUse hook | 1.5 | Tool call 該被擋（refund > $500、缺 prerequisite）| ✅ Fully deterministic |
| | PostToolUse hook | 1.5 | Tool result 要 normalize / 轉換 | ✅ Fully deterministic |
| | Programmatic prerequisite gate | 1.4 | 順序保證（get_customer 必須先於 process_refund）| ✅ Fully deterministic |
| **L4 架構改造** | Multi-pass review | 4.6 | Single-pass attention dilution、self-review 偏差 | ✅ Architectural |
| | Multi-instance（second independent Claude）| 4.6 | Self-review 看不到自己的問題 | ✅ Architectural |
| | Subagent delegation | 1.2, 1.3, 5.4 | Context overflow、separation of concerns | ✅ Architectural |
| | Coordinator-subagent pattern | 1.2 | Multi-agent 協調 | ✅ Architectural |
| **L5 後驗校準** | Field-level confidence + calibrated threshold | 5.5 | High-confidence extraction 還是有誤 | ✅ With labeled set |
| | Stratified random sampling | 5.5 | Aggregate metric 掩蓋特定 segment 差表現 | ✅ Sampling guarantee |
| **L6 流程控制** | Plan mode | 3.4 | Complex / architectural decision 先想再做 | ❌ Process |
| | Iterative refinement / mid-phase validation | 3.5 | Costly rework | ❌ Process |
| | Batch API | 4.5 | Cost / latency tolerant | ✅ Cost-deterministic |
| | Escalation pattern | 5.2 | 超出 agent 能力範圍要 human | ❌ Probabilistic |

---

## 6 組最容易混的對照

### ① Hook vs Validation+retry（最常考）

| | Hook（L3）| Validation + retry（L2）|
|---|---|---|
| **時機** | Tool call **發生前** 攔截 | LLM **回完之後** 檢查 |
| **保證** | ✅ Deterministic（直接 block）| ❌ Probabilistic（model 重試還是可能錯）|
| **適用** | 業務規則、財務、合規、順序 | Format / semantic 錯誤 |
| **題目線索** | "must / mandatory / financial / compliance / order" | "validation fails / self-correction / format mismatch" |

→ **記法**：要 100% 保證的事用 hook；可接受重試的 quality 問題用 validation+retry。

---

### ② Schema (tool_use) vs Validation+retry

| | tool_use + JSON schema（L1）| Validation + retry（L2）|
|---|---|---|
| **解什麼錯** | **Syntax** 錯（壞 JSON、missing field）| **Semantic** 錯（加總錯、欄位放錯、邏輯衝突）|
| **執行時機** | API 回應就保證 schema 對 | API 回應後，跑 validator，錯了再送 retry |
| **題目線索** | "JSON syntax errors / strict parser / guarantee structure" | "values don't sum / wrong field / inconsistent" |

→ **記法**：tool_use 解的是「JSON 對不對」；validation 解的是「JSON 內容對不對」。

---

### ③ Few-shot vs Hook（CCAF 高頻 distractor pair）

| 場景 | 答案 | 為什麼 |
|---|---|---|
| 「Order 順序錯，12% 跳過 get_customer」 | **Hook**（Q1 答 A）| 涉及財務，要 deterministic |
| 「Format 飄，detailed instruction 不夠」 | **Few-shot** | 是 quality 問題，可 probabilistic |
| 「該 escalate 沒 escalate」 | **Few-shot + criteria**（Q3 答 A）| Quality 問題，沒財務硬約束 |

→ **規則**：題目有 financial / compliance / safety / 順序 字眼 → 永遠 hook。其他用 prompt-level（criteria/few-shot）。

---

### ④ Tool description (2.1) vs Few-shot (4.2)

兩者都修「行為錯誤」，但層次不同：

| | Tool description | Few-shot |
|---|---|---|
| **修的是** | Model 對「tool 是什麼」的理解 | Model 對「該怎麼推理」的理解 |
| **首選順序** | **First step**（low-effort, high-leverage）| 通常是 description 改完還不夠才加 |
| **題目線索（distractor pattern）** | "first step / minimal / similar tools 混淆" → 選 description | 「加 5-8 examples」是 distractor when description is the root cause |

→ **規則**：題目寫「first step」+ description 弱 → **永遠選 expand description**，few-shot 是 distractor（Q2 經典題）。

---

### ⑤ Restrict allowedTools (2.3) vs PreToolUse hook (1.5)

兩者都「阻止 tool 被用」，但層次不同：

| | Restrict allowedTools | PreToolUse hook |
|---|---|---|
| **層次** | Agent 看不到那個 tool（gone）| Agent 看得到，但 call 時被攔 |
| **適用** | Subagent 該專精 → 給少 tool | 動態判斷（refund < $500 才放）|
| **靜態 vs 動態** | 靜態（agent 設定時決定）| 動態（runtime 條件決定）|

→ **記法**：靜態權限分配 → allowedTools；動態條件攔截 → hook。

---

### ⑥ Self-reported confidence (4.1 / 5.2 反對) vs Calibrated confidence (5.5 推薦)

⚠️ **這個最容易踩雷**——「confidence」字眼出現在兩個對立的地方。

| | Self-reported confidence | Calibrated confidence |
|---|---|---|
| **誰評的** | Model 自己評（"I'm 90% sure"）| Model 給 score，但**用 labeled validation set 校準過 threshold**|
| **可信嗎** | ❌ Poorly calibrated | ✅ Calibrated by data |
| **CCAF 立場** | **反對**（Q3 選項 B 是 distractor）| **推薦**（5.5 主場）|
| **題目線索** | "agent self-reports confidence and routes by threshold" | "field-level confidence + labeled validation set + stratified sampling" |

→ **記法**：題目寫「self-report confidence + threshold」沒提 calibration → distractor，刪。寫「calibrated with labeled set」→ 是 5.5 正解。

---

## Domain 4 額外 7 組對照（學完 4.3-4.6 後新增）

### ⑦ tool_use 兩種用途（Domain 1 vs Domain 4.3）

| | 用途 A：傳統 agent loop | 用途 B：4.3 妙用（structured output）|
|---|---|---|
| **Tool 對應真實 function** | ✅ 有 implementation | ❌ 沒，純擺設 |
| **執行 function** | ✅ Client 執行 | ❌ 不執行 |
| **回 tool_result** | ✅ 是（agent loop 繼續）| ❌ 否（一次 call 結束）|
| **可在 batch API 跑** | ❌ 不行（multi-turn）| ✅ 行（single-turn）|

→ **記法**：tool_use 的 `input_schema` = function signature contract。借這個 contract 可以做兩件事：(A) 觸發外部動作；(B) 強迫 structured output。

---

### ⑧ Outbound vs Inbound normalization

| | Outbound（4.3 prompt rule）| Inbound（1.5 PostToolUse hook）|
|---|---|---|
| **誰生 heterogeneous data** | Claude 自己 | External tool |
| **誰被 normalize** | Claude 的 output | Tool 的 output（餵給 Claude 之前）|
| **方向** | Claude → 你 code | Tool → Claude |
| **Deterministic?** | ❌ Probabilistic | ✅ Deterministic |

→ **記法**：要 normalize **Claude 自己生的** → prompt（4.3）；要 normalize **external tool 給 Claude 的** → PostToolUse hook（1.5）。

---

### ⑨ Sync API vs Batch API

| | Sync API | Batch API |
|---|---|---|
| **Latency** | 即時（秒級）| 最壞 24h |
| **Cost** | 標準 | -50% |
| **SLA** | ✅ | ❌ 無保證 |
| **Multi-turn agent loop** | ✅ 支援 | ❌ 不支援 |
| **入口** | SDK / HTTP | SDK / HTTP（**不是 CLI**）|
| **適用** | Pre-merge / chatbot / blocking | Overnight / weekly / batch extraction |

→ **記法**：有人在等 → sync；沒人在等 + 沒 multi-turn → batch。SLA 數學：`max submission interval = SLA - 24h`。

---

### ⑩ 4.4 validation+retry vs 4.6 verification pass

兩者都「事後檢查」但目的完全不同：

| | 4.4 validation+retry | 4.6 verification pass |
|---|---|---|
| **誰檢查** | 你 code（Pydantic / 業務 validator）| **另一個 Claude call** |
| **檢查什麼** | Schema 對嗎、加總對嗎、邏輯一致嗎 | 每個 finding 的 confidence |
| **失敗做什麼** | Retry 帶 error feedback 修正 | 加 metadata（不修正，只標）|
| **目的** | 修錯 | 提供 routing signal |

→ **記法**：要修錯用 4.4，要評分給 routing 用 4.6。

---

### ⑪ 「Format 飄」的兩種層次

| | 4.2 場景的 format 飄 | 4.3 場景的 format 飄 |
|---|---|---|
| **飄在哪** | 整個 output 的 macro shape（無 schema 約束）| Schema 內某欄位的 micro value |
| **修法** | Few-shot 示範整體 shape | Prompt rule（或 few-shot）規定 value normalization |

→ **考試不會獨立考兩者寫法區別**，只會考層次差別（L0 prompt vs L1 schema vs L3 hook）。

---

### ⑫ Confidence 三層觀（最完整版）

| 層次 | 立場 | 出現在 |
|---|---|---|
| 用 self-confidence 當**唯一 filter**（hard cutoff，沒校準也沒結合其他 signal）| ❌ 強烈反對 | 4.1, 5.2 |
| 用 self-confidence 當 **routing signal**（多 input 之一）| ⚠️ 接受（短期方案）| **4.6** |
| Calibrated confidence + labeled validation set + 結合其他 signal | ✅ 推薦（正規）| **5.5** |

#### 「Sole filter」反 pattern 的真正三條件

```
條件 ① 沒驗證 calibration（不知 0.8 是不是真的 80% 正確）
條件 ② 沒結合其他 signal（純 confidence 一條過）
條件 ③ Hard cutoff (`if conf > 0.9: show`)

三條都犯 = CCAF 強烈反對
```

#### Prompt-based self-calibration 為何不算 calibration

你可能想：「prompt 裡叫 model 'be conservative with confidence' / 餵過往 data 給它調」——技術上 model 會被 prompt 影響，**但 CCAF 不把這稱為 calibration**：

1. 4.1 明文反對 "be conservative" 類 general instruction
2. Model 沒 feedback loop 真實驗證
3. Model 沒 introspective access 知道該壓多少
4. Probabilistic 修法不可量測收斂

→ **CCAF calibration 特指外部 code 用 labeled data 建 mapping**。

---

### ⑬ Routing / Dispatching 同義

| 詞 | 意思 |
|---|---|
| **Routing / dispatching** | 把每個 finding 分流到不同 downstream（auto-post / human review / drop / escalate）|
| **Calibrated routing** | 分流的 threshold 是用 labeled data 校準過的 |
| **Verification pass** | 第二次 Claude call 對 findings 標 confidence，給 routing 用 |
| **Mapping function** | 你 code 持有的 `raw_confidence → calibrated_accuracy` 對照表 |

→ **記法**：4.6 提供 raw signal、5.5 校準 signal、最後 routing 把 finding 送對地方。

---

## 終極決策樹（考試一張流程圖）

```
看到 production 問題，按順序問：

Q1: 是涉及 financial / compliance / safety / 順序保證 嗎？
   ✅ Yes → ★ PreToolUse hook / programmatic prerequisite（L3）
                          ↓ no
Q2: 是 JSON syntax 壞 / 要保證 structured 嗎？
   ✅ Yes → tool_use + JSON schema（L1）
                          ↓ no
Q3: 是 semantic 錯（加總錯、值不一致）能 retry 修嗎？
   ✅ Yes → Validation + retry with error feedback（L2）
                          ↓ no
Q4: 是 fabricate（編造資料）？
   ✅ Yes → Nullable schema（L1）+ few-shot null example（L0）
                          ↓ no
Q5: Root cause 是 tool description 弱？
   ✅ Yes → ★ Expand tool descriptions（L0）— first step!
                          ↓ no
Q6: Root cause 是 decision boundary 模糊（FP 高 / over-flag）？
   ✅ Yes → ★ Explicit criteria（4.1）+ optional few-shot（4.2）
                          ↓ no
Q7: 是 format 飄 / ambiguous reasoning 不一致？
   ✅ Yes → ★ Few-shot examples（4.2）
                          ↓ no
Q8: 是 single-pass review attention dilution？
   ✅ Yes → Multi-pass / multi-instance review（4.6）
                          ↓ no
Q9: 是 long-running session context degrade？
   ✅ Yes → Scratchpad / subagent delegation / /compact（5.4）
                          ↓ no
Q10: 是 multi-agent coordinator 太窄 / decomposition 錯？
   ✅ Yes → 修 coordinator decomposition（1.2）
```

---

## ★ 三條鐵律（記住這三條，60% 的迷惘消失）

### 鐵律 1：「First step / proportionate / low-effort」 → 永遠選最便宜的

順序是 **L0 prompt → L1 schema → L2 retry → L3 hook → L4 architecture**。

題目這幾個字出現 → 答案幾乎在 L0 / L1。

### 鐵律 2：Financial / compliance / safety / 順序 → 永遠 deterministic

**只要題目有「order / refund / financial / compliance / mandatory / must」字眼**，答案幾乎是 hook / programmatic prerequisite。

→ 此時 prompt-level（criteria / few-shot）一律是 distractor，再「explicit」也不行。

### 鐵律 3：「換大 model / 大 context window / 換 temperature」 → 永遠錯

CCAF 反對 capacity 解架構問題。看到這類選項基本都刪。

---

## 速查：學過的工具 + 還沒教的

| 工具 | 已學 | 還沒教 |
|---|---|---|
| Hook（PreToolUse / PostToolUse） | ✅ Domain 1.4, 1.5 | |
| Tool description | ✅ Domain 2.1 | |
| `allowedTools` restrict | ✅ Domain 2.3 | |
| `tool_choice` 三模式 | ✅ Domain 2.3, 4.3 | |
| Explicit criteria | ✅ Domain 4.1 | |
| Few-shot | ✅ Domain 4.2 | |
| **tool_use + JSON schema** | ✅ Domain 4.3 | |
| **Nullable schema + enum extensibility** | ✅ Domain 4.3 | |
| **Validation + retry with feedback 三件套** | ✅ Domain 4.4 | |
| **Self-correction validation flow**（calculated/stated/conflict）| ✅ Domain 4.4 | |
| **`detected_pattern` 欄位** | ✅ Domain 4.4 | |
| **Pydantic validator** | ✅ Domain 4.4 | |
| **Batch API（含 `custom_id`、SLA 數學、multi-turn 限制）**| ✅ Domain 4.5 | |
| **Multi-pass / multi-instance review** | ✅ Domain 4.6 | |
| **Verification pass** | ✅ Domain 4.6 | |
| Persistent case facts / trim tool output | | ⏭️ 5.1 |
| Escalation pattern | | ⏭️ 5.2 |
| Structured error context | | ⏭️ 5.3 |
| Scratchpad / `/compact` / subagent | | ⏭️ 5.4 |
| **Calibrated confidence + stratified sampling** | | ⏭️ 5.5 |
| Claim-source mapping | | ⏭️ 5.6 |

---

## Domain 4 新增 term 速記（已釐清概念）

| Term | 意思 |
|---|---|
| **`tool_use` block** | Claude 回應的「請 call function」結構化 request；不是真的執行 |
| **`input_schema`** | tool 的 args 規格——**Claude 給 client 的東西**必須符合（不是反向）|
| **`tool_choice: "auto"`** | Model 可選擇要不要 call（**不保證** structured）|
| **`tool_choice: "any"`** | Model 必 call tool 但自選——適合「多 schema + 不知 doc type」|
| **`tool_choice: forced`** | `{"type":"tool","name":"X"}` 強制特定 tool |
| **API layer** | Anthropic 伺服器端，schema enforcement 在這發生（你 code 看不到但有效）|
| **Envelope** | `--output-format json` 包 metadata（session_id/cost）的外殼，**內容仍是文字**——要結構化內容用 `--json-schema`|
| **Nullable schema** | Field 設 `["string", "null"]` 防 fabrication |
| **`"unclear"` enum** | Ambiguous case 的 escape value |
| **`"other"` + detail** | Extensible categorization |
| **Pydantic** | Python schema validator 庫，可加 `@model_validator` 業務 rule |
| **`stated_X` vs `calculated_X`** | Self-correction 對照欄位——讀字 vs 算術兩種任務不會一起錯 |
| **`conflict_detected`** | Source 內部矛盾的 boolean flag |
| **`detected_pattern`** | Aggregate dismissals 用的 enum 欄位（4.4 → 4.1 閉環）|
| **三件套** | Retry prompt 必含：原 doc + previous extraction + specific error |
| **`custom_id`** | Batch request/response 對應 label（你設、API 原樣傳）|
| **5-turn = 5 天** | Batch + multi-turn 的最壞 case（每 turn 24h）|
| **Verification pass** | 第二次 Claude call 對 findings 標 confidence，給 routing 用（不修正）|
| **Mapping function** | 你 code 持有的 `raw_confidence → calibrated_accuracy` 對照表 |
| **Outbound normalization** | 4.3 prompt rule，normalize Claude 自己的 output |
| **Inbound normalization** | 1.5 PostToolUse hook，normalize tool 給 Claude 的 result |
| **Routing = Dispatching** | 把 finding 分流到 downstream（auto-post / human / drop / escalate）|

---

## Domain 4 反 pattern 速記（已釐清的 distractor 收集）

| 反 pattern | 屬於 |
|---|---|
| 「Schema 也解 semantic error」 | 4.3/4.4 分界線陷阱 |
| `tool_choice: "auto"` 期待保證 structured | 4.3 |
| 「強化 prompt 解 syntax 飄」 | 4.3 |
| Required field + 期待別 fabricate | 4.3（要 nullable）|
| 盲目 retry 不附 error feedback | 4.4 |
| Information absent 還在 retry | 4.4（要 null 或 escalate）|
| 「更嚴格 schema 解 sum mismatch」 | 4.4 |
| Pre-merge check 改 batch | 4.5 |
| Multi-turn agent loop 上 batch | 4.5（會在第一個 tool_use 處停）|
| 「Batch + fallback to sync」 | 4.5（雙倍 cost + 複雜度）|
| 「換大 context window 解 attention dilution」 | 4.6（鐵律 3）|
| 叫 dev 把 PR 拆小 | 4.6（shift burden）|
| Majority voting 解 review 不一致 | 4.6（反而壓制 detection）|
| Extended thinking 解 self-review bias | 4.6（同 session 還是 anchored）|
| Prompt 裡叫 model 「be conservative with confidence」 | 4.1 + 4.6（不算 calibration）|
| Self-rate confidence + threshold + only show > X | 4.1/5.2/4.6 — 三條件都犯（沒校準/沒多 signal/hard cutoff）|

---

## 跟其他學習筆記的關係

- 這份是**橫切表**（cross-cutting）——抽出散落各 domain 的「修 output」工具集中比較
- 各 domain 的 deep-dive 在 `CCAF_Domain_X_Summary.md`
- 觀念釐清在 `CCAF_Handoff_Context.md`
- 學完 4.3-4.6 + Domain 5 後，這份表會更完整、可以正式驗收

---

# 🆕 Domain 5 整合（學完 5.1-5.6 後新增）

## L5 / L6 工具補完（Domain 5 主場）

| Layer | 工具 | Domain | 解什麼問題 | Deterministic? |
|---|---|---|---|---|
| **L0 Prompt** | Case facts block（transactional fact 抽出） | 5.1 | Long conversation 後數字/日期飄 | ❌ Probabilistic（但結構性對抗）|
| | Escalation criteria + few-shot | 5.2 | Agent escalate 太多/太少 | ❌ Probabilistic |
| **L1 Schema** | Field-level confidence schema | 5.5 | 每欄位獨立 routing decision | ✅ Per-field |
| | Structured error context dict | 5.3 | Coordinator 拿到 4 要素做 recovery | ✅ Structural |
| | Claim-source mapping schema | 5.6 | Synthesis preserve source attribution | ✅ Structural |
| | Manifest schema | 5.4 | Crash recovery 重建 agent state | ✅ Structural |
| **L3 Hook** | Trim verbose tool output（PostToolUse 落地）| 5.1, 1.5 | Tool result 40+ fields 累積吃 token | ✅ Deterministic |
| **L4 Architecture** | Subagent delegation for verbose exploration | 5.4 | Long codebase exploration context degradation | ✅ Architectural |
| | Multi-issue structured layer | 5.1 | 多 issue session 互相污染 | ✅ Structural |
| | Coverage annotation in synthesis | 5.3 | Multi-agent gap 透明化 | ✅ Process |
| **L5 後驗校準** | ★ Field-level confidence + calibrated threshold | 5.5 | High-confidence extraction 還是有誤 | ✅ With labeled set |
| | ★ Stratified random sampling | 5.5 | Aggregate metric 掩蓋 segment 災難 + 偵測 novel pattern | ✅ Sampling guarantee |
| | By-segment by-field accuracy analysis | 5.5 | Aggregate 97% 可能掩蓋 segment 30% | ✅ Verified |
| **L6 流程** | Scratchpad files | 5.4 | Externalize key finding 跨 context boundary | ❌ Process |
| | `/compact` 即時壓縮 | 5.4 | Context 塞太多時補救 | ❌ Process |
| | Phase summarization | 5.4 | Long exploration 連貫性 | ❌ Process |
| | Crash recovery via manifest | 5.4 | Resume 不從頭來 | ✅ Recovery |
| | Escalation pattern | 5.2 | Agent → human handoff | ❌ Probabilistic |
| | Document analysis 不替 coordinator decide | 5.6 | Conflict 全帶上去 annotate | ❌ Process |
| | Render by content type | 5.6 | Financial → table、news → prose | ❌ Format |

---

## Domain 5 額外 6 組對照

### ⑭ 5.1 case facts vs 5.4 scratchpad（兩種 externalize memory）

| | 5.1 case facts block | 5.4 scratchpad files |
|---|---|---|
| **場景** | Multi-turn 對話（customer support） | Long-running exploration（codebase） |
| **存什麼** | Transactional fact（amount/date/ID）| Architectural finding（class/flow path） |
| **存哪裡** | 在 prompt 內每次帶 | 外部檔（`.md` / `.json`）|
| **更新** | 對話中 update | Phase 之間 update |
| **共通** | **Externalize 不能飄的東西到 prompt 外** |

→ **記法**：對話的 fact → case facts block；exploration 的 finding → scratchpad files。

---

### ⑮ 5.2 escalation criteria vs 1.4 hook（escalation 兩 layer）

| | 5.2 escalation criteria | 1.4 programmatic prerequisite / PreToolUse hook |
|---|---|---|
| **解什麼** | Judgment call（policy gap、明示 human） | Hard rule（refund > $500 必擋）|
| **Layer** | L0 prompt-level | L3 hook |
| **Deterministic** | ❌ Probabilistic | ✅ Fully |
| **何時用** | Soft escalation decision | Hard business constraint |

→ **記法**：題目「financial / mandatory / must」→ hook；「該轉 human 嗎」→ escalation criteria。

---

### ⑯ 4.4 conflict_detected vs 5.6 multi-source 衝突

| | 4.4 conflict_detected | 5.6 multi-source annotation |
|---|---|---|
| **衝突在哪** | **單一** source 內部（invoice header vs footer）| **多** source 之間（paper A vs paper B）|
| **修法** | Boolean flag + retry / human | 並列 annotate + 不選 |
| **Report 處理** | Validation 失敗 retry | Contested findings section |

→ **記法**：單 source 內部矛盾 → 4.4；跨 source 不一致 → 5.6。

---

### ⑰ 5.3 coverage annotation vs 5.6 source attribution（透明度兩面向）

| | 5.3 coverage annotation | 5.6 source attribution |
|---|---|---|
| **標什麼** | 哪 topic 沒 source（**有 gap**）| 每 claim 對應哪 source（**有 evidence**）|
| **角度** | 從錯誤面 | 從證據面 |
| **共通** | **transparent synthesis 兩面向** |

→ **記法**：完整 transparent synthesis = 5.3 coverage gap + 5.6 source attribution。

---

### ⑱ 5.1 metadata producer vs 5.6 metadata consumer（兩端配合）

| | 5.1 Skills bullet 5 | 5.6 Knowledge bullet 4 |
|---|---|---|
| **講什麼** | Subagent **必含** metadata | Synthesis **必 preserve** metadata |
| **階段** | Production 端（產生）| Synthesis 端（保留）|
| **共通** | 兩端都做才不會丟 metadata |

→ **記法**：5.1 上游強制產出 + 5.6 下游強制保留 = 完整 metadata chain。

---

### ⑲ 4.6 verification pass vs 5.5 calibrated confidence（confidence 短期 vs 長期）

| | 4.6 verification pass | 5.5 calibrated confidence |
|---|---|---|
| **Confidence 哪來** | Model self-rate（uncalibrated）| Model self-rate + **labeled validation set 校準** |
| **Labeled data** | 不需要 | **必需** |
| **量測 calibration** | 沒做 | **明確做** |
| **適用** | 短期方案、快速加 routing signal | 長期正規、要 reduce human review |

→ **記法**：4.6 是過渡，5.5 是終點。看到 "labeled validation set" → 5.5 正解。

---

## 終極決策樹（更新版含 Domain 5）

```
看到 production 問題，按順序問：

Q1: 是涉及 financial / compliance / safety / 順序保證 嗎？
   ✅ Yes → ★ PreToolUse hook / programmatic prerequisite（L3）
                          ↓ no
Q2: 是 JSON syntax 壞 / 要保證 structured 嗎？
   ✅ Yes → tool_use + JSON schema（L1）
                          ↓ no
Q3: 是 semantic 錯（加總錯、值不一致）能 retry 修嗎？
   ✅ Yes → Validation + retry with error feedback（L2）
                          ↓ no
Q4: 是 fabricate（編造資料）？
   ✅ Yes → Nullable schema（L1）+ few-shot null example（L0）
                          ↓ no
Q5: Root cause 是 tool description 弱？
   ✅ Yes → ★ Expand tool descriptions（L0）— first step!
                          ↓ no
Q6: Root cause 是 decision boundary 模糊（FP 高 / over-flag）？
   ✅ Yes → ★ Explicit criteria（4.1）+ optional few-shot（4.2）
                          ↓ no
Q7: 是 escalation 判斷錯（該轉沒轉 / 不該轉卻轉）？
   ✅ Yes → ★ Escalation criteria + few-shot（5.2）
                          ↓ no
Q8: 是 format 飄 / ambiguous reasoning 不一致？
   ✅ Yes → ★ Few-shot examples（4.2）
                          ↓ no
Q9: 是 single-pass review attention dilution？
   ✅ Yes → Multi-pass / multi-instance review（4.6）
                          ↓ no
Q10: 是 long conversation 後 numerical/日期飄？
   ✅ Yes → ★ Case facts block + trim tool output（5.1）
                          ↓ no
Q11: 是 long-running exploration context degrade？
   ✅ Yes → ★ Scratchpad + subagent delegation + /compact（5.4）
                          ↓ no
Q12: 是 multi-agent subagent 失敗訊息怎麼傳？
   ✅ Yes → ★ Structured error context 4 要素（5.3）
                          ↓ no
Q13: 是想 reduce human review 但怕 segment 災難？
   ✅ Yes → ★ Field-level calibrated confidence + by-segment analysis + stratified sampling（5.5）
                          ↓ no
Q14: 是 multi-source synthesis source attribution 丟掉？
   ✅ Yes → ★ Claim-source mapping schema + report 分段（5.6）
                          ↓ no
Q15: 是 multi-agent coordinator decomposition 太窄？
   ✅ Yes → 修 coordinator decomposition（1.2）
```

---

## 速查：學過的工具 — 完整 ✅

| 工具 | 已學 |
|---|---|
| Hook（PreToolUse / PostToolUse） | ✅ Domain 1.4, 1.5 |
| Tool description | ✅ Domain 2.1 |
| `allowedTools` restrict | ✅ Domain 2.3 |
| `tool_choice` 三模式 | ✅ Domain 2.3, 4.3 |
| Explicit criteria | ✅ Domain 4.1 |
| Few-shot | ✅ Domain 4.2 |
| tool_use + JSON schema | ✅ Domain 4.3 |
| Nullable schema + enum extensibility | ✅ Domain 4.3 |
| Validation + retry with feedback 三件套 | ✅ Domain 4.4 |
| Self-correction validation flow | ✅ Domain 4.4 |
| `detected_pattern` 欄位 | ✅ Domain 4.4 |
| Pydantic validator | ✅ Domain 4.4 |
| Batch API（custom_id, SLA 數學, multi-turn 限制） | ✅ Domain 4.5 |
| Multi-pass / multi-instance review | ✅ Domain 4.6 |
| Verification pass | ✅ Domain 4.6 |
| **Case facts block + trim tool output** | ✅ Domain 5.1 |
| **Position-aware（beginning + section header）** | ✅ Domain 5.1 |
| **Escalation criteria + few-shot** | ✅ Domain 5.2 |
| **Multiple matches → 問 identifier** | ✅ Domain 5.2 |
| **Structured error context（4 要素）** | ✅ Domain 5.3 |
| **Local recovery first + propagate 救不了的** | ✅ Domain 5.3 |
| **Coverage annotation in synthesis** | ✅ Domain 5.3 |
| **Scratchpad files** | ✅ Domain 5.4 |
| **Subagent delegation for verbose exploration** | ✅ Domain 5.4 |
| **`/compact` + crash recovery manifest** | ✅ Domain 5.4 |
| **Field-level confidence + calibrated threshold** | ✅ Domain 5.5 |
| **Stratified random sampling** | ✅ Domain 5.5 |
| **By-segment by-field accuracy analysis** | ✅ Domain 5.5 |
| **Claim-source mapping schema** | ✅ Domain 5.6 |
| **Report 分段 well-established / contested** | ✅ Domain 5.6 |
| **Render by content type** | ✅ Domain 5.6 |

→ **🎉 全部學完，toolkit 正式驗收**。

---

## Domain 5 新增 term 速記

| Term | 意思 |
|---|---|
| **Case facts block** | Transactional fact 抽出每次 prompt 原樣帶 |
| **Trim verbose tool output** | Tool result 進 context 前先砍多餘欄位 |
| **Lost in the middle** | LLM 對長 input 中段 attention 弱、頭尾可靠 |
| **Position-aware** | 重要 finding 放開頭 + section header |
| **Escalation criteria + few-shot** | ESCALATE/RESOLVE rule + 邊界 case 示範 |
| **Policy gap** | Policy silent on customer's specific request |
| **Structured error context** | 4 要素 dict（failure_type/attempted_query/partial_results/alternatives）|
| **Access failure vs valid empty** | 沒查到 vs 查了沒料（不同 status code）|
| **Local recovery first** | Subagent 先 retry transient，救不了才 propagate |
| **Coverage annotation** | Synthesis 標哪 topic 有 gap |
| **Scratchpad file** | 外部 `.md`/`.json` 持久化 key finding |
| **Subagent delegation** | 隔離 verbose exploration output |
| **`/compact`** | Claude Code 內建 context 壓縮指令 |
| **Manifest** | Crash recovery 用的 agent state index |
| **Aggregate metric 風險** | Overall 97% 可能掩蓋 segment 30% |
| **Stratified random sampling** | 各 segment 各抽 sample 確保 coverage |
| **Field-level confidence** | 每欄位獨立 confidence（vs overall）|
| **Calibrated threshold** | Labeled validation set 反推對應目標 accuracy 的 threshold |
| **Calibration curve** | Raw confidence → actual accuracy 對照（每 field 不同）|
| **Claim-source mapping** | 每 claim 結構化 link 到 source（URL/date/excerpt）|
| **Well-established vs contested** | Report 分段顯示 epistemic 狀態 |
| **Render by content type** | Financial → table、news → prose、technical → list |

---

## Domain 5 反 pattern 速記

| 反 pattern | 屬於 |
|---|---|
| Progressive summarization 壓掉 numerical fact | 5.1 |
| 換大 context window 解 long-conversation 飄 | 5.1, 5.4（鐵律 3）|
| 純 prompt instruction「記住數字 / 記得 class 名」 | 5.1, 5.4（vague guidance）|
| Self-reported confidence 當 escalation trigger | 5.2 |
| Sentiment analysis → escalate | 5.2（sentiment ≠ complexity）|
| 訓練 classifier 預測 escalation | 5.2（out-of-scope）|
| Customer 明示要 human 還先 investigate | 5.2 |
| Multiple matches 用 heuristic 猜 | 5.2 |
| Subagent generic error message（"unavailable"）| 5.3 |
| Subagent return [] as success | 5.3（silent suppression）|
| Subagent throw exception terminate workflow | 5.3 |
| 「叫 user restart session / 拆 task」| 5.4（shift burden）|
| Aggregate accuracy 直接決定全自動 | 5.5（招牌風險）|
| Self-rate confidence + 拍腦袋 threshold | 5.5（無 calibration）|
| Synthesis 把 source 砍掉「為了精簡」 | 5.6 |
| Conflict 自己選一個 silently | 5.6 |
| 沒 publication date | 5.6（時序差被誤判）|
| Generic「based on multiple sources」disclaimer | 5.6（要 specific）|
| 全 prose 寫財務 / 全 list 寫 narrative | 5.6（render 該按 content type）|

---

## 三條鐵律（含 Domain 5 適用情境）

### 鐵律 1：「First step / proportionate / low-effort」 → 永遠選最便宜的
- 順序：L0 prompt → L1 schema → L2 retry → L3 hook → L4 architecture → L5 calibration
- Domain 5 例：先 case facts block（5.1）→ 不夠才上 multi-instance review（4.6）

### 鐵律 2：Financial / compliance / safety / 順序 → 永遠 deterministic
- 此時 prompt-level（criteria / few-shot / escalation criteria）一律 distractor
- Hook（1.4 / 1.5）才是答案

### 鐵律 3：「換大 model / 大 context window / 換 temperature」 → 永遠錯
- Domain 5 特別常見：long conversation / long exploration 容易誤選「換大 context」
- 但 lost-in-the-middle / attention quality 不是容量問題

---

# 🎓 Toolkit 正式驗收

學完整個 CCAF 5 個 Domain 後，這份 toolkit 涵蓋：

| 範圍 | 數量 |
|---|---|
| 工具 layer | L0 - L6 七層 |
| 主要工具 | 30+ |
| 對照組（容易混的）| 19 組 |
| 終極決策樹節點 | 15 個 Q |
| 鐵律 | 3 條 |
| Term 速記 | 50+ |
| 反 pattern 速記 | 40+ |

**用法建議**：
- 練習題遇到不知選哪個 → 看終極決策樹
- 看到熟悉但記不清的詞 → 翻 term 速記
- 看到答案說不上來為何錯 → 翻反 pattern 速記
- 模擬考前最後溫習 → 看三條鐵律 + Confidence 三層觀

---

# 🎯 3 秒判斷法：Few-shot vs Hook（A vs C 永不卡關）

> 練習題實戰時最常卡的就是 **A (few-shot/prompt)** vs **C (hook/programmatic)**。這套判斷法直接秒殺。

## 核心心法（一句話）

> **「題目有錢的味道 → hook；題目只有質量問題 → prompt-level」**

「錢的味道」字眼包括：
`refund`, `payment`, `billing`, `charge`, `account`, `financial`, `identity`, `compliance`, `mandatory`, `must`, `always`, `never`, `prerequisite`, `order/sequence guarantee`, `% failure with real harm`, `misidentified`, `incorrect [transaction]`

---

## 為什麼會一直卡 A vs C？

**根本原因**：A 跟 C **看起來都「合理」**，都會降低 failure rate。但 CCAF 考的不是「降低」，是「**降到夠不夠**」。

```
Prompt-level (A few-shot, B prompt instruction):
   12% → 5% → 2% → 1% → 0.5%
   永遠 > 0%
   ↓
   財務場景: ❌ 不可接受

Hook (C programmatic prerequisite):
   12% → 0%
   ↓
   財務場景: ✅ 唯一答案
```

---

## 3 秒判斷流程

問自己**一個問題**：

> **「失敗的代價是什麼？」**

```
失敗會：虧錢 / 違法 / 違反 compliance / 破壞 safety / 順序錯導致業務崩
       ↓
   ★ 必 deterministic → Hook / programmatic prerequisite (C 類)
   prompt-level (A/B 類) 全部 distractor

失敗會：output 飄一點 / 偶爾不一致 / 風格不統一 / 質量稍差但沒實質傷害
       ↓
   可 probabilistic → Few-shot / criteria / description (A/B 類)
   Hook over-engineered
```

---

## Keyword 觸發詞速查

### Deterministic 觸發詞（看到 → 必 hook）

| Keyword | 為什麼 |
|---|---|
| **financial / refund / payment / charge / billing** | 虧錢 |
| **compliance / policy / mandatory / must / always / never** | 合規硬約束 |
| **safety / security / identity / verification / authentication** | 安全硬約束 |
| **prerequisite / order / sequence / "X first" / "before Y"** | 順序保證 |
| **incorrect refund / wrong charge / misidentified account** | 已產生實質傷害 |
| **% failure rate + financial action** | 任何 > 0% 不可接受 |

### Probabilistic 觸發詞（看到 → 可 prompt-level）

| Keyword | 為什麼 |
|---|---|
| **format consistency / output style / inconsistent format** | Quality 問題 |
| **ambiguous reasoning / edge case judgment** | 教 reasoning |
| **null behaviour / fabrication on missing field** | Quality + nullable |
| **detailed instruction not enough** | Few-shot 主場 |

### Root cause 在別的地方（A/C 都錯）

| Keyword | 真正答案 |
|---|---|
| **tool selection wrong (similar tools confused)** | ★ Expand description (2.1) — A/C 都是 distractor |
| **decision boundary unclear (over-flag)** | Explicit criteria (4.1) |

---

## 同類題目變形（CCAF 喜歡這個題型）

**所有這些題目的 A 都是 distractor，C/類似 hook 才是正解**：

| 題目場景 | A (distractor) | 正解 (hook 類) |
|---|---|---|
| Agent 12% 跳過 verification | Few-shot 教 verification first | Programmatic prerequisite |
| Agent 偶爾 refund > $500 | Prompt 加「max $500 rule」 | PreToolUse hook 擋 |
| Agent 不該 call 某 tool 卻 call | Few-shot 教邊界 | `allowedTools` restrict 或 hook |
| 8% 跳過 KYC 直接交易 | Prompt 強調 KYC mandatory | Programmatic prerequisite |
| 5% case forgot to log audit | Few-shot 加 logging step | PostToolUse hook 自動 log |
| Agent 偶爾把 PII 傳到 external API | Prompt 加 PII rule | PreToolUse hook 擋 PII |

→ **看到 % failure rate + 業務 critical action**，**永遠 hook**.

---

## 反過來：什麼時候 A few-shot 才是正解？

兩個條件**都成立**才選 few-shot：

**條件 1**：題目**完全沒**:
- ❌ Financial / refund / payment 字眼
- ❌ Compliance / mandatory / must 字眼
- ❌ Safety / verification 字眼
- ❌ % failure rate + 實質業務傷害

**條件 2**：題目**有**:
- ✅ "Output format inconsistent"
- ✅ "Ambiguous case reasoning unclear"
- ✅ "Style 飄"
- ✅ "Edge case judgment"
- ✅ "Detailed instruction not enough to make consistent"

---

## 實戰：Sample Question 1 拆解

> Production data shows that in 12% of cases, your agent skips `get_customer` entirely and calls `lookup_order` using only the customer's stated name, occasionally leading to **misidentified accounts** and **incorrect refunds**. What change would most effectively address this reliability issue?
>
> A) Add few-shot examples showing the agent always calling get_customer first
> B) Enhance the system prompt: customer verification mandatory
> **C) ★ Programmatic prerequisite blocking lookup_order and process_refund until get_customer returns verified ID** ✅
> D) Routing classifier enabling subset of tools

**3 秒判斷流程**：

1. 找「錢的味道」字眼：
   - ⚠️ "**refunds**" → 財務 ✅
   - ⚠️ "**misidentified accounts**" → 客戶身份錯誤 ✅
   - ⚠️ "**agent skips X entirely**" → 順序保證失敗 ✅
   - **3 個都中** → **必 hook**

2. 刪所有 prompt-level：
   - A (few-shot) ❌ — 12% → 1% 但仍 > 0%
   - B (system prompt) ❌ — 同樣 probabilistic

3. 剩下 hook 類：
   - C (programmatic prerequisite) ✅
   - D (routing classifier) ❌ — 解錯問題（tool availability 不是 ordering）

→ **正解 C**

---

## 一句話收掉

> **看到 refund / payment / financial / mandatory / order guarantee / % critical failure → 直接刪 A/B (prompt-level)，剩下 hook / programmatic prerequisite 那個就是答案**.
>
> **不要在 A/C 之間「比較哪個好」**——hook 在這場景是 0% failure，prompt 永遠 > 0%，**單向必殺**.
