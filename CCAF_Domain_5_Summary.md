# CCAF Domain 5: Context Management & Reliability

**佔比 15%**

整個 domain 在解 **「LLM 在長對話 / 多 agent / 多 source 場景下，怎麼保持可靠？」**——不是 single-shot quality（Domain 4 主場），而是「**跨時間、跨 agent、跨 source**」的 reliability。

**Domain 5 vs Domain 4：**
- Domain 4 解 single-call 的 quality 設計
- Domain 5 解 multi-turn / multi-agent / multi-source 的 reliability 設計

**對應 exam scenarios：**
- **Scenario 1**: Customer Support Resolution Agent（5.1, 5.2 主場）
- **Scenario 3**: Multi-Agent Research System（5.3, 5.6 主場）
- **Scenario 4**: Developer Productivity / Codebase Exploration（5.4 主場）
- **Scenario 6**: Structured Data Extraction（5.5 主場）

**6 個 task statements：**
- 5.1 Manage conversation context across long interactions
- 5.2 Escalation & ambiguity resolution patterns
- 5.3 Error propagation across multi-agent systems
- 5.4 Large codebase exploration context management
- 5.5 Human review workflows + confidence calibration
- 5.6 Multi-source synthesis: provenance + uncertainty

---

## Task Statement 5.1 — Manage conversation context across long interactions

### 整個 task 在解決什麼問題

Multi-turn 對話越長，critical fact 越會「飄、漏、被稀釋」。Customer support agent 跑 30 turn 後，原本精確的 `$89.99 / order #12345 / 5/3` 變成「approximately $90 / recent order / some date」。

### 四個 Knowledge of 重點

1. **★★★ Progressive summarization 風險**：把 numerical / date / customer-stated expectation 壓成 vague summary → reasoning 全部走樣
2. **★★ Lost in the middle 效應**：long input 頭尾可靠，**中段 finding 容易漏**
3. **★★ Tool result 累積 disproportionate 吃 token**：`lookup_order` 40+ fields 但只用 5
4. **★ Multi-turn 必傳完整 conversation history**：API stateless，client 要負責累積

### 六個 Skills in 重點

1. **★★★ "Case Facts" block** — 把 transactional fact（amount/date/ID/status）抽出來，**每次 prompt 原樣帶**（不進 summary）
2. **★★ Multi-issue session structured layer** — 多 issue 時各 issue 獨立保留 critical data
3. **★★★ Trim verbose tool output** — 在累積前砍（`lookup_order` 只留 return-relevant 5 個欄位）
4. **★★ Position-aware** — Key findings summary 放**開頭**（official 用詞），section header 切段對抗 lost-in-the-middle
5. **★ Subagent metadata** — 加 date / source location / methodology context（給 5.6 synthesis 用）
6. **★★ Upstream agent 改 structured data**（不是 verbose narrative）— downstream context budget 有限時

### 已釐清概念

**Position-aware 原文只說「beginning」**：official Skills bullet 4 只寫 `at the beginning`。Knowledge bullet 2 才講「beginning AND end 都可靠」。考試**保險答案 = beginning**。

**Upstream 改 structured 的三種實作**：
- **A. tool_use + input_schema**（4.3）— upstream agent 用 schema 強制
- **B. Criteria / few-shot**（4.1/4.2）— upstream system prompt 要求
- **C. PostToolUse hook**（1.5）— 事後 transform（適合 upstream 是 external tool）
- 5.1 主指 A+B（"modifying upstream agents"），不是 C

### 易踩陷阱

| 陷阱 | 為什麼錯 |
|---|---|
| Progressive summarization 解 long context | Summarize 會壓掉 numerical fact |
| 換大 context window | 鐵律 3，且 lost-in-the-middle 不會因 context 大而消失 |
| 純 prompt instruction「記住數字」 | Vague guidance（4.1 反對）|
| 所有 conversation 都靠 model 記得 | Stateless → client 要傳完整 history |

### 考試判斷規則

| 題目線索 | 答案方向 |
|---|---|
| Long conversation 後 numerical / 日期飄 | Case facts block |
| Tool result 40+ fields 但只用 5 | Trim verbose tool output |
| 多個 issue 在同 session 互相污染 | Structured issue layer |
| Aggregated input 中段 finding 漏掉 | Place summary 開頭 + section header |
| Multi-agent downstream context 不夠 | Upstream 改回 structured |
| Synthesis 需要判斷時間/來源 | Subagent 加 metadata |

### ★ 核心 mental model

> **Long conversation context 分兩層：transactional fact 層（不能飄 → 結構化每次原樣帶）+ narrative 層（可以濃縮）。Tool output 在累積前 trim。Aggregated input 重點放開頭 + section header 對抗 lost-in-the-middle。永遠不要 summarize 數字、日期、ID、status——那是「資料」不是「敘事」。**

---

## Task Statement 5.2 — Escalation & ambiguity resolution patterns

### 整個 task 在解決什麼問題

Customer support agent 在「該轉 human」跟「該自己解」之間判斷錯——standard case 也轉、policy gap 卻硬解、明示要 human 還在 investigate。**5.2 = 4.1 在 escalation 場景的應用**——明確 categorical criteria > vague judgment / sentiment / self-confidence。

### 四個 Knowledge of 重點

1. **★★★ 三個 valid escalation triggers**：
   - ① Customer 明示要 human
   - ② Policy exception/gap（**not just complex case**）
   - ③ 無法 meaningful progress
2. **★★ Customer 明示 human → 立刻轉**，不要先 investigate
3. **★★★ Sentiment / self-reported confidence 不可靠**（CCAF 招牌反 pattern）
4. **★ Multiple customer matches → 問 identifier，不要 heuristic 猜**

### 五個 Skills in 重點

1. **★★★ Explicit escalation criteria + few-shot**（4.1+4.2 在 escalation 場景組合）
2. **★★ 尊重明示要 human** — 立刻轉不要 investigate
3. **★ Frustration acknowledgment + offer resolution** — customer **重申**才轉
4. **★★★ Policy gap → escalate**（policy 沒提的 specific request 不要替 company 自創規則）
5. **★★ Multiple matches → 問額外 identifier**

### 易踩陷阱

| 陷阱 | 為什麼錯 |
|---|---|
| Self-reported confidence threshold 當 escalation trigger | 5.2 Knowledge bullet 3 直接反對 |
| Sentiment analysis → escalate | Sentiment ≠ complexity |
| 訓練 classifier 預測 escalation | Out-of-scope + over-engineered |
| 「Complex case 一律 escalate」 | 不是 valid trigger（policy gap 才是）|
| Customer 明示要 human 還先 investigate | 違反尊重明示原則 |

### 考試判斷規則

| 題目線索 | 答案方向 |
|---|---|
| Agent escalate 太多 / 太少 | Explicit escalation criteria + few-shot |
| Customer 明示「want human」| 立即 escalate（不 investigate） |
| Customer frustration 但沒明示要 human | Acknowledge + offer resolve；reiterate 才轉 |
| Policy 沒涵蓋的 specific request | Escalate（policy gap） |
| Tool 回 multiple matches | 問額外 identifier |
| Self-rated confidence + threshold | ❌ Distractor |
| Sentiment analysis | ❌ Distractor |

### ★ Sample Question 3（Page 26-27）— 5.2 主場

55% first-contact resolution，escalate 太多 standard case + 沒 escalate policy exception。

**正解 A**：Explicit escalation criteria + few-shot demo when to escalate vs resolve。
- B（self-rate confidence）❌ 5.2 反對
- C（trained classifier）❌ over-engineered + out-of-scope
- D（sentiment analysis）❌ sentiment ≠ complexity

### ★ 核心 mental model

> **5.2 = 4.1 在 escalation 場景的應用**。三個 valid trigger：明示要 human / policy gap / 無法 progress。三個必刪 distractor：self-rated confidence / sentiment / 訓練 classifier。**Multiple matches 問 identifier 不猜；明示要 human 立刻轉不 investigate**。

---

## Task Statement 5.3 — Error propagation across multi-agent systems

### 整個 task 在解決什麼問題

Multi-agent system 中 subagent 失敗時，怎麼把訊息傳回 coordinator 讓它做 intelligent recovery？**反 pattern 兩端**：silent suppression（return empty as success）vs terminate workflow（一個死全部死）。**正解中間**：propagate **structured error context**。

### 四個 Knowledge of 重點

1. **★★★ Structured error context 四要素**：failure_type + attempted_query + partial_results + alternatives
2. **★★★ Access failure vs valid empty result 區分**：sounds same, completely different meaning
3. **★★ Generic error 隱藏 valuable context**（"search unavailable" 沒救）
4. **★★★ 兩個對立反 pattern**：silent suppression / terminate workflow

### 四個 Skills in 重點

1. **★★★ 設計 structured error context dict**（4 要素都要包）
2. **★★ Access failure / valid empty 用不同 status code 區分**
3. **★★★ Local recovery first，只 propagate 救不了的**（subagent 先試自救）
4. **★★ Synthesis output 含 coverage annotation**（哪些 topic 有 gap 明示）

### 易踩陷阱

| 陷阱 | 為什麼錯 |
|---|---|
| Generic error message（"search unavailable"）| 隱藏 context |
| Empty result marked as success | Silent suppression |
| Throw exception terminate workflow | 其他 subagent work 白做 |
| Catch error return [] 標 success | 同 silent suppression |

### 考試判斷規則

| 題目線索 | 答案方向 |
|---|---|
| Subagent 失敗訊息傳 coordinator | Structured error context（4 要素）|
| 怎麼分「沒查到 vs 真沒料」 | Access failure vs valid empty 不同 status |
| Subagent 怎麼處理 transient error | Local recovery first，propagate 救不了的 |
| Synthesis 報告涵蓋有缺口 | Coverage annotation 明示 |

### ★ Sample Question 8（Page 30）— 5.3 主場

Web search subagent timeout 怎麼傳訊息給 coordinator。

**正解 A**：Structured error context（failure_type, attempted_query, partial_results, alternatives）。
- B（generic "search unavailable"）❌ 隱藏 context
- C（empty result as success）❌ silent suppression
- D（terminate workflow）❌ 一個死全部死

### ★ 核心 mental model

> **三條鐵律：不 silent suppression、不 terminate workflow、要 structured error context（4 要素）。Subagent 先 local recovery transient，救不了才 propagate。Access failure ≠ valid empty 用不同 status 區分。Synthesis 階段標 coverage annotation 不要假裝沒事。**

---

## Task Statement 5.4 — Large codebase exploration context management

### 整個 task 在解決什麼問題

用 Claude 探索 legacy codebase 跑 60 turn 後 → context degradation（model 開始說「typical pattern」忘掉具體 class）+ verbose tool output 塞爆 context + crash 怕 lose all progress。

### 四個 Knowledge of 重點

1. **★★★ Context degradation 徵兆**：model 給 inconsistent answer、引用「typical patterns」而不是之前發現的具體 class
2. **★★★ Scratchpad files** — 持久化 key finding 到外部檔，跨 context boundary
3. **★★★ Subagent delegation** — 隔離 verbose exploration output，main 維持高層
4. **★ Structured state persistence (manifest)** — 每個 agent export state，coordinator resume 載 manifest

### 五個 Skills in 重點

1. **★★★ Spawn subagent 投問特定 questions**（"find all test files"）
2. **★★★ Maintain scratchpad files** 對抗 context degradation
3. **★★ Phase 之間 summarize → inject 進下一 phase**（這個 summarize OK，因為是 architectural finding 不是 transactional fact）
4. **★ Crash recovery manifest 設計**
5. **★★ `/compact` 指令** — Claude Code 內建即時壓縮 context

### 已釐清概念

**5.4 的 summarize 跟 5.1 反對的 summarize 不衝突**：
- 5.1 反對 **summarize transactional fact**（數字/日期會飄）
- 5.4 推薦 **summarize architectural finding**（class 名/flow path）— 結構性內容濃縮 OK

**`/compact` 跟其他工具關係**：
- `/compact` = 已塞太多的補救
- Subagent delegation = 預防塞太多
- Scratchpad = 跨 compact 還記得的長期記憶

### 易踩陷阱

| 陷阱 | 為什麼錯 |
|---|---|
| 換大 context window | 鐵律 3，lost-in-the-middle 不會解 |
| 「叫 user 拆 task / restart session」 | Shift burden 反 pattern |
| Vague instruction「叫 model 記得」 | 4.1 反對 |
| 全部塞 main agent 跑 | Verbose 污染 → degradation 加劇 |

### 考試判斷規則

| 題目線索 | 答案方向 |
|---|---|
| Long exploration 後「typical pattern」字眼 | Scratchpad +
 subagent delegation |
| Verbose tool output 塞 context | Subagent delegation（隔離）+ `/compact` |
| Crash 怕 lose progress | Structured state manifest |
| Phase 之間需要連貫 | Summarize each phase + inject next phase |
| Context 已塞 90% | `/compact` 即時補救 |

### ★ 核心 mental model

> **Long exploration 三招防衛：預防（subagent delegation 隔離 verbose output）+ 記憶（scratchpad files externalize key finding）+ 補救（`/compact` 即時壓縮 + manifest crash recovery）。Symptom 「typical pattern」字眼 → context degradation 警報。不要靠「叫 model 記得 / 換大 model / restart 從頭探」——三個都是 distractor。**

---

## Task Statement 5.5 — Human review workflows + confidence calibration

### 整個 task 在解決什麼問題

Aggregate accuracy 97% 想 stop human review → 危險。可能是「90% 容易 case 99% 對 + 10% 難 case 30% 對」加權出來。**5.5 = 4.1/4.6 confidence 觀念的正規版**——用 labeled validation set 校準 + by-segment 驗證 + 持續 stratified sampling 才能安全 reduce review。

### 四個 Knowledge of 重點

1. **★★★ Aggregate accuracy 風險** — 97% overall 可能掩蓋 multilingual 45% / handwritten 30% 的災難
2. **★★★ Stratified random sampling** — 各 segment 各抽 sample（保證 minor segment 也被抽到）→ 量測各 stratum + 偵測 novel error pattern
3. **★★★ Field-level confidence + labeled validation set 校準**（5.5 招牌）— 三關鍵字：field-level / calibrated / labeled
4. **★★ By-segment + by-field 驗證 才能 reduce review** — 每個 (doc-type × field) cell 都過關才能自動

### 四個 Skills in 重點

1. **★★★ Stratified random sampling 實作** — 抽 high-confidence 那部分檢查（low-conf 已 routing 給 human 不用抽）
2. **★★★ By-segment + by-field accuracy 分析**（不能只看 aggregate）
3. **★★★ Field-level confidence + threshold calibration**（招牌 skill）：
   - Step 1: schema 加 field-level confidence
   - Step 2: 收 labeled validation set
   - Step 3: 分桶量測 raw confidence vs actual accuracy
   - Step 4: 訂 threshold 對應目標 accuracy（不拍腦袋）
   - Step 5: production routing
4. **★★ Routing — low confidence + ambiguous source + conflict_detected → human review**

### 已釐清概念

**Field-level + threshold calibration 簡單例子**：
- Model output `total_confidence: 0.87`
- 拿 50 張人工 label
- 分桶：raw 0.9-1.0 桶 → actual 100%；0.8-0.9 桶 → actual 93%
- 想要 95% accuracy → threshold = 0.9（從 data 反推，不拍腦袋）
- Production: `if conf >= 0.9: auto else human`

**為什麼 field-level**：每個欄位 calibration curve 不同。vendor_name 容易抽對 threshold 可低；total 算術錯多 threshold 高；due_date normalize 容易錯 threshold 最高。

**精髓**：threshold **不是拍腦袋**，**有 data support 的 confidence**。

### 易踩陷阱

| 陷阱 | 為什麼錯 |
|---|---|
| Aggregate metric 決定全自動 | 招牌風險 — segment 災難被掩蓋 |
| Self-reported confidence + 拍腦袋 threshold | 4.1/5.2 反對；無校準 |
| 「先 disable review，看出問題再加回」 | 災難 risk |
| 換大 model 解 accuracy | 鐵律 3 |
| Threshold 直接訂 0.9（沒查校準曲線）| 不知對應真實 accuracy |

### 考試判斷規則

| 題目線索 | 答案方向 |
|---|---|
| Aggregate 97% 想 reduce review | By-segment + by-field analysis + calibrated threshold + stratified sampling |
| 想知道某類 doc 是不是表現差 | Stratified random sampling |
| Threshold 怎麼訂 | Labeled validation set 反推 |
| Confidence 用法 | Field-level + calibrated |
| Reviewer 人手有限 | Routing low-confidence + ambiguous → human |
| 「Stratified random sampling」字眼 | ✅ 5.5 正解線索 |
| 「Labeled validation set」字眼 | ✅ 5.5 正解線索 |

### ★ 核心 mental model

> **5.5 = confidence 用法的正規版**。三條鐵律：(1) Aggregate metric 是陷阱必拆 by doc-type × by field 看。(2) Threshold 不拍腦袋——用 labeled set 反推對應目標 accuracy。(3) 持續 stratified random sampling 監測各 segment。**Routing 給 human：low confidence + ambiguous source + conflict_detected 任一觸發**。**「校準 + 多 signal 結合」才完整正解**。

---

## Task Statement 5.6 — Multi-source synthesis: provenance + uncertainty

### 整個 task 在解決什麼問題

Multi-agent research synthesis 寫出漂亮 report 但 source attribution 在 summarization 過程被洗掉、conflict 被 silently 抹掉、temporal context 沒提（讓 reader 把 2019 data 當現況）。**5.6 = research-grade 透明度設計**——claim-source mapping preserve through synthesis、conflict 並列 annotate、temporal date 必含。

### 四個 Knowledge of 重點

1. **★★★ Source attribution 在 summarization 步驟被洗掉**（每層為了精簡砍 metadata）
2. **★★★ Structured claim-source mapping 必須 preserve through synthesis**（不 compress 掉）
3. **★★★ Conflicting sources annotate 不選邊**（並列 + reader 判斷）
4. **★★ Temporal data 必含**（避免時序差異被誤判為 contradiction）

### 五個 Skills in 重點

1. **★★★ Subagent output 強制含 claim-source mapping**（URL / date / excerpt / methodology）
2. **★★★ Report 分段 — well-established / contested / temporally bounded**
3. **★★ Document analysis — conflict 全 inclusive 不 select**（讓 coordinator 決定 reconcile）
4. **★★ Subagent 強制含 publication / data collection date**
5. **★ 不同 content type 不同 rendering**（financial → table、news → prose、technical → list）

### 已釐清概念

**4.4 conflict_detected vs 5.6 multi-source 衝突**：
- **4.4**：single source 內部矛盾（一份 invoice header vs footer）
- **5.6**：multi-source 之間（兩份 paper 各自內部都對但數字不同）

**5.1 metadata 跟 5.6 的關係**：
- 5.1 是 producer 端要求（subagent 必含 metadata）
- 5.6 是 consumer 端要求（synthesis 必 preserve metadata）
- 兩端都做才不會丟

**5.3 coverage annotation 跟 5.6 的關係**：
- 5.3 從錯誤面（哪個 topic 沒 source）
- 5.6 從證據面（每個 claim 對應哪個 source）
- 拼起來 = 完整 transparent synthesis

### 易踩陷阱

| 陷阱 | 為什麼錯 |
|---|---|
| Synthesis 把 source 砍掉「為了精簡」| Source 是 metadata 不是 narrative |
| Conflict 自己選一個 silently | 應該並列 annotate |
| 沒 publication date | 時序差被誤判為 contradiction |
| Generic「based on multiple sources」 disclaimer | 5.6 要 specific source 不要 vague |
| 全 prose 寫財務 | 該用 table |
| 換大 model 解 attribution | 鐵律 3，是設計問題不是容量問題 |

### 考試判斷規則

| 題目線索 | 答案方向 |
|---|---|
| Final report 沒 source citation | Structured claim-source mapping preserve |
| 兩個 credible source 數字不同 | 並列 annotate（不 arbitrarily 選）|
| 「數字飄」其實是 temporal 不同 | Subagent 強制含 publication date |
| Synthesis 全部同一格式 | Render by content type |
| 多 source 想分共識 vs 爭議 | Report 分段 well-established / contested |
| 單一 source 內部矛盾 | 4.4 conflict_detected（不是 5.6）|
| Multi-source 間矛盾 | 5.6 source-attributed annotation |

### ★ 核心 mental model

> **Multi-source synthesis 三鐵律：(1) 每 claim 帶 source mapping，每層 preserve 不 compress。(2) Conflict 不選邊——並列 annotate。(3) Temporal date 永遠帶——避免時序差被誤判 contradiction。Synthesis report 分段（well-established / contested / temporal）顯示 epistemic 狀態。Document analysis 不替 coordinator 做選擇——把 conflict 帶上去 annotate。不同 content type 不同 rendering。**

---

# Domain 5 反模式速查表

| 反模式 | 為什麼錯 | 屬於 |
|---|---|---|
| Progressive summarization 壓掉 numerical fact | 數字會飄 | 5.1 |
| 換大 context window 解 long-conversation 問題 | 鐵律 3 + lost-in-the-middle | 5.1, 5.4 |
| 純 prompt instruction「記住數字 / 記得 class 名」 | Vague guidance | 5.1, 5.4 |
| 整 conversation 完全靠 model 記 | API stateless | 5.1 |
| Self-reported confidence 當 escalation trigger | 5.2 反對 | 5.2 |
| Sentiment analysis → escalate | Sentiment ≠ complexity | 5.2 |
| 訓練 classifier 預測 escalation | Out-of-scope | 5.2 |
| Customer 明示要 human 還先 investigate | 違反尊重明示 | 5.2 |
| Multiple matches 用 heuristic 猜 | 該問 identifier | 5.2 |
| Subagent generic error message | 隱藏 context | 5.3 |
| Subagent return [] as success | Silent suppression | 5.3 |
| Subagent throw exception terminate workflow | 一個死全部死 | 5.3 |
| 「叫 user 拆 task / restart session」 | Shift burden | 5.4 |
| Aggregate accuracy 直接決定全自動 | 招牌風險 | 5.5 |
| Self-rate confidence + 拍腦袋 threshold | 4.1/5.5 反對 | 5.5 |
| Threshold 直接訂 0.9 沒查校準曲線 | 不知對應真實 accuracy | 5.5 |
| Synthesis 把 source 砍掉「為了精簡」| Source 是 metadata | 5.6 |
| Conflict 自己選一個 silently | 應該並列 annotate | 5.6 |
| 沒 publication date | 時序差被誤判 contradiction | 5.6 |
| Generic「based on multiple sources」disclaimer | 要 specific 不要 vague | 5.6 |

---

# 高頻必背 Sample Questions（Domain 5）

| Sample Q | Domain | 考點 | 正解核心 |
|---|---|---|---|
| **Q3** | 5.2 | Escalation calibration | Explicit criteria + few-shot |
| **Q8** | 5.3 | Subagent error propagation | Structured error context（4 要素）|

（Q1, Q7, Q9 雖在其他 domain 主場，但 5.x 觀念也有交集）

---

# Domain 5 在「全 toolkit」框架的位置

```
L0 Prompt    : Case facts block, escalation criteria + few-shot
L1 Schema    : Field-level confidence, claim-source mapping
              Structured error context dict, manifest schema
L2 Validation: -
L3 Hook      : Trim verbose tool output (PostToolUse)
L4 Architecture: ★ Subagent delegation, multi-issue layer
              Multi-agent error propagation
L5 後驗校準  : ★★★ Calibrated confidence + stratified sampling
              By-segment by-field accuracy analysis
L6 Process   : Phase summarization, scratchpad, /compact
              Crash recovery manifest, escalation pattern
              Document analysis 不替 coordinator decide
              Render by content type
```

→ Domain 5 涵蓋 L0/L1/L3/L4/L5/L6——是 multi-turn / multi-agent 系統可靠性的核心，特別是 L5 calibration 完全是 5.5 主場。

---

# Confidence 跨 Domain 三層觀（最終整合版）

這是 CCAF 整個 exam 反復出現的觀念脈絡：

| 層次 | 立場 | 出現在 |
|---|---|---|
| Self-reported confidence 當**唯一 hard filter**（拍腦袋 threshold + 沒校準 + 沒結合其他 signal）| ❌ **強烈反對** | 4.1, 5.2 |
| Self-reported confidence 當 **routing signal**（多 input 之一）| ⚠️ **接受**（短期方案）| **4.6** |
| **Field-level + calibrated by labeled validation set + stratified sampling + 配合其他 signal** | ✅ **推薦**（正規）| **5.5** |

→ **三層遞進**。考題用詞辨識：
- 「self-rate + threshold」沒提 calibration → distractor
- 「routing signal alongside severity」→ 4.6 接受
- 「field-level + labeled validation set + stratified」→ 5.5 正解

---

# Domain 5 整合：6 個 task 互相關係

```
跨時間（5.1 long conversation）
跨 agent（5.3 multi-agent error propagation）
跨 source（5.6 multi-source synthesis）
跨 long-task（5.4 long codebase exploration）
跨 human-in-loop（5.2 escalation, 5.5 review routing）

共通哲學：
- 不靠 model self-rating（5.2/5.5 反對 self-confidence）
- Externalize critical metadata（5.1 case facts、5.4 scratchpad、5.6 source attribution）
- 結構化錯誤訊息（5.3 structured error context）
- 不要假裝沒事（5.3 coverage annotation、5.6 conflict annotation、5.5 by-segment validation）
```

---

# Domain 5 新增 term 速記（已釐清概念）

| Term | 意思 |
|---|---|
| **Case facts block** | 把 transactional fact 抽出每次 prompt 原樣帶 |
| **Trim verbose tool output** | Tool result 進 context 前先砍多餘欄位 |
| **Lost in the middle** | LLM 對長 input 中段 attention 弱、頭尾可靠 |
| **Position-aware** | 重要 finding 放開頭（official 用詞）+ section header |
| **Escalation criteria + few-shot** | 寫死 ESCALATE/RESOLVE rule + 邊界 case 示範 |
| **Policy gap** | Policy silent on customer's specific request → 該 escalate |
| **Structured error context** | 4 要素：failure_type / attempted_query / partial_results / alternatives |
| **Access failure vs valid empty** | 沒查到（要 retry）vs 查了沒料（proceed） |
| **Local recovery first** | Subagent 先 retry transient，救不了才 propagate |
| **Coverage annotation** | Synthesis 標哪 topic 有 gap |
| **Scratchpad file** | Externalize key finding 到外部檔，跨 context boundary |
| **Subagent delegation** | 隔離 verbose exploration output |
| **`/compact`** | Claude Code 內建 context 壓縮指令 |
| **Manifest** | Crash recovery 用的 agent state index |
| **Aggregate metric 風險** | Overall 97% 可能掩蓋 segment 30% |
| **Stratified random sampling** | 各 segment 各抽 sample 確保 coverage + 偵測 novel pattern |
| **Field-level confidence** | 每欄位獨立 confidence（vs overall）|
| **Calibrated threshold** | 從 labeled validation set 反推對應目標 accuracy 的 threshold |
| **Calibration curve** | Raw confidence → actual accuracy 的對照（每 field 不同）|
| **Claim-source mapping** | 每 claim 結構化 link 到 source（URL/date/excerpt）|
| **Well-established vs contested findings** | Report 分段顯示 epistemic 狀態 |
| **Render by content type** | Financial → table、news → prose、technical → list |

---

# 終極 Mental Model

> **Domain 5 的核心邏輯：multi-turn / multi-agent / multi-source 場景下，**「**externalize critical state**」**+** 「**結構化 metadata 不 compress**」**+** 「**不假裝沒事（coverage gap、source conflict、segment 不均）**」。
>
> **三件事決定考題答案：**
> **1. Critical fact 會不會飄？需要 externalize（case facts / scratchpad / source mapping）**
> **2. 失敗訊息怎麼傳？需要 structured（不 silent suppress 不 terminate）**
> **3. Confidence 怎麼用？需要 calibrated（不拍腦袋 + 不 sole filter）**
>
> **Self-rating（confidence、sentiment）一律可疑——要嘛當 signal（4.6 短期）、要嘛 calibrated by labeled set（5.5 正規）、永遠不能當 sole filter（4.1/5.2 強烈反對）。**
